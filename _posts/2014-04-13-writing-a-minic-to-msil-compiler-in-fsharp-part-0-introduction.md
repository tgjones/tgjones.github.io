---
title:   "Writing a MiniC-to-MSIL compiler in F# - Part 0 - Introduction"
date:    2014-04-13 10:34:00 UTC
excerpt: "Last year, I learnt F# by using it to write a compiler for a subset of C. In this series, I will explain how each part of the compiler works. This post is the introduction."
---

* You can find the code for this series in [the Mini-C GitHub repository](https://github.com/tgjones/mini-c/).

### Introduction

Last year, I learnt F# by using it to write a compiler. (I also read [Programming F# 3.0](http://www.amazon.com/Programming-F-3-0-Chris-Smith/dp/1449320295/ref=sr_1_1?ie=UTF8&qid=1389175939&sr=8-1&keywords=programming+F%23+3.0) by Chris Smith, a book I recommend.) Writing a compiler is, after all, the [hello world](http://stackoverflow.com/questions/2668288/is-writing-a-compiler-hello-world-for-f) program for F#. After finishing the code, I decided to write this series of blog posts about it - mainly for my own benefit, because it's difficult to truly understand something until you've explained it to somebody else. (Actually I finished the code a year ago. What can I say? I procrastinated and [did some other things](http://www.lingohq.com) in the meantime.)

This isn't going to be a tutorial on F# - although if you don't know F#, you should still be able to follow along. F#'s syntax is initially confusing (for a C# programmer like me), then elegant, and then you don't want to go back to C#... but that's another story. This isn't even a tutorial on compilers. I'm going to explain how this particular compiler works - with the hope that it's of interest to others.

It's worth mentioning that there are a number of other "write a compiler in F#" articles out there. Here's a few:

* [Lisp Compiler in F#](http://www.partario.com/blog/2009/05/lisp-compiler-in-f-introduction.html)
* [A Java to x86 compiler written in F#](http://blogs.msdn.com/b/chrsmith/archive/2008/04/18/a-java-to-x86-compiler-written-in-f.aspx)
* [Let's Build A Compiler... in F#!](http://blogs.teamb.com/craigstuntz/2013/12/12/38766/)
* [Writing a Compiler with F#](http://codegoeshere.blogspot.tw/2009/03/writing-compiler-with-f.html)
* and, of course, the [F# compiler itself is open source](https://github.com/fsharp/fsharp).

### The language

I wanted to keep the compiler as simple as possible. To that end, I googled around and landed on a simple grammar based on a subset of the C programming language called [Mini-C](http://jamesvanboxtel.com/projects/minic-compiler/minic.pdf). This subset includes most of the basics of C, but is missing (among other things) pointers, structs and macros. Since this compiler is never actually going to be used by anybody, that doesn't really matter - the subset is complicated enough to make for an interesting learning exercise.

Here's a sample Mini-C program. As a subset of C, it unsurprisingly looks exactly like C:

``` c
int fib(int n) {
  if (n == 0) 
    return 0;
  if (n == 1)
    return 1;
  return fib(n - 1) + fib(n - 2);
}

int main(void) {
  return fib(10);
}
```

I should say up-front that I possess no expert knowledge about compilers. I vaguely remember attending an undergraduate compilers course, but not much of it stuck. I have read a few books on compilers since then, because I love languages and language implementations. But I'm definitely an enthusiast, rather than an expert. If you spot any mistakes, please let me know!

### The compiler

Compilers have the nice property that they are easy to split into logical parts. That's how my compiler is implemented, and it's also how I'm going to structure this series of blog posts. I'll update the links as I write each part:

* Part 0 - Introduction - this post
* [Part 1 - Defining the abstract syntax tree (AST)](/blog/archive/2014/04/20/writing-a-minic-to-msil-compiler-in-fsharp-part-1-defining-the-abstract-syntax-tree)
* [Part 2 - Lexing and parsing](/blog/archive/2014/05/29/writing-a-minic-to-msil-compiler-in-fsharp-part-2-lexing-and-parsing)
* [Part 3 - Semantic analysis](/blog/archive/2014/06/20/writing-a-minic-to-msil-compiler-in-fsharp-part-3-semantic-analysis)
* [Part 4 - Building the intermediate representation (IR)](/blog/archive/2014/08/23/writing-a-minic-to-msil-compiler-in-fsharp-part-4-building-the-intermediate-representation)
* [Part 5 - Code generation](/blog/archive/2014/09/13/writing-a-minic-to-msil-compiler-in-fsharp-part-5-code-generation)
* [Part 6 - Conclusion](/blog/archive/2014/09/14/writing-a-minic-to-msil-compiler-in-fsharp-part-6-conclusion)

What we'll end up with is a compiler that accepts [a subset of] C source code, parses into an AST, performs some semantic analysis on the AST, builds an intermediate representation, and finally writes MSIL out to a .NET executable.

As a little preview, here is a screenshot of a tool I wrote to accompany the compiler. When you edit the source code in the left panel, the AST and IL panels update in real-time.

![Mini-C GUI](/assets/52cd6886f51f2758e7000011/standard/mini-c.png)

Stay tuned for [Part 1](/blog/archive/2014/04/20/writing-a-minic-to-msil-compiler-in-fsharp-part-1-defining-the-abstract-syntax-tree)!