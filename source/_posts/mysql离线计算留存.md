---
title: mysql离线计算留存
date: 2019-08-27 17:38:42
tags:
- mysql
- 留存
- join
- olap
categories:
- mysql
---

# 表结构

| id | spu_code | wk |
|---|---|---|
|  1 | 1 | 1 |
|  2 | 1 | 2 |
|  3 | 1 | 3 |
|  4 | 1 | 1 |
|  5 | 1 | 2 |
|  6 | 1 | 7 |
|  7 | 1 | 11 |
|  8 | 2 | 7 |
|  9 | 1 | 11 |

- 唯一标识：spu_code
- 时间字段：wk

# 计算sql

```
SELECT s.spu_code AS spu_code, s.wk AS wk, t.wk AS k_wk
FROM (
	SELECT spu_code, wk
	FROM cal_cohort_demo
	GROUP BY spu_code, wk
) s
	LEFT JOIN (
		SELECT *
		FROM cal_cohort_demo
		GROUP BY spu_code, wk
	) t
	ON s.spu_code = t.spu_code
		AND t.wk > s.wk
WHERE t.spu_code IS NOT NULL
ORDER BY s.spu_code, s.wk + 0, t.wk + 0;
```

# 计算结果

| spu_code | wk | k_wk |
|---|---|---|
| 1 | 1    | 2    |
| 1 | 1    | 3    |
| 1 | 1    | 7    |
| 1 | 1    | 11   |
| 1 | 2    | 3    |
| 1 | 2    | 7    |
| 1 | 3    | 7    |
| 1 | 11   | 2    |
| 1 | 11   | 3    |
| 1 | 11   | 7    |
