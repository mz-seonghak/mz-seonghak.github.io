---
layout: post
title: "Apache Iceberg × BigLake × BigQuery: 메타데이터 구조부터 쓰기 성능 튜닝까지"
date: 2026-03-24
tags: [iceberg, biglake, bigquery, gcp, data-engineering, spark, partition, pruning, compaction]
---

GCP에서 Apache Iceberg를 BigLake/BigQuery와 연동하면, Spark로 적재하고 BigQuery로 분석하는 유연한 데이터 아키텍처를 구현할 수 있습니다. 하지만 실제 운영 과정에서 예상치 못한 스캔량 차이, 쓰기 성능 저하, 한글 컬럼 이슈 등을 마주하게 됩니다.

이 글에서는 Iceberg의 메타데이터 구조를 먼저 이해한 뒤, BigQuery에서의 스캔량 최적화, BigLake 연동 방식 비교, 그리고 Spark 쓰기 성능 튜닝까지 실무에서 필요한 내용을 정리합니다.

---

## 1. Iceberg는 서비스가 아니라 "테이블 포맷"이다

Iceberg를 처음 접하면 Hive Metastore처럼 별도 데몬이 돌아가는 것으로 오해하기 쉽습니다. Iceberg는 실행되는 프로세스가 아니라, **메타데이터 파일 구조의 규격(specification)**입니다.

```
gs://bucket/warehouse/my_table/
├── metadata/
│   ├── v1.metadata.json          ← 테이블 스키마, partition-spec, 스냅샷 목록
│   ├── snap-xxxxx.avro           ← 스냅샷 → manifest list
│   └── manifest-xxxxx.avro       ← 각 데이터 파일의 위치 + 컬럼별 min/max 통계
└── data/
    ├── p_yyyymm=202501/
    │   ├── file1.parquet
    │   └── file2.parquet
    └── p_yyyymm=202502/
        └── file3.parquet
```

비유하자면 Iceberg 메타데이터는 **책의 목차 + 색인**입니다. 책(데이터)을 읽는 사람(BigQuery)이 색인을 보고 필요한 페이지만 펼치는 것이지, 색인이 스스로 뭔가를 하는 게 아닙니다.

### 메타데이터 3단 구조

| 계층 | 파일 | 담고 있는 것 |
|------|------|-------------|
| metadata.json | 테이블 정의 | 스키마, partition-spec, 현재 스냅샷 ID |
| manifest-list | `snap-xxxxx.avro` | 어떤 manifest 파일들이 있는지 목록 + 파티션 범위 요약 |
| manifest file | `manifest-xxxxx.avro` | 데이터 파일별 경로, 행 수, **컬럼별 min/max**, null count |

핵심은 **manifest file**입니다. 여기에 각 Parquet 파일별로 **모든 컬럼의 min/max, null count 등의 통계**가 기록되어 있고, 이것이 쿼리 엔진의 pruning에 활용됩니다.

---

## 2. BigQuery에서 스캔량 확인하는 방법

Iceberg 테이블의 pruning 효과를 제대로 측정하려면 BigQuery의 스캔량을 정확히 확인해야 합니다.

### 2-1. Dry Run으로 예상 스캔량 사전 확인

```bash
bq query --dry_run --use_legacy_sql=false \
  'SELECT col1, col2 FROM `project.dataset.table` WHERE partition_date = "2025-01-01"'
```

Console에서는 쿼리 편집기 우측 상단에 예상 스캔량이 자동으로 표시됩니다.

### 2-2. 실행 후 Job 메타데이터로 확인

```sql
SELECT
  job_id,
  query,
  total_bytes_processed,
  total_bytes_billed,
  cache_hit
FROM `region-asia-northeast3`.INFORMATION_SCHEMA.JOBS
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)
ORDER BY creation_time DESC;
```

`total_bytes_processed`가 실제 스캔된 바이트 수이고, `total_bytes_billed`가 과금 기준 바이트 수입니다. **캐시 히트 시 0**으로 나올 수 있으니 `cache_hit` 컬럼도 함께 확인해야 합니다.

### 2-3. 캐시를 비활성화하고 비교

최적화 전후 효과를 정확히 비교하려면 캐시를 꺼야 합니다.

```bash
bq query --use_legacy_sql=false --nouse_cache \
  'SELECT ... FROM table WHERE ...'
```

Console에서는 **More → Query Settings → Cache 체크 해제**로 설정할 수 있습니다.

### 2-4. 테이블 스토리지 용량 확인

```sql
SELECT
  table_name,
  total_rows,
  total_logical_bytes,
  total_physical_bytes
FROM `project.dataset.INFORMATION_SCHEMA.TABLE_STORAGE`
WHERE table_name = 'my_table';
```

---

## 3. 파티션 컬럼이 아닌데 스캔량이 줄어드는 이유

`p_yyyymm`이 파티션 컬럼이고 `기준년월`이 일반 컬럼인 Iceberg 외부 테이블에서, `기준년월`으로 필터해도 스캔량이 줄어드는 현상이 발생할 수 있습니다. 이건 BigQuery 파티션 pruning이 아니라 **Iceberg의 파일 pruning**입니다.

### Column Statistics 기반 Data Skipping

Iceberg는 파티션과 별개로, 각 데이터 파일(Parquet)에 대해 **컬럼별 min/max 통계를 manifest에 저장**합니다. BigQuery가 쿼리를 실행하면 다음 순서로 동작합니다:

1. `metadata.json`을 읽어서 현재 스냅샷의 manifest list 확인
2. manifest file을 읽어서 각 데이터 파일의 컬럼별 min/max 통계 확인
3. `WHERE 기준년월 = '202501'` 조건과 각 파일의 `기준년월` min/max를 대조
4. 해당 범위에 걸리지 않는 파일은 **아예 읽지 않음**
5. 나머지 파일만 GCS에서 읽어서 처리

```
WHERE 기준년월 = '202501' 쿼리 시:

p_yyyymm=202501/file1.parquet → 기준년월 min:202501, max:202501 → ✅ 읽음
p_yyyymm=202501/file2.parquet → 기준년월 min:202501, max:202501 → ✅ 읽음
p_yyyymm=202502/file3.parquet → 기준년월 min:202502, max:202502 → ❌ skip
p_yyyymm=202502/file4.parquet → 기준년월 min:202502, max:202503 → ❌ skip
```

이 전체 과정을 **BigQuery 엔진이 직접 수행**합니다. Iceberg 쪽에서 별도로 뭔가 돌아가는 게 아니라, BigQuery가 Iceberg 메타데이터 파일을 해석할 줄 아는 것입니다.

### 왜 파티션 컬럼보다 일반 컬럼이 스캔량이 적을 수 있는가

직관과 반대이지만, **pruning 단위가 다르기 때문에** 충분히 발생할 수 있습니다.

| | p_yyyymm (파티션 컬럼) | 기준년월 (일반 컬럼) |
|---|---|---|
| pruning 단위 | **디렉토리 단위** (거친 단위) | **파일 단위** (세밀한 단위) |
| 동작 방식 | 파티션 디렉토리 전체를 읽음 | manifest의 파일별 min/max로 판단 |

`p_yyyymm=202501/` 디렉토리 안에 `기준년월`이 `202412`인 데이터가 섞여 있다면, `WHERE p_yyyymm = '202501'`은 해당 디렉토리 전체를 읽지만, `WHERE 기준년월 = '202501'`은 여러 디렉토리에서 해당 값이 있는 파일만 골라서 읽습니다.

확인하는 가장 확실한 방법은 두 컬럼의 값 불일치를 직접 조회하는 것입니다:

```sql
SELECT p_yyyymm, 기준년월, COUNT(*) as cnt
FROM `project.dataset.iceberg_table`
WHERE p_yyyymm = '202501'
GROUP BY 1, 2
ORDER BY 2;
```

---

## 4. Iceberg Hidden Partitioning

### Hive 스타일 vs Iceberg 스타일

Hive 스타일 파티셔닝은 사용자가 별도의 파티션 컬럼을 직접 관리해야 하지만, Iceberg의 Hidden Partitioning은 **원본 컬럼에 transform 함수를 적용**해서 파티션을 만듭니다.

```sql
-- Hive 스타일: 별도 파티션 컬럼을 사용자가 직접 관리
CREATE TABLE orders (...) PARTITIONED BY (p_yyyymm STRING);
-- 쿼리 시 파티션 컬럼을 명시해야 pruning 작동
SELECT * FROM orders WHERE p_yyyymm = '202501';

-- Iceberg hidden partition: 원본 컬럼에 transform 적용
CREATE TABLE orders (...) PARTITIONED BY (month(order_date));
-- 쿼리 시 원본 컬럼으로 필터하면 자동 pruning
SELECT * FROM orders WHERE order_date >= '2025-01-01' AND order_date < '2025-02-01';
```

### 사용 가능한 Transform 함수

| Transform | 입력 타입 | 예시 | 파티션 값 |
|-----------|-----------|------|-----------|
| `year(col)` | DATE, TIMESTAMP | `year(order_date)` | 2025 |
| `month(col)` | DATE, TIMESTAMP | `month(order_date)` | 2025-01 |
| `day(col)` | DATE, TIMESTAMP | `day(order_date)` | 2025-01-15 |
| `hour(col)` | TIMESTAMP | `hour(event_ts)` | 2025-01-15-09 |
| `bucket(N, col)` | 모든 타입 | `bucket(16, customer_id)` | 0~15 |
| `truncate(col, W)` | STRING, INT 등 | `truncate(zipcode, 3)` | 100, 200 |
| `identity(col)` | 모든 타입 | `identity(region)` | 원본 값 그대로 |

### metadata.json에서 확인

```json
{
  "partition-specs": [{
    "spec-id": 0,
    "fields": [
      {
        "name": "order_date_month",
        "transform": "month",
        "source-id": 3,
        "field-id": 1000
      }
    ]
  }]
}
```

### 한글 컬럼명의 Hidden Partition 제약

한글 컬럼을 원본으로 hidden partition을 지정하면 문제가 발생합니다. 이는 Iceberg + Parquet의 **컬럼명 sanitization** 때문입니다.

Iceberg의 Java 라이브러리는 Parquet 파일에 데이터를 쓸 때 non-ASCII 문자를 `_xHH` 형태로 인코딩합니다. `기준년월`이라는 컬럼명은 Parquet 파일 내부에서 각 한글 바이트가 인코딩되어 전혀 다른 문자열이 됩니다.

```
metadata.json의 partition-spec
  → source column: "기준년월" (원본 이름으로 저장)

Parquet 파일 내부 컬럼명
  → "_xEA_xB8_xB0_xEC_xA4_x80_xEB_x85_x84_xEC_x9B_x94" (sanitized)

엔진이 partition transform 적용 시
  → metadata의 "기준년월"과 Parquet의 sanitized 이름 매칭 실패
```

Iceberg 스펙상 컬럼은 이름이 아닌 **field ID**로 매칭되어야 하지만, 실제 구현체에서 이 매핑이 완벽하지 않아 non-ASCII 컬럼명에서 문제가 발생합니다.

**대응 방안:** 파티션 관련 컬럼은 영문으로 유지하는 것이 가장 안전합니다. 한글 원본 컬럼은 그대로 두고, 파티션용 영문 컬럼(`p_yyyymm` 등)을 별도로 관리하는 현재 구조가 올바른 설계입니다.

---

## 5. BigLake × BigQuery 연동 방식 3가지

GCP에서 Iceberg 테이블을 다루는 방식은 메타데이터 저장 위치에 따라 세 가지로 나뉩니다.

### 5-1. BigLake Managed Iceberg Table

**메타데이터:** BigQuery 내부 (Big Metadata)  
**데이터:** GCS (Parquet)

```sql
CREATE TABLE project.dataset.my_table (
  id INT64,
  name STRING,
  created_at TIMESTAMP
)
WITH CONNECTION `project.region.connection_name`
OPTIONS (
  file_format = 'PARQUET',
  table_format = 'ICEBERG',
  storage_uri = 'gs://my-bucket/my-table'
);
```

**특징:**

- BigQuery가 메타데이터를 내부적으로 관리하며, 표준 BigQuery 테이블과 동일한 사용 경험 제공
- 데이터는 고객 소유의 GCS 버킷에 저장
- 자동 파일 크기 최적화, 클러스터링, 메타데이터 컴팩션, 고아 파일 GC 자동 수행
- DML(`INSERT`, `UPDATE`, `DELETE`, `MERGE`) 완전 지원
- 외부 엔진에서 읽으려면 `EXPORT TABLE METADATA` 필요

### 5-2. BigLake External Iceberg Table

**메타데이터:** GCS (Iceberg 표준 metadata.json)  
**데이터:** GCS (Parquet)

```sql
CREATE EXTERNAL TABLE project.dataset.my_external_table
WITH CONNECTION `project.region.connection_name`
OPTIONS (
  format = 'ICEBERG',
  uris = ["gs://my-bucket/warehouse/table/metadata/iceberg.metadata.json"]
);
```

**특징:**

- Spark/Flink 등 외부 엔진이 GCS에 직접 Iceberg 표준 메타데이터를 관리
- BigQuery에서는 **읽기 전용** (쓰기는 Spark 등에서)
- `metadata.json` URI를 수동으로 업데이트해야 할 수 있음

### 5-3. BigLake Metastore + REST Catalog

**메타데이터:** BigLake Metastore (관리형 서비스)  
**데이터:** GCS (Parquet)

```python
spark.conf.set("spark.sql.catalog.my_catalog", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.my_catalog.type", "rest")
spark.conf.set("spark.sql.catalog.my_catalog.uri",
    "https://biglake.googleapis.com/iceberg/v1/restcatalog")
spark.conf.set("spark.sql.catalog.my_catalog.warehouse", "gs://my-bucket")
```

**특징:**

- Spark, BigQuery 양쪽에서 읽기/쓰기 가능
- 메타스토어를 직접 관리할 필요 없음 (서버리스)
- 표준 Iceberg REST Catalog API 지원

### 비교 요약

| | Managed (BigQuery 내부) | External (GCS 메타) | BigLake Metastore |
|---|---|---|---|
| 메타 위치 | BigQuery 내부 | GCS metadata.json | BigLake Metastore 서비스 |
| BigQuery DML | 전체 지원 | 읽기 전용 | 읽기 + 쓰기 |
| Spark 쓰기 | Storage Write API | 직접 가능 | REST Catalog로 가능 |
| 자동 최적화 | compaction, GC 자동 | 없음 (직접 관리) | 부분 지원 |
| 메타 동기화 | 자동 | 수동 URI 업데이트 | 자동 |
| 멀티엔진 | export 필요 | 네이티브 지원 | 네이티브 지원 |

Spark로 GCS에 데이터를 올리고 BigQuery에서 읽는 구조라면 **BigLake Metastore + REST Catalog** 방식이 가장 적합합니다. Spark에서 직접 쓰기가 가능하면서 BigQuery에서도 바로 접근이 되고, metadata.json을 수동으로 업데이트할 필요도 없습니다.

---

## 6. Spark → Iceberg 쓰기 성능 문제와 해결

### 문제: External Table 3~4분 → Iceberg 4시간

기존 Hive 스타일 external table에서 3~4분 걸리던 적재가 Iceberg external table로 변경 후 4시간으로 늘어나는 현상이 발생할 수 있습니다.

### 핵심 원인: write.distribution-mode의 Shuffle

Iceberg의 기본 `write.distribution-mode`가 `hash`이기 때문에, 기존 external table에서는 없던 대규모 shuffle이 발생합니다.

```
기존 external table:  데이터 → 바로 Parquet 쓰기 → 끝
Iceberg table:        데이터 → hash shuffle(파티션 키 기준) → 정렬 → Parquet 쓰기 → 메타 커밋
```

하나의 파티션에 대량 데이터가 몰리면, 파티션당 하나의 태스크만 쓰기를 담당하게 되어 쓰기 속도가 극도로 느려집니다.

### 해결: distribution mode 변경

```sql
ALTER TABLE catalog.db.my_table SET TBLPROPERTIES (
  'write.distribution-mode' = 'none',
  'write.spark.fanout.enabled' = 'true'
);
```

이 설정은 Iceberg 테이블의 `metadata.json`에 기록되므로, 한 번 설정하면 이후 어떤 Spark 세션에서든 동일하게 적용됩니다.

현재 설정값 확인:

```sql
SHOW TBLPROPERTIES catalog.db.my_table ('write.distribution-mode');
```

### hash vs none 모드의 파일 차이

100개 Spark Task가 12개 파티션에 10GB를 쓴다고 가정할 때:

| | hash (기본값) | none |
|---|---|---|
| 파일 수 | ~12개 (파티션당 1~2개) | ~1,200개 (Task × 파티션 조합) |
| 파일 크기 | 균일 (400~800MB) | 들쭉날쭉 (수KB ~ 수십MB) |
| 파티션 내 정렬 | 정렬됨 | 정렬 안 됨 |
| 쓰기 속도 | 느림 (shuffle 있음) | 빠름 (shuffle 없음) |
| 읽기 성능 | 좋음 | 나쁨 (small files) |

`none` 모드에서는 각 Task가 자기가 가진 데이터를 파티션별로 나눠서 쓰기 때문에, 하나의 파티션에 여러 개의 작은 파일이 생성됩니다. 또한 하나의 파일에 여러 파티션의 데이터가 섞이면 min/max 범위가 넓어져서 data skipping 효과도 줄어듭니다.

### 추가 Spark 튜닝

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.shuffle.partitions", "200")
```

---

## 7. Compaction으로 사후 재정돈

`none` 모드로 빠르게 쓰면 small files가 발생하지만, Iceberg는 이를 위한 테이블 유지보수 프로시저를 제공합니다.

### 7-1. rewrite_data_files — 데이터 파일 병합

```sql
-- 기본 compaction
CALL catalog.system.rewrite_data_files('db.my_table');

-- 특정 파티션만 compaction
CALL catalog.system.rewrite_data_files(
  table => 'db.my_table',
  where => 'p_yyyymm = "202501"'
);

-- 파일 사이즈 기준 지정
CALL catalog.system.rewrite_data_files(
  table => 'db.my_table',
  options => map(
    'target-file-size-bytes', '536870912',
    'min-file-size-bytes', '67108864',
    'max-file-size-bytes', '1073741824'
  )
);
```

```
Before (none 모드 적재 직후):
  파티션 202501/
    ├── file_001.parquet (8MB)   ← 기준년월 min:202412, max:202503
    ├── file_002.parquet (3MB)   ← 기준년월 min:202501, max:202502
    └── ... (100개)

After (rewrite_data_files 실행 후):
  파티션 202501/
    └── file_merged_001.parquet (500MB)  ← 기준년월 min:202501, max:202501
```

파일이 합쳐지면서 **column statistics도 타이트해지므로** data skipping 효과도 복원됩니다.

### 7-2. sort 전략으로 정렬까지 적용

```sql
-- 정렬 기준 설정
ALTER TABLE catalog.db.my_table
WRITE ORDERED BY 기준년월, customer_id;

-- sort 전략으로 compaction
CALL catalog.system.rewrite_data_files(
  table => 'db.my_table',
  strategy => 'sort',
  sort_order => '기준년월 ASC, customer_id ASC'
);
```

### 7-3. rewrite_manifests — 메타데이터 정돈

```sql
CALL catalog.system.rewrite_manifests('db.my_table');
```

### 7-4. 오래된 스냅샷 및 고아 파일 정리

```sql
-- 스냅샷 정리
CALL catalog.system.expire_snapshots(
  table => 'db.my_table',
  older_than => TIMESTAMP '2026-02-25 00:00:00',
  retain_last => 5
);

-- 고아 파일 정리
CALL catalog.system.remove_orphan_files('db.my_table');
```

### 핵심: 재정돈 중에도 테이블은 정상 작동

Iceberg의 스냅샷 격리 덕분에, 재정돈 중에도 테이블 읽기가 정상 동작합니다. 재정돈이 완료되면 새 스냅샷으로 커밋되고, 이후 쿼리부터 정돈된 파일을 사용하게 됩니다. 다운타임 없이 온라인으로 수행 가능합니다.

---

## 8. 실무 운영 패턴

### 시간이 "이동"하는 것 아닌가?

전체 작업량으로 보면 그렇지 않습니다.

```
hash 모드:       [shuffle + 정렬 + 쓰기]  = 4시간 (한 덩어리)
none + 재정돈:    [쓰기] 4분  +  [compaction] 30분~1시간  = 총 34분~64분
```

hash 모드의 4시간 중 상당 부분이 **단일 Task에 데이터가 몰리는 비효율** 때문입니다. compaction은 이미 쓰여진 파일을 **병렬로** 읽고 병합하므로 같은 작업이 훨씬 효율적으로 수행됩니다.

### 결정적 차이: 적재와 재정돈의 분리

```
hash 모드:
  09:00 적재 시작 → 13:00 적재 완료 → 13:00 데이터 사용 가능
                     (4시간 동안 데이터 없음)

none + 재정돈:
  09:00 적재 시작 → 09:04 적재 완료 → 09:04 데이터 사용 가능 (비최적 상태)
                                      09:04 compaction 시작
                                      09:40 compaction 완료 → 최적 상태
```

**4분 후부터 데이터를 바로 쿼리할 수 있습니다.** small files 상태라서 최적은 아니지만, 급한 리포트나 확인 작업을 기다릴 필요가 없습니다.

### 매일 배치 적재 패턴

```python
# 1. 빠르게 적재 (none 모드, 3~4분)
spark.sql("INSERT INTO catalog.db.my_table SELECT * FROM source")

# 2. 당일 파티션만 compaction
spark.sql("""
  CALL catalog.system.rewrite_data_files(
    table => 'db.my_table',
    where => 'p_yyyymm = "202503"'
  )
""")

# 3. manifest 재정돈
spark.sql("CALL catalog.system.rewrite_manifests('db.my_table')")

# 4. 주 1회: 오래된 스냅샷 정리
spark.sql("""
  CALL catalog.system.expire_snapshots(
    table => 'db.my_table',
    older_than => TIMESTAMP '2026-02-25 00:00:00',
    retain_last => 10
  )
""")
```

### 상황별 권장

| 상황 | 권장 |
|------|------|
| 적재 후 바로 대시보드/리포트에 사용 | none + 즉시 compaction |
| 적재 후 다음 날 아침에 사용 | none + 야간 compaction |
| 적재 빈도가 높고 읽기는 가끔 | none + 주 1회 compaction |
| 적재 빈도 낮고 읽기가 핵심 | hash 모드 유지 |

---

## 9. Iceberg Manifest 파일 직접 확인하는 방법

pruning이 실제로 동작하는지 확인하려면 manifest 파일을 직접 열어봐야 합니다. manifest 파일은 Avro 형식이므로 전용 도구가 필요합니다.

### avro-tools (Java 기반)

```bash
wget https://repo1.maven.org/maven2/org/apache/avro/avro-tools/1.11.3/avro-tools-1.11.3.jar
gsutil cp gs://bucket/path/to/metadata/manifest-xxxxx.avro /tmp/

java -jar avro-tools-1.11.3.jar tojson /tmp/manifest-xxxxx.avro | python3 -m json.tool
```

### Python fastavro

```bash
pip install fastavro
```

```python
import fastavro
import json

with open('/tmp/manifest-xxxxx.avro', 'rb') as f:
    reader = fastavro.reader(f)
    for record in reader:
        print(json.dumps(record, indent=2, default=str))
```

### 주로 봐야 할 필드

```json
{
  "data_file": {
    "file_path": "gs://bucket/data/p_yyyymm=202501/file1.parquet",
    "partition": { "p_yyyymm": "202501" },
    "record_count": 50000,
    "lower_bounds": { "1": "202501", "2": "A0001" },
    "upper_bounds": { "1": "202501", "2": "Z9999" }
  }
}
```

`lower_bounds`와 `upper_bounds`에 `기준년월` 컬럼의 min/max가 기록되어 있다면, BigQuery가 이를 보고 파일 pruning을 하고 있는 것입니다. 컬럼 ID와 실제 컬럼명의 매핑은 `metadata.json`의 `schemas` 섹션에서 확인할 수 있습니다.

---

## 마무리

| 주제 | 핵심 |
|------|------|
| Iceberg 메타데이터 | 서비스가 아닌 파일 구조 규격. manifest에 파일별 column statistics 포함 |
| 스캔량 최적화 | 파티션 컬럼이 아니어도 manifest의 min/max로 data skipping 가능 |
| Hidden Partition | transform 함수 기반 자동 pruning. 한글 컬럼은 sanitization 이슈로 사용 불가 |
| BigLake 연동 | Managed / External / BigLake Metastore 세 가지 방식. 멀티엔진이면 Metastore 권장 |
| 쓰기 성능 | `write.distribution-mode=none`으로 shuffle 제거 후 사후 compaction |
| 운영 패턴 | 빠른 적재 → compaction 분리로 데이터 가용성과 읽기 성능 모두 확보 |

Iceberg는 "빠르게 쓰고 나중에 정돈한다"는 운영 방식이 의도된 설계이며, 이를 위한 도구가 잘 갖춰져 있습니다. GCP 환경에서는 BigLake를 통해 Spark 쓰기와 BigQuery 분석을 자연스럽게 연결할 수 있으므로, 각 연동 방식의 특성을 이해하고 환경에 맞는 조합을 선택하는 것이 중요합니다.
