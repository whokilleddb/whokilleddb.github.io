---
title: Dynamic Function Table RF_CALLBACK Injection 
tags: ["windows", "shellcode"]
categories: []
date: 2026-09-06 23:43:15
---

# Dynamic Function Tables & the RF_CALLBACK (Type 2) Mechanism

On x64 Windows, [stack unwinding for exceptions and SEH is table-driven, not frame-pointer-based](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170). Every function that touches the stack must have a [RUNTIME_FUNCTION](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170#struct-runtime_function) entry (.pdata) describing its prologue/epilogue so the unwinder can walk the stack.

For statically compiled modules, [`RtlLookupFunctionEntry()`](https://learn.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-rtllookupfunctionentry) finds these tables via the PE's `IMAGE_DIRECTORY_ENTRY_EXCEPTION`. But for JIT-compiled code (CLR, Java, JavaScript engines, dynamically generated stubs), Windows exposes an API to register unwind info at runtime:

- `RtlAddFunctionTable()` — Type 1 (static table in memory)
- `RtlInstallFunctionTableCallback()` — Type 2 (RF_CALLBACK) ← the interesting one
- `RtlAddGrowableFunctionTable()` — Type 3

From [Microslop's own documentation](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170):

> For dynamically generated functions [JIT compilers], the runtime to support these functions must either use RtlInstallFunctionTableCallback or RtlAddFunctionTable to provide this information to the operating system. Failure to do so results in unreliable exception handling and debugging of processes.

Internally these all populate the process-global list `ntdll!RtlpDynamicFunctionTable` (a doubly linked list guarded by `RtlpDynamicFunctionTableLock`, an SRW lock).

<img src="/images/dyn_func_inject/RtlpDynamicFunctionTable.png" style="display:block; margin:0;" />

## The `DYNAMIC_FUNCTION_TABLE` Structure

Each node roughly looks like (undocumented, reconstructed):

```c
typedef struct _DYNAMIC_FUNCTION_TABLE {
    LIST_ENTRY                 Links;
    PRUNTIME_FUNCTION          FunctionTable;   // or NULL for Type 2
    LARGE_INTEGER              TimeStamp;
    ULONG64                    MinimumAddress;
    ULONG64                    MaximumAddress;
    ULONG64                    BaseAddress;
    PGET_RUNTIME_FUNCTION_CALLBACK Callback;    // <-- RF_CALLBACK
    PVOID                      Context;
    PWSTR                      OutOfProcessCallbackDll;
    FUNCTION_TABLE_TYPE        Type;            // RF_SORTED=0, RF_UNSORTED=1, RF_CALLBACK=2
    ULONG                      EntryCount;
    RTL_BALANCED_NODE          TreeNode;
} DYNAMIC_FUNCTION_TABLE;
```

For Type 2 (`RF_CALLBACK`), FunctionTable is `NULL` and instead the unwinder must call `Callback(ControlPc, Context)` every time it needs a `RUNTIME_FUNCTION` for an address in `[MinimumAddress, MaximumAddress)`.
