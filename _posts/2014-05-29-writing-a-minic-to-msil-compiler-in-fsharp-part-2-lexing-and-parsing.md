---
title:   "Writing a MiniC-to-MSIL compiler in F# - Part 2 - Lexing and parsing"
date:    2014-05-29 10:05:00 UTC
excerpt: "Last year, I learnt F# by using it to write a compiler for a subset of C. In this series, I will explain how each part of the compiler works. This post discusses parsing Mini-C source code into a tree of objects (or AST), which makes later compiler stages much simpler to implement."
---

* This post is part of the series [Writing a MiniC-to-MSIL compiler in F#](/blog/archive/2014/04/13/writing-a-minic-to-msil-compiler-in-fsharp-part-0-introduction).
* You can find the code for this series in [the Mini-C GitHub repository](https://github.com/tgjones/mini-c/).

### Introduction

In the [last post](/blog/archive/2014/04/20/writing-a-minic-to-msil-compiler-in-fsharp-part-1-defining-the-abstract-syntax-tree), we defined an abstract syntax tree (AST) for our compiler. Now we'll focus on turning Mini-C source code like this:

``` c
int main(bool b) {
  while (b)
    break;
  return 1;
}
```

into a tree of objects (or, an abstract syntax tree) like this:

``` fsharp
[Ast.FunctionDeclaration(
  Ast.Int, "main",
  [ Ast.ScalarVariableDeclaration (Ast.Bool, "b") ],
  (
    [],
    [
      Ast.WhileStatement(
        Ast.IdentifierExpression { Identifier = "b" },
        Ast.BreakStatement
      )
      Ast.ReturnStatement(Some(Ast.LiteralExpression (Ast.IntLiteral 1)))
    ]
  )
)]
```

(That's an actual example from [one of Mini-C's unit tests](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler.Tests/ParserTests.fs#L313).)

Compilers traditionally break this transformation down into two steps: *lexing* and *parsing*.

#### Lexing

A lexer takes the raw source code, and breaks it up into "tokens". For example, if you take this source code:

``` c
while (b)
  break;
```

and pass it through a C lexer, you will get something like this back:

`while keyword` `openParen` `identifier` `closeParen` `break keyword` `semicolon`

In C-based languages, including Mini-C, white space is insignificant, so the lexer throws it away.

Note that the lexer doesn't understand or verify the language syntax; for example, if you left out a `)` in the previous example, the lexer would still work. Validating syntax one of the tasks performed by a parser.

#### Parsing

A parser takes the stream of tokens output by a lexer, and builds some sort of structural representation, validating syntax in the process. For example, given this stream of tokens from the lexer:

`while keyword` `openParen` `identifier` `closeParen` `break keyword` `semicolon`

Mini-C's parser will build this abstract syntax tree:

``` fsharp
Ast.WhileStatement(
  Ast.IdentifierExpression { Identifier = "b" },
  Ast.BreakStatement
)
```

If there are any syntax errors - for example, a missing `)` - the parser will (hopefully) detect that and return the relevant error message.

That's as much detail as I'm going to go into on lexing and parsing in the abstract - it's a fascinating area though, and I recommend [LL and LR parsing demystified](http://blog.reverberate.org/2013/07/ll-and-lr-parsing-demystified.html) as a good introduction to parsing.

#### Hand-coded or generated?

It's perfectly possible (and quite fun) to write a lexer and parser from scratch. I did just that for [StitchUp](https://github.com/tgjones/stitchup/tree/master/src/StitchUp.Content.Pipeline/FragmentLinking/Parser) a few years ago. For simple languages, it's not that hard to write a [recursive descent parser](http://en.wikipedia.org/wiki/Recursive_descent_parser).

Perhaps the most popular (?) option is to use a parser generator. [ANTLR](http://www.antlr.org/), [Bison](http://www.gnu.org/software/bison/) and [Yacc](http://dinosaur.compilertools.net/) are some of the most used parser generators. [FsLex and FsYacc](http://fsprojects.github.io/FsLexYacc/) are lexer and parser generator tools for use with F#. I actually did originally use FsLex and FsYacc for Mini-C, but I didn't like the separate steps required to first build the parser generator code, and then compile it into my project. It's a very context-specific decision, though - for a "real" language project, this type of tool would probably be my go-to.

A third option is to use a runtime library, where you build the language grammar using the same language as you're using for the rest of the compiler. I chose this option for Mini-C; specifically, I used the [Piglet parsing library](http://dervall.github.io/Piglet/) for both lexing and parsing. This is a small and elegant library which allows the language grammar to be written in F# code, with no separate build step needed.

### Configuring Piglet

To parse source code using Piglet, we have to tell Piglet about the the following types of entity. I'll talk more about each one in the following sections.

* Nonterminal symbols
* Terminal symbols
* Precedence
* Productions
* Entry point

Before any of that, Piglet requires us to create an `IParserConfigurator` object:

``` fsharp
let configurator = ParserFactory.Configure<obj>()
```

#### Nonterminal symbols

Nonterminal symbols are composed of other (nonterminal or terminal) symbols. For example, in Mini-C, a `Statement` is a nonterminal symbol. It's nonterminal because a statement is itself composed of other symbols, such as expressions, assignments, local variable declarations, etc.

Here is how we define nonterminal symbols using Piglet. I'm only including a few here - the full list is in the [source code](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Parser.fs#L14). (I wrote some [F#-friendly wrappers](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/ParsingUtilities.fs) on top of Piglet's API, which I'm using here; they make for a more pleasant experience when using Piglet from F#.)

``` fsharp
let nonTerminal<'T> () = new NonTerminalWrapper<'T>(configurator.CreateNonTerminal())

let program                   = nonTerminal<Program>()
let declarationList           = nonTerminal<Declaration list>()
let declaration               = nonTerminal<Declaration>()
let staticVariableDeclaration = nonTerminal<VariableDeclaration>()
let functionDeclaration       = nonTerminal<FunctionDeclaration>()
```

There isn't yet any connection between these nonterminals - that's what productions are for. We'll get to them shortly.

#### Terminal symbols

Terminal symbols are those symbols which are *not* composed of other symbols. If you imagine the source code as a tree, terminal symbols are the "leaves". Keywords, string literals, number literals and punctuation are all terminal symbols.

First, we define a couple of helper methods - don't pay too much attention to them, they just allow us to write terminal symbols in a strongly-typed way, which Piglet itself doesn't really support:

``` fsharp
let terminalParse<'T> regex (onParse : (string -> 'T)) =
    new TerminalWrapper<'T>(configurator.CreateTerminal(regex, (fun s -> box (onParse s))))

let terminal regex =
    new TerminalWrapper<string>(configurator.CreateTerminal(regex))
```

Now we can define some terminals. I'll include a few of each type here; again, the rest are in the [source code](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Parser.fs#L49).

``` fsharp
// Keywords
let ifKeyword    = terminal      "if"
let elseKeyword  = terminal      "else"
let whileKeyword = terminal      "while"

// Operators
let minus        = terminal      "-"
let exclamation  = terminal      "!"

// Literals
let intLiteral   = terminalParse @"\d+"      (fun s -> Ast.IntLiteral(int32 s))
let trueLiteral  = terminalParse "true"      (fun s -> Ast.BoolLiteral(true))
let falseLiteral = terminalParse "false"     (fun s -> Ast.BoolLiteral(false))
let boolKeyword  = terminalParse "bool"      (fun s -> Ast.Bool)

// Identifier (i.e. variable name, function name, parameter name, etc.)
let identifier   = terminalParse "[a-zA-Z_][a-zA-Z_0-9]*"  (fun s -> s)

// Punctuation
let semicolon    = terminal      ";"
let comma        = terminal      ","
```

#### Precedence

Next, we have to tell Piglet about precedence. To simplify somewhat, precedence is essentially how we resolve situations like this:

``` c
1 + 2 * 3
```

Does this mean `1 + (2 * 3)` or `(1 + 2) * 3`? Precedence rules define exactly how to resolve this kind of ambiguity.  C-based languages have long-standing conventions for precedence order. For example, multiplications and divisions are evaluated before additions and subtractions.

With Piglet, we call the `LeftAssociative` method multiple times in increasing order of precedence. For each call, we pass in the terminal symbols that should be grouped together at the same precedence level. Here is the code to tell Piglet that multiplications and divisions have a *greater precedence* than additions and subtractions:

``` fsharp
configurator.LeftAssociative(downcast exclamation.Symbol,
                             downcast plus.Symbol,
                             downcast minus.Symbol)
                             |> ignore
configurator.LeftAssociative(downcast asterisk.Symbol,
                             downcast forwardSlash.Symbol,
                             downcast percent.Symbol)
                             |> ignore
```

Here is the [full code responsible for configuring Mini-C's precedence rules](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Parser.fs#L92).

#### Productions

Now that we've created our nonterminals and terminals, we need to tell Piglet how they relate to each other. We also need to tell Piglet how to build the abstract syntax tree. Both these tasks are handled using productions (or equivalently, production rules).

We already know from [Mini-C's grammar](/blog/archive/2014/04/20/writing-a-minic-to-msil-compiler-in-fsharp-part-1-defining-the-abstract-syntax-tree) how each element in the syntax connects together. We just need to express this in code.

Let's start with the first production: the one for a `program`. This is the relevant line from Mini-C's grammar:

```
program → decl_list
```

And here is how we defined the abstract syntax tree node for a program:

``` fsharp
type Program = Declaration list
```

Now, here is the production rule:

``` fsharp
program.AddProduction(declarationList).SetReduceToFirst()
```

Let's break it down:

* `program` and `declarationList` are both nonterminals that we declared above.
* A `program` nonterminal is composed of one symbol, a `declarationList` nonterminal. 
* Piglet is a [shift-reduce parser](http://en.wikipedia.org/wiki/Shift-reduce_parser), which I won't get into here, but that means that we need to tell Piglet "after you find some code which matches this list of symbols, here is how you convert that list of symbols into something useful".
* In this case, there is only one symbol for the `program` production (`declarationList`), so `SetReduceToFirst` tells Piglet to simply return `declarationList`.

Let's keep going - hopefully things will become clearer once we've looked at some more examples. First, here is the grammar rule for declaration lists:

```
decl_list → decl_list decl | decl
```

This is a recursive definition. A declaration list is composed of either:

* a declaration list, followed by a declaration, or
* a declaration.

And here is how we turn that into F# code:

``` fsharp
declarationList.AddProduction(declarationList, declaration)
  .SetReduceFunction (fun a b -> a @ [b])
declarationList.AddProduction(declaration).SetReduceFunction (fun a -> [a])
```

Hopefully you can see that there's a marked similarity between that code and the grammar rule above.

The `SetReduceFunction` method calls provide functions that indicate how to build the AST from the nonterminals. The first one - `fun a b -> a @ [b]` - appends declaration to the end of the declaration list that we're in the middle of building up. The second one wraps a single declaration into a list. We have to perform these acrobatics in order to get both productions to return the same type - `Declaration list`.

Let's do one more: here is the grammar rule for declarations:

```
decl → var_decl | fun_decl
```

And here are the productions:

``` fsharp
declaration.AddProduction(staticVariableDeclaration)
  .SetReduceFunction (fun a -> Ast.StaticVariableDeclaration a)
declaration.AddProduction(functionDeclaration)
  .SetReduceFunction (fun a -> Ast.FunctionDeclaration a)
```

Finally, we can see a concrete use of the abstract syntax tree types that defined in the [previous part](/blog/archive/2014/04/20/writing-a-minic-to-msil-compiler-in-fsharp-part-1-defining-the-abstract-syntax-tree).

Let's skip ahead a bit to production rules for expressions. Here are the grammar rules for expressions:

```
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
```

And here are (some of) the production rules for expressions:

``` fsharp
expression.AddProduction(expression, doublePipes, expression)
  .SetReduceFunction (fun a _ c -> Ast.BinaryExpression(a, Ast.ConditionalOr, c))
expression.AddProduction(expression, doubleEquals, expression)
  .SetReduceFunction (fun a _ c -> Ast.BinaryExpression(a, Ast.Equal, c))

expression.AddProduction(expression, plus, expression)
  .SetReduceFunction (fun a _ c -> Ast.BinaryExpression(a, Ast.Add, c))
expression.AddProduction(expression, minus, expression)
  .SetReduceFunction (fun a _ c -> Ast.BinaryExpression(a, Ast.Subtract, c))

expression.AddProduction(openParen, expression, closeParen)
  .SetReduceFunction (fun _ b _ -> b)

expression.AddProduction(identifier)
  .SetReduceFunction (fun a -> Ast.IdentifierExpression ({ Identifier = a }))
expression.AddProduction(identifier, openSquare, expression, closeSquare)
  .SetReduceFunction (fun a _ c _ -> Ast.ArrayIdentifierExpression({ Identifier = a }, c))
```

Hopefully it's starting to make sense! Some of the symbols in these productions - i.e. `identifier` - is a terminal, and some are nonterminals. Every nonterminal needs to have at least one production associated with it. If we start with `program`, and follow the productions for each nonterminal, we eventually get down to a production rule that only contains terminals (i.e. keywords, literals, etc).

Here is the [full code for configuring productions](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Parser.fs#L124).

#### Entry point

The final step is to define an entry point to the parser. This turns out to be straightforward:

``` fsharp
let parser = configurator.CreateParser()

let parse (s : string) =
  try
    parser.Parse(s) :?> Program
  with
    | :? Piglet.Lexer.LexerException as ex ->
      raise (lexerError ex.Message)
    | :? Piglet.Parser.ParseException as ex ->
      raise (parserError ex.Message)
```

### A worked example

Let's use a practical example to see how the parser takes in source code, and builds an abstract syntax tree.

We'll use this code - it's a simple top-level variable declaration, but it is also a complete Mini-C program, even if it doesn't do a lot...

``` c
int a;
```

And here is how Piglet will parse this code (more accurately, it is first lexing it, but Piglet hides that step behind its parser API):

1. Shift-reduce parsers start from the "bottom" and work their way up. The first symbol that Piglet will parse will be `int`. This matches the `intKeyword` terminal.

    ``` fsharp
    let intKeyword = terminalParse "int" (fun s -> Ast.Int)
    ```

2. Next, Piglet sees that the `typeSpec` nonterminal has a production rule composed of (only) the `intKeyword` terminal:

    ``` fsharp
    typeSpec.AddProduction(intKeyword).SetReduceToFirst()
    ```

3. Piglet calls that production's reduce function, resulting in:

    ``` fsharp
    Ast.Int
    ```

4. Next, Piglet sees that `a` matches the `identifier` terminal:

    ``` fsharp
    let identifier = terminalParse "[a-zA-Z_][a-zA-Z_0-9]*" (fun s -> s)
    ```

5. Next, Piglet parses `;` and finds that it matches the `semicolon` terminal:

    ``` fsharp
    let semicolon = terminal ";"
    ```

6. So Piglet has now matched three symbols - `typeSpec`, `identifier` and `semicolon`. It sees that one of the nonterminals has a production rule that matches these three terminals (order is important):

    ``` fsharp
    staticVariableDeclaration.AddProduction(typeSpec, identifier, semicolon)
      .SetReduceFunction (fun a b _ -> Ast.ScalarVariableDeclaration(a, b))
    ```

7. Piglet calls that production's reduce function, resulting in:

    ``` fsharp
    Ast.ScalarVariableDeclaration(Ast.Int, "a")
    ```

8. The `declaration` nonterminal has a production rule which matches a single `staticVariableDeclaration` symbol:

    ``` fsharp
    declaration.AddProduction(staticVariableDeclaration)
      .SetReduceFunction (fun a -> Ast.StaticVariableDeclaration a)
    ```

9. Piglet calls that production rule's reduce function, resulting in: 

    ``` fsharp
    Ast.StaticVariableDeclaration(Ast.ScalarVariableDeclaration(Ast.Int, "a"))
    ```

10. The `declarationList` nonterminal has a matching production rule:

    ``` fsharp
    declarationList.AddProduction(declaration).SetReduceFunction (fun a -> [a])
    ```

11. Calling `declarationList`'s reduce function gives us: 

    ``` fsharp
    [Ast.StaticVariableDeclaration(Ast.ScalarVariableDeclaration(Ast.Int, "a"))]
    ```

12. Finally, the top-level `program` nonterminal has a production rule which matches `declarationList`:

    ``` fsharp
    program.AddProduction(declarationList).SetReduceToFirst()
    ```

13. `SetReduceToFirst` gives us the same result as `declarationList`'s reduce function:

    ``` fsharp
    [Ast.StaticVariableDeclaration(Ast.ScalarVariableDeclaration(Ast.Int, "a"))]
    ```

**And we're done!** We've gone from the source code, to an abstract syntax tree. Whew, this post was a bit longer than I anticipated... You'll find all the code for this part in [ParsingUtilities.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/ParsingUtilities.fs) and [Parser.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/Parser.fs) in the GitHub repo.

All we've done so far is check that the code is syntactically correct, and built an AST. But the program as a whole might still be "wrong" (or more technically, it might be semantically invalid). For example, the code might reference a variable that hasn't been declared, or it might assign a string to an integer variable. This kind of verification and type checking is called semantic analysis, and will be the subject of the [next part in this series](/blog/archive/2014/06/20/writing-a-minic-to-msil-compiler-in-fsharp-part-3-semantic-analysis).