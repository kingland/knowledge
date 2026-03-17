# Calling Conventions in Windows

A **calling convention** defines the rules for how functions receive parameters and return values — essentially a contract between caller and callee governing the stack, registers, and cleanup responsibilities.

---

## x86 (32-bit) Calling Conventions

Windows x86 has several conventions, and choosing the wrong one causes stack corruption.

### `__cdecl` (C Declaration)
- **Parameter passing:** pushed right-to-left onto the stack
- **Stack cleanup:** **caller** cleans up
- **Return value:** `EAX` (or `EDX:EAX` for 64-bit values)
- **Name mangling:** `_functionName`
- **Use case:** C runtime, variadic functions (`printf`), default for C/C++

```cpp
int __cdecl add(int a, int b); // caller cleans: add esp, 8 after call
```

### `__stdcall` (Standard Call)
- **Parameter passing:** right-to-left onto stack
- **Stack cleanup:** **callee** cleans up (via `ret N`)
- **Return value:** `EAX`
- **Name mangling:** `_functionName@bytes` (e.g., `_add@8`)
- **Use case:** Win32 API (`WINAPI` is `__stdcall`), COM interfaces

```cpp
int __stdcall add(int a, int b); // callee does: ret 8
```

### `__fastcall`
- **Parameter passing:** first two args in `ECX`, `EDX`; rest on stack
- **Stack cleanup:** callee
- **Name mangling:** `@functionName@bytes`
- **Use case:** performance-sensitive internal functions

### `__thiscall`
- **Parameter passing:** `this` pointer in `ECX`; args on stack right-to-left
- **Stack cleanup:** callee (caller for variadics)
- **Use case:** non-static C++ member functions (MSVC default)

---

## x64 (64-bit) — Microsoft x64 ABI

x64 Windows uses **one unified calling convention** — no `__cdecl`/`__stdcall` distinction.

### Register Usage

| Purpose | Registers |
|---|---|
| Integer/pointer args (1–4) | `RCX`, `RDX`, `R8`, `R9` |
| Float/double args (1–4) | `XMM0`–`XMM3` |
| Return value (int) | `RAX` |
| Return value (float) | `XMM0` |
| Caller-saved (volatile) | `RAX`, `RCX`, `RDX`, `R8`–`R11`, `XMM0`–`XMM5` |
| Callee-saved (non-volatile) | `RBX`, `RBP`, `RDI`, `RSI`, `R12`–`R15`, `XMM6`–`XMM15` |

### Key Rules

**1. Shadow Space (Home Space)**
The caller must allocate **32 bytes** (4 × 8) on the stack *before* the call, even if fewer than 4 args are passed. The callee may spill register args here.

```asm
sub rsp, 40        ; 32 bytes shadow + 8 to align to 16
mov rcx, arg1
mov rdx, arg2
call MyFunction
add rsp, 40
```

**2. Stack Alignment**
The stack must be **16-byte aligned** at the point of `call` (so `RSP` is 16n+8 just before `call` pushes the return address, making it 16n at function entry).

**3. Args 5+**
Pushed on the stack right-to-left, above the shadow space.

**4. Large return values**
If the return type doesn't fit in `RAX`, the caller allocates space and passes a pointer in `RCX` (shifting all other args right).

---

## Function Prologue / Epilogue (x64)

```asm
; Prologue
push rbp
mov  rbp, rsp
sub  rsp, N          ; allocate locals (N must keep RSP 16-byte aligned)
; save non-volatile registers if used
push rbx
push rdi

; ... function body ...

; Epilogue
pop  rdi
pop  rbx
mov  rsp, rbp        ; or: add rsp, N
pop  rbp
ret
```

---

## Practical Summary

| Convention | Platform | Cleanup | Arg passing |
|---|---|---|---|
| `__cdecl` | x86 | Caller | Stack R→L |
| `__stdcall` | x86 Win32 API | Callee | Stack R→L |
| `__fastcall` | x86 | Callee | ECX/EDX + stack |
| `__thiscall` | x86 C++ | Callee | ECX=this, stack |
| MS x64 ABI | x64 Windows | Caller | RCX/RDX/R8/R9 + shadow space |

---

## Why It Matters in C++

```cpp
// Explicit convention needed when interfacing with Win32 or DLLs
typedef int (WINAPI *PFN_MSGBOX)(HWND, LPCSTR, LPCSTR, UINT);

// __stdcall mismatch → stack corruption on x86
// On x64 it's all unified, but shadow space still required in raw ASM
```

Key takeaway: on **x86**, mismatched conventions silently corrupt the stack. On **x64**, the ABI is unified but you must respect shadow space and register preservation rules — especially important in your compiler/codegen work when emitting call sequences.