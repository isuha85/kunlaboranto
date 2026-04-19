---
layout: default
title: 2026-03-01-PG_OOM_ADJUST_VALUE.md
---

## 사족

멀티-프로세스 구조인, PostgreSQL 은 /proc/<pid>/oom_score_adj 설정값을 조정하여, 주요 데몬을 더 강하게 보호할 수 있다. ( **OOM Killer** )

`결과예`

```BASH

$ pe post
UID          PID    PPID  C STIME TTY          TIME CMD
postgres     322       1  0 13:49 ?        00:00:02 /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/17/main -c config_file=/etc/postgresql/17/main/postgresql.conf
postgres     553     322  0 13:49 ?        00:00:00 postgres: 17/main: checkpointer 
postgres     554     322  0 13:49 ?        00:00:00 postgres: 17/main: background writer 
postgres     584     322  0 13:49 ?        00:00:00 postgres: 17/main: walwriter 
postgres     585     322  0 13:49 ?        00:00:00 postgres: 17/main: autovacuum launcher 
postgres     588     322  0 13:49 ?        00:00:00 postgres: 17/main: logical replication launcher 
postgres   15149     322  0 21:17 ?        00:00:00 postgres: 17/main: test test 127.0.0.1(44320) idle
core       15151    

$ grep ^ /proc/15149/oom_score*
/proc/15149/oom_score:668
/proc/15149/oom_score_adj:0

$ grep ^ /proc/322/oom_score*
/proc/322/oom_score:73
/proc/322/oom_score_adj:-900

$ grep ^ /proc/554/oom_score*
/proc/554/oom_score:667
/proc/554/oom_score_adj:0

$ grep ^ /proc/553/oom_score*
/proc/553/oom_score:668
/proc/553/oom_score_adj:0

```

---

## From Gemini

**`PG_OOM_ADJUST_FILE`**와 **`PG_OOM_ADJUST_VALUE`**는 리눅스 환경에서 PostgreSQL 서버(Postmaster)가 시스템의 **OOM(Out of Memory) Killer**에 의해 갑자기 종료되는 것을 방지하기 위해 사용하는 환경 변수입니다.

리눅스 커널은 메모리가 부족할 때 어떤 프로세스를 죽일지 결정하는데, 데이터베이스처럼 중요한 프로세스가 죽는 것을 막으려는 목적이 큽니다.

---

### 1. 주요 의미와 작동 원리

리눅스 시스템에는 `/proc/[PID]/oom_score_adj`라는 파일이 존재합니다. 이 파일의 값을 수정하면 특정 프로세스가 OOM Killer의 대상이 될 우선순위를 조정할 수 있습니다.

* **`PG_OOM_ADJUST_FILE`**: 커널의 OOM 점수를 기록할 **대상 파일 경로**를 지정합니다. (보통 `/proc/self/oom_score_adj` 또는 구형 커널의 경우 `/proc/self/oom_adj`)
* **`PG_OOM_ADJUST_VALUE`**: 해당 파일에 써넣을 **점수(값)**입니다.
    * **0**: 기본값.
    * **음수 (예: -1000)**: OOM Killer가 이 프로세스를 죽일 확률을 낮춥니다. (절대 죽이지 않으려면 -1000)
    * **양수**: OOM Killer가 이 프로세스를 먼저 죽이게 합니다.



---

### 2. 왜 사용하나요? (DBA의 관점)

PostgreSQL은 다중 프로세스 아키텍처를 사용합니다. 만약 메모리가 부족할 때 리눅스 커널이 **부모 프로세스(Postmaster)**를 죽여버리면 전체 데이터베이스 서비스가 중단됩니다. 

따라서 보통 다음과 같은 전략을 씁니다:
1.  **부모 프로세스(Postmaster)**: OOM 점수를 매우 낮게(`-1000`) 설정하여 절대 죽지 않게 보호합니다.
2.  **자식 프로세스(Backend)**: 개별 쿼리를 수행하는 프로세스들은 상대적으로 점수를 높여, 정말 급할 때 서비스 전체가 아닌 특정 쿼리 프로세스만 종료되도록 유도합니다.

---
