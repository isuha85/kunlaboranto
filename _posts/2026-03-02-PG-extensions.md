---
layout: default
title: 2026-03-02-PG-extensions.md
---

## 목적

PG 17.x 버전기준으로 extension 사용예를 수집


## 더미 (정리전)

* (부정확) extention 의 설치위치가 각 DATABASE 단위인 경우도 있고, postgres DATABASE 에서만 조회되는 경우도 있다.

### `postgresql.conf`

* main/postgresql.conf 파일

```BASH
 shared_preload_libraries = 'pg_stat_statements,auto_explain,pg_hint_plan'

# (2026.04.20 - Azure PostgreSQL 17.6 )
#shared_preload_libraries = 'auto_explain,pg_cron,pg_hint_plan',pg_squeeze,pg_stat_statements,pgaudit,azure,pg_qs,pgaadauth,pgms_stats,pgms_wait_sampling,pg_availabliity'
```
