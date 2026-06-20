---
title: Why does `add rax, rbx` encode to `48 01 D8` on x86_64?
tags: []
categories: []
date: 2026-06-21 02:25:54
---

While learning more about x86_64, I went down a rabbit hole recently, and it all started with this:

<img src="../images/add-rax-rbx/defucse.png" style="display:block; margin:0;" />

<!-- more -->

For the first time, I stopped and asked myself - _"Where are these values are coming from?"_. This blog is an attempt to answer these questions for myself. A thing to note that is for the sake of simplicity and my own mental sanity, I will focus on _JUST THIS ONE ASSEMBLY_; x86_64 assembly has a lot of moving parts which change with instructions, value types and more - covering it all on a blog would be very difficult. So, we stick to just this one instruction and break down all the relevant concepts. 

## The Bible of x86_64

For this guide, we would refer to [Intel® 64 and IA-32 Architectures Software Developer’s Manual (Combined Volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4)](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) which I will refer to as _THE BOOK_ throughout this guide.

## The Three Bytes

{% codeblock line_number:false %}
48          01          D8
REX prefix  Opcode      ModR/M byte
{% endcodeblock %}

There are three bytes we need to consider. Starting with the first one:

## The REX Prefix: `0x48`

First things first: _WTH is a REX prefix?_ 

Searching for the phrase "REX Prefix" in the book gives us 248 results. Here are some snippets which helped me understand what it does:


> [3-2 Vol. 1] REX prefixes allow a 64-bit operand to be specified when operating in 64-bit mode. By using this mechanism, many existing instructions have been promoted to allow the use of 64-bit registers and 64-bit addresses.

> [Vol. 1 3-19] REX prefixes consist of 4-bit fields that form 16 different values. The W-bit field in the REX prefixes is referred to as REX.W. If the REX.W field is properly set, the prefix specifies an operand size override to 64 bits. 

Before going further, lets see how `0x48` looks as a Base2 number:

```
(0x48)₁₆ =  (01001000)₂
```

We will divide this in two parts: `0100` and `1000`

The first `0100` is fixed and identifies this byte as a REX prefix. Why `0100`? Mostly due to [historical recycling choices](https://stackoverflow.com/a/36510865). 

This leaves out the `1000` part. From `Vol. 1 3-19` we can see that they are a part of the 4 bit field in the following order:

```
Bits:       | 3 | 2 | 1 | 0 |
            +---+---+---+---+ 
REX Bit:    | W | R | X | B |
            +---+---+---+---+ 
Value:      | 1 | 0 | 0 | 0 |
```

So what are `REX.W`, `REX.R`, `REX.X` and `REX.B`?

- `REX.W` (Width): Changes the operand size from the legacy 32-bits to 64-bits. This is the only bit that alters execution size rather than just targeting a register. Since we are looking at 64 bit code - this is set to `1` (See: Vol. 1 3-19)

- `REX.R` (Register): Extends the `modR/M` reg field. It allows the instruction to access the higher 8 General Purpose Registers (GPRs) `R8–R15`, as well as extended XMM/YMM registers - we dont need any of this complex stuff for this particular case as we just working with `rax` and `rbx` - so it is set to `0`.

- `REX.X` (Index): Extends the `SIB` index field. It is used specifically in complex memory addressing modes to utilize the extended GPRs as the scale/index register. Again, we dont use any of these complex fields, so it is set to `0`.

- `REX.B` (Base): Extends the `modR/M r/m` field or the `SIB` base field. Similar to `REX.R`, this allows the source/destination operands to be mapped to the `R8–R15` registers, and therefore, for our case, it is `0`.

So, this is why we get the `1000` value. So together with `0100`, we get `01001000` - the `0x48` byte we see. That's one mystery solved. Time to move to the next byte.

## The Opcode: `0x01`

