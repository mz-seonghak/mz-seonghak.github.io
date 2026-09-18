---
layout: post
title: "BigQuery: too many subqueries or query is too complex"
date: 2023-02-06
tags: [bigquery, gcp, sql, troubleshooting]
---

[![](https://euriion.com/wp-content/uploads/2023/02/image-1.png)](https://euriion.com/wp-content/uploads/2023/02/image-1.png)

BigQuery에서 with 구문을 많이 사용하거나 Sub query, Inline view를 과도하게 사용하면 나오는 오류입니다.

### 해결 방법

쿼리를 분할하거나 단계를 간단하게 줄이는 방법밖에 없습니다.

join할 때 on 구문에 조건이 많을 때에도 이 에러가 나오므로 view를 생성하거나 하는 방법으로 회피할 수 있습니다.
