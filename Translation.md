# Translation and metaprogramming

WebSharper provides several ways to customize the way functions and
values are compiled to JavaScript:

* [Directly providing JavaScript code](#javascript);
* [Customizing the compiled name of the value](#name);
* [Transforming the F# code](#meta) during compilation, a concept
  known as metaprogramming.

<a name="javascript"></a>
## Embedding JavaScript

There are two ways of directly inserting JavaScipt code into a WebSharper project.

### JavaScript function body

The `Direct` attribute takes a JavaScript expression as a string parameter.
This will get parsed by the compiler and any syntax errors will be reported in a
compile error message.
This parsed expression will be used as a function body, `return` is automatically
added for the last value.

If you don't want the use function from .NET, the `X<'T>` type function in
`WebSharper.JavaScript` throws an exception of type `WebSharper.JavaScript.ClientSideOnly` with the message
"This function is intended for client-side use only.".

You can use placeholders for the function or method arguments.
For named parameters, the name with a `$` prepended is recognised.
For example:

    [<Direct "$x + $y" >]
    let add (x: int) (y: int) = X<int>

Also you can access parameters by index.
In let-bound functions in modules and static methods of classes, the parameters
are indexed from 0.

    [<Direct "$0 + $1" >]
    let add (x: int) (y: int) = X<int>
    
In instance methods, `$0` translates to the self indentifier, and method parameters
are indexed from 1.
(You can also use `$this` for the self identifier, but this recommended against, as
a parameter named `this` can override it, and it does not work for extension members
which are actually static methods in translated form.)

    [<Direct "Math.sqrt($0.x * $0.x + $0.y * $0.y)" >]
    member this.GetLength() = X<float>

### Inlined JavaScript code

The `Inline` attribute  takes a JavaScript expression as a string parameter.
(It can also be used together with the `JavaScript` attribute to inline a function
translated from F#.)
This will be parsed, and inlined to all call sites of this function.
Only a subset of JavaScript operators and keywords can be used which can be translated
to the "core" AST used internally by WebSharper to optimize output.

Parameter placeholders work exactly as with `Direct`. 

    [<Inline "$x + $y" >]
    let add (x: int) (y: int) = X<int>

## Inlines accessing global object

If you want a non-cached access to the global `window` object, use the `$global` fake variable inside inline strings.

For example, `[<Inline "myLibrary.doSomething()">]` assumes that `myLibrary` is initialized before the current script starts running (WebSharper itself takes care of this, if you use the `Require` attribute) and will not change, so access is safe to shorten.
On the other hand, `[<Inline "$global.myLibrary.doSomething()">]` always accesses `myLibrary` as a property of `window.` 

### Inline Helper

The `JS.Inline` function parses its first parameter at compile-time as JS code and includes
that in the result. It can contain holes, named `$0`, `$1`, ... and variable arguments will
be passed to the inline. Examples:

    open WebSharper.JavaScipt
    
    let zeroDate = JS.Inline("new Date()")
    let date = JS.Inline("new Date($0)", 1472226125177L)
	
### Constant

The `Constant` attribute takes a literal value as parameter.
It can annotate a property or a union case which will be translated to the literal provided.

<a name="name"></a>
## Naming

The `Name` attribute takes a string parameter and allows specifying
the name of a function or class in the translation.
For example:

    [<Name "add" >]
    let OriginalNameForAdd (x: int) (y: int) = x + y

<a name="naming-abstract"></a>
### Naming abstract members

If you set a fixed translated name with the `Name` attribute on an abstract member of a class
or interface, all inheriting and overriding members will have that exact translated name.
If a class is overriding or implementing two abstract members that has the same fixed name,
it will result in a compile-time error, and you have to change one of the fixed names to resolve it.

Automatically, interface members have a unique long translated name generated that contains the full type name.
This guarantees no conflicts without using the `Name` attribute.
If you want to shorten it for readability of the JS output and making it smaller,
you can use the `Name` attribute on the interface type to specify a short name for the interface.
It is recommended that it is unique across your solution.
You can also use `[<Name "">]` on the interface type to make all interface methods have the same translated name
as their original .NET name (if not specified otherwise by a `Name` on the member).

If you use the `[<JavaScipt false>]` attribute on an interface or abstract class, its member's naming will not be tracked by the compiler, this is sometimes useful for divergent behavior of the code in .NET and in JavaScipt.
	
<a name="meta"></a>
## Metaprogramming

Currently there are two ways of adding special logic to WebSharper translation.
The `Generated` attribute can be used to create a function's body by evaluating
an expression at compile-time.
The `Macro` attribute can be used to translate all call sites of a function with
a custom logic.

### Generated body

The `Generated` attribute takes a `System.Type` value as a parameter.
The type specified must have a constructor with no arguments and implement the
`WebSharper.Core.Macros.IGenerator` interface.

This interface has a single property called `Body` which must return a
`GeneratedBody` value, a union type of three cases: `QuotationBody`,
`CoreBody`, `SyntaxBody`.

With `QuotationBody`, you can return an F# quotation value on which WebSharper translation
will be used.
You must provide the quotation for the whole function as a lambda.
For example:

    type HelloQuotationGenerator() =
        interface IGenerator with
            member this.Body =
                <@@ fun w -> "Hello " + w + "!" @@>
                |> QuotationBody

    [<Generated(typeof<HelloQuotationGenerator>)>]
    let hello1 (w: string) = X<string>

With `CoreBody`, you can return an AST in the internal "core" language of WebSharper
which is used as an intermediate step for running optimizations between the F# quotation
and JavaScript syntax levels.
This expression type is found in the `WebSharper.Core.JavaScript.Core` module.
For example:

    module C = WebSharper.Core.JavaScript.Core

    type HelloCoreGenerator() =
        interface IGenerator with
            member this.Body =
                let w = C.Id "w"
                C.Lambda (None, [w], !~(C.String "Hello ") + C.Var w + !~(C.String "!"))
                |> CoreBody

    [<Generated(typeof<HelloCoreGenerator>)>]
    let hello2 (w: string) = X<string>

With `SyntaxBody`, you can return an AST for the JavaScript language, so all JavaScript
syntax is usable.
For example:

    module S = WebSharper.Core.JavaScript.Syntax
    module P = WebSharper.Core.JavaScript.Parser

    type HelloSyntaxGenerator() =
        interface IGenerator with
            member this.Body =
                S.Lambda (None, ["w"], 
                    [ S.Action (S.Return (Some (
                        !~(S.String "Hello ") + S.Var "w" + !~(S.String "!")))) ]
                )
                // equivalent to
                // "function(w) { return 'Hello' + w + '!'; }"
                //  |> P.Source.FromString 
                //  |> P.ParseExpression
            |> SyntaxBody

    [<Generated(typeof<HelloJSGenerator>)>]
    let hello3 (w: string) = X<string>

These 3 samples produce the same function body.

### Macros

WebSharper macros allow a custom translation of all call sites of a function between the
quotation and "core" levels.

The `Macro` attribute takes a `System.Type` value as a parameter.
The type specified must have a constructor with no arguments and implement the
`WebSharper.Core.Macros.IMacro` interface.

This interface has a single method called `Translate` which takes a code quotation and an
inner translator function.
The code quotation is of the type `WebSharper.Core.Quotations.Expression`, which is
similar in structure to the `Microsoft.FSharp.Quotations.Expr` type only it uses less
eager reflection of types involved so that WebSharper translation can be faster.
The macro must return a `WebSharper.Core.JavaScript.Core.Expression` value.
The provided function can be used to translate inner parts of the quotation.

Example:

    module Q = WebSharper.Core.Quotations

    type NameOfMacro() =
        interface IMacro with
            member this.Translate(q, _) =
                match q with
                | Q.CallModule (c, []) ->
                    match c.Generics with
                    | [t] -> !~(C.String t.FullName) 
                    | _ -> failwith "NameOfMacro error"
                | _ -> failwith "NameOfMacro error"

    [<Macro(typeof<NameOfMacro>)>]
    let nameof<'a> = X<string>

This macro analyzes the type argument of the type function call and translates to
the .NET name of that type.
So `nameof<string>` will appear as `'System.String'` in the JavaScript translation.
Here the inner translator is not needed.

Example for using inner translator:

    [<Sealed>]
    type AddMacro() =
        interface IMacro with
            member this.Translate(q, tr) =
                match q with
                | Q.CallModule (_, [a; b]) ->
                    match a, b with
                    | Q.Value (Q.Int ai), Q.Value (Q.Int bi) ->
                        !~ (C.Integer (int64 (ai + bi)))
                    | _ ->
                        tr a + tr b // equivalent to C.Binary(tr a, C.BinaryOperator.``+``, tr b)
                | _ -> failwith "AddMacro error"

    [<Macro(typeof<AddMacro>)>]
    let add (a: int) (b: int) = X<int>

This macro examines if both of its argument are a constant, in this case it adds 
the numbers compile-time, otherwise defaults to using the `+` operator, translating
the `a` and `b` argument expressions separately.

## Fallback

If you use the provided translator function in the quotation itself,
WebSharper will default to translating the call as it would without the Macro annotation.

Example:

    type AddMacro() =
        interface IMacro with
            member this.Translate(q, tr) =
                match q with
                | Q.CallModule (_, [a; b]) ->
                    match a, b with
                    | Q.Value (Q.Int ai), Q.Value (Q.Int bi) ->
                        !~ (C.Integer (int64 (ai + bi)))
                    | _ -> tr q
                | _ -> failwith "AddMacro error"

    [<Macro(typeof<AddMacro>)>]
    let add a b = a + b 

This `add` function works identical to the previous implementation, only
it translates to the addition by a fallback.