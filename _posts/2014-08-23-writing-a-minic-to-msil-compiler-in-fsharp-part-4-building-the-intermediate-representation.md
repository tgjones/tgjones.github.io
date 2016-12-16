---
title:   "Writing a MiniC-to-MSIL compiler in F# - Part 4 - Building the intermediate representation"
date:    2014-08-23 15:30:00 UTC
excerpt: "Last year, I learnt F# by using it to write a compiler for a subset of C. In this series, I will explain how each part of the compiler works. This post describes building the intermediate representation (IR), which in this compiler is an abstract representation of MSIL."
---

* This post is part of the series [Writing a MiniC-to-MSIL compiler in F#](/blog/archive/2014/04/13/writing-a-minic-to-msil-compiler-in-fsharp-part-0-introduction).
* You can find the code for this series in [the Mini-C GitHub repository](https://github.com/tgjones/mini-c/).

### Introduction

Let's review where we've got to with our compiler. By the end of the [parsing and lexing stage](/blog/archive/2014/05/29/writing-a-minic-to-msil-compiler-in-fsharp-part-2-lexing-and-parsing), we were able to take source code like this:

``` c
int a;
int main(bool b) {
  float c[];
  return 1;
}
```

and turn it into an abstract syntax tree like this:

``` ocaml
[
  Ast.StaticVariableDeclaration(Ast.ScalarVariableDeclaration(Ast.Int, "a")),
  Ast.FunctionDeclaration(
    Ast.Int, "main",
    [ Ast.ScalarVariableDeclaration(Ast.Bool, "b") ],
    (
      [ Ast.ArrayVariableDeclaration(Ast.Float, "c") ],
      [ Ast.ReturnStatement(Some(Ast.LiteralExpression(Ast.IntLiteral 1))) ]
    )
  )
]
```

Then in [semantic analysis](/blog/archive/2014/06/20/writing-a-minic-to-msil-compiler-in-fsharp-part-3-semantic-analysis), we created a symbol table:

| Identifier | Variable declaration                           |
| ---------- | ---------------------------------------------- |
| `a`        | `Ast.ScalarVariableDeclaration(Ast.Int, "a")`  |
| `b`        | `Ast.ScalarVariableDeclaration(Ast.Bool, "b")` |
| `c`        | `Ast.ArrayVariableDeclaration(Ast.Float, "c")` |

and a function table:

| Identifier | Function declaration                           |
| ---------- | ---------------------------------------------- |
| `main`     | `{`<br>&nbsp;&nbsp;`ReturnType = Ast.Int,`<br>&nbsp;&nbsp;`ParameterTypes = [ { Type = Ast.Bool; IsArray = false } ]`<br>`}` |

and an expression types table:

| Expression | Expression type                           |
| ---------- | ---------------------------------------------- |
| `Ast.LiteralExpression(`<br>&nbsp;&nbsp;`Ast.IntLiteral(1)`<br>`)` | `{ Type = Ast.Int; IsArray = false }` |

I haven't talked much about the end goal of this compiler. Just like any other .NET language (C#, F#, etc.), it outputs executable files (.exe) containing [Microsoft Intermediate Language (MSIL)](http://en.wikipedia.org/wiki/Common_Intermediate_Language), an intermediate language that is used by the [Common Language Runtime (CLR)](http://en.wikipedia.org/wiki/Common_Language_Runtime) to run .NET applications. (Unlike those proper languages, there's no option to output .dll's, although that wouldn't be hard to add.) MSIL is (visually, at least) closer to assembly code than to a high level language like C#, but it's still high level enough to have concepts like classes and methods. The .NET virtual machine is stack-based, so most of the MSIL we use will push to, operate on, or pop from the stack. If you haven't seen MSIL before, hopefully it will become clearer when looking at some concrete examples.

Instead of building an .exe file directly from the AST (which is possible), we're going to first build an in-memory representation of the MSIL. This intermediate representation (IR) is easier to test and reason about than if we skipped straight to code generation. In a "real" compiler, the IR is also a good place to implement optimisations. In the next part, we'll write this IR out as real MSIL to an executable file.

By the end of this part, we'll be able to build this intermediate representation of the sample code above:

``` ocaml
{ // ILClass
  Fields  = [ { Type = typeof<int>; Name = "a" } ];
  Methods = [
    { // ILMethod
      Name       = "main";
      ReturnType = typeof<int>;
      Parameters = [ { Type = typeof<bool>; Name = "b" } ];
      Locals     = [ { Type = typeof<single[]>; Name = "c" } ];
      Body       = [ IL.Ldc_I4(1); IL.Ret ];
    }
  ]
} 
```

The code is in [IL.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/IL.fs) and [ILBuilder.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/ILBuilder.fs) in the GitHub repository.

### Defining the intermediate representation

Our intermediate representation needs to be rich enough to express all the MSIL that we could possibly want to generate for any given Mini-C program. Fortunately, Mini-C is quite simple, so we only need to use a small subset of MSIL.

For any Mini-C program, we will generate one (and only one) MSIL class. This is really just a container of global functions, as far as Mini-C is concerned.

Then, for each function in the Mini-C program, we'll generate an MSIL method. For each global variable declaration, we'll generate a field.

Similar to the abstract syntax tree, we're going to use F# records and discriminated unions to define our intermediate representation. F# is famous for enabling functional programming, but the flip side of that is its strong support for a much wider variety of data types than are in, say, C#. (It looks like [C# 6 might be getting records](https://roslyn.codeplex.com/discussions/560339), though: welcome to the future, C#.)

At the top level we have `ILClass`:

``` ocaml
type ILClass = 
  {
    Fields  : ILVariable list;
    Methods : ILMethod list;
  }
```

Variables are represented with `ILVariable`, whose `Type` field is a `System.Type` instance (we're getting closer to MSIL, so it will make things easier if we switch to `System.Type` rather than our own `Ast.Type` that we've been using until now):

``` ocaml
and ILVariable =
  {
    Type  : Type;
    Name  : string;
  } 
```

Methods are represented with, you guessed it, `ILMethod`:

``` ocaml
and ILMethod =
  {
    Name       : string;
    ReturnType : Type;
    Parameters : ILVariable list;
    Locals     : ILVariable list;
    Body       : ILOpCode list;
  } 
```

We're going to make use of [MSIL labels](http://msdn.microsoft.com/en-us/library/system.reflection.emit.ilgenerator.marklabel(v=vs.110).aspx) to handle branching:

``` ocaml
and ILLabel = int
```

And finally, `ILOpCode` contains the actual opcodes used in method bodies. These map one-to-one to [MSIL opcodes](http://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.aspx).

``` ocaml
and ILOpCode =
  | Add
  | Br of ILLabel
  | Brfalse of ILLabel
  | Brtrue of ILLabel
  | Call of string
  | CallClr of System.Reflection.MethodInfo
  | Ceq
  | Cge
  | Cgt
  | Cle
  | Clt
  | Dup
  | Div
  | Label of ILLabel
  | Ldarg of int16
  | Ldc_I4 of int
  | Ldc_R8 of float
  | Ldelem of Type
  | Ldlen
  | Ldloc of int16
  | Ldsfld of ILVariable
  | Mul
  | Neg
  | Newarr of Type
  | Pop
  | Rem
  | Ret
  | Starg of int16
  | Stelem of Type
  | Stloc of int16
  | Stsfld of ILVariable
  | Sub 
```

Some of these cases have fields: for example, `Br` needs to know the `ILLabel` to break to. The cases that don't have fields mostly take their operands directly from the stack.

And that's it for our abstract representation of MSIL - using ILClass, ILVariable, ILMethod, ILLabel and ILOpCode, we can represent all possible Mini-C programs.

### Building the intermediate representation

As noted above, the output of this compiler stage will always be a single `ILClass`. Within that class, we can have multiple fields and multiple methods. Fields are quite easy to generate - they are more or less direct mappings from `Ast.StaticVariableDeclaration` objects in the abstract syntax tree. Methods are harder, which makes sense, because that's where the the real work happens - we're going from a higher-level AST representation (which has things like `while` loops) to a lower-level [abstraction of an] MSIL representation (which doesn't).

#### Helpers

We'll need a few helper types and functions. The first is `ILVariableScope`, which stores whether a variable has field, argument or local scope. The `int16` values are used as indices in the various MSIL opcodes that deal with arguments and locals.

``` ocaml
type private ILVariableScope =
  | FieldScope of ILVariable
  | ArgumentScope of int16
  | LocalScope of int16
```

We're going to need a mapping between `Ast.VariableDeclaration`s and `ILVariableScope`s:

``` ocaml
type private VariableMappingDictionary = Dictionary<Ast.VariableDeclaration, ILVariableScope>
```

And finally, some helper functions to get the .NET type from an `Ast.Type`, and to create an `ILVariable` from an `Ast.VariableDeclaration`:

``` ocaml
module private ILBuilderUtilities =
  let typeOf =
    function
    | Ast.Void  -> typeof<System.Void>
    | Ast.Bool  -> typeof<bool>
    | Ast.Int   -> typeof<int>
    | Ast.Float -> typeof<float>

  let createILVariable =
    function
    | Ast.ScalarVariableDeclaration(t, i) as d ->
      {
        ILVariable.Type = typeOf t; 
        Name = i;
      }
    | Ast.ArrayVariableDeclaration(t, i) as d ->
      {
        ILVariable.Type = (typeOf t).MakeArrayType(); 
        Name = i;
      }
```

#### Building `ILMethod`s

Before we get into the nitty gritty of building the IR, let's briefly talk about how the .NET virtual machine works. It is stack-based, which means that expressions (such as `1 + 2`) are evaluated by pushing operands onto the stack, popping operands off the stack when evaluating an instruction (such as `add`), and then pushing the result back onto the stack. (This is different from how CPUs actually work. Real computers are (always?) register-based, not stack-based, but fortunately for us the .NET CLR takes care of that little detail.)

Let's look at an example. Let's say we want to evaluate `1 + 2`. Here is the MSIL for that:

``` c
ldc.i4 1   // Pushes "1" onto the stack as an int32
ldc.i4 2   // Pushes "2" onto the stack as an int32
add        // Pops "1" and "2" from the stack, adds them, and pushes the result back onto the stack
```

After this executes, we'll have the result (hopefully `3`) on top of the stack, and a subsequent instruction can use that result.

Some MSIL instructions push to the stack, some pop from the stack, some do both, and some don't affect the stack at all. 

Most of our conversion to MSIL will be straightforward, but `while` statements deserve some explanation. In Mini-C, a while statement looks like this:

``` c
while (i > 3) {
  i -= 1;
}
```

In MSIL, that same code looks like this:

``` c
  br condition // Always branch to "condition" label
start:
  ldloc 0      // Push local with index 0 ("i") onto stack
  ldc.i4 1     // Push "1" onto stack
  sub          // Pop and subtract
  stloc 0      // Store result back into local with index 0 ("i")
condition:
  ldloc 0      // Push local with index 0 ("i") onto stack
  ldc.i4 3     // Push "3" onto stack
  cgt          // Pop and compare the two values; push "1" if first is greater than second, otherwise "0"
  brtrue start // Branch to "start" label if comparison pushed "1"
end:
  // Next instruction
```

The lines beginning with a word, followed by a colon, are labels. They are used as targets for branching instructions. We will dynamically generate these labels for each `while` statement that we convert to MSIL representation.

(The `end` label isn't used in this example, but if we had a `break` statement inside the `while` statement body, it would branch to the `end`.)

`while` statements are the exception - most of the abstract syntax tree will map much more directly to our IR. We just need to recurse through the AST, building a list of MSIL instructions as we go. Let's get started.

``` ocaml
type ILMethodBuilder(semanticAnalysisResult : SemanticAnalysisResult,
                     variableMappings : VariableMappingDictionary) =
  let mutable argumentIndex = 0s
  let mutable localIndex = 0s
  let arrayAssignmentLocals = Dictionary<Ast.Expression, int16>()
  let mutable labelIndex = 0
  let currentWhileStatementEndLabel = Stack<ILLabel>()
```

First, we've got some messy book-keeping declarations. `argumentIndex` and `localIndex` will keep track of the current index for method arguments and local variables, respectively.

Array assignment expressions (`i[0] = j`), being an expression, have a value. For example, you could write `k = i[0] = j`, and `k` would be assigned the value of the array assignment expression. `arrayAssignmentLocals` is used to keep track of the local indices of these values - we could cope without it, but we'd have to store the value in the array, and then immediately load it again from the array, which is probably less efficient than using a local (admittedly, I haven't done any profiling).

It's possible to have nested `while` statements. In order to correctly handle `break` statements, we need to keep track of the end label for the "current" while statement, so that we can branch to the right place.

``` ocaml
let lookupILVariableScope identifierRef =
  let declaration = semanticAnalysisResult.SymbolTable.[identifierRef]
  variableMappings.[declaration]
```

Given an identifier, this function finds its variable scope (a value indicating whether it is a field, argument or local, along with its index for the latter two).

``` ocaml
let makeLabel() =
  let result = labelIndex
  labelIndex <- labelIndex + 1
  result
```

Labels, at least in our IR, are really just integers. This helper function returns a unique label index.

``` ocaml
let rec processBinaryExpression =
  function
  | (l, Ast.ConditionalOr, r) ->
    let leftIsFalseLabel = makeLabel()
    let endLabel = makeLabel()
    List.concat [ processExpression l
                  [ Brfalse leftIsFalseLabel ]
                  [ Ldc_I4 1 ]
                  [ Br endLabel ]
                  [ Label leftIsFalseLabel ]
                  processExpression r
                  [ Label endLabel ] ]
  | (l, Ast.ConditionalAnd, r) ->
    let leftIsTrueLabel = makeLabel()
    let endLabel = makeLabel()
    List.concat [ processExpression l
                  [ Brtrue leftIsTrueLabel ]
                  [ Ldc_I4 0 ]
                  [ Br endLabel ]
                  [ Label leftIsTrueLabel ]
                  processExpression r
                  [ Label endLabel ] ]
  | (l, op, r) -> List.concat [ (processExpression l);
                                (processExpression r);
                                [ processBinaryOperator op ] ]
```

Finally, we're generating our actual IR. This function handles binary expressions. Note the `rec` keyword - this function, and several that follow it, call into each other, which naturally follows from the recursive nature of expressions.

I'll explain how "conditional or" expressions work; "conditional and" expressions are left as an [exercise for the reader](http://www.quickmeme.com/img/a3/a3fc3115766602b69975dd27c0c86d7d2ecc335eb97fbfaf3bdb3d5b6e5e6e98.jpg).

Here's an example of a "conditional or" expression in Mini-C:

``` c
bool a;
bool b;
bool c;

b = true;
c = false;

a = b || c; // The RHS here is the actual "conditional or" expression
```

When converting this to MSIL, we need to linearise it as follows:

* First, evaluate the LHS (`b`).
* If the LHS is true, we're done - push `1` onto the stack, which is how we represent `true` on the stack.
* If the LHS is false, we need to evaluate the RHS (`c`).

We can accomplish these steps using a combination of labels and branches, as follows:

``` ocaml
let leftIsFalseLabel = makeLabel()
let endLabel = makeLabel()
List.concat [ processExpression l          // 1. Evaluate the LHS
              [ Brfalse leftIsFalseLabel ] // 2. If the LHS is false, branch to leftIsFalseLabel
              [ Ldc_I4 1 ]                 // 3. If the LHS is true, push "1" to the stack
              [ Br endLabel ]              // 4. We're done - branch to endLabel
              [ Label leftIsFalseLabel ]
              processExpression r          // 5. Evaluate the RHS, which will leave a "1" or "0" on the stack.
              [ Label endLabel ] ]
```

``` ocaml
and processBinaryOperator =
  function
  | Ast.Add -> Add
  | Ast.Divide -> Div
  | Ast.Multiply -> Mul
  | Ast.Modulus -> Rem
  | Ast.Subtract -> Sub
  | Ast.Equal -> Ceq
  | Ast.Greater -> Cgt
  | Ast.GreaterEqual -> Cge
  | Ast.Less -> Clt
  | Ast.LessEqual -> Cle
  | _ -> failwith "Shouldn't be here"
```

The `processBinaryOperator` function converts binary operators from their AST to their respective MSIL opcodes. Implicit in this conversion process is that the stack-based CLR uses a reverse Polish notation instruction set, in which operators follow their operands.

``` ocaml
and processIdentifierLoad identifierRef =
  match lookupILVariableScope identifierRef with
  | FieldScope(v)    -> [ Ldsfld v ]
  | ArgumentScope(i) -> [ Ldarg i ]
  | LocalScope(i)    -> [ Ldloc i ]

and processIdentifierStore identifierRef =
  match lookupILVariableScope identifierRef with
  | FieldScope(v)    -> [ Stsfld v ]
  | ArgumentScope(i) -> [ Starg i ]
  | LocalScope(i)    -> [ Stloc i ]
```

This pair of functions generates the instructions necessary to load and store values associated with an identifier. We use different MSIL instructions for working with fields, arguments, and local variables. Arguments and locals are referred to by an index. When we're actually writing the MSIL out, we will refer to fields by a `FieldInfo` object; for now, we use an `ILVariable` as the IR version of that.

#### Mapping expressions from AST to IR

The two functions to convert expressions and statements to IR are the biggest of this stage. I'll break them into smaller chunks, instead of doing one big code dump.

``` ocaml
and processExpression expression =
  match expression with
  | Ast.ScalarAssignmentExpression(i, e) ->
    List.concat [ processExpression e
                  [ Dup ]
                  processIdentifierStore i ]
  ...
```

Our basic strategy for converting expressions to MSIL is this:

1. Convert the expression itself (recursively, because expressions can contain expressions).
2. Make sure that the thing left on top of the stack afterwards is the value of the expression.

For a scalar assignment expression (`i = 1`) we need to make sure that the value of the expression (`1`) is left on top of the stack, so that the expression can be chained (`j = i = 1`).

We do that with the `dup` MSIL instruction, which copies the current topmost item on the stack, and pushes the copy onto the stack. Then, `processIdentifierStore` generates, for example, a `stloc` instruction, which will pop one of the copies from the stack, leaving the other copy intact on top of the stack.

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.ArrayAssignmentExpression(i, e1, e2) as ae ->
    List.concat [ processIdentifierLoad i
                  processExpression e1
                  processExpression e2
                  [ Dup ]
                  [ Stloc arrayAssignmentLocals.[ae] ]
                  [ Stelem (typeOf (semanticAnalysisResult.SymbolTable.GetIdentifierTypeSpec i).Type) ]
                  [ Ldloc arrayAssignmentLocals.[ae] ] ]
  ...
```

Array assignments (`a[i] = b`) are somewhat more complicated than scalar assignments. We need to generate instructions to:

* Push the field, argument or local onto the top of the stack
* Process `e1`, which is the array index expression. We'll be left with the value of the expression on top of the stack, and "underneath" that will be the field, argument or local that we pushed in the previous step
* Process `e2` - after this, we'll have 3 items on the stack (or rather, 3 items relevant to this particular expression)
* Duplicate the value on top of the stack - i.e. the result of `e2`
* Pop the duplicate and store it in a "temp" local variable
* Pop the original and store it in the array. `Stelem` pops three items off the stack: the array, the array index, and the value to store. It takes as a parameter the type of the value to store.
* Push the "temp" local variable back to the top of the stack. This is how we make the RHS of this array assignment expression available to a calling expression (i.e. `c = a[i] = b`).

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.BinaryExpression(a, b, c) -> processBinaryExpression (a, b, c)
  | Ast.UnaryExpression(op, e) ->
    List.concat [ processExpression e
                  processUnaryOperator op]
  ...
```

We've already covered `processBinaryExpression`. There's no special reason for it to be in a separate function; it just made `processExpression` a bit large if it was all included inline.

Unary expressions are handled by first pushing the result of the expression on to the stack, and then the instruction representing the unary operation. Here is `processUnaryOperator`:

``` ocaml
and processUnaryOperator =
  function
  | Ast.LogicalNegate -> [ Ldc_I4 0; Ceq ]
  | Ast.Negate        -> [ Neg ]
  | Ast.Identity      -> [ ]
```

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.IdentifierExpression(i) -> processIdentifierLoad i
  | Ast.ArrayIdentifierExpression(i, e) ->
    List.concat [ processIdentifierLoad i
                  processExpression e
                  [ Ldelem (typeOf (semanticAnalysisResult.SymbolTable.GetIdentifierTypeSpec i).Type) ] ]
  ...
```

We've already looked at `processIdentifierLoad`. Array identifier expressions (`a[i]`) are handled by:

* Pushing the array onto the stack
* Pushing the array index expression onto the stack
* Calling `Ldelem`, which pops the array and array index from the stack, and pushes the value of the array at that index onto the stack

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.FunctionCallExpression(i, a) ->
    List.concat [ a |> List.collect processExpression
                  [ Call i ] ]
  ...
```

Function calls, as with most of the rest of MSIL, are handled in the reverse order to higher level code. First, we push all the arguments onto the stack, and then we issue the `call` instruction. The parameter to the `call` instruction is the name of the function to call; later, we'll map this to a `MethodInfo`.

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.ArraySizeExpression(i) ->
    List.concat [ processIdentifierLoad i
                  [ Ldlen ] ]
  ...
```

For array size expressions (`myArray.size`), we push the array onto the stack, and then call `ldlen`, which pops the array and pushes the array length onto the stack.

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.LiteralExpression(l) ->
    match l with
    | Ast.IntLiteral(x)   -> [ Ldc_I4(x) ]
    | Ast.FloatLiteral(x) -> [ Ldc_R8(x) ]
    | Ast.BoolLiteral(x)  -> [ (if x then Ldc_I4(1) else Ldc_I4(0)) ]
  ...
```

MSIL has separate instructions for loading each type of literal. We only need to use two:

* `ldc_i4` loads `System.Int32` values
* `ldc_r8` loads `System.Double` values

The .NET CLR doesn't have a representation of a boolean type on the stack, so we just use `int32` values: `1` for `true` and `0` for `false`.

``` ocaml
and processExpression expression =
  match expression with
  ...
  | Ast.ArrayAllocationExpression(t, e) ->
    List.concat [ processExpression e
                  [ Newarr (typeOf t) ] ]
```

Array allocations (`new float[2]`) are handled by:

* Pushing the result of the expression representing the size of the array, onto the stack
* Calling `newarr`, which pops the required array length from the stack and pushes the newly-created array.

#### Mapping statements from AST to IR

The next big function is `processStatement`, which maps statements from their AST representation to their intermediate representation. We'll break it down into (hopefully) more readable chunks.

``` ocaml
and processStatement =
  function
  | Ast.ExpressionStatement(x) ->
    match x with
    | Ast.Expression(x) ->
      let isNotVoid = semanticAnalysisResult.ExpressionTypes.[x].Type <> Ast.Void
      List.concat [ processExpression x
                    (if isNotVoid then [ Pop ] else []) ]
    | Ast.Nop -> []
  ...
```

Expression statements (which are, roughly, an expression with a semicolon on the end) have an interesting wrinkle. We have to be careful about the stack; we can't leave extra values lying around if they're not going to be used. An expression statement, by definition, doesn't return a value to anything else, so it shouldn't leave anything on the stack. We make sure of that by checking if the expression returns a value. If it does, then do an extra `Pop` to throw away the topmost item on the stack.

``` ocaml
and processStatement =
  function
  ...
  | Ast.CompoundStatement(_, s) -> s |> List.collect processStatement
  ...
```

Compound statements are easy; we just collect all the instructions generated by recursively calling `processStatement` for each of the statements in the compound statement.

``` ocaml
and processStatement =
  function
  ...
  | Ast.IfStatement(e, s1, Some(s2)) ->
    let thenLabel = makeLabel()
    let endLabel = makeLabel()
    List.concat [ processExpression e
                  [ Brtrue thenLabel ]
                  processStatement s2
                  [ Br endLabel ]
                  [ Label thenLabel ]
                  processStatement s1
                  [ Label endLabel ] ]
  | Ast.IfStatement(e, s1, None) ->
    let thenLabel = makeLabel()
    let endLabel = makeLabel()
    List.concat [ processExpression e
                  [ Brtrue thenLabel ]
                  [ Br endLabel ]
                  [ Label thenLabel ]
                  processStatement s1
                  [ Label endLabel ] ]
  ...
```

The two types of `if` statement are handled separately.

First, there's the type that has an `else` block. Whenever we're dealing with branching, we use labels as the target of branch instructions. We proceed as follows:

* Push the value of the `if` condition onto the stack
* If it's true, branch to `thenLabel` (we could equivalently implement `if` statements by using `brfalse` and inverting the logic; I don't know enough to know if there's a good reason for implementing it either way)
* If we're still "here", it means the condition evaluated to false, so process the `else` statement
* After the `else` statement, branch to `endLabel`
* If the condition was true, we will have branched to `thenLabel`, so process the `then` statement
* Either way, we end up at `endLabel`

There's also the type of `if` statement without an `else` block. This is handled very similarly.

``` ocaml
and processStatement =
  function
  ...
  | Ast.WhileStatement(e, s) ->
    let startLabel = makeLabel()
    let conditionLabel = makeLabel()
    let endLabel = makeLabel()
    currentWhileStatementEndLabel.Push endLabel
    let result = List.concat [ [ Br conditionLabel ]
                               [ Label startLabel ]
                               processStatement s
                               [ Label conditionLabel ]
                               processExpression e
                               [ Brtrue startLabel ]
                               [ Label endLabel ] ]
    currentWhileStatementEndLabel.Pop() |> ignore
    result
  ...
```

While statements are interesting. While statements can be nested, and if there is a `break` statement in the body, it needs to branch to the end of the correct while loop. If Mini-C had other types of loop (which it doesn't) the `break` statement would need to branch to the end of whichever type of loop it is immediately parented by.

This tracking of the current while loop is done with the `currentWhileStatementEndLabel` stack.

``` ocaml
and processStatement =
  function
  ...
  | Ast.BreakStatement ->
    [ Br (currentWhileStatementEndLabel.Peek()) ]
```

And here is where `currentWhileStatementEndLabel` is used - we branch to the `endLabel` of whichever while loop is "closest" to this break statement.

``` ocaml
and processStatement =
  function
  ...
  | Ast.ReturnStatement(x) ->
    match x with
    | Some(x) -> (processExpression x) @ [ Ret ]
    | None    -> [ Ret ]
  ...
```

Return statements are straightforward: if there's a value to return, then push it to the stack, and call `Ret`. Otherwise, call `Ret` directly. Note that MSIL methods must end with `Ret`, even if there's no value to return, otherwise you get a CLR exception - I know that because it took me a while to figure out why my executables were crashing, and that turned out to be the reason.

#### Collecting local declarations

Now that we've got functions to build an abstract representation of MSIL for our method bodies, let's turn our attention to local declarations. MSIL requires us to declare all required local variables at the start of each method. Mini-C, on the other hand, has two sources of local variables, one explicit and the other implicit:

* Mini-C allows local declarations within compound statements, and compound statements can be nested.
* Evaluation of array assignment expressions makes use of a temporary local variable, as discussed above. This local variable isn't in the Mini-C source code, but we need it to make our MSIL work correctly.

The following functions extract all the required local declarations from a function. They're actually only pulling local declarations out from the two sources just mentioned; the rest of these functions handle traversing down through the AST. (There's potential for separating out the tree traversal code from the actual operation being performed.)

``` ocaml
let processVariableDeclaration (mutableIndex : byref<_>) f d =
  let v = createILVariable d
  variableMappings.Add(d, f mutableIndex)
  mutableIndex <- mutableIndex + 1s
  v

let processLocalDeclaration declaration =
  processVariableDeclaration &localIndex (fun i -> LocalScope i) declaration
let processParameter declaration =
  processVariableDeclaration &argumentIndex (fun i -> ArgumentScope i) declaration

let rec collectLocalDeclarations statement =
  let rec fromStatement =
    function
    | Ast.ExpressionStatement(es) ->
      match es with
      | Ast.Expression(e) -> fromExpression e
      | Ast.Nop -> []
    | Ast.CompoundStatement(localDeclarations, statements) ->
      List.concat [ localDeclarations |> List.map processLocalDeclaration;
                    statements |> List.collect collectLocalDeclarations ]
    | Ast.IfStatement(e, s1, Some(s2)) ->
      List.concat [ fromExpression e
                    collectLocalDeclarations s1
                    collectLocalDeclarations s2 ]
    | Ast.IfStatement(e, s1, None) ->
      List.concat [ fromExpression e
                    collectLocalDeclarations s1 ]
    | Ast.WhileStatement(e, s) ->
      List.concat [ fromExpression e
                    collectLocalDeclarations s ]
    | Ast.ReturnStatement(Some(e)) ->
      List.concat [ fromExpression e ]
    | _ -> []

  and fromExpression =
    function
    | Ast.ScalarAssignmentExpression(i, e) -> fromExpression e
    | Ast.ArrayAssignmentExpression(i, e1, e2) as ae ->
      let v = {
        ILVariable.Type = typeOf ((semanticAnalysisResult.SymbolTable.GetIdentifierTypeSpec i).Type); 
        Name = "ArrayAssignmentTemp" + string localIndex;
      }
      arrayAssignmentLocals.Add(ae, localIndex);
      localIndex <- localIndex + 1s
      List.concat [ [ v ]; fromExpression e2 ]
    | Ast.BinaryExpression(l, op, r)      -> List.concat [ fromExpression l; fromExpression r; ]
    | Ast.UnaryExpression(op, e)          -> fromExpression e
    | Ast.ArrayIdentifierExpression(i, e) -> fromExpression e
    | Ast.FunctionCallExpression(i, a)    -> a |> List.collect fromExpression
    | Ast.ArrayAllocationExpression(t, e) -> fromExpression e
    | _ -> []

  fromStatement statement
```

We're almost there, at least for methods. Here is `ILMethodBuilder`'s only public method. It creates a record with a complete intermediate representation of the MSIL we're going to output in the next part.

``` ocaml
member x.BuildMethod(returnType, name, parameters, (localDeclarations, statements)) =
  {
    Name       = name;
    ReturnType = typeOf returnType;
    Parameters = parameters |> List.map processParameter;
    Locals     = List.concat [ localDeclarations |> List.map processLocalDeclaration;
                               statements |> List.collect collectLocalDeclarations ]
    Body       = statements |> List.collect processStatement;
  }
```

#### Building `ILClass`

So far, we've seen the `ILMethodBuilder` type, which can turn a function declaration from the AST into an `ILMethod`. Now, let's look at `ILBuilder`, which converts a whole program into its IR:

``` ocaml
type ILBuilder(semanticAnalysisResult) =
  let variableMappings = new VariableMappingDictionary(HashIdentity.Reference)

  let processStaticVariableDeclaration d =
    let v = createILVariable d
    variableMappings.Add(d, FieldScope(v))
    v

  let processFunctionDeclaration functionDeclaration =
    let ilMethodBuilder = new ILMethodBuilder(semanticAnalysisResult, variableMappings)
    ilMethodBuilder.BuildMethod functionDeclaration
```

`variableMappings`, as the name suggests, stores the mapping from `Ast.VariableDeclaration` to `ILVariableScope`. `processStaticVariableDeclaration` adds static variable declarations to this dictionary. `processFunctionDeclaration` is the entry point into `ILMethodBuilder.BuildMethod`.

`ILBuilder`'s only public method is `BuildClass`, which starts by creating hard-coded `ILMethod` objects for Mini-C's built-in methods. In a real language, you wouldn't want to do this. Instead, you'd provide a way to import classes and / or methods from arbitrary .NET namespaces. But Mini-C doesn't have any of that; the only functions you can call are either the ones you write yourself, or these four:

``` ocaml
member x.BuildClass (program : Ast.Program) =
  let builtInMethods = [
    {
      Name = "iread";
      ReturnType = typeof<int>;
      Parameters = [];
      Locals = [];
      Body = [ CallClr(typeof<System.Console>.GetMethod("ReadLine"))
               CallClr(typeof<System.Convert>.GetMethod("ToInt32", [| typeof<string> |]))
               Ret ];
    };
    {
      Name = "fread";
      ReturnType = typeof<float>;
      Parameters = [];
      Locals = [];
      Body = [ CallClr(typeof<System.Console>.GetMethod("ReadLine"))
               CallClr(typeof<System.Convert>.GetMethod("ToDouble", [| typeof<string> |]))
               Ret ];
    };
    {
      Name = "iprint";
      ReturnType = typeof<System.Void>;
      Parameters = [ { Type = typeof<int>; Name = "value"; }];
      Locals = [];
      Body = [ Ldarg(0s)
               CallClr(typeof<System.Console>.GetMethod("WriteLine", [| typeof<int> |]))
               Ret ];
    };
    {
      Name = "fprint";
      ReturnType = typeof<System.Void>;
      Parameters = [ { Type = typeof<float>; Name = "value"; }];
      Locals = [];
      Body = [ Ldarg(0s)
               CallClr(typeof<System.Console>.GetMethod("WriteLine", [| typeof<float> |]))
               Ret ];
    }
  ]

  ...
```

`ILClass` stores fields and methods separately, because that's what we're going to need when we turn our IR into real executable files. So let's separate those out:

``` ocaml
member x.BuildClass (program : Ast.Program) =
  ...

  let variableDeclarations =
    program
    |> List.choose (fun x ->
      match x with
      | Ast.StaticVariableDeclaration(x) -> Some(x)
      | _ -> None)
    
  let functionDeclarations =
    program
    |> List.choose (fun x ->
      match x with
      | Ast.FunctionDeclaration(_, _, _, _ as a) -> Some a
      | _ -> None)
```

And, *finally*, we can create an `ILClass` record, which wraps up our entire IR for a single Mini-C program:

``` ocaml
``` ocaml
member x.BuildClass (program : Ast.Program) =
  ...
  
  {
    Fields  = variableDeclarations |> List.map processStaticVariableDeclaration;
    Methods = List.concat [ builtInMethods
                            functionDeclarations |> List.map processFunctionDeclaration ];
  }
```

### Summary

Here's that example from the beginning, again; hopefully it makes more sense now. Putting together everything we've covered so far, we're able to take this Mini-C source code:

``` c
int a;
int main(bool b) {
  float c[];
  return 1;
}
```

... and turn it into this intermediate representation (IR):

``` ocaml
{ // ILClass
  Fields  = [ { Type = typeof<int>; Name = "a" } ];
  Methods = [
    { // ILMethod
      Name       = "main";
      ReturnType = typeof<int>;
      Parameters = [ { Type = typeof<bool>; Name = "b" } ];
      Locals     = [ { Type = typeof<single[]>; Name = "c" } ];
      Body       = [ IL.Ldc_I4(1); IL.Ret ];
    }
  ]
} 
```

The hard work is now behind us. Because the IR we chose is very close to MSIL, turning it into a real .NET executable is fairly straightforward. That will be the topic of the [next part](/blog/archive/2014/09/13/writing-a-minic-to-msil-compiler-in-fsharp-part-5-code-generation), which I hope will take less time for me to write than this one has! See you next time.