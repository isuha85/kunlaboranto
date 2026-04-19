---
layout: default
title: 2026-01-03-윈도환경-LD_PRELOAD.md
---

## 사족

```
DOS> chcp 65001
DOS> psql .. 
WARNING: Console code page (65001) differs from Windows code page (949)
..
```

메시지를 지우는 방법을 검색하다가 기록만 해둠
과거 리눅스 환경에서 PBT (ProBlem-Tracking) 등으로 사용했던 LD_PRELOAD 방식이 윈도우는 어떨까 검색만 해봄.

---

## by Gemini

윈도우 환경에서 리눅스의 `LD_PRELOAD`와 유사하게 특정 API(`GetACP`)를 가로채는(Hooking) 방법은 몇 가지가 있습니다. 윈도우에는 `LD_PRELOAD`라는 직접적인 환경 변수는 없지만, **DLL Injection**이나 **IAT(Import Address Table) Hooking** 기술을 사용하여 동일한 효과를 낼 수 있습니다.

사용자님의 목적이 `psql`이 읽어가는 시스템 코드 페이지 값을 속여서 경고를 없애는 것이라면, 가장 대중적이고 강력한 방법들을 소개해 드립니다.

---

### 1. Detours 라이브러리 활용 (가장 표준적인 방법)
Microsoft에서 만든 **Detours**는 윈도우 API 후킹을 위한 가장 안정적인 라이브러리입니다.

* **원리:** 실행 중인 프로세스의 메모리에 직접 들어가 `GetACP` 함수의 시작 부분을 `jmp` 명령어로 수정하여 사용자가 정의한 함수로 보냅니다.
* **구현 단계:**
    1.  `GetACP`를 대신할 커스텀 함수가 포함된 DLL을 작성합니다. (예: 항상 `65001`을 반환하도록 설정)
    2.  `DetourCreateProcessWithDll` API를 사용하여 `psql.exe`를 실행할 때 해당 DLL을 함께 로드합니다.

### 2. `withdll.exe` 도구 사용 (LD_PRELOAD와 가장 유사한 경험)
Detours 샘플에 포함된 `withdll.exe`를 사용하면 커맨드 라인에서 리눅스의 `LD_PRELOAD`처럼 간단하게 실행할 수 있습니다.

```cmd
:: 리눅스의 LD_PRELOAD=./hook.so ./psql 과 유사한 방식
withdll.exe /d:my_hook.dll psql.exe "host=127.0.0.1 ..."
```

### 3. IAT (Import Address Table) Hooking
`psql.exe`가 실행될 때 `kernel32.dll`에서 `GetACP`의 주소를 찾아가는 테이블을 조작하는 방식입니다.

* **장점:** 함수 본문을 수정하지 않아 안정적입니다.
* **단점:** 직접 구현하려면 PE 파일 구조에 대한 이해가 필요합니다.

---

### 4. 코드 예시 (C++)
`GetACP`를 65001로 강제 반환하는 간단한 DLL 구조입니다.

```cpp
#include <windows.h>
#include "detours.h"

// 원래 함수의 포인터 저장
static UINT (WINAPI * TrueGetACP)(VOID) = GetACP;

// 우리가 만든 가짜 함수
UINT WINAPI HookGetACP(VOID) {
    return 65001; // UTF-8 코드 페이지 강제 반환
}

BOOL WINAPI DllMain(HINSTANCE hinst, DWORD dwReason, LPVOID reserved) {
    if (dwReason == DLL_PROCESS_ATTACH) {
        DetourTransactionBegin();
        DetourUpdateThread(GetCurrentThread());
        DetourAttach(&(PVOID&)TrueGetACP, HookGetACP);
        DetourTransactionCommit();
    }
    return TRUE;
}
```

---

### 5. 더 쉬운 대안: MS 공식 "App Compatibility Toolkit"
코딩 없이 특정 앱의 시스템 응답 값을 속이고 싶다면 **Compatibility Administrator (Shim Infrastructure)**를 사용할 수 있습니다.

1.  **Windows ADK**를 설치합니다.
2.  **Compatibility Administrator**를 실행합니다.
3.  `psql.exe`에 대해 **"UseUnicodeProperties"** 또는 **"ForceUTF8"** 관련 Shim을 적용합니다.
4.  이렇게 하면 윈도우가 해당 프로세스에 한해 API 응답을 유니코드 친화적으로 조작해 줍니다.

### ⚠️ 주의사항
* **보안 프로그램:** 회사 PC나 보안 솔루션이 설치된 환경에서는 API 후킹 시도를 멀웨어로 오진하여 차단할 수 있습니다.
* **부작용:** `GetACP`가 65001을 반환하게 되면, `psql` 외에 해당 DLL이 로드된 다른 함수들도 영향을 받아 예상치 못한 한글 깨짐이 발생할 수 있습니다.

`MinTool4PG` 툴킷의 배포 편의성을 생각하신다면 후킹보다는 이전 답변에서 언급한 **환경 변수(`PGCLIENTENCODING`)**나 **배치 파일**을 통한 우회법이 사용자들에게 더 안전한 선택지가 될 것입니다. 하지만 개인 환경에서 완벽한 깔끔함을 원하신다면 **Detours**를 이용한 DLL 주입이 가장 확실한 "LD_PRELOAD" 방식의 구현입니다.

직접 DLL을 빌드하여 테스트해 보실 계획인가요? 도움이 필요하시면 빌드 환경 설정에 대해 더 알려드릴 수 있습니다.