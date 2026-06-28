---
title: "Learning to write a LLVM Pass for Code Obfuscation #2"
tags: []
categories: []
date: 2026-06-26 03:17:58
---

This blog is a continuation of the [previous blog](https://whokilleddb.github.io/2026/06/07/Learning-to-write-a-LLVM-Pass-for-Code-Obfuscation-1/) where we covered some basics of C++, created a demo pass and prints function names following the LLVM guide. In this second part, we are gonna [Leeroy Genkins](https://www.youtube.com/watch?v=hooKVstzbz0) it and dive straight into writing LLVM passes. 

![](../images/llvm/leeroy-jenkins.png)

<!--more-->

# LLVM, IR and Clang

What even is LLVM IR? I am hoping that people reading this have _some_ idea about what an IR is, but we need to see it in action. So, let's start with a simple C code:

```c
// classify.c

bool classify(int x) {
    if (x > 0) return true;
    else return false;
}
```

Then, we can produce the IR with clang as follows:

```
clang -O0 -emit-llvm -S classify.c -o classify.ll
```

where,

`clang` is the llvm compiler frontend (for, just the _compiler_)
`-O0` turns off optimizations
`-emit-llvm` tells clang to produce LLVM IR rather than native machine code
`-S` tells clang to emit IR in text form 

This should produce a `classify.ll` file as follows:

```c
; ModuleID = 'classify.c'
source_filename = "classify.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"

; Function Attrs: noinline nounwind optnone uwtable
define dso_local zeroext i1 @classify(i32 noundef %0) #0 {
  %2 = alloca i1, align 1
  %3 = alloca i32, align 4
  store i32 %0, ptr %3, align 4
  %4 = load i32, ptr %3, align 4
  %5 = icmp sgt i32 %4, 0
  br i1 %5, label %6, label %7

6:                                                ; preds = %1
  store i1 true, ptr %2, align 1
  br label %8

7:                                                ; preds = %1
  store i1 false, ptr %2, align 1
  br label %8

8:                                                ; preds = %7, %6
  %9 = load i1, ptr %2, align 1
  ret i1 %9
}

attributes #0 = { noinline nounwind optnone uwtable "frame-pointer"="all" "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }

!llvm.module.flags = !{!0, !1, !2, !3, !4}
!llvm.ident = !{!5}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 8, !"PIC Level", i32 2}
!2 = !{i32 7, !"PIE Level", i32 2}
!3 = !{i32 7, !"uwtable", i32 2}
!4 = !{i32 7, !"frame-pointer", i32 2}
!5 = !{!"Debian clang version 19.1.7 (3+b1)"}
```

What we would be focusing on for right now is this part of the code:

```
; Function Attrs: noinline nounwind optnone uwtable
define dso_local zeroext i1 @classify(i32 noundef %0) #0 {
  %2 = alloca i1, align 1
  %3 = alloca i32, align 4
  store i32 %0, ptr %3, align 4
  %4 = load i32, ptr %3, align 4
  %5 = icmp sgt i32 %4, 0
  br i1 %5, label %6, label %7

6:                                                ; preds = %1
  store i1 true, ptr %2, align 1
  br label %8

7:                                                ; preds = %1
  store i1 false, ptr %2, align 1
  br label %8

8:                                                ; preds = %7, %6
  %9 = load i1, ptr %2, align 1
  ret i1 %9
}
```

Lets take this line-by-line: 

`; Function Attrs: noinline nounwind optnone uwtable`

This specifies the function's attributes(duh!):

`noinline`: The function must never be inlined into its callers. The optimizer is forbidden from substituting the body at call sites.
`nounwind`:  The function does not throw / unwind exceptions. This is standard for C code and lets the compiler skip generating exception-handling (unwind) tables and bookkeeping.
`optnone`: Do not optimize this function. It tells optimization passes to leave the function alone.
`uwtable`: Generate an unwind table for it anyway — stack-frame metadata so debuggers/profilers and stack unwinders can walk the call stack (needed for backtraces, even though the function itself won't throw).

`nounwind` and `uwtable` reflect the language/platform: C functions don't unwind (`nounwind`), but the Linux x86-64 ABI still wants unwind tables for reliable backtraces (`uwtable`).

`define dso_local zeroext i1 @classify(i32 noundef %0) #0`

This is the function definition in IR form:

`define`: this tells the compiler that this is the actual function definition (with a body) instead of a function declaration. 
`dso_local`: this means that the function will live in the same DSO(dynamic shared object) as any code referencing it - which means the compiler can use direct calls without needing to go through PLT/GOT 
`zeroext i1`: Since we are returning a bool - which is a 1 bit value (signified by `i1`) - we need to _zero extend_ the wider register where it is places (ret values are usually returned by placing them in `rax`)
`@classify`: This is the global symbol name for our function. In LLVM, global symbols are denoted by the `@` prefix and local symbols are prefixed by `%`
`i32 noundef %0`: Remember how the `classify()` function took an integer as an input? This block specifies exactly that. The `i32` means the input is an integer which can never be undefined(`noundef`) - and %0 is just the variable name
`#0` : This refers to the attribute group `attributes #0` at the bottom - we would leave this for now - and might come back to this later if we need to. 

```
%2 = alloca i1, align 1
%3 = alloca i32, align 4
store i32 %0, ptr %3, align 4
%4 = load i32, ptr %3, align 4
%5 = icmp sgt i32 %4, 0
br i1 %5, label %6, label %7
```

This code block is called the _entry block_ - all calls to the function hHAVE TO pass through this block regardless of the input value. Now one thing to note: LLVM usually assigns labels to code blocks(what is a code block? We will get to a more formal definition later). However when generating the textual IR, it skips naming the entry block but internally, it still consumes a label. LLVM IR, every code block and every value needs a name. We can specify an explicit name, but in our case, we did not, so LLVM assigns sequential integers: %0, %1, %2, and so on. Since we used `%0` to denote the input variable, this block uses the (hidden) label `1`, so the right definition would be:

```
1: 
    %2 = alloca i1, align 1
    %3 = alloca i32, align 4
    store i32 %0, ptr %3, align 4
    %4 = load i32, ptr %3, align 4
    %5 = icmp sgt i32 %4, 0
    br i1 %5, label %6, label %7
```

Now, getting to the actual code block:

```
%2 = alloca i1, align 1
%3 = alloca i32, align 4
store i32 %0, ptr %3, align 4
```

The first line reserves a 1 bit address on the stack to hold the return value(`%2`). The second value(`%3`) creates a local variable to store the value of the input value locally. The third line copies the value of the input parameter into the slot.

```
%4 = load i32, ptr %3, align 4
%5 = icmp sgt i32 %4, 0
br i1 %5, label %6, label %7
```

The value stored in `%3` is immediately popped back into `%4` (yes, this is redundant code - remember we are not optimizing things as all?). `%5` stores the value of the operation `x>0`. The `icmp` part stands for integers (signed) and the `sgt` means _signed greater than_ - so this makes sure we are doing a signed comparision against 0 and stores the result. The last line reads as follows: If the 1 bit(`i1`) stored at `%5` is true, jump to `%6`, else jump to `%7`. The `br` keyword represents that the code is branching from the given statement.

The key thing to understand: `-O0` puts everything on the stack. It doesn't try to keep values in registers, so each variable gets an `alloca` (stack allocation) and is shuffled in/out via store/load. Most optimizations should get rid of the redundant operations. 

Next, we have the branches:

```
6:                                                ; preds = %1
  store i1 true, ptr %2, align 1
  br label %8

7:                                                ; preds = %1
  store i1 false, ptr %2, align 1
  br label %8

```

First, we have the branch labels `%6` and `%7` (DONT!). Notice the `preds = %1` comment? It shows which blocks can reach the respective block. Since both blocks are just preceeded by the entry block aka `%1` it says `preds=1` (`preds`=predecessor).  Each branch corresponds to each arm of the if/else condition. It stores a 1 bit (`i1`) value in `%2`, and then makes an unconditional jump to the code block represented by `%8`.  

The final block is: 

```
%9 = load i1, ptr %2, align 1
ret i1 %9
```

We load the value stored in `%2` into another variable `%9` and return it. Pretty simple, right? 

But this code was very inefficient. Let's see what happens when we turn up the optimization. After compiling with `-O3` for examples, the resulting IR becomes:

```
define dso_local noundef zeroext i1 @classify(i32 noundef %0) local_unnamed_addr #0 {
  %2 = icmp sgt i32 %0, 0
  ret i1 %2
}
```

Notice how much more efficient the code becomes? LLVM allows us to write our own optimizations, but that's out of the scope (for now atleast?). 

**SSA - Single Static Assignment**

One rule of LLVM IR is SSA which means one very simple thing: Every variable is assigned _ONLY ONCE_. What does this mean? Let's see with some code:

```c
int x = 1;
x = x+1;
x = x*5;
```

The IR for this looks as follows:

```
%1 = alloca i32, align 4
store i32 1, ptr %1, align 4
%2 = load i32, ptr %1, align 4
%3 = add nsw i32 %2, 1
store i32 %3, ptr %1, align 4
%4 = load i32, ptr %1, align 4
%5 = mul nsw i32 %4, 5
store i32 %5, ptr %1, align 4
%6 = load i32, ptr %1, align 4
ret i32 %6
```

See, how no variable is reused? A new variable is used with every store/load instruction. This is a small rule which we need to keep in mind for future references.

---------------

Now that we understand a bit of IR, time to move onto the next topic: Code blocks 

# Basic Blocks in LLVM

A **basic block** is a maximal sequence of instructions with:
- **exactly one entry point** (the first instruction — you can only jump *to* the top of the block), and
- **exactly one exit point** (the last instruction — control leaves *only* from the bottom).

This idea creates a bunch of paradigms:

- A block has NO BRANCHES in the middle - all code must enter from the _entrypoint_ aka the first instruction and nothing except the last instruction can transfer control elsewhere
- Each block has exactly ONE TERMINATOR and it is at the very end. A terminator instruction is one which controls where the control flow goes next. Stuff like if/else, switch case, goto, returns

One thing to note is that a basic block _CAN CONTAIN_ a function call - A normal `call` is an ordinary instruction. Control returns right after it, so it does not end the block. A `call` is just another instruction — execution flows into it and out the other side. Only *terminators* end blocks. (The one exception is `invoke`, used for exceptions, which *is* a terminator because it has two outgoing edges: normal return and exception.)

# Phi Nodes

When two different paths through the Control Flow Graph(CFG) produce a value that is needed in a block that both paths reach, LLVM IR uses a `phi` instruction to select the correct value. Let's take an example:

```c
int x;
if (condition) {
  x = 10;
} else {
  x = 20;
}
int y = x + 1;
```

In SSA form, we cannot assign to `x` twice. So LLVM generates:

```
entry:
  br i1 %condition, label %true_branch, label %false_branch

true_branch:
  br label %merge           ; x "is" 10 on this path

false_branch:
  br label %merge           ; x "is" 20 on this path

merge:
  %x = phi i32 [ 10, %true_branch ], [ 20, %false_branch ]
  %y = add i32 %x, 1
```

The `phi` instruction essentially says: _"My value is 10 if I was reached from `%true_branch`, or 20 if I was reached from `%false_branch`."_

As a rule, if a basic block has a `phi` instruction - it must be the first instruction of that block.

# Dominators and Dominator Trees

(_Sigh_)

