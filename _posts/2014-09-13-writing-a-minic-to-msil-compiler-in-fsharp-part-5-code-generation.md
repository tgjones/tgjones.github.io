---
title:   "Writing a MiniC-to-MSIL compiler in F# - Part 5 - Code generation"
date:    2014-09-13 15:30:00 UTC
excerpt: "Last year, I learnt F# by using it to write a compiler for a subset of C. In this series, I will explain how each part of the compiler works. This post describes creating .NET executable files by generating MSIL."
---

* This post is part of the series [Writing a MiniC-to-MSIL compiler in F#](/blog/archive/2014/04/13/writing-a-minic-to-msil-compiler-in-fsharp-part-0-introduction).
* You can find the code for this series in [the Mini-C GitHub repository](https://github.com/tgjones/mini-c/).

## Introduction

Let's do a quick recap. By the end of the last part on [building an intermediate representation](/blog/archive/2014/08/23/writing-a-minic-to-msil-compiler-in-fsharp-part-4-building-the-intermediate-representation), we were able to take source code like this:

``` c
int main() {
  return 123 + 456;
}
```

and turn it into an intermediate representation (IR) like this:

``` fsharp
{ // ILClass
  Fields  = [ { Type = typeof<int>; Name = "a" } ];
  Methods = [
    { // ILMethod
      Name       = "main";
      ReturnType = typeof<int>;
      Parameters = [];
      Locals     = [];
      Body       = [ IL.Ldc_I4(123); IL.Ldc_I4(456); IL.Add; IL.Ret ];
    }
  ]
} 
```

In this part, we'll focus on [code generation](http://en.wikipedia.org/wiki/Code_generation_(compiler)):

> In computing, code generation is the process by which a compiler's code generator converts some intermediate representation of source code into a form (e.g., machine code) that can be readily executed by a machine.

We'll be converting our intermediate representation into MSIL. Fortunately, our IR is already very similar to MSIL - a choice designed to make code generation easier.

Let's get started. The code for this part is in [CodeGenerator.fs](https://github.com/tgjones/mini-c/blob/master/src/MiniC.Compiler/CodeGenerator.fs) in the GitHub repository.

## Emitting MSIL

MSIL is a bytecode format - if you opened a .NET DLL or executable file in a binary viewer, all you'd see would be a long sequence of bytes. Nobody (presumably) wants to build their .NET applications by writing bytes.

The next level up is writing IL yourself. Microsoft provides a tool, `ilasm.exe`, which assembles IL into .NET bytecode (or a Portable Executable (PE), as it's called). If you were to write IL by hand for the trivial example program above, it would look like this (I have removed some items for brevity):

``` c
.assembly extern mscorlib
{
  .publickeytoken = (B7 7A 5C 56 19 34 E0 89 )
  .ver 4:0:0:0
}
.assembly TrivialExample
{
  .hash algorithm 0x00008004
  .ver 1:0:0:0
}
.module TrivialExample.exe

.class private auto ansi beforefieldinit TrivialExample.Program
       extends [mscorlib]System.Object
{
  .method private hidebysig static int32 
          Main() cil managed
  {
    .entrypoint
    .maxstack  2
    IL_0000:  ldc.i4 123
    IL_0001:  ldc.i4 456
    IL_0002:  add
    IL_0003:  ret
  }
}
```

Whenever a .NET language like C#, or Mini-C, is compiled, the compiler is essentially creating one of these IL files, and then assembling it into .NET bytecode. But to save you the trouble of building IL text files yourself, the .NET framework includes a nice API for generating IL, in the `System.Reflection.Emit` namespace. Instead of, for example, writing this line in an IL file:

```
ldc.i4 123
```

we can just call this method:

``` fsharp
ilGenerator.Emit(OpCodes.Ldc_I4, 123)
```

This is quite [meta](http://thats-so-meta.tumblr.com/): we're using a .NET API to generate (another) .NET application.

The main `System.Reflection.Emit` types that we'll need are:

* [ModuleBuilder](http://msdn.microsoft.com/en-us/library/system.reflection.emit.modulebuilder.aspx)
* [TypeBuilder](http://msdn.microsoft.com/en-us/library/system.reflection.emit.typebuilder.aspx)
* [MethodBuilder](http://msdn.microsoft.com/en-us/library/system.reflection.emit.methodbuilder.aspx)

## Generating MSIL methods

In the Mini-C compiler, at least, most of the hard work in code generation is expended in generating methods. So that's where we'll start. First, a couple of helpful type aliases:

``` fsharp
type MethodMappingDictionary = Dictionary<string, MethodInfo>
type FieldMappingDictionary = Dictionary<ILVariable, FieldInfo>
```

These will help us keep track of the generated versions of methods and fields.

We'll wrap all the code to generate methods into a `MethodGenerator` type (I'll discuss in the next part my feelings on my use of classes / types in this compiler. I think I'd put the pieces together differently 
if I was doing it now. I have been a C# developer for a long time, and I think it shows. There are often better ways of doing things in F#.)

``` fsharp
type MethodGenerator(typeBuilder : TypeBuilder, ilMethod : ILMethod,
                     methodMappings : MethodMappingDictionary,
                     fieldMappings : FieldMappingDictionary) =
  ...
```

The `ILMethod` object is the intermediate representation that we built in the last part.

First, we'll create a new `MethodBuilder`:

``` fsharp
let methodAttributes = MethodAttributes.Public ||| MethodAttributes.Static
let methodBuilder = typeBuilder.DefineMethod(ilMethod.Name, methodAttributes)
```

Later on, when we're generating the IL for a method call, we need to pass as the argument a `MethodInfo` object. To support that, we need to keep track of all the `MethodInfo` objects we've created for the functions in our program. That's what the `methodMappings` dictionary is for.

``` fsharp
do methodMappings.Add(ilMethod.Name, methodBuilder)
```

(Rather confusingly, the `*Builder` types inherit from the `*Info` types; i.e. `MethodBuilder` inherits from `MethodInfo`.)

``` fsharp
let ilGenerator = methodBuilder.GetILGenerator()
```

`ilGenerator` is the key object here; we'll use it to generate all the IL instructions for methods.

Next, we need some code to handle labels. In the IL examples above, the labels appear at the start of IL instructions (for example, `IL_0001:`). If you're writing IL by hand, you can give meaningful names to labels. We don't need to do that, but we do need to use labels as branch targets. For example, at the end of a `while` loop, we need to branch back to the beginning of the loop to test the condition again. The thing we're branching back to is a label.

We'll keep track of labels using another dictionary:

``` fsharp
let labelMappings = new Dictionary<IL.ILLabel, System.Reflection.Emit.Label>()
```

And then for a given `IL.ILLabel`, we'll create a corresponding `System.Reflection.Emit.Label` if we haven't already, and return it:

``` fsharp
let getLabel ilLabel =
  if labelMappings.ContainsKey ilLabel then
    labelMappings.[ilLabel]
  else
    let label = ilGenerator.DefineLabel()
    labelMappings.Add(ilLabel, label)
    label
```

Now we get to one of the core functions in this whole compiler - the function that actually generates IL instructions from the intermediate representation. In a real-world compiler, this function would be *much* larger. But in Mini-C, we can get away with using a small subset of the [available MSIL instructions](http://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.aspx).

``` fsharp
let emitOpCode (ilGenerator : ILGenerator) = function
  | Add        -> ilGenerator.Emit(OpCodes.Add)
  | Br(l)      -> ilGenerator.Emit(OpCodes.Br, getLabel l)
  | Brfalse(l) -> ilGenerator.Emit(OpCodes.Brfalse, getLabel l)
  | Brtrue(l)  -> ilGenerator.Emit(OpCodes.Brtrue, getLabel l)
  | Call(n)    -> ilGenerator.Emit(OpCodes.Call, methodMappings.[n])
  | CallClr(m) -> ilGenerator.Emit(OpCodes.Call, m)
  | Ceq        -> ilGenerator.Emit(OpCodes.Ceq)
  | Cge        -> ilGenerator.Emit(OpCodes.Clt)
                  ilGenerator.Emit(OpCodes.Ldc_I4_0)
                  ilGenerator.Emit(OpCodes.Ceq)
  | Cgt        -> ilGenerator.Emit(OpCodes.Cgt)
  | Cle        -> ilGenerator.Emit(OpCodes.Cgt)
                  ilGenerator.Emit(OpCodes.Ldc_I4_0)
                  ilGenerator.Emit(OpCodes.Ceq)
  | Clt        -> ilGenerator.Emit(OpCodes.Clt)
  | Dup        -> ilGenerator.Emit(OpCodes.Dup)
  | Div        -> ilGenerator.Emit(OpCodes.Div)
  | Label(l)   -> ilGenerator.MarkLabel(getLabel l)
  | Ldarg(i)   -> ilGenerator.Emit(OpCodes.Ldarg, i)
  | Ldc_I4(i)  -> ilGenerator.Emit(OpCodes.Ldc_I4, i)
  | Ldc_R8(r)  -> ilGenerator.Emit(OpCodes.Ldc_R8, r)
  | Ldelem(t)  -> ilGenerator.Emit(OpCodes.Ldelem, t)
  | Ldlen      -> ilGenerator.Emit(OpCodes.Ldlen)
  | Ldloc(i)   -> ilGenerator.Emit(OpCodes.Ldloc, i)
  | Ldsfld(v)  -> ilGenerator.Emit(OpCodes.Ldsfld, fieldMappings.[v])
  | Mul        -> ilGenerator.Emit(OpCodes.Mul)
  | Neg        -> ilGenerator.Emit(OpCodes.Neg)
  | Newarr(t)  -> ilGenerator.Emit(OpCodes.Newarr, t)
  | Pop        -> ilGenerator.Emit(OpCodes.Pop)
  | Rem        -> ilGenerator.Emit(OpCodes.Rem)
  | Ret        -> ilGenerator.Emit(OpCodes.Ret)
  | Starg(i)   -> ilGenerator.Emit(OpCodes.Starg, i)
  | Stelem(t)  -> ilGenerator.Emit(OpCodes.Stelem, t)
  | Stloc(i)   -> ilGenerator.Emit(OpCodes.Stloc, i)
  | Stsfld(v)  -> ilGenerator.Emit(OpCodes.Stsfld, fieldMappings.[v])
  | Sub        -> ilGenerator.Emit(OpCodes.Sub)
```

The only two that aren't direct mappings are `IL.Cge` (`>=`) and `IL.Cle` (`<=`). That's because these instructions don't exist in IL. So for `Cge`, for example, we instead:

* Compare using the less-than operator. If the first value is greater than or equal to the second value, this will push 0 onto the stack.
* Load the constant `0` onto the stack.
* If these two values are equal, then the first value must be greater than or equal to the second value.

(As an aside: I don't know why MSIL doesn't have a `Cge` instruction. Do any readers know?)

And finally, let's bring it all together:

``` fsharp
member x.Generate() =
  methodBuilder.SetReturnType ilMethod.ReturnType
  methodBuilder.SetParameters (List.toArray (ilMethod.Parameters |> List.map (fun p -> p.Type)))
        
  let defineParameter index name = 
    methodBuilder.DefineParameter(index, ParameterAttributes.In, name) |> ignore
  ilMethod.Parameters |> List.iteri (fun i p -> defineParameter (i + 1) p.Name)

  let emitLocal (ilGenerator : ILGenerator) variable =
    ilGenerator.DeclareLocal(variable.Type).SetLocalSymInfo(variable.Name)
  ilMethod.Locals |> List.iter (emitLocal ilGenerator)

  ilMethod.Body |> List.iter (emitOpCode ilGenerator)

  let rec last =
    function
    | head :: [] -> head
    | head :: tail -> last tail
    | _ -> failwith "Empty list."
  if (last ilMethod.Body) <> Ret then
    ilGenerator.Emit(OpCodes.Ret)
```

(Apparently there's some unwritten rule that you can't have a proper functional program without having a function somewhere in it that uses recursion to get the last item in a list. So I've obliged.)

The only wrinkle here is that if there isn't an explicit `return` at the end of the function in the original source code, we need to make sure we still add one to the generated code, otherwise our executable will fail runtime verification.

That's quite a whirlwind tour through IL generation. Obviously, we've only scratched the surface of what's possible, but hopefully this gives you a rough idea of what's involved.

## Generating MSIL types

We can now generate IL for each of the functions in our Mini-C programs. Next, we need to generate IL for fields, and bring it all together into a .NET type. Mini-C doesn't have types, but MSIL methods can only exist inside a type. We'll generate one type as a container for all the fields and functions in any given Mini-C program.

First, some code to generate fields, and store the mapping from `IL.Field` to `System.Reflection.FieldInfo` (when we refer to fields inside method IL instructions, we need to use the `FieldInfo` object):

``` fsharp
type CodeGenerator(moduleBuilder : ModuleBuilder, ilClass : ILClass, moduleName : string) =
  let fieldMappings = new FieldMappingDictionary()

  let generateField (typeBuilder : TypeBuilder) (ilField : ILVariable) =
    let fieldAttributes = FieldAttributes.Public ||| FieldAttributes.Static
    let fieldBuilder = typeBuilder.DefineField(ilField.Name, ilField.Type, fieldAttributes)
    fieldMappings.Add(ilField, fieldBuilder)

  ...
```

Next, the all-important method that creates a new type, generates the fields and methods for that type, and finally returns both the type and the `main` method that will act as the program entry point:

``` fsharp
type CodeGenerator(moduleBuilder : ModuleBuilder, ilClass : ILClass, moduleName : string) =
  ...
  
  member x.GenerateType() =
    let typeAttributes = TypeAttributes.Abstract ||| TypeAttributes.Sealed ||| TypeAttributes.Public
    let typeBuilder = moduleBuilder.DefineType(moduleName + ".Program", typeAttributes)

    ilClass.Fields |> List.iter (generateField  typeBuilder)

    let methodMappings = new MethodMappingDictionary()
    let generateMethod ilMethod =
      let methodGenerator = new MethodGenerator(typeBuilder, ilMethod, methodMappings, fieldMappings)
      methodGenerator.Generate()
    ilClass.Methods |> List.iter generateMethod

    (typeBuilder.CreateType(), typeBuilder.GetMethod("main"))
```

And that's pretty much it! It took me quite a lot of trial and error to get this working. Some bugs were hard to fix, because if you get IL generation wrong, your executable files can crash in odd and unhelpful ways. If you pop from the stack more times than you push, or if you don't have a `Ret` at the end of your methods, or... several other categories of error, you'll get an exception at runtime. I think I could have used [PEVerify](http://msdn.microsoft.com/en-us/library/62bwd2yd.aspx) to catch some of these errors without running the program, but I didn't know about that at the time.

We've almost reached the end of this little journey through a .NET compiler. In the [next and final part](/blog/archive/2014/09/14/writing-a-minic-to-msil-compiler-in-fsharp-part-6-conclusion), I'll show the compiler entry point (the function that actually takes Mini-C source code as a string, and compiles it into a .NET assembly). I'll also discuss how I would have done things differently with the benefit of hindsight. See you then!