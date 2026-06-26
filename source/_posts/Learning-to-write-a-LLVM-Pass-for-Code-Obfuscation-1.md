---
title: "Learning to write a LLVM Pass for Code Obfuscation #1"
tags: []
categories: []
date: 2026-06-07 05:13:57
---

This series of blogs is supposed to document my learning journey from a C/gcc nerd to a C++/LLVM chad. Expect this blog to be very informal, with the occasional rant, but it will document _everything_ that I have learnt, including C++ internals, gimmicks, LLVM quirks and other references.

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSyC5W-orQb4vFVY1wjh5ffQ5_jxETifNbpGENj81FQDuxI7Lbd8dUObEa4-WyUFB1_g7f2CgdSQMAOker1unrwc0qc8ihv_QB3FjlB0go&s=10)

<!-- more -->

# Setup 

First thing first, we need to do some basic setup, starting with fetching the source code for the LLVM project:

```bash
git clone https://github.com/llvm/llvm-project.git  
```

At the time of writing this, the latest commit is ` 8aafa50c7a2dfb8ca1d5cdf8980f7f2d259779f5` - incase you wanna follow along the exact version and stuff, just do:

```bash
git checkout 8aafa50c7a2dfb8ca1d5cdf8980f7f2d259779f5
```


Next, we install some basics with:

```
sudo apt -y ninja-build install build-essential subversion cmake python3-dev libncurses5-dev libxml2-dev libedit-dev swig doxygen graphviz xz-utils clang gdb git vim tmux
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

Well, we can all agree that's a lot of repeated code. While the main function body remains identicat, we have to maintain different functions just due to the nature of the data types we are dealing with. C++ attempts to solve this problem with templates. The same code in C++ would be something like:

```cpp
#include <iostream>

template <typename T> T maxVal(T x, T y) { return (x > y)? x:y;}

int main(){
	std::cout << "Max int:   between 6, 7 is: " << maxVal(6,7) << std::endl;
	std::cout << "Max float: between 6.0, 7.0 is: " << maxVal(6.0,7.0) << std::endl;
	std::cout << "Max int: between '6', '7' is: " << maxVal('6','7') << std::endl;
	return 0;
}
```

So, templates just help us write generic functions which can be used by any valid type `T`. A _slightlty inaccurate but easy-to-understand_ way to understand templates is to consider them as functions which take the data type(can be a class, composite data and more as well) as well, along with the values of the type. 


### Template Classes

Templates go beyond just functions - we can have template classes as well. Consider the following terrible C code:

```c
#include <stdio.h>
#include <assert.h>

typedef struct _int_func {
  int val;
  int (*fn)(struct _int_func *);
} int_func, *p_int_func;

int inc_func(p_int_func p_int){
  p_int->val = p_int->val + 1;
  return p_int->val;
} 

typedef struct _float_func {
  float val;
  float (*fn)(struct _float_func *);
} float_func, *p_float_func;

float float_inc(p_float_func p_float) {
  p_float->val = p_float->val+1;
  return p_float->val;
}

int main() {
  int_func ifunc = {0};
  ifunc.val = 6;
  ifunc.fn = inc_func;
  int iret = ifunc.fn(&ifunc);

  printf("[+] ifunc: 0x%p | ifunc->val: %d | ifunc->fn: 0x%p | iret: %d\n", &ifunc, ifunc.val, (void *)(ifunc.fn), iret);

  float_func ffunc = {0};
  ffunc.val = 6.0006;
  ffunc.fn = float_inc;
  float fret = ffunc.fn(&ffunc);

  printf("[+] ffunc: 0x%p | ffunc->val: %f | ffunc->fn: 0x%p | fret: %f\n", &ffunc, ffunc.val, (void *)(ffunc.fn), fret);

  return 0;
}
```

A C++/OOPs based approach would be: 

```cpp
#include <iostream>

class IntFunc {
  public:
    IntFunc(int x)
      : val{x} {}

    int inc_func() {
      val = val + 1;
      return val;
    }
    
    int get_num() { return val;}
  private:
    int val;
};

class FloatFunc {
  public:
    FloatFunc(float x)
      : val{x} {}

    float inc_func() {
      val = val + 1;
      return val;
    }
    
    float get_num() { return val;}
  private:
    float val;
};


int main() {
  IntFunc ifunc(6);
  FloatFunc ffunc(6.0006);

  std::cout << "ifunc.val: " << ifunc.get_num() << " | inc_func(): " << ifunc.inc_func() << " | new ifunc.val: " << ifunc.get_num() << std::endl;
  std::cout << "ffunc.val: " << ffunc.get_num() << " | inc_func(): " << ffunc.inc_func() << " | new ffunc.val: " << ffunc.get_num() << std::endl;
 
  return 0;
}
```

But C++ allows us to optimize this further using templates. If you notice, `IntFunc` and `FloatFunc` share a lot of the same code - this is where templates come in.

```cpp
#include <iostream>

template <class T>
class NumFunc {
  public: 
    NumFunc(T x):
      val(x){}

    T inc_func() {
      val = val+1;
      return val;
    }

    T get_num() { return val;}

  private:
    T val;
};

int main() {
  NumFunc<int> ifunc(6);
  NumFunc<float> ffunc(6.006);

  std::cout << "ifunc.val: " << ifunc.get_num() << " | inc_func(): " << ifunc.inc_func() << " | new ifunc.val: " << ifunc.get_num() << std::endl;
  std::cout << "ffunc.val: " << ffunc.get_num() << " | inc_func(): " << ffunc.inc_func() << " | new ffunc.val: " << ffunc.get_num() << std::endl;
  return 0;
}
```

So - you can reuse the same class with different data types - and this will come in handy soon.

## Mixins

First thing to know: _Mixins are classes_. It's just that they are a special category of classes which have some specific properties. So we can say that:

> All Mixins are classes, but not all classes are Mixins

A mixin is a class that provides a specific set of functionalities and is intended to be used in conjunction with other classes through multiple inheritance. Mixins are typically abstract or incomplete in themselves, meaning they do not represent a complete object but rather provide a modular way to add features to a class hierarchy. Think of Mixins as the _ketchup_ of classes. You dont have ketchup on it's own (hopefully), you add it on top of a HotDog to make it better. Extending this analogy, you can have the same ketchup go on a hotdog, fries or your chicken wings. While the base class of food items remain different - our Mixin aka ketchup remains the same to add to their taste(functionality) - hope that made sense. 

In more technical terms, key characteristics of C++ Mixins are:

1. **Non-Instantiable**: Mixins often contain pure virtual functions (abstract methods) and may not have any concrete data members. This makes them unsuitable for standalone instantiation.
  
2. **Functionality Addition**: The primary purpose of mixins is to add specific behaviors or functionalities to a class without affecting its primary inheritance hierarchy.

3. **Multiple Inheritance Utilization**: Mixins are often used in conjunction with multiple inheritance, allowing a class to inherit from both a main base class and one or more mixins. This composition allows the class to have features from all its base classes.

4. **Code Reusability**: By using mixins, developers can reuse functionality across different class hierarchies without creating complex inheritance trees.

5. **Avoiding Object Identity**: Mixins do not represent a complete object model on their own. They are meant to be part of a larger class hierarchy and should not be instantiated directly.

[This is an excellent example of C++ Mixins.](https://gist.github.com/MangaD/93a79aae04324c85f2b8d691b66441ff) I would highly recommend going through atleast the first two examples. We will see more of this in the next example.

## CRTP

C++ has this curious thing called Curiously Recurring Template Pattern (CRTP). It is a C++ idiom where a derived class inherits from a base class template, passing itself as the template argument. This technique enables static polymorphism, allowing the base class to call methods in the derived class at compile time without the performance overhead of virtual functions (VTables). 

But what does that jargon mean - coming from C? Take a look at the example code:

```c
#include <stdio.h>

typedef struct Shape Shape;

struct Shape {
	double (*area)(Shape *self);
};  

typedef struct {
	Shape base;
	double radius;
} Circle;

typedef struct {
	Shape base;
	double side;
} Square;

double circle_area(Shape *self) {
	Circle *c = (Circle *)self;
	return 3.14159 * c->radius * c->radius;
}

double square_area(Shape *self) {
	Square *s = (Square *)self; 
	return s->side * s->side;
}   

double get_area(Shape *s) {
	return s->area(s);
}

int main(void) {
	Circle c = { .base = { .area = circle_area }, .radius = 5.0 };
	Square s = { .base = { .area = square_area }, .side  = 4.0 };

	Shape *shapes[] = { (Shape *)&c, (Shape *)&s };

	for (int i = 0; i < 2; i++)
		printf("area = %.2f\n", get_area(shapes[i]));
		
	return 0;
}
```

We have a generic "Class" called `Shape` which has a `vtable` pointer and we have a function called `get_area()` which takes variables of this _class_. Now the shape can be a `Square` or a `Circle` - the function has no knowledge of this. It is upto those two _classes_ to implement their respective area functions. 

If we have to write the same in C++ using CRTP we would write something like:

```cpp
#include <cstdio>

template <typename Derived>
struct Shape {
	double area() {
		return static_cast<Derived *>(this)->area_impl();
	}
};

struct Circle : Shape<Circle> {
	double radius;
	Circle(double r) : radius(r) {}
	double area_impl() { return 3.14159 * radius * radius; }
};

struct Square : Shape<Square> {
	double side;
	Square(double s) : side(s) {}
	double area_impl() { return side * side; }
};  

template <typename T>
double get_area(Shape<T> &s) {
	return s.area();
}

int main() {
	Circle c(5.0);
	Square s(4.0);
	
	printf("circle area = %.2f\n", get_area(c));
	printf("square area = %.2f\n", get_area(s));

	// NOTE: can't do Shape<?>* shapes[] = {&c, &s} — that's the CRTP tradeoff.
	// Shape<Circle> and Shape<Square> are unrelated types to the compiler.
	return 0;
}
```

So now that we see the C++ code - it makes more sense. Why do we need this? Becuase standard dynamic polymorphism uses virtual functions, which require runtime pointer lookups via a VTable. CRTP resolves these calls at compile time. This makes it heavily utilized in high-frequency trading (HFT) and embedded systems where every CPU cycle matters.

---

So now back to LLVM. We left at the following definition:

```c++
class HelloWorldPass : public OptionalPassInfoMixin<HelloWorldPass>
```

From the official guide:

> This creates the class for the pass with a declaration of the run() method which actually runs the pass. Inheriting from OptionalPassInfoMixin<PassT> or RequiredPassInfoMixin<PassT> sets up some more boilerplate so that we don’t have to write it ourselves. RequiredPassInfoMixin should be used for passes that cannot be skipped (e.g. AlwaysInlinerPass), while OptionalPassInfoMixin should be used for passes that can be skipped (e.g. optimization passes).

With the context we have of `Mixins` and CRTP at this point we should be good to go. We might dive into `OptionalPassInfoMixin` and `RequiredPassInfoMixin` later if the need be.

Looking at the actual implementation of the `run()` function (because remember - we had to implement the area function for each _class_ ourselves?), we see:

```c++
PreservedAnalyses HelloWorldPass::run(Function &F,
                                      FunctionAnalysisManager &AM) {
  errs() << F.getName() << "\n";
  return PreservedAnalyses::all();
}
```

Which is pretty simple - we just print the function names. Now go follow the rest of the [Official Guide on how to write a LLVM pass](https://llvm.org/docs/WritingAnLLVMNewPMPass.html) till the FAQ section and come back.

------------------

Hi! You are back! So at this point you should have a basic idea of how the thing works and written your first pass. But this doesn't do anything useful. Also, in the example, we see they are using IR code - not something super useful. Ideally, we would want a clang flag or some other way to pass our C/C++ code directly. So I am planning to in the long run (as in by the end of this series) to implement something of that sort. Let's see how it goes! Wish me luck and see you soon!

## References

1. [Official Guide on how to write a LLVM pass](https://llvm.org/docs/WritingAnLLVMNewPMPass.html)
2. [Extending LLVM for Code Obfuscation (1 of 2)](https://www.praetorian.com/blog/extending-llvm-for-code-obfuscation-part-1/)
3. [Templates in C++](https://www.geeksforgeeks.org/cpp/templates-cpp/)
4. [Combining Static and Dynamic Polymorphism with C++ Mixin classes](https://michael-afanasiev.github.io/2016/08/03/Combining-Static-and-Dynamic-Polymorphism-with-C++-Template-Mixins.html)
5. [What are Mixins (as a concept)](https://stackoverflow.com/questions/18773367/what-are-mixins-as-a-concept)