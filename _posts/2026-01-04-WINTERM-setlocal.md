2026-01-04-WINTERM-setlocal.md
---
layout: default
title: 2026-01-03-WINTERM-setlocal.md
---

## 사족

윈도셀 (cmd)에서 ` setlocal .. endlocal ` 구문을 발견

---

## by Gemini

`setlocal`은 윈도우 배치 스크립트(`.bat`, `.cmd`) 내에서 **환경 변수의 범위를 현재 스크립트(또는 특정 블록)로 제한**하겠다는 선언입니다. 

쉽게 말해, 스크립트 안에서 변수를 만들고 값을 바꿔도 **스크립트가 끝나면 원래 시스템 환경에는 아무런 영향을 주지 않도록 보호막을 치는 것**입니다.

---

### 1. 주요 특징
* **지역화(Localization):** `setlocal` 이후에 정의된 모든 변수는 `endlocal`을 만나거나 스크립트가 종료되면 메모리에서 사라집니다.
* **환경 보호:** 시스템 전역 변수나 사용자 변수를 실수로 덮어쓰는 것을 방지합니다.
* **쌍을 이루는 명령:** 보통 스크립트 상단에 `setlocal`, 하단에 `endlocal`을 배치합니다.

---

### 2. 왜 사용해야 하나요? (예시)

만약 `MinTool4PG`라는 툴을 만드는데, 내부적으로 `PGPORT`라는 변수를 사용한다고 가정해 보겠습니다.

#### **`setlocal`이 없을 때**
1.  사용자가 원래 `PGPORT=5432`를 쓰고 있었습니다.
2.  사용자님의 스크립트가 실행되면서 내부적으로 `set PGPORT=5433`으로 바꿨습니다.
3.  스크립트 종료 후에도 사용자의 터미널에는 `PGPORT`가 **5433**으로 남아버립니다. (다른 접속 시 오류 발생 원인)

#### **`setlocal`이 있을 때**
```batch
@echo off
setlocal
:: 여기서 바꾸는 값은 이 스크립트 안에서만 유효함
set PGPORT=5433
echo 스크립트 내부 포트: %PGPORT%
endlocal

:: 스크립트 종료 후
echo 원래 포트로 복구됨: %PGPORT%
```

---

### 3. `setlocal EnableDelayedExpansion` (중요 팁)
DBA로서 복잡한 루프(Loop) 문을 작성할 때 `setlocal` 뒤에 자주 붙이는 옵션입니다.

* **지연된 확장:** 루프(`for`문 등) 안에서 변수 값이 실시간으로 변할 때, `%var%` 대신 `!var!`를 사용하여 변경된 값을 즉시 참조할 수 있게 해줍니다.
* **예시:**
    ```batch
    setlocal EnableDelayedExpansion
    set COUNT=0
    for %%i in (*.sql) do (
        set /a COUNT+=1
        echo 현재 !COUNT!번째 파일 처리 중...
    )
    endlocal
    ```

---

### 4. `MinTool4PG` 적용 가이드
사용자님이 만드시는 툴킷 배치 파일의 표준 템플릿은 다음과 같은 형태가 되어야 안전합니다.

```batch
@echo off
:: 1. 환경 변수 오염 방지 및 지연 확장 활성화
setlocal EnableDelayedExpansion

:: 2. 경로 이동 (popd로 돌아올 수 있게)
pushd %~dp0

:: [여기에 툴킷 로직 작성]
set TEST_VAR=Secret

:: 3. 원래 경로로 복귀 및 변수 해제
popd
endlocal
:: 이제 TEST_VAR는 존재하지 않음
```

---

### 💡 요약
`setlocal`은 **"여기서 벌어지는 일은 여기서 끝내라"**는 선언입니다. 툴킷을 배포했을 때 사용자의 PC 환경을 망가뜨리지 않으려면 배치 파일 상단에 반드시 넣어주는 것이 '매너'이자 '기술'입니다. 
