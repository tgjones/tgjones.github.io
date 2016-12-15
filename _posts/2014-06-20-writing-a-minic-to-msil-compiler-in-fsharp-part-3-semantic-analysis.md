---
title:   "Writing a MiniC-to-MSIL compiler in F# - Part 3 - Semantic analysis"
date:    2014-06-20 09:00:00 UTC
excerpt: "Last year, I learnt F# by using it to write a compiler for a subset of C. In this series, I will explain how each part of the compiler works. This post describes the semantic analysis stage, including type checking and building a symbol table."
---

* This post is part of the series [Writing a MiniC-to-MSIL compiler in F#](/blog/archive/2014/04/13/writing-a-minic-to-msil-compiler-in-fsharp-part-0-introduction).
* You can find the code for this series in [the Mini-C GitHub repository](https://github.com/tgjones/mini-c/).

### Introduction

So far in this [series](/blog/archive/2014/04/13/writing-a-minic-to-msil-compiler-in-fsharp-part-0-introduction), we have covered the [abstract syntax tree](/blog/archive/2014/04/20/writing-a-minic-to-msil-compiler-in-fsharp-part-1-defining-the-abstract-syntax-tree) and [lexing and parsing](/blog/archive/2014/05/29/writing-a-minic-to-msil-compiler-in-fsharp-part-2-lexing-and-parsing). By the end of the previous part, we were able to take source code like this:

``` c
int main(bool b) {
  while (b)
    break;
  return 1;
}
```

and turn it into an abstract syntax tree like this:

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

In this part, we'll do some semantic analysis on this abstract syntax tree (AST). Semantic analysis refers to understanding the meaning of the code, as opposed to understanding the syntax (which is the job of the parser). Specifically, we will:

* Create a table of all symbols (i.e. names of variables) in the program.
* Create a table of all function calls in the program.
* Check that functions are only defined once.
* Check that types match (for example, check that the `return` statement in a function returns a type that matches the function declaration).
* Create a table of the types of all expressions in the program.
* Check for the existence of a `main` method.

The tables that we create will be used by later stages in our compiler pipeline.

The code for all this is in [SemanticAnalysis.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/SemanticAnalysis.fs) in the GitHub repo.

My last post was quite heavy on the theory, and also didn't have much F# code - I'll try to redress the balance in this post.

F# makes working with an AST - in fact, trees of any kind - really elegant. If I had written this compiler in C#, the code would have undoubtedly been much longer. Pattern matching, in particular, is an awesome feature, and (now that I know it exists) something I really miss when I code in C#.

Another F# characteristic that I liked when working on this compiler is that the order of F# files in a project is significant, and you can only use constructs that appear earlier in the list of files than where you're trying to use them. A compiler is basically a pipeline - source code comes in at one end, undergoes various transformations, and is spit out at the other end, in this case in the form of an executable .NET console application. Each stage in the pipeline only needs to know about the output of the stage before it. This works really well in F#, where each stage in the pipeline can be a file. Simply by looking at the ordered list of project files in Visual Studio, you immediately know in what order the pipeline executes - in other words, the file order reifies how the compiler itself works.

(That said, I'm not sure if I like the strict compilation order requirement when working on large object-oriented projects; it's certainly possible to workaround it, but it feels a bit like swimming upstream. The new [folder organisation](http://fsprojects.github.io/VisualFSharpPowerTools/folderorganization.html) feature in [Visual F# Power Tools](http://fsprojects.github.io/VisualFSharpPowerTools) will help, but it's only a partial solution. I'd love to know what F# experts do - maybe they just don't write large object-oriented codebases. I haven't seen any significant open source examples of such a thing.)

### Building the symbol table

The [symbol table](http://en.wikipedia.org/wiki/Symbol_table) is a structure that maps each identifier to its type. We'll need the symbol table both later on in semantic analysis, as well as the next stage of our compiler (building the intermediate representation).

I originally started off with F#'s [map](http://en.wikibooks.org/wiki/F_Sharp_Programming/Sets_and_Maps#Maps) type, but I found that creating the symbol table using only immutable data structures was hard work. So `SymbolTable` will subclass .NET's `Dictionary` type:

``` fsharp
type SymbolTable(program) as self =
  inherit Dictionary<IdentifierRef, VariableDeclaration>(HashIdentity.Reference)

  …
```

The keys in the symbol table are `IdentifierRef`s. This is a [record](http://en.wikibooks.org/wiki/F_Sharp_Programming/Tuples_and_Records#Defining_Records) type that we defined as part of the AST:

``` fsharp
let IdentifierRef = { Identifier : string; }
```

Records are reference types, not value types, but by default they use structural equality. That's no good here, because we might have more than one `IdentifierRef` instance with the same `Identifier` - for example, if a local variable `foo` is defined in more than one method. So when we're storing and fetching symbol types based on `IdentifierRef` instances, we want to use reference equality - that's what [HashIdentity.Reference](http://msdn.microsoft.com/en-us/library/ee353660.aspx) does for us.

The values in the symbol table are `VariableDeclaration`s, which are defined as:

``` fsharp
let VariableDeclaration = 
  | ScalarVariableDeclaration of TypeSpec * Identifier
  | ArrayVariableDeclaration of TypeSpec * Identifier
```

So our symbol table will map identifiers - i.e. names of static variables, parameters and local variables - to their corresponding `VariableDeclaration` objects. As a concrete example, for this program:

``` c
int a;
int main(bool b) {
  float c[];
  return 1;
}
```

our symbol table will look like this:

| Identifier | Variable declaration                           |
| ---------- | ---------------------------------------------- |
| `a`        | `Ast.ScalarVariableDeclaration(Ast.Int, "a")`  |
| `b`        | `Ast.ScalarVariableDeclaration(Ast.Bool, "b")` |
| `c`        | `Ast.ArrayVariableDeclaration(Ast.Float, "c")` |

To build the symbol table, we're going to need some helper functions and types. We'll define a type to store all the variable declarations for a single "scope", where scope means what it normally does in programming languages.

``` fsharp
type private SymbolScope(parent : SymbolScope option ) =
  let mutable list = List.empty<VariableDeclaration>

  let identifierFromDeclaration =
    function
    | ScalarVariableDeclaration(_, i)
    | ArrayVariableDeclaration(_, i) -> i

  let declaresIdentifier (identifierRef : IdentifierRef) declaration =
    (identifierFromDeclaration declaration) = identifierRef.Identifier

  member x.AddDeclaration declaration =
    let ifd = identifierFromDeclaration
    if List.exists (fun x -> ifd x = ifd declaration) list then
      raise (variableAlreadyDefined (identifierFromDeclaration declaration))
    list <- declaration :: list

  member x.FindDeclaration identifierRef =
    let found = List.tryFind (fun x -> declaresIdentifier identifierRef x) list
    match found with
    | Some(d) -> d
    | None ->
      match parent with
      | Some(ss) -> ss.FindDeclaration identifierRef
      | None -> raise (nameDoesNotExist (identifierRef.Identifier)) 
```

Let's walk through it:

* First, we declare an empty list of `VariableDeclaration` objects.
* Then there are a couple of helper functions: `identifierFromDeclaration` takes in a `VariableDeclaration` and returns its identifier.
* `declaresIdentifier` checks whether a given declaration declares an identifier.
* `AddDeclaration` adds a declaration to this scope. We first make sure that this identifier hasn't already been declared in this scope (our first bit of actual semantic analysis!).
* `FindDeclaration` returns the `VariableDeclaration` object for a given identifier. It recurses up through the parent symbol scopes, following standard language conventions for name lookup.

Next, we'll declare a `SymbolScopeStack` type to, as the name suggests, keep track of a stack of symbol scopes. For example, in this program, there would be four symbol scopes, as noted in the comments:

``` c
int a;             // 1. Global scope
int main(bool b) { // 2. Function scope
  float c[];       // 3. Implicit compound statement scope inside function
  {
    bool d;        // 4. Explicit compound statement scope inside function
  }
  return 1;
}
```

Here is `SymbolScopeStack`:

``` fsharp
type private SymbolScopeStack() =
  let stack = new Stack<SymbolScope>()
  do stack.Push(new SymbolScope(None))

  member x.CurrentScope = stack.Peek()

  member x.Push() = stack.Push(new SymbolScope(Some(stack.Peek())))
  member x.Pop() = stack.Pop() |> ignore
  member x.AddDeclaration declaration = stack.Peek().AddDeclaration declaration
```

Let's carry on with defining `SymbolTable`:

``` fsharp
type SymbolTable(program) as self =
  …
  let whileStatementStack = Stack<WhileStatement>()
  let symbolScopeStack = new SymbolScopeStack() 
```

We have to keep track of nested while statements, so that we know which while statement a `break` statement applies to.

`symbolScopeStack` will grow and shrink as we initialise the symbol table.

``` fsharp
type SymbolTable(program) as self =
  …
  let rec scanDeclaration =
    function
    | StaticVariableDeclaration(x) -> symbolScopeStack.AddDeclaration x
    | FunctionDeclaration(x)       -> scanFunctionDeclaration x 
```

Straightforward enough. When we hit a static variable declaration, add it to the current symbol scope. But we'll have to dig deeper into function declarations…

``` fsharp
type SymbolTable(program) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, parameters, compoundStatement) =
    let rec scanCompoundStatement (localDeclarations, statements) =
      symbolScopeStack.Push()
      localDeclarations |> List.iter (fun d -> symbolScopeStack.AddDeclaration d)
      statements |> List.iter scanStatement
      symbolScopeStack.Pop() |> ignore 
```

Compound statements (i.e. a group of statements nested inside curly braces) get their own symbol scope. We add each local declaration to that scope, and then scan each statement inside the compound statement. (Compound statements might be nested, so `scanStatement` might end up calling `scanCompoundStatement`, hence the `rec` keyword.) Note also that we're making use of F#'s ability to declare functions within functions - it's really nice for this sort of helper function that is only useful within its parent function.

``` fsharp
type SymbolTable(program) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, parameters, compoundStatement) =
    …
    and scanStatement =
      function
      | ExpressionStatement(es) ->
        match es with
        | Expression(e) -> scanExpression e
        | Nop -> ()
      | CompoundStatement(x) -> scanCompoundStatement x
      | IfStatement(e, s1, Some(s2)) ->
        scanExpression e
        scanStatement s1
        scanStatement s2
      | IfStatement(e, s1, None) ->
        scanExpression e
        scanStatement s1
      | WhileStatement(e, s) ->
        whileStatementStack.Push (e, s)
        scanExpression e
        scanStatement s
        whileStatementStack.Pop() |> ignore
      | ReturnStatement(Some(e)) ->
        scanExpression e
      | ReturnStatement(None) ->
        if functionReturnType <> Void then
          raise (cannotConvertType (Void.ToString()) (functionReturnType.ToString()))
      | BreakStatement ->
        if whileStatementStack.Count = 0 then
          raise (noEnclosingLoop()) 
```

Most of these are straightforward. We 'walk' down the AST, breaking each type of statement into its component parts. Declarations can only occur at the global scope, function scope, or compound statement scope, but usages can occur almost everywhere. When we encounter a declaration, we add it to the current symbol scope. When we encounter a usage, we check that a declaration with that name exists in the current symbol scope or any of its parents, and then add it to the symbol table.

The only thing we need to do for a `BreakStatement` is check that there is a parent `WhileStatement` to break out of. That's not really related to building a symbol table, so [SRP](http://en.wikipedia.org/wiki/Single_responsibility_principle) purists might balk at it, but… this is my compiler, I'll break SRP if I want to :-).

``` fsharp
type SymbolTable(program) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, parameters, compoundStatement) =
    …
    and addIdentifierMapping identifierRef =
      let declaration = symbolScopeStack.CurrentScope.FindDeclaration identifierRef
      self.Add(identifierRef, declaration) 
```

`addIdentifierMapping` adds an entry to the symbol table, where the key is an `IdentifierRef`, and the value is the declaration of that identifier. Because we set our symbol table up to use referential equality for `IdentifierRef` instances, we can have multiple usages of the same identifier, resulting in multiple entries in the symbol table.

``` fsharp
type SymbolTable(program) as self =
  …  
  and scanFunctionDeclaration (functionReturnType, _, parameters, compoundStatement) =
    …
    and scanExpression =
      function
      | ScalarAssignmentExpression(i, e) ->
        addIdentifierMapping i
        scanExpression e
      | ArrayAssignmentExpression(i, e1, e2) ->
        addIdentifierMapping i
        scanExpression e1
        scanExpression e2
      | BinaryExpression(e1, _, e2) ->
        scanExpression e1
        scanExpression e2
      | UnaryExpression(_, e) ->
        scanExpression e
      | IdentifierExpression(i) ->
        addIdentifierMapping i
      | ArrayIdentifierExpression(i, e) ->
        addIdentifierMapping i
        scanExpression e
      | FunctionCallExpression(_, args) ->
        args |> List.iter scanExpression
      | ArraySizeExpression(i) ->
        addIdentifierMapping i
      | LiteralExpression(l) -> ()
      | ArrayAllocationExpression(_, e) ->
        scanExpression e 
```

Here we scan each type of expression, looking for symbol usages. Many of these are recursive. A `ScalarAssignmentExpression` (for example, `i = j + 1`) would call `scanExpression` for the `j + 1` part, which would match the `BinaryExpression` pattern, which in turn would call `scanExpression` and match `IdentifierExpression` and `LiteralExpression` for each side of the binary expression, respectively.

Now that we've got all our `scanFunctionDeclaration` helper functions, we can write the actual implementation:

``` fsharp
type SymbolTable(program) as self =
  …  
  and scanFunctionDeclaration (functionReturnType, _, parameters, compoundStatement) =
    …
    symbolScopeStack.Push()
    parameters |> List.iter symbolScopeStack.AddDeclaration
    scanCompoundStatement compoundStatement
    symbolScopeStack.Pop() |> ignore 
```

We've almost finished `SymbolTable` - here are the final parts:

``` fsharp
type SymbolTable(program) as self =
  …  
  do program |> List.iter scanDeclaration

  member x.GetIdentifierTypeSpec identifierRef =
    typeOfDeclaration self.[identifierRef] 

```

We kick off the symbol table initialization by calling `scanDeclaration` with the top-level `program` object.

And we're done with `SymbolTable`! Now let's build a function table.

### Building the function table

The function table is going to serve a similar purpose to the symbol table. But where the symbol table mapped variable identifiers to variable declarations, the function table maps function calls to function declarations.

As usual, we need some helpers. We'll start with a record, and a related helper function:

``` fsharp
type VariableType =
  {
    Type    : TypeSpec;
    IsArray : bool;
  }
  override x.ToString() =
    x.Type.ToString() + (if x.IsArray then "[]" else "")

let typeOfDeclaration =
  function
  | Ast.ScalarVariableDeclaration(t, _) -> { Type = t; IsArray = false }
  | Ast.ArrayVariableDeclaration(t, _)  -> { Type = t; IsArray = true }
```

The `typeOfDeclaration` function builds a `VariableType` record from a variable declaration.

Next, we'll need a type to store as the "value" in the function table dictionary. We'll use another record:

``` fsharp
type FunctionTableEntry =
  {
    ReturnType     : TypeSpec;
    ParameterTypes : VariableType list;
  }
```

Finally, we can define the function table itself:

``` fsharp
type FunctionTable(program) as self =
  inherit Dictionary<Identifier, FunctionTableEntry>()

  let rec scanDeclaration =
    function
    | StaticVariableDeclaration(x)    -> ()
    | FunctionDeclaration(t, i, p, _) ->
      if self.ContainsKey i then
        raise (functionAlreadyDefined i)
      self.Add(i, { ReturnType = t; ParameterTypes = List.map typeOfDeclaration p; })

  do
    // First add built-in methods
    self.Add("iread",  { ReturnType = Int; ParameterTypes = []; })
    self.Add("iprint", { ReturnType = Void; ParameterTypes = [ { Type = Int; IsArray = false } ]; })
    self.Add("fread",  { ReturnType = Float; ParameterTypes = []; })
    self.Add("fprint", { ReturnType = Void; ParameterTypes = [ { Type = Float; IsArray = false } ]; })
    program |> List.iter scanDeclaration 
```

Unlike the symbol table, the function table uses value-equality. This is because Mini-C functions are global, and we don't need to differentiate between functions at different scopes. A function name *always* refers to the same function.

Because functions can only be declared at the global scope, `scanDeclaration` is quite simple. It protects against duplicate function declarations, and then adds the function declaration (with its corresponding return type and parameter types) to the function table.

Then there are some (hard-coded) built-in methods - `iread`, `iprint`, etc. These are available to every Mini-C program. In a real language, obviously, this hard-coding wouldn't be necessary. You'd be able to import, say,  the `System` namespace, and use the `Console` class. But Mini-C doesn't have namespaces or classes, so these hard-coded functions are the only way to communicate with the world outside.

To give a concrete example, for this program:

``` c
int a;
int main(bool b) {
  return 1;
}
```

our function table will look like this:

| Identifier | Function declaration                           |
| ---------- | ---------------------------------------------- |
| `main`     | `{ ReturnType = Ast.Int, ParameterTypes = [ { Type = Ast.Bool; IsArray = false } ] }` |

Armed with symbol and function tables, we can turn our attention to the next, and final, table: the expression types table.

### Building the expression types table

The expression types table maps expressions to expression types. For example, given this program:

``` c
int main() {
  int i = 2 + 3;
  return 1;
}
```

we'd end up with this expression types table:

| Expression | Expression type                           |
| ---------- | ---------------------------------------------- |
| `Ast.BinaryExpression(`<br>&nbsp;&nbsp;`Ast.LiteralExpression(Ast.IntLiteral(2)),`<br>&nbsp;&nbsp;`Ast.BinaryOperator(Ast.Add),`<br>&nbsp;&nbsp;`Ast.LiteralExpression(Ast.IntLiteral(3))`<br>`)` | `{`<br>&nbsp;&nbsp;`Type = Ast.Int;`<br>&nbsp;&nbsp;`IsArray = false`<br>`}` |
| `Ast.LiteralExpression(Ast.IntLiteral(1))` | `{`<br>&nbsp;&nbsp;`Type = Ast.Int;`<br>&nbsp;&nbsp;`IsArray = false`<br>`}` |

We initialize the expression types table by walking all the way through the tree, from top to bottom, looking at and storing the type of every expression. Let's get started:

``` fsharp
type ExpressionTypeDictionary(program, functionTable : FunctionTable, symbolTable : SymbolTable) as self =
  inherit Dictionary<Expression, VariableType>(HashIdentity.Reference)
  …
```

We're going to make use of the function table and symbol table that we made earlier.

``` fsharp
type ExpressionTypeDictionary(program, functionTable : FunctionTable, symbolTable : SymbolTable) as self =
  …
  let rec scanDeclaration =
    function
    | FunctionDeclaration(x) -> scanFunctionDeclaration x
    | _ -> () 

```

Nothing too exciting here; we handle function declarations, and ignore global variable declarations, because there are no expressions in global variable declarations (in Mini-C).

``` fsharp
type ExpressionTypeDictionary(program, functionTable : FunctionTable, symbolTable : SymbolTable) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, _, compoundStatement) =
    let rec scanCompoundStatement (_, statements) =
      statements |> List.iter scanStatement 
```

This structure might look familiar from the symbol table code. It is quite similar, but we're producing a different result. In C#, we might be tempted to reach for a [visitor](http://en.wikipedia.org/wiki/Visitor_pattern) object, in order to share the traversal code, but F#'s pattern matching makes that unnecessary, in my opinion.

Now the code to peek inside statements:

``` fsharp
type ExpressionTypeDictionary(program, functionTable : FunctionTable, symbolTable : SymbolTable) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, _, compoundStatement) =
    …
    and scanStatement =
      function
      | ExpressionStatement(es) ->
        match es with
        | Expression(e) -> scanExpression e |> ignore
        | Nop -> ()
      | CompoundStatement(x) -> scanCompoundStatement x
      | IfStatement(e, s1, Some(s2)) ->
        scanExpression e |> ignore
        scanStatement s1
        scanStatement s2
      | IfStatement(e, s1, None) ->
        scanExpression e |> ignore
        scanStatement s1
      | WhileStatement(e, s) ->
        scanExpression e |> ignore
        scanStatement s
      | ReturnStatement(Some(e)) ->
        let typeOfE = scanExpression e
        if typeOfE <> scalarType functionReturnType then
          raise (cannotConvertType (typeOfE.ToString()) (functionReturnType.ToString()))
      | _ -> () 
```

Apart from walking further down the tree, we do another semantic analysis check for return statements: the type of the expression that follows the `return` keyword must match the declared return type of the containing function.

Next, the biggest function so far: the function to extract an expression type from an expression. Because expressions can contain expressions, this is quite recursive in nature. Instead of doing one big code dump, I'll break it down into separate parts. Here is the basic structure:

``` fsharp
type ExpressionTypeDictionary(program, functionTable : FunctionTable, symbolTable : SymbolTable) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, _, compoundStatement) =
    …
    and scanExpression expression =
      let checkArrayIndexType e =
        let arrayIndexType = scanExpression e
        if arrayIndexType <> scalarType Int then
          raise (cannotConvertType (arrayIndexType.ToString()) (Int.ToString()))

      let expressionType =
        match expression with
        … // Lots of expression type patterns - see next section.
```

We'll use the `checkArrayIndexType` helper function later, to ensure that expressions used as array indices are of integer type.

#### Scanning expressions

Let's go through each expression type.

``` fsharp
| ScalarAssignmentExpression(i, e) -> // e.g. i = 1
  let typeOfE = scanExpression e
  let typeOfI = symbolTable.GetIdentifierTypeSpec i
  if typeOfE <> typeOfI then raise (cannotConvertType (typeOfE.ToString()) (typeOfI.ToString()))
  typeOfI
```

Here we recursively call `scanExpression` to get the type of the right-hand side of the assignment. Then, we lookup the identifier on the left-hand side in the symbol table. We take the opportunity to check that these two types match. Finally, we return the type of the left-hand side (which we now know is exactly as the type of the right-hand side).

``` fsharp
| ArrayAssignmentExpression(i, e1, e2) -> // e.g. j[i] = 3
  checkArrayIndexType e1

  let typeOfE2 = scanExpression e2
  let typeOfI = symbolTable.GetIdentifierTypeSpec i

  if not typeOfI.IsArray then
    raise (cannotApplyIndexing (typeOfI.ToString()))

  if typeOfE2.IsArray then
    raise (cannotConvertType (typeOfE2.ToString()) (typeOfI.Type.ToString()))

  if typeOfE2.Type <> typeOfI.Type then raise (cannotConvertType (typeOfE2.ToString()) (typeOfI.Type.ToString()))

  scalarType typeOfI.Type
```

Array assignments are a bit more involved than scalar assignments. We need to:

* check that the array index expression is of type integer
* check in the symbol table that the variable on the left-hand side is an array
* check that the expression on the right-hand side is a scalar (in Mini-C, you can't have multi-dimensional arrays)
* check that the type of the right-hand side matches the array type

``` fsharp
| BinaryExpression(e1, op, e2) -> // e.g. 1 + 2
  let typeOfE1 = scanExpression e1
  let typeOfE2 = scanExpression e2
  match op with
  | ConditionalOr | ConditionalAnd ->
    match typeOfE1, typeOfE2 with
    | { Type = Bool; IsArray = false; }, { Type = Bool; IsArray = false; } -> ()
    | _ -> raise (operatorCannotBeApplied (op.ToString()) (typeOfE1.ToString()) (typeOfE2.ToString()))
    scalarType Bool
  | Equal | NotEqual ->
    match typeOfE1, typeOfE2 with
    | { Type = a; IsArray = false; }, { Type = b; IsArray = false; } when a = b && a <> Void -> ()
    | _ -> raise (operatorCannotBeApplied (op.ToString()) (typeOfE1.ToString()) (typeOfE2.ToString()))
    scalarType Bool
  | LessEqual | Less | GreaterEqual | Greater ->
    match typeOfE1, typeOfE2 with
    | { Type = Int; IsArray = false; }, { Type = Int; IsArray = false; }
    | { Type = Float; IsArray = false; }, { Type = Float; IsArray = false; } ->
      ()
    | _ -> raise (operatorCannotBeApplied (op.ToString()) (typeOfE1.ToString()) (typeOfE2.ToString()))
    scalarType Bool
  | Add | Subtract | Multiply | Divide | Modulus ->
    typeOfE1
```

F#'s pattern matching again comes to the rescue (exercise for the reader: how much code would this take in C#?). Armed with a knowledge of how binary expressions work in C, hopefully this function makes sense. As an example, for the `Ast.Equal` (i.e. `==`), we make sure that both sides are scalar, both sides are the same type, and that type is not `Ast.Void`.

``` fsharp
| UnaryExpression(_, e) -> // e.g. -1
  scanExpression e
| IdentifierExpression(i) -> // e.g. i
  symbolTable.GetIdentifierTypeSpec i
| ArrayIdentifierExpression(i, e) -> // e.g. i[0]
  checkArrayIndexType e
  scalarType (symbolTable.GetIdentifierTypeSpec i).Type
```

We use the symbol table to lookup variable types. This is why we needed to make the symbol table use referential equality; if we keyed on a string, then the same variable name in various different scopes would resolve to the same type, which would be wrong. In Mini-C (and C), you can declare multiple variables with the same name, in different scopes.

``` fsharp
| FunctionCallExpression(i, a) -> // e.g. myFunc(1, "a")
  if not (functionTable.ContainsKey i) then
    raise (nameDoesNotExist i)
  let calledFunction = functionTable.[i]
  let parameterTypes = calledFunction.ParameterTypes
  if List.length a <> List.length parameterTypes then
    raise (wrongNumberOfArguments i (List.length parameterTypes) (List.length a))
  let argumentTypes = a |> List.map scanExpression
  let checkTypesMatch index l r =
    if l <> r then raise (invalidArguments i (index + 1) (l.ToString()) (r.ToString()))
  List.iteri2 checkTypesMatch argumentTypes parameterTypes
  scalarType calledFunction.ReturnType
```

Function calls are more interresting. We check:

* that the function exists
* that the number and type of the arguments in the function call match the number and type of the parameters in the function declaration

``` fsharp
| ArraySizeExpression(i) -> // e.g. myArray.size
  scalarType Int
| LiteralExpression(l) -> // e.g. 1
  match l with
  | BoolLiteral(b)  -> scalarType Bool
  | IntLiteral(i)   -> scalarType Int
  | FloatLiteral(f) -> scalarType Float
| ArrayAllocationExpression(t, e) -> // e.g. new float[2]
  checkArrayIndexType e
  { Type = t; IsArray = true }
```

`LiteralExpression`s are another of the leaves of the tree - instead of recursing, we return concrete scalar types.

Now we can finish writing the `scanExpression` function:

``` fsharp
type ExpressionTypeDictionary(program, functionTable : FunctionTable, symbolTable : SymbolTable) as self =
  …
  and scanFunctionDeclaration (functionReturnType, _, _, compoundStatement) =
    …
    and scanExpression expression =
      …      
      self.Add(expression, expressionType)

      expressionType 
```

And finally, we initialize the expression types table by calling `scanDeclaration` with the top-level program:

``` fsharp
do program |> List.iter scanDeclaration 
```

### Wrapping up

We're almost there! We're going to send some of the results of this semantic analysis to the next compiler stage. Let's declare a record for that purpose:

``` fsharp
type SemanticAnalysisResult =
  {
    SymbolTable     : SymbolTable;
    ExpressionTypes : ExpressionTypeDictionary;
  }
```

And last but not least, we'll define the entry point to this whole semantic analysis stage:

``` fsharp
let analyze program =
  let symbolTable   = new SymbolTable(program)
  let functionTable = new FunctionTable(program)

  if not (functionTable.ContainsKey "main") then
    raise (missingEntryPoint())

  let expressionTypes = new ExpressionTypeDictionary(program, functionTable, symbolTable)

  {
    SymbolTable     = symbolTable;
    ExpressionTypes = expressionTypes;
  }
```

Now we're starting to get somewhere. In addition to the AST, we've now got a symbol table, containing the type and scope of every variable used in the program. And we've got an expression types table, containing the type of every expression in the program. We're almost ready to generate some output. But instead of going straight from what we have now to writing out MSIL, we'll build up an intermediate representation. The intermediate representation will map very closely to MSIL, but will make it easier to build up the output in-memory, before writing out our final executable file.

Until [next time](/blog/archive/2014/08/23/writing-a-minic-to-msil-compiler-in-fsharp-part-4-building-the-intermediate-representation)!

(If you have any feedback about either the content or style of this series, please let me know in the comments!)