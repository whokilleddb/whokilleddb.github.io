---
title: "Learning to write a LLVM Pass for Code Obfuscation #1"
tags: []
categories: []
date: 2026-06-07 05:13:57
---

This series of blogs is supposed to document my learning journey from a C/gcc nerd to a C++/LLVM chad. Expect this blog to be very informal, with the occasional rant, but it will document _everything_ that I have learnt, including C++ internals, gimmicks, LLVM quirks and other references.

# Setup 

First thing first, we need to do some basic setup, starting with fetching the source code for the LLVM project:

```bash
$ git clone https://github.com/llvm/llvm-project.git  
```

At the time of writing this, the latest commit is ` 8aafa50c7a2dfb8ca1d5cdf8980f7f2d259779f5` - incase you wanna follow along the exact version and stuff, just do:

```bash
$ git checkout 8aafa50c7a2dfb8ca1d5cdf8980f7f2d259779f5
```


Next, we install some basics with:

```
$ sudo apt -y ninja-build install build-essential subversion cmake python3-dev libncurses5-dev libxml2-dev libedit-dev swig doxygen graphviz xz-utils clang gdb git vim tmux
```

I would recommend running everything in a `tmux` session because some of these compilations take a while. That being said, let's talk about LLVM (while my code compiles in the background).

## Why LLVM?

All of this started with the following set of tweets:

<blockquote class="twitter-tweet" data-theme="dark"><p lang="en" dir="ltr">use clang like a real man 😼</p>&mdash; 5pider (@C5pider) <a href="https://x.com/C5pider/status/1993692843026899167?ref_src=twsrc%5Etfw">November 26, 2025</a></blockquote> <script async src="https://platform.x.com/widgets.js" charset="utf-8"></script>

So, all the big boys were playing with `clang` and I wanted to do so as well. I had played a bit with it in the past when compiling stuff in C for Mac but never really looked into it. To me it was just another _compiler_. 

That was lesson 1: `clang` is technically the _fronend_ component of the LLVM compiler for C/C++ source code. It's job is to take the source code, parse it into an AST and then lower that AST into LLVM IR. 

This LLVM IR is what is of interest to us. Our goal is to write a LLVM IR pass which messes with this IR in a way that makes Reverse Engineering and static detections difficult, for which, unfortunately, we have to learn some C++. I am thinking about exploring the concepts as they come about.

# Following the official guide

At this point, you should start following the _[Official Guide on how to write a LLVM pass](https://llvm.org/docs/WritingAnLLVMNewPMPass.html)_. Follow the guide till you reach the [FAQ section](https://llvm.org/docs/WritingAnLLVMNewPMPass.html#faqs) and then come back here.

-----
Welcome back! I am assuming at this point you have written `HelloWorld` pass, seen it in action and also written a test for it. 

Okay, time to talk about the code we just wrote, starting with the header file:

```cpp
#ifndef LLVM_TRANSFORMS_HELLONEW_HELLOWORLD_H
#define LLVM_TRANSFORMS_HELLONEW_HELLOWORLD_H

#include "llvm/IR/PassManager.h"

namespace llvm {

class HelloWorldPass : public OptionalPassInfoMixin<HelloWorldPass> {
public:
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &AM);
};

} // namespace llvm

#endif // LLVM_TRANSFORMS_HELLONEW_HELLOWORLD_H
```

First, we take a look at:

```cpp
class HelloWorldPass : public OptionalPassInfoMixin<HelloWorldPass>
```

Let's break it down:
- `class HelloWorldPass`: We are declaring a new class with the name `HelloWorldPass`
- `public OptionalPassInfoMixin<HelloWorldPass>`: Here we see some C++ bullshit. Time for a detour and learn about a couple of C++ things.

## Templates

Coming from C, C++ templates was something completely new to me. These are probably the simplest things to understand. 

I would recommend going through [GFG's guide](https://www.geeksforgeeks.org/cpp/templates-cpp/) for this. But to simplify it, imagine this: You want to write a C program which can compare: two numbers, two decimals or two characters. You would end up writing some code like:

```c
#include <stdio.h>

int maxInt(int a, int b) {return (a > b)? a:b;}
float maxFloat(float a, float b) {return (a > b)? a:b;}
char maxChar(char a, char b) {return (a > b)? a:b;}

int main(){
	printf("Max int:   between 6, 7 is: %d\n", maxInt(6,7));
	printf("Max float: between 6.0, 7.0 is: %f\n", maxFloat(6.0,7.0));
	printf("Max int: between '6', '7' is: %c\n", maxChar('6','7'));
	return 0;
}
```

Well, we can all agree that's a lot of repeated code. While the main function body remains identicat, we have to maintain different functions just due to the nature of the data types we are dealing with. C++ attempts to solve this problem with templates.


## Mixins

## CRTP

---

Resuming


## References

1. [Official Guide on how to write a LLVM pass](https://llvm.org/docs/WritingAnLLVMNewPMPass.html)
2. [Extending LLVM for Code Obfuscation (1 of 2)](https://www.praetorian.com/blog/extending-llvm-for-code-obfuscation-part-1/)
3. [Templates in C++](https://www.geeksforgeeks.org/cpp/templates-cpp/)