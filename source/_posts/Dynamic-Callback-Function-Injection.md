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
- `RtlInstallFunctionTableCallback()` — Type 2 (RF_CALLBACK) aka the interesting one
- `RtlAddGrowableFunctionTable()` — Type 3

From [Microslop's own documentation](https://learn.microsoft.com/en-us/cpp/build/exception-handling-x64?view=msvc-170):

> For dynamically generated functions [JIT compilers], the runtime to support these functions must either use RtlInstallFunctionTableCallback or RtlAddFunctionTable to provide this information to the operating system. Failure to do so results in unreliable exception handling and debugging of processes.

Internally these all populate the process-global list `ntdll!RtlpDynamicFunctionTable` (a doubly linked list guarded by `RtlpDynamicFunctionTableLock`, an [SRW lock](https://learn.microsoft.com/en-us/windows/win32/sync/slim-reader-writer--srw--locks)).

<img src="/images/dyn_func_inject/RtlpDynamicFunctionTable.png" style="display:block; margin:0;" />

## The `DYNAMIC_FUNCTION_TABLE` Structure

[Microsoft defines](https://learn.microsoft.com/pl-pl/windows/win32/devnotes/dynamic_function_table_type) `DYNAMIC_FUNCTION_TABLE` as:

> The `DYNAMIC_FUNCTION_TABLE` structure is an entry in a linked list which is used by a debugger to discover all of the dynamic function tables in a target process. 

Each node roughly looks like (thank you [ReactOS](https://fossies.org/dox/reactos-0.4.16/struct__DYNAMIC__FUNCTION__TABLE.html)):

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
} DYNAMIC_FUNCTION_TABLE;
```

For Type 2 (`RF_CALLBACK`), FunctionTable is `NULL` and instead the unwinder must call `Callback(ControlPc, Context)` every time it needs a `RUNTIME_FUNCTION` for an address in `[MinimumAddress, MaximumAddress)`.

Looking into the definition of [`PGET_RUNTIME_FUNCTION_CALLBACK`](https://fossies.org/dox/reactos-0.4.16/rtlsupportapi_8h_source.html#l00056):

```c
typedef PRUNTIME_FUNCTION (CALLBACK *PGET_RUNTIME_FUNCTION_CALLBACK)(DWORD_PTR,PVOID);
```

If this callback function returns `NULL` it means _"no unwind info here"_ - the callback still ran, which is the whole point.


## The Registration API

```c
BOOLEAN RtlInstallFunctionTableCallback(
    DWORD64  TableIdentifier,      // must have low bit set: (base|0x3)
    DWORD64  BaseAddress,
    DWORD    Length,
    PGET_RUNTIME_FUNCTION_CALLBACK Callback,
    PVOID    Context,
    PCWSTR   OutOfProcessCallbackDll  // optional
);
```

The signature of the callback is:

PRUNTIME_FUNCTION Callback(DWORD64 ControlPc, PVOID Context);

That Callback pointer is stored as a raw function pointer in the caller's own heap-tracked list inside ntdll. It is not CFG-guarded on the way in, and it lives at a data address (not in a code section).

The Invocation Path — Where the Callback Fires

The interesting question for injection research is: "what makes ntdll actually call that function pointer?"

The chain is:

RtlLookupFunctionEntry(ControlPc, &ImageBase, &HistoryTable)
  └─> RtlpLookupDynamicFunctionEntry
        └─> walks RtlpDynamicFunctionTable list
              └─> if node.Type == RF_CALLBACK
                    └─> INVOKES node.Callback(ControlPc, node.Context)

RtlLookupFunctionEntry is called any time the OS needs to unwind:

1. Any exception raised by any thread whose PC falls in the registered range — RtlDispatchException → RtlVirtualUnwind → RtlLookupFunctionEntry.
2. RtlCaptureStackBackTrace / StackWalk64 when the walker crosses the range.
3. RtlUnwindEx during C++ EH or SEH unwinds.
4. RtlRaiseException — including from DbgPrint, SetThreadName (0x406D1388), OutputDebugString on some builds.
5. Certain profiling / ETW stack collection paths.

So the primitive is: "if I can make the process throw/unwind through an address I own, my callback runs — with the ControlPc and my Context as arguments."
