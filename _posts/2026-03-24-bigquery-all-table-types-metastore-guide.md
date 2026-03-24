---
layout: post
title: "BigQuery 테이블 유형 완전 가이드: Native부터 External까지, 메타스토어 관리 방식별 총정리"
date: 2026-03-24
tags: [bigquery, biglake, gcs, iceberg, delta-lake, external-table, metastore, dataproc, gcp, data-engineering]
---

BigQuery에서 데이터를 다루는 방법은 크게 **Native Table**(BigQuery 내부 저장)과 **External Table**(GCS 등 외부 저장)로 나뉩니다. 특히 External Table은 메타스토어를 누가, 어떻게 관리하느냐에 따라 6가지 이상의 방식이 존재하며, 각각 기능·성능·운영 부담이 다릅니다.

이 글에서는 BigQuery에서 사용할 수 있는 **모든 테이블 유형**을 나열하고, 각 방식의 구조·특징·장단점을 비교합니다.

---

## 전체 구조 한눈에 보기

```
BigQuery에서 데이터를 사용하는 방법
│
├── 1. Native Table (BigQuery 내부 저장)
│
├── 2. External Table (GCS 파일 참조)
│   │
│   ├── 메타스토어 없음 (BigQuery 카탈로그만)
│   │   ├── 2-A. 기본 External Table
│   │   └── 2-B. BigLake External Table (flat files)
│   │
│   ├── BigQuery가 메타스토어 관리
│   │   └── 2-C. BigLake Managed Iceberg Table
│   │
│   ├── GCS 파일 자체가 메타스토어 (자체 관리)
│   │   └── 2-D. BigLake External Table (Iceberg / Delta Lake / Hudi)
│   │
│   ├── GCP 관리형 메타스토어
│   │   ├── 2-E. BigLake Metastore (REST Catalog)
│   │   └── 2-F. Dataproc Metastore (Hive Metastore Service)
│   │
│   └── 자체 호스팅 메타스토어
│       └── 2-G. Self-hosted Hive Metastore
│
└── 3. Object Table (비정형 데이터)
```

---

## 1. BigQuery Native Table

**데이터 위치:** BigQuery 내부 (Capacitor 포맷)
**메타스토어:** BigQuery 카탈로그 (완전 관리형)

BigQuery의 기본이자 가장 성능이 좋은 방식입니다. 데이터가 BigQuery 내부 스토리지에 Capacitor라는 독자 컬럼형 포맷으로 저장됩니다.

```sql
CREATE TABLE project.dataset.sales (
  order_id INT64,
  customer_id STRING,
  amount NUMERIC,
  order_date DATE
)
PARTITION BY order_date
CLUSTER BY customer_id;
```

### 핵심 기능

| 기능 | 지원 여부 |
|------|-----------|
| DML (INSERT/UPDATE/DELETE/MERGE) | 완전 지원 |
| Streaming Insert | 지원 (Storage Write API) |
| 파티셔닝 | 지원 (시간/범위/정수) |
| 클러스터링 | 지원 |
| Time Travel | 최대 7일 |
| Fail-safe | 추가 7일 (삭제 복구) |
| Row/Column 수준 보안 | 지원 |
| 캐싱 | 자동 (쿼리 결과 캐시) |
| Materialized View | 지원 |

### 장점

- **최고 성능**: BigQuery 엔진에 최적화된 Capacitor 포맷, 자동 파티션 pruning과 클러스터링
- **완전 관리형**: 인프라 관리 불필요, 자동 스토리지 최적화
- **풍부한 기능**: Time Travel, 스냅샷, 복제, CDC 등 엔터프라이즈 기능 전체 사용 가능
- **비용 모델 단순**: 스토리지 비용 + 쿼리 비용 (on-demand 또는 슬롯 기반)

### 단점

- **데이터 이동 필수**: GCS 등 외부에 있는 데이터를 BigQuery로 로드해야 함
- **독점 포맷**: Capacitor는 BigQuery 전용이라 Spark 등 다른 엔진에서 직접 읽을 수 없음
- **스토리지 비용 이중화**: 원본 소스와 BigQuery 양쪽에 데이터가 존재하면 비용 증가
- **대용량 로드 시간**: 초기 데이터 적재에 시간 소요

### 적합한 경우

- BigQuery가 **유일한 분석 엔진**인 환경
- 최고 쿼리 성능과 SLA가 필요한 프로덕션 대시보드
- DML이 빈번한 워크로드 (실시간 업데이트, CDC)
- 데이터가 이미 BigQuery에 있거나, 적재 파이프라인이 확립된 환경

---

## 2-A. 기본 External Table (메타스토어 없음)

**데이터 위치:** GCS
**메타스토어:** 없음 (BigQuery 카탈로그에 스키마만 등록)
**BigLake Connection:** 사용하지 않음

가장 단순한 External Table입니다. GCS의 파일을 직접 URI로 참조하며, BigLake Connection 없이 사용자의 GCS 권한으로 직접 접근합니다.

```sql
CREATE EXTERNAL TABLE project.dataset.ext_sales
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bucket/sales/*.parquet']
);
```

CSV, JSON(newline-delimited), Avro, Parquet, ORC 등 다양한 파일 포맷을 지원합니다.

### 장점

- **설정 최소화**: URI만 지정하면 바로 쿼리 가능
- **데이터 이동 없음**: GCS에 있는 파일을 그대로 사용
- **비용 절감**: BigQuery 스토리지 비용 없음 (쿼리 비용만 발생)
- **ETL 파이프라인 간소화**: 별도 로드 단계 불필요

### 단점

- **성능 제한**: 매 쿼리마다 GCS에서 파일을 읽으므로 Native 대비 느림
- **DML 불가**: 읽기 전용 (INSERT/UPDATE/DELETE 불가)
- **메타데이터 캐싱 없음**: 쿼리마다 파일 목록을 다시 탐색
- **보안 제한**: Row/Column 수준 보안 미지원
- **권한 관리 복잡**: 쿼리 실행자가 GCS 버킷에 직접 권한이 있어야 함
- **파일 관리 부담**: 파일 추가/삭제 시 URI 패턴에 맞춰야 함
- **Hive 파티션 탐색 제한**: `_FILE_NAME` 가상 컬럼을 사용하거나 Hive 파티셔닝 옵션을 별도로 설정해야 함

### 적합한 경우

- 빠른 PoC나 일회성 분석
- 소규모 데이터셋 (수 GB 이하)
- 데이터 파이프라인 초기 단계에서 빠르게 테스트

---

## 2-B. BigLake External Table (Flat Files)

**데이터 위치:** GCS
**메타스토어:** 없음 (BigQuery 카탈로그에 스키마만 등록)
**BigLake Connection:** 사용

기본 External Table에 BigLake Connection을 추가한 형태입니다. 구조는 동일하지만, **접근 위임(Access Delegation)**이 핵심 차이입니다.

```sql
-- BigLake Connection 생성 (1회)
-- Console: BigQuery → External Connections → Cloud Resource Connection

CREATE EXTERNAL TABLE project.dataset.bl_sales
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bucket/sales/*.parquet']
);
```

BigLake Connection의 서비스 계정이 GCS에 접근하므로, 개별 사용자에게 GCS 권한을 부여할 필요가 없습니다.

### 2-A 대비 추가 장점

- **접근 위임**: 사용자에게 GCS 직접 권한 불필요 → 데이터 유출 위험 감소
- **Row/Column 수준 보안**: 정책 태그와 행 수준 보안 필터 적용 가능
- **메타데이터 캐싱**: BigQuery가 파일 목록과 스키마를 캐싱하여 쿼리 계획 시간 단축
- **BigQuery 옵티마이저 통합**: 더 나은 쿼리 최적화 가능
- **통합 거버넌스**: Data Catalog, DLP 등과 연계

### 단점

- **여전히 읽기 전용**: DML 미지원
- **BigLake Connection 설정 필요**: 초기 설정이 기본 External Table보다 복잡
- **쿼리 성능**: Native Table 대비 여전히 느림 (GCS I/O)
- **메타데이터 캐시 갱신**: 파일이 자주 변경되면 캐시 무효화 전략 필요

### 적합한 경우

- **프로덕션 환경**에서 GCS 데이터를 BigQuery로 조회해야 하는 경우
- 데이터 거버넌스와 보안이 중요한 조직
- 다수의 사용자가 동일 GCS 데이터를 조회하는 환경

---

## 2-C. BigLake Managed Iceberg Table

**데이터 위치:** GCS (Parquet)
**메타스토어:** BigQuery 내부 (Big Metadata)
**BigLake Connection:** 사용

BigQuery가 Iceberg 메타데이터를 **내부적으로 완전 관리**하는 방식입니다. 사용자 입장에서는 Native Table과 거의 동일한 경험을 제공하면서, 데이터는 고객 소유의 GCS 버킷에 열린 포맷(Parquet)으로 저장됩니다.

```sql
CREATE TABLE project.dataset.managed_sales (
  order_id INT64,
  customer_id STRING,
  amount NUMERIC,
  order_date DATE
)
CLUSTER BY customer_id
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  file_format = 'PARQUET',
  table_format = 'ICEBERG',
  storage_uri = 'gs://my-bucket/managed-sales'
);

-- Native Table과 동일하게 DML 사용 가능
INSERT INTO project.dataset.managed_sales
VALUES (1, 'C001', 150.00, '2026-01-15');

UPDATE project.dataset.managed_sales
SET amount = 200.00
WHERE order_id = 1;
```

### 장점

- **Native Table 수준의 사용성**: DML, Streaming, Time Travel 전부 지원
- **열린 포맷**: 데이터가 GCS에 Parquet으로 저장되어 벤더 종속 최소화
- **자동 최적화**: 파일 크기 최적화, 클러스터링, 메타데이터 컴팩션, 고아 파일 GC 자동 수행
- **Row/Column 보안**: 완전 지원
- **외부 엔진 접근 가능**: `EXPORT TABLE METADATA`로 Iceberg 메타데이터를 내보내면 Spark 등에서 읽기 가능

### 단점

- **외부 엔진 읽기가 즉시적이지 않음**: Spark에서 읽으려면 `EXPORT TABLE METADATA` 실행 필요
- **외부 엔진 쓰기 불가**: BigQuery만 쓰기 가능, Spark에서 직접 INSERT 불가
- **BigQuery 종속**: 메타데이터가 BigQuery 내부에 있으므로 BigQuery 없이는 테이블 관리 불가
- **BigLake Connection 필수**: 초기 설정 비용

### 적합한 경우

- BigQuery가 **주 분석 엔진**이면서, 가끔 Spark 등에서 데이터를 읽어야 하는 경우
- Native Table의 기능이 필요하면서 데이터를 열린 포맷으로 유지하고 싶은 경우
- 벤더 종속을 줄이면서도 BigQuery의 편의성을 포기하고 싶지 않은 경우

---

## 2-D. BigLake External Table (Open Table Format)

**데이터 위치:** GCS (Parquet)
**메타스토어:** GCS의 메타데이터 파일 (자체 관리)
**BigLake Connection:** 사용

Spark, Flink 등 외부 엔진이 GCS에 직접 Open Table Format(Iceberg, Delta Lake, Hudi)으로 데이터를 쓰고, BigQuery에서 이를 읽는 방식입니다. 메타데이터 파일이 GCS에 존재하며, **데이터를 쓰는 엔진이 메타스토어를 직접 관리**합니다.

### Iceberg

```sql
CREATE EXTERNAL TABLE project.dataset.ext_iceberg_sales
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  format = 'ICEBERG',
  uris = ["gs://my-bucket/warehouse/sales/metadata/v3.metadata.json"]
);
```

Spark 등이 GCS에 Iceberg 표준 메타데이터(`metadata.json`, manifest list, manifest file)를 직접 관리합니다. BigQuery는 이 메타데이터를 읽어서 쿼리합니다.

### Delta Lake

```sql
CREATE EXTERNAL TABLE project.dataset.ext_delta_sales
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  format = 'DELTA_LAKE',
  uris = ["gs://my-bucket/delta/sales"]
);
```

Delta Lake의 트랜잭션 로그(`_delta_log/`)를 BigQuery가 직접 해석합니다.

### Apache Hudi

```sql
CREATE EXTERNAL TABLE project.dataset.ext_hudi_sales
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  format = 'HUDI',
  uris = ["gs://my-bucket/hudi/sales"]
);
```

Hudi 테이블은 manifest 파일 기반으로 BigQuery에서 조회할 수 있습니다.

### 장점

- **외부 엔진 주도**: Spark/Flink가 자유롭게 데이터를 쓰고, BigQuery에서 즉시 조회
- **추가 인프라 불필요**: GCS만 있으면 되므로 별도 메타스토어 서비스 없음
- **오픈 포맷 호환**: 표준 Iceberg/Delta/Hudi 포맷이라 다양한 엔진에서 접근 가능
- **비용 효율**: 메타스토어 서비스 비용 없음

### 단점

- **BigQuery에서 읽기 전용**: BigQuery로는 DML 불가
- **메타데이터 수동 관리**: Iceberg의 경우 `metadata.json` URI를 직접 관리해야 하고, 스냅샷이 업데이트될 때마다 URI를 갱신해야 할 수 있음
- **스키마 진화 복잡**: BigQuery 외부 테이블 정의와 실제 메타데이터 간 불일치 가능
- **컴팩션 직접 수행**: small files 문제를 Spark 등에서 직접 해결해야 함
- **동시성 제어 제한**: 여러 엔진이 동시에 쓰면 충돌 가능 (특히 Catalog 없이 사용 시)

### Open Table Format 비교

| | Iceberg | Delta Lake | Hudi |
|---|---|---|---|
| BigQuery 지원 수준 | 가장 성숙 | 네이티브 지원 | manifest 기반 |
| Time Travel (BigQuery) | 제한적 | 제한적 | 제한적 |
| Schema Evolution | 지원 | 지원 | 지원 |
| 메타 관리 복잡도 | metadata.json URI 관리 | _delta_log 자동 | manifest 관리 |
| Partition Pruning | manifest 기반 file pruning | 통계 기반 | 제한적 |
| GCP 생태계 통합 | 최고 (BigLake 완전 지원) | 좋음 | 기본 |

### 적합한 경우

- Spark/Flink가 **주 적재 엔진**이고 BigQuery는 분석 전용인 환경
- 별도 메타스토어 서비스를 운영하고 싶지 않은 소규모 팀
- 이미 Iceberg/Delta/Hudi로 데이터를 관리하고 있어 BigQuery에서 조회만 하면 되는 경우

---

## 2-E. BigLake Metastore + REST Catalog

**데이터 위치:** GCS (Parquet)
**메타스토어:** BigLake Metastore (GCP 관리형 서비스)
**BigLake Connection:** 사용

GCP가 제공하는 **관리형 Iceberg REST Catalog** 서비스입니다. Iceberg의 표준 REST Catalog API를 지원하므로, Spark·Flink·BigQuery·Databricks·Trino 등 여러 엔진이 동일한 메타스토어를 공유할 수 있습니다.

### Spark에서 테이블 생성

```python
spark.conf.set("spark.sql.catalog.my_catalog", "org.apache.iceberg.spark.SparkCatalog")
spark.conf.set("spark.sql.catalog.my_catalog.type", "rest")
spark.conf.set("spark.sql.catalog.my_catalog.uri",
    "https://biglake.googleapis.com/iceberg/v1beta/restcatalog")
spark.conf.set("spark.sql.catalog.my_catalog.warehouse",
    "projects/PROJECT/locations/REGION/catalogs/CATALOG")
spark.conf.set("spark.sql.catalog.my_catalog.token", "<access_token>")

spark.sql("""
  CREATE TABLE my_catalog.db.sales (
    order_id BIGINT,
    customer_id STRING,
    amount DECIMAL(10,2),
    order_date DATE
  ) USING iceberg
  PARTITIONED BY (month(order_date))
""")
```

### BigQuery에서 조회

BigLake Metastore에 등록된 테이블은 BigQuery에서 자동으로 보입니다. 별도의 `CREATE EXTERNAL TABLE` 없이도 BigQuery 카탈로그에 나타납니다.

```sql
SELECT * FROM `project.dataset.sales`
WHERE order_date >= '2026-01-01';
```

### 장점

- **진정한 멀티엔진**: Spark, BigQuery, Databricks, Flink, Trino 모두에서 읽기/쓰기 가능
- **서버리스**: 메타스토어 인프라 관리 불필요
- **자동 동기화**: 한 엔진에서 변경한 메타데이터가 다른 엔진에서 즉시 반영
- **표준 API**: Iceberg REST Catalog 표준을 따르므로 벤더 종속 최소
- **Credential Vending**: 세밀한 접근 제어 지원
- **metadata.json URI 수동 관리 불필요**: 카탈로그가 자동 추적

### 단점

- **Iceberg 전용**: Delta Lake, Hudi는 지원하지 않음
- **서비스 비용**: BigLake Metastore 사용에 따른 추가 비용
- **Region 제약**: 지원 Region이 제한적일 수 있음
- **비교적 새로운 서비스**: 아직 GA 전이거나 기능이 빠르게 변화 중
- **설정 복잡도**: Spark Catalog 설정, 인증 토큰 관리 등 초기 설정이 복잡

### 적합한 경우

- **멀티엔진 레이크하우스**: Spark로 적재하고 BigQuery + Databricks로 분석하는 환경
- Iceberg 기반 데이터 레이크를 구축하면서 메타스토어 운영 부담을 줄이고 싶은 경우
- 여러 팀/서비스가 동일 데이터를 다양한 엔진으로 접근하는 대규모 조직

---

## 2-F. Dataproc Metastore (Hive Metastore Service)

**데이터 위치:** GCS (Parquet, ORC 등)
**메타스토어:** Dataproc Metastore (GCP 관리형 Hive Metastore)
**BigLake Connection:** 사용 가능

GCP가 관리하는 **Hive Metastore 호환 서비스**입니다. 기존 Hadoop/Hive 생태계와의 호환성이 핵심입니다.

### Dataproc에서 테이블 생성

```sql
-- Dataproc Spark SQL
CREATE TABLE db.sales (
  order_id BIGINT,
  customer_id STRING,
  amount DECIMAL(10,2)
)
PARTITIONED BY (order_date STRING)
STORED AS PARQUET
LOCATION 'gs://my-bucket/hive/sales';
```

### BigQuery에서 연동

Dataproc Metastore에 등록된 Hive 테이블을 BigQuery에서 외부 테이블로 연결할 수 있습니다.

```sql
CREATE EXTERNAL TABLE project.dataset.hive_sales
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my-bucket/hive/sales/*'],
  hive_partition_uri_prefix = 'gs://my-bucket/hive/sales',
  require_hive_partition_filter = true
);
```

### 장점

- **Hive 호환**: 기존 Hive 쿼리, Spark SQL, Presto 등과 완벽 호환
- **관리형 서비스**: Hive Metastore를 직접 운영할 필요 없음
- **Dataproc 통합**: Dataproc 클러스터와 자동 연결
- **성숙한 생태계**: 수년간 검증된 Hive Metastore 프로토콜
- **다양한 테이블 포맷**: Hive, Iceberg, Delta Lake 등 여러 포맷의 메타데이터 저장 가능

### 단점

- **BigLake Metastore 대비 레거시**: Google은 BigLake Metastore로의 마이그레이션을 권장
- **인스턴스 관리 필요**: 서버리스가 아닌 인스턴스 기반이라 크기 조정 필요
- **비용**: 인스턴스 비용이 발생 (사용하지 않아도 과금)
- **BigQuery 연동 제한**: BigQuery에서 직접 Hive Metastore를 참조하지 못하고, 외부 테이블을 별도로 생성해야 함
- **메타데이터 동기화**: BigQuery 외부 테이블의 스키마가 Hive 테이블 변경을 자동 반영하지 않을 수 있음

### 적합한 경우

- **기존 Hadoop/Hive 워크로드**를 GCP로 마이그레이션한 환경
- Hive 호환 메타스토어가 필요한 레거시 시스템과의 연계
- Dataproc 클러스터를 주로 사용하는 팀

---

## 2-G. Self-hosted Hive Metastore

**데이터 위치:** GCS (Parquet, ORC 등)
**메타스토어:** 직접 구축한 Hive Metastore (GCE/GKE에서 운영)
**BigLake Connection:** 사용 가능

Hive Metastore를 GCE VM이나 GKE 위에 직접 설치하고 운영하는 방식입니다.

### 구성 예시

```
┌─────────────────────────────────────┐
│ GCE VM / GKE Pod                    │
│  ├── Hive Metastore Service         │
│  └── Backend DB (MySQL / PostgreSQL)│
└──────────────┬──────────────────────┘
               │ Thrift Protocol
    ┌──────────┼──────────┐
    │          │          │
  Spark     Presto    BigQuery
                    (External Table)
```

### 장점

- **완전한 제어**: 버전, 설정, 플러그인 등을 자유롭게 커스터마이징
- **비용 최적화 가능**: 소규모 환경에서는 작은 VM으로 운영 가능
- **특수 요구사항 대응**: 커스텀 Serde, UDF 등 특수 기능 사용 가능

### 단점

- **운영 부담 최대**: 가용성, 백업, 업그레이드, 모니터링 전부 직접 관리
- **단일 장애점**: Metastore 다운 시 모든 연관 워크로드 영향
- **스케일링 직접 관리**: 부하 증가에 따른 스케일업/아웃 직접 수행
- **BigQuery 연동 번거로움**: BigQuery에서 직접 참조 불가, 별도 외부 테이블 생성 필요
- **보안 관리**: 네트워크, 인증 등 보안 구성 직접 수행

### 적합한 경우

- **온프레미스에서 마이그레이션 중**이고 Hive Metastore를 그대로 가져온 경우
- Dataproc Metastore의 기능이나 Region 지원이 부족한 특수 환경
- 이미 Hive Metastore 운영 노하우가 있는 팀

---

## 3. Object Table (비정형 데이터)

**데이터 위치:** GCS (이미지, PDF, 텍스트, 오디오 등)
**메타스토어:** 없음

구조화된 데이터가 아닌 **비정형 데이터**를 BigQuery에서 다루기 위한 특수한 테이블 유형입니다.

```sql
CREATE EXTERNAL TABLE project.dataset.product_images
WITH CONNECTION `project.region.my-connection`
OPTIONS (
  object_metadata = 'SIMPLE',
  uris = ['gs://my-bucket/images/*']
);

-- 파일 메타데이터 조회
SELECT uri, content_type, size, updated
FROM project.dataset.product_images;

-- BigQuery ML과 연동하여 이미지 분류
SELECT uri, ml_predict_row.label
FROM ML.PREDICT(
  MODEL `project.dataset.vision_model`,
  TABLE `project.dataset.product_images`
);
```

### 장점

- **비정형 데이터 통합**: SQL로 이미지, 문서 등의 메타데이터 조회 가능
- **BigQuery ML 연동**: 비정형 데이터에 대한 ML 추론 파이프라인 구성 가능
- **Vertex AI 통합**: 비전 모델 등과 연계하여 분석

### 단점

- **데이터 자체를 읽지는 않음**: 파일의 **메타데이터**만 BigQuery에서 조회
- **특수 목적**: 일반적인 데이터 분석과는 용도가 다름

---

## 전체 비교 요약

### 기능 비교

| 방식 | DML | Streaming | Row/Col 보안 | Time Travel | 파티션 Pruning |
|------|-----|-----------|-------------|-------------|---------------|
| Native Table | 전체 지원 | 지원 | 지원 | 7일 | 지원 |
| 기본 External | 불가 | 불가 | 미지원 | 없음 | Hive 파티션만 |
| BigLake External (flat) | 불가 | 불가 | 지원 | 없음 | Hive 파티션만 |
| Managed Iceberg | 전체 지원 | 지원 | 지원 | 지원 | 지원 |
| External Iceberg/Delta | 불가 | 불가 | 지원 | 제한적 | Manifest 기반 |
| BigLake Metastore | Spark 쓰기 | 불가 | 지원 | 지원 | 지원 |
| Dataproc Metastore | Spark 쓰기 | 불가 | 제한적 | 엔진 의존 | Hive 파티션 |
| Self-hosted HMS | Spark 쓰기 | 불가 | 미지원 | 엔진 의존 | Hive 파티션 |

### 메타스토어 관리 비교

| 방식 | 메타스토어 관리 주체 | 관리 부담 | 멀티엔진 | 비용 |
|------|---------------------|-----------|----------|------|
| Native Table | BigQuery (내부) | 없음 | BigQuery 전용 | 스토리지+쿼리 |
| 기본 External | 없음 (URI 직접) | 최소 | BigQuery 전용 | 쿼리만 |
| BigLake External (flat) | BigQuery 카탈로그 | 낮음 | BigQuery 전용 | 쿼리만 |
| Managed Iceberg | BigQuery (내부) | 없음 | export 필요 | 스토리지+쿼리 |
| External Iceberg/Delta | 데이터 쓰기 엔진 | **높음** (수동) | 제한적 | GCS만 |
| BigLake Metastore | GCP (서버리스) | 낮음 | **완전 지원** | 서비스+GCS |
| Dataproc Metastore | GCP (인스턴스) | 중간 | Hive 호환 | 인스턴스+GCS |
| Self-hosted HMS | **직접 운영** | **최대** | Hive 호환 | VM+GCS |

### 성능 비교 (상대적)

| 방식 | 쿼리 성능 | 적재 성능 | 스캔 최적화 |
|------|-----------|-----------|-------------|
| Native Table | ★★★★★ | ★★★★ | 파티션+클러스터 |
| 기본 External | ★★☆☆☆ | N/A | 제한적 |
| BigLake External (flat) | ★★★☆☆ | N/A | 메타데이터 캐싱 |
| Managed Iceberg | ★★★★☆ | ★★★★ | 파티션+클러스터 |
| External Iceberg | ★★★★☆ | 외부 엔진 | Manifest pruning |
| BigLake Metastore | ★★★★☆ | 외부 엔진 | Manifest pruning |

---

## 의사결정 가이드

### 질문 1: BigQuery 외에 다른 엔진이 필요한가?

```
BigQuery만 사용
├── DML이 필요한가?
│   ├── Yes → Native Table 또는 Managed Iceberg
│   └── No  → BigLake External Table (flat files)
│
Spark/Flink/Databricks도 사용
├── 어느 엔진이 데이터를 쓰는가?
│   ├── BigQuery가 씀 → Managed Iceberg + EXPORT METADATA
│   ├── Spark가 씀   → BigLake Metastore (REST Catalog) 또는 External Iceberg
│   └── 양쪽 다 씀   → BigLake Metastore (REST Catalog)
```

### 질문 2: 메타스토어 운영에 얼마나 투자할 수 있는가?

```
전혀 관리하고 싶지 않다
├── BigQuery 중심 → Native Table
└── 멀티엔진      → BigLake Metastore (서버리스)

최소한으로 관리하겠다
├── 메타데이터 파일만 → External Iceberg (GCS)
└── Hive 호환 필요   → Dataproc Metastore

전부 직접 제어하겠다
└── Self-hosted Hive Metastore
```

### 질문 3: 기존 환경은 무엇인가?

| 기존 환경 | 권장 방식 |
|-----------|-----------|
| 온프레미스 Hive → GCP 마이그레이션 | Dataproc Metastore → 점진적으로 BigLake Metastore |
| 신규 데이터 레이크 구축 | BigLake Metastore (REST Catalog) |
| BigQuery만 사용, 외부 데이터 간헐적 조회 | BigLake External Table (flat files) |
| BigQuery 중심, 데이터 portability 필요 | Managed Iceberg Table |
| Spark 중심, BigQuery는 분석 전용 | External Iceberg 또는 BigLake Metastore |
| PoC / 일회성 분석 | 기본 External Table |

---

## 자주 하는 오해

### "External Table이면 다 같은 거 아닌가요?"

아닙니다. BigLake Connection 유무에 따라 보안 모델이 완전히 달라지고, Open Table Format 사용 여부에 따라 pruning 성능이 크게 차이납니다. 같은 "External Table"이라는 이름이지만, 기본 External Table과 BigLake Metastore 기반 Iceberg 테이블은 **아키텍처적으로 완전히 다른 방식**입니다.

### "Managed Iceberg이면 Native Table이랑 뭐가 다른가요?"

사용성은 비슷하지만, 데이터가 **고객 소유 GCS 버킷에 Parquet으로 저장**된다는 점이 핵심 차이입니다. Native Table은 BigQuery 내부 Capacitor 포맷이라 다른 엔진에서 읽을 수 없지만, Managed Iceberg는 메타데이터를 export하면 Spark 등에서도 읽을 수 있습니다. 반면 쿼리 성능은 Native Table이 조금 더 좋습니다.

### "BigLake Metastore와 Dataproc Metastore는 같은 건가요?"

다릅니다. Dataproc Metastore는 **Hive Metastore 호환** 서비스이고, BigLake Metastore는 **Iceberg REST Catalog 표준** 서비스입니다. Google은 BigLake Metastore로의 마이그레이션을 권장하고 있으며, 이를 위한 마이그레이션 도구도 제공합니다.

### "GCS에 Parquet만 올려두면 바로 쿼리할 수 있나요?"

기본 External Table이나 BigLake External Table(flat files)을 만들면 가능합니다. 하지만 이 방식은 **파일 수준에서만** 작동하므로, 테이블 수준의 스키마 진화, ACID 트랜잭션, Time Travel 등은 지원되지 않습니다. 이런 기능이 필요하면 Iceberg 같은 Open Table Format을 사용해야 합니다.

---

## 마무리

| 핵심 판단 기준 | 권장 방식 |
|---------------|-----------|
| 최고 성능, BigQuery만 사용 | **Native Table** |
| BigQuery 중심 + 데이터 portability | **Managed Iceberg** |
| 소규모, 빠른 시작, 읽기 전용 | **기본 External Table** |
| 프로덕션 외부 데이터 + 거버넌스 | **BigLake External (flat)** |
| Spark 적재 + BigQuery 분석, 간단한 구성 | **External Iceberg** |
| 멀티엔진 레이크하우스 | **BigLake Metastore (REST Catalog)** |
| Hive 레거시 마이그레이션 | **Dataproc Metastore** |

BigQuery에서 데이터를 다루는 방법은 생각보다 많고, 각 방식마다 메타스토어 관리 주체·성능·기능·운영 부담이 다릅니다. "어떤 방식이 제일 좋은가"보다는 **"우리 조직의 데이터 흐름에서 메타스토어를 누가, 어떻게 관리하는 것이 가장 자연스러운가"**를 기준으로 선택하는 것이 핵심입니다.
