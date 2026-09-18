---
layout: post
title: "BigQuery How to extract domain from URL"
date: 2023-01-26
tags: [bigquery, gcp, sql]
---

[![](https://euriion.com/wp-content/uploads/2023/08/image-33.png)](https://euriion.com/wp-content/uploads/2023/08/image-33.png)

If you want extract a domain from provided URL strings in BigQuery.

It’s very easy. You can use `net.Host` function.

```
WITH
  examples AS (
  SELECT "https://some.domain.com/path?query=param#hash" AS example
  UNION ALL
  SELECT "some.domain.com/path?query=param#hash" AS example)
SELECT
  NET.HOST(example)
FROM
  examples
```
