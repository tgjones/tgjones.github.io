---
title:   "Writing a MiniC-to-MSIL compiler in F# - Part 1 - Defining the abstract syntax tree"
date:    2014-04-20 14:04:00 UTC
excerpt: "Last year, I learnt F# by using it to write a compiler for a subset of C. In this series, I will explain how each part of the compiler works. This post defines the grammar of our mini-language, as well as the abstract syntax tree (AST) used by the parser."
---

* This post is part of the series [Writing a MiniC-to-MSIL compiler in F#](/blog/archive/2014/04/13/writing-a-minic-to-msil-compiler-in-fsharp-part-0-introduction).
* You can find the code for this series in [the Mini-C GitHub repository](https://github.com/tgjones/mini-c/).

### Introduction

So we're going to write a compiler - great. Let's start by deciding on the language we want to compile. The standard way to do this is by defining a language grammar. The grammar will define precisely what source code our compiler will accept as valid input - and implicitly, what isn't valid input.

I chose to use a grammar I found in [this paper](http://jamesvanboxtel.com/projects/minic-compiler/minic.pdf) (which looks like a university course assignment). It defines a language it calls Mini-C, which is a subset of C that omits macros, pointers and structs (and probably lots of other things). There's enough in MiniC to making writing a compiler for it an interesting exercise.

Here's an example of Mini-C source code. By the end of this series, our compiler will be able to turn this source code into an executable .NET application:

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

### The Mini-C grammar

Enough talking. Here's the MiniC grammar, from the [paper](http://jamesvanboxtel.com/projects/minic-compiler/minic.pdf) I mentioned above. It consists of multiple rules. Each rule starts with the rule name, followed by an arrow, followed by one or more tokens. Capital letters indicate a literal - i.e. `BOOL` means the literal string "bool".

```
program       → decl_list
decl_list     → decl_list decl | decl
decl          → var_decl | fun_decl
var_decl      → type_spec IDENT  ; | type_spec IDENT [ ] ;
type_spec     → VOID | BOOL | INT | FLOAT
fun_decl      → type_spec IDENT ( params ) compound_stmt
params        → param_list | VOID
param_list    → param_list , param | param
param         → type_spec IDENT | type_spec IDENT [ ]
stmt_list     → stmt_list stmt | ε
stmt          → expr_stmt | compound_stmt | if_stmt | while_stmt | 
                return_stmt | break_stmt
expr_stmt     → expr ; | ;
while_stmt    → WHILE ( expr ) stmt
compound_stmt → { local_decls stmt_list }
local_decls   → local_decls local_decl | ε
local_decl    → type_spec IDENT ; | type_spec IDENT [ ] ;
if_stmt       → IF ( expr ) stmt | IF ( expr ) stmt ELSE stmt
return_stmt   → RETURN ; | RETURN expr ;

The following expressions are listed in order of increasing precedence:
expr → IDENT = expr | IDENT [ expr ] = expr
     → expr OR expr
     → expr EQ expr | expr NE expr 
     → expr LE expr | expr < expr | expr GE expr  | expr > expr
     → expr AND expr
     → expr + expr | expr - expr 
     → expr * expr | expr / expr  | expr % expr
     → ! expr | - expr | + expr
     → ( expr )
     → IDENT | IDENT [ expr ] | IDENT ( args ) | IDENT . size
     → BOOL_LIT | INT_LIT  | FLOAT_LIT | NEW type_spec [ expr ]

arg_list → arg_list , expr | expr
args     → arg_list | ε
```

If that makes your eyes hurt, don't worry. Let's break it down, taking the `var_decl` rule as an example:

```
var_decl → type_spec IDENT  ; | type_spec IDENT [ ] ;
```

And here's a line of Mini-C code, which is valid according to the `var_decl` rule:

``` c
bool MyVar[];
```

The vertical bar (`|`) in the middle of the `var_decl` rule indicates an OR relationship: source code will match the rule if it looks like the left side OR the right side. Let's look at the right side:

* First we have a `type_spec`. The `type_spec` rule is defined in the grammar just below `var_decl`. A `type_spec` can be any of the following strings: `void`, `bool`, `int` or `float`.
* Then there's an identifier. Mini-C uses the same rules as standard C for valid identifiers.
* Finally, there's a pair of square brackets: `[` followed by `]`.

This level of precision in the grammar will be necessary when we come to write the parser. But before we can write the parser, we need to define the hierarchy of objects that the parser will store its results in. This is called the abstract syntax tree (AST).

### The Abstract Syntax Tree (AST)

Now that we have a grammar for Mini-C, we need to turn it into code. The code won't do anything yet - it's just how we will represent source code after it has been parsed.

Wikipedia defines an [abstract syntax tree](http://en.wikipedia.org/wiki/Abstract_syntax_tree) (AST) as

> ... a tree representation of the abstract syntactic structure of source code written in a programming language.

When you're writing an F# program, as soon as you hear the word "tree", you can smile. F# was built for such things. In C#, we'd probably use a hierarchy of classes, but F#'s [discriminated unions](http://msdn.microsoft.com/en-us/library/dd233226.aspx) provide a more elegant solution. Hopefully you'll see that F# lets us write code that looks surprisingly similar to the grammar. You'll find this code in [Ast.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Ast.fs) in the [Mini-C GitHub repository](https://github.com/tgjones/mini-c).

Let's write some code. I'm not going to explain F#'s syntax, because there are plenty of other places that do a better job than I could. If you're not familiar with F#, it might look a bit weird at first, but it grows on you.

``` ocaml
type Program = Declaration list
```

This makes `Program` essentially an alias for a list of declarations. (In C#, we'd write `List<Declaration>`.)

``` ocaml
and Declaration =
  | StaticVariableDeclaration of VariableDeclaration
  | FunctionDeclaration of FunctionDeclaration
```

A declaration is either a static variable declaration or a function declaration.

``` ocaml
and TypeSpec =
  | Void
  | Bool
  | Int
  | Float
```

The supported types in Mini-C are `void`, `bool`, `int` and `float`.

``` ocaml
and VariableDeclaration = 
    | ScalarVariableDeclaration of TypeSpec * Identifier
    | ArrayVariableDeclaration of TypeSpec * Identifier
```

A variable declaration can either be a scalar variable declaration or an array variable declaration. Both types of variable declaration consist of a type and an identifier. The `*` character is what F# uses to define tuples. In C#, we might write `Tuple<TypeSpec, Identifier>` (except we wouldn't because nobody uses tuples in C#).

``` ocaml
and FunctionDeclaration = TypeSpec * Identifier * Parameters * CompoundStatement
```

A function declaration consists of a type, an identifier, some function parameters, and a "compound statement". We'll see what compound statements are shortly.

``` ocaml
and Identifier = string

and Parameters = VariableDeclaration list

and IdentifierRef = { Identifier : string; }
```

An identifier is simply a string. The `Parameters` type (as used in `FunctionDeclaration`) is an alias for a list of variable declarations. It's worth noting that the `VariableDeclaration` type is used for both function parameters and global variable declarations. Although the grammar is a bit different in these two contexts, the actual data we need to store after parsing is the same.

``` ocaml
and Statement =
  | ExpressionStatement of ExpressionStatement
  | CompoundStatement of CompoundStatement
  | IfStatement of IfStatement
  | WhileStatement of WhileStatement
  | ReturnStatement of Expression option
  | BreakStatement
```

Here are examples of each type of Mini-C statement:

| Statement Type       | Mini-C Code                     |
|----------------------|---------------------------------|
| Expression Statement | `i = 2 + 3;`                    |
| Compound Statement   | `{ int j; j = 5; j = j + 1; }`  |
| If Statement         | `if (i == 2) { return 3; }`     |
| While Statement      | `while (i < 3) { i = i + 1; } ` |
| Return Statement     | `return true;`                  |
| Break Statement      | `break;`                        |

``` ocaml
and ExpressionStatement =
  | Expression of Expression
  | Nop
```

Many languages, including Mini-C, differentiate between *expressions* and *statements*. At a *very* high level:

* **Expressions** evaluate to a value.
* **Statements** do something.

**Expression statements** are where the two come together. An expression statement is simply a statement composed of an expression. We'll soon see what expressions are.

``` ocaml
and CompoundStatement = LocalDeclarations * Statement list

and LocalDeclarations = VariableDeclaration list
```

A compound statement is composed of a list of local variable declarations, and a list of statements. You might notice that there's some recursion going on here: one of the `Statement` types is `CompoundStatement`, which itself is composed of a list of `Statement`s. That simplifies AST construction, and will also make working with the AST easier later on.

``` ocaml
and IfStatement = Expression * Statement * Statement option
```

An `if` statement is composed of:

* an expression - i.e. the condition to test
* a statement to execute if the condition evaluates to true, which could be either a single statement or a compound statement
* an optional statement to execute if the condition evaluates to false - this is what you'd write in the `else` part of the if statement

Note that adding `option` after a type such as `Statement` is roughly analogous to writing `Statement?` or `Nullable<Statement>` in C#, except that F# is more awesome and allows the use of `option` for reference types too.

``` ocaml
and WhileStatement = Expression * Statement
```

While statements, at least far as the AST goes, are similar to if statements. They are composed of an expression - which is the condition to test on each time through the loop, and a statement (which again, could be a compound statement).

``` ocaml
and Expression =
  | ScalarAssignmentExpression of IdentifierRef * Expression
  | ArrayAssignmentExpression of IdentifierRef * Expression * Expression
  | BinaryExpression of Expression * BinaryOperator * Expression
  | UnaryExpression of UnaryOperator * Expression
  | IdentifierExpression of IdentifierRef
  | ArrayIdentifierExpression of IdentifierRef * Expression
  | FunctionCallExpression of Identifier * Arguments
  | ArraySizeExpression of IdentifierRef
  | LiteralExpression of Literal
  | ArrayAllocationExpression of TypeSpec * Expression
```

In partnership with statements, expressions make up the core of most languages. Mini-C has a simple grammar, and correspondingly few expression types. Here are examples of each type of Mini-C expression:

| Expression Type   | Mini-C Code       |
|-------------------|-------------------|
| Scalar assignment | `i = 2 + 3`       |
| Array assignment  | `j[0] = 2 + 3;`   |
| Binary            | `i == 2`          |
| Unary             | `-i`              |
| Identifier        | `i`               |
| Array identifier  | `j[0]`            |
| Function call     | `myFunc(0, true)` |
| Array size        | `j.size`          |
| Literal           | `true`            |
| Array allocation  | `new int[3]`      |

Note that many of these definitions are recursive. `ScalarAssignmentExpression`, for example, is composed of an `IdentifierRef` and an `Expression`. If you think about it, this makes sense - on the right hand side of an assignment, you want to be able to use any arbitrary expression, including other assignments (at least in C-like languages).

``` ocaml
and BinaryOperator =
  | ConditionalOr
  | Equal
  | NotEqual
  | LessEqual
  | Less
  | GreaterEqual
  | Greater
  | ConditionalAnd
  | Add
  | Subtract
  | Multiply
  | Divide
  | Modulus

and UnaryOperator =
  | LogicalNegate
  | Negate
  | Identity
```

Binary and unary operators are hopefully self-explanatory, and comparable to the operators in most C-based languages.

``` ocaml
and Literal =
  | BoolLiteral of bool
  | IntLiteral of int
  | FloatLiteral of float
```

Mini-C supports these literals:

* `BoolLiteral` - `true` or `false`
* `IntLiteral` - `1`, `-2`, etc.
* `FloatLiteral` - `1.23`, `-0.5`, etc.

### Until next time...

And that's it! You'll find all that code in [Ast.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Ast.fs) in the GitHub repository. F#'s discriminated unions are perfectly suited for defining an abstract syntax tree.

[Next time](/blog/archive/2014/05/29/writing-a-minic-to-msil-compiler-in-fsharp-part-2-lexing-and-parsing), we'll build a parser capable of turning Mini-C source code into a tree of objects, using the AST we've defined here.