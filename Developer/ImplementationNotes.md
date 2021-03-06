# Introduction

This document describes the implementation of the Cobra compiler in just enough detail to make you a dangerous contributor. It does not go into exhaustive detail because the source code already has embedded docs, comments and, hopefully, well crafted identifiers. Additionally, the source should be fairly readable in most, but not all, places.

This document will give you some important facts about the structure of the implementation, some of the reasons for it, debugging tips and more.

This is primarily a plain text document. I don't feel like messing with HTML, LaTex or even CobraText for this one. There are some CobraTextisms used here and there. Paragraphs are not wrapped so your editor should do so.

It's not easy to talk about one part of the compiler without talking about the other parts. I've done the best I can, but don't be surprised if you have to read this more than once, preferably mixed with some dives into the code.

Terms:

- "CLR" means "Common Language Runtime", although I mostly use it as a shortcut for "Microsoft .NET or Novell Mono".
- "grok" sort of means "understand". Read "Stranger in a Strange Land" by Robert A. Heinlein in order to grok "grok". Then read all the other Heinlein books. And Frank Herbert. And Asimov. And Sagan. William Poundstone. Richard Feynman. Kurzweil. De Grey. ...

The Cobra docs are located at http://Cobra-Language.com/docs/. You should be familiar with the language to have any hope of understanding the compiler, so be sure you've browsed around there.

Also, reading the implementation is not the best way to learn the language. Reading the test cases isn't even the best way to learn the language. Read the How To's, the Samples and the documentation online. And write some code.

## Set Up

If you develop on the Cobra compiler itself then you should set the environment variable COBRA_IS_DEV_MACHINE to 1.

Here are commands that I often use when getting a new workspace going. There are some things here for me to do, like get the test cases imported to svn and push the repository up to a server.

    svn co file:///Users/chuck/Projects/Cobra/Repository/product/trunk Workspace-Foo
    cd Workspace-Foo
    mate .  # my text editor

    cd Source
    bin/build
    cobra hello
    cobra -build-standard-library
    cobra -ert:no hello
    bin/testify

On Windows systems, you should use bin\build.bat and bin\testify.bat instead.

The "testify" command runs the entire test suite. There are some test cases that currently fail on Windows. So if you see a few failures out of the greater than 590 test cases, that's currently normal.

The GTK and MySQL tests will obviously fail if you don't have those set up. But if you're not interested in them, don't worry about it.

If you'd like to just run the basic tests you can use:

    testify ..\Tests\100-basics

    # or on Unix-like systems:
    testify ../Tests/100-basics

Note that `-bsl` is an abbreviation for `-build-standard-library`.

To update the system:

    cd Workspace-Foo
    svn up
    cd Source
    bin/build
    cobra hello
    cobra -bsl
    cobra -ert:no hello

Or if you are already in the Source directory:

    cd ..
    svn up
    cd Source
    bin/build
    cobra hello
    cobra -bsl
    cobra -ert:no hello

The -turbo option excludes all contracts, asserts, nil checks and unit tests. It also turns on optimization. If you're building Cobra just to have the latest (as opposed to doing development) you may wish to build with this option:

    bin/build -turbo
    cobra -bsl -turbo

## Types

Types include:

    Primitive Types
        bool
        int
        char
        etc.
    Boxes
        Class
        Interface
        Struct
    Enums
    Wrapped Types
        Nilable
        Array
        Vari
        Out (future)
        InOut (future)
        Pointer (future)

Cobra has its own set of classes for representing types rather than using System.Type directly. The reasons?

1. It keeps the door wide open on other editions of Cobra such as Cobra/JVM and Cobra/Native.
2. Cobra has at least one additional special type, NilableType, which System.Type cannot represent.
3. System.Type is retarded for mushing all types into one class. I owe you one "detailed rant".

All types implement IType. The CobraType class is an abstract ancestor of many of the Cobra type classes, but it is not a requirement. Only IType conformance is required.

ITypeProxy is the interface for anything that represents a type which can be determined at a later point:

    interface ITypeProxy
        inherits INode        

        get realType as IType

So the proxy does not take action in place of the type. It is not indefinite, but temporary. It will be asked for its .realType and then largely ignored afterwards.

The most common kind of proxy, which was also the main motivation for creating ITypeProxy, is AbstractTypeIdentifier (or rather its descendants), which are produced by the parser. The Cobra language, like many modern languages such as C#, allows references to types that are located in other files, or declared further down in the same file, or that create cycles (example: B inherits A which has a variable of type B). Consequently, the parser often creates "type identifiers" instead of actual types. These type identifiers always descend from AbstractTypeIdentifier and always implement ITypeProxy.

So reversing that order:

1. The parser often cannot determine types.
2. So it produces "type identifiers" which are objects that
3. implement ITypeProxy.
4. Later the proxies will resolve to types when asked for their .realType

As a minor convenience, CobraTypes implement ITypeProxy and return `this` for .realType. Just so you can pass a type where an ITypeProxy is expected. Just in case you need to or like to.

## Boxes

Classes, interfaces and structs are major players in the CLR universe. They have much in common. They can all contain properties and methods. They can implement any number of interfaces. They are considered to be descendants of System.Object whether directly or indirectly.

But they also have lots of differences. Classes and structs have initializers (a Cobra term; CLR calls them "constructors"), but interfaces do not. Classes can inherit classes, but interfaces and structs cannot. And so on.

The compiler has classes to represents these things. And they have a common base class to capture what they have in common:

    class Box
        class Class
        class Interface
        class Struct

So the term "box" is used to refer to "class or interface or struct". It is used in the source code in comments, embedded docs and even identifiers such as `parentBox` and `BoxMember`.

Why "box"? Because I couldn't think of anything else that was short and sweet. So its just an arbitrary, but useful name.

## Phases

The compiler divides its work into major phases. All phases except code generation may result in the detection of a user error (such as bad command line option, bad syntax, unknown type, bad semantics, run-time error, etc.). An error in any phase halts the process. Hence, any phase can assume that previous phases completed successfully.

The phases are:

- Process command line args
- Tokenize
- Parse
- Bind use
- Bind inheritance
- Bind interface
- Bind implementation
- Run

Of course, if the command line options indicate something else, such as "-h" for "display help", then the other phases do not execute. Also, if "-c" or "-compile" is passed then the "Run" phase is skipped. Also, once the Run phase starts, a new process is launched and the Cobra program now has control--the compiler is out of the picture at that point. (Its effects may still be felt in terms of its code generation, and a program may invoke the compiler like it might invoke any other program.)

But in a typical "cobra foo.cobra" invocation, the phases will be as above.

## Compiler object

There is a compiler class which is instantiated by the CommandLine class and told to do its business. It is the compiler that manages the phases listed above. It does not, for example, parse command line args or print command line help.

The compiler also houses the types. For example, you will expressions in the source code such as .compiler.intType and .compiler.boolType where .compiler is a property of Node which gives access to the compiler. (From a decoupling perspective, it may be useful to "hide" the compiler from the nodes and have interface-based properties like .typeProvider.boolType (or .types.boolType) and .options).

### Process command line args

See CommandLine.cobra. The arguments are driven by so-called "metadata" which describes the config options. The descriptions are interpreted when parsing the command line options and when printing help.

If you're looking for the .main function, it resides in "cobra.cobra" and is rather small as it mostly just hands control off to an instance of CommandLine. It also has a last ditch exception handler and implements the -timeit command line option to make it as accurate as possible.

### Tokenize phase

The tokenizer slurps up the raw text of the source code and spits out ITokens for consumption by the parser.

The tokenizer is well organized and makes good use of CLR regular expressions. But it also turned up as a major performance bottleneck when profiling the compiler with the Red Gate ANTS profiler. I doubt that much more speed can be squeezed from the current design. It may be necessary to rewrite the tokenizer either by hand or using a tokenizer-generator.

### Parse phase

The parser consumes the tokens and outputs abstract syntax tree nodes, also called "AST nodes", also called "nodes". The file Node.cobra defines the base interfaces and classes for nodes. For typing purposes, the rest of the source code should be referring to interfaces such as INode, ISyntaxNode, ITypeProxy and IType with the exception of Stmt and Expr which are classes (and of course, the exception of having something specific to refer to!).

The parser is a top-down, recursive-descent parser which I'm pretty happy with. I like writing parsing logic in Cobra and will not be using a parser-generator tool.

The top level objects that the parser produces are modules.

The modules in turn contain namespaces (even if they are global namespaces) which in turn contain declarations such as classes and subnamespaces. All of these--modules, namespaces and classes--are nodes.

Node.cobra lays down the high level interfaces and ancestor classes such an INode, Node, ISyntaxNode and SyntaxNode. The syntax nodes simply retain a token so they know where they came from (example: file Foo.cobra, line 12, column 10).

Getting back to modules there are multiple types:

- CobraModule for Cobra source files (foo.cobra)
- AssemblyModule for DLLs (Foo.Bar.dll)
- SharpModule for C# source files (foo.cs)

.sidebox

C# source files??? Yes. Cobra has a little trick where you can include C# code with your Cobra code and compile it into one big happy family:

    cobra foo.cobra bar.cs

Since Cobra does not grok C# you must tell it what's in there with a "fake" class. This is analagous to ANSI C function prototypes, but for classes:

    class Utils
        is fake
        def doSomething(s as String, i as int)
            pass

And your C# class better match. This feature is an escape hatch if you need to do something that you cannot do in Cobra, but using this is fairly rare for the following reasons:

- Cobra is maturing all the time so the list of things you cannot do is shrinking all the time.
- You can embed C# *expressions* in your code with $sharp('blah blah'). See the docs.
- You could use a DLL!

## Bind use phase

Cobra embraces CLR namespaces and like any CLR language must provide a statement to access them:

    use System.Drawing

This is equivalent to C#'s "using" statement at the global level (not the C# "using" statement at the method body level which has the same name, but an *entirely* different purpose!).

So this is the first of the "bind foo phases" where "foo" is use, inheritance, interface or implementation. These "bind" phases execute by the compiler telling all the modules to perform the binding and the modules telling their namespaces and the namespaces telling their declarations and so on. In other words, the compiler does this:

    for module in _modules
        module.bindUse

And then `bindUse` cascades top-down from there. The compiler also detects errors so it will refrain from subsequent phases. There is also some state management like asserting that the previous phase took place and setting .isBindingFoo to true in case nodes would like to know that.

In the case of `bindUse`, it doesn't have to cascade very far. Only modules and namespaces need it. But the Compiler doesn't know or care. It tells the modules what to do and after that everything is "self responsibility" in the classic OO fashion.

So what did .bindUse actually *do*? It found the references to actual nodes that match the use directive. Given the example, somewhere among all the nodes there is a "System.Drawing" namespace node (implemented by a NameSpace node). Now wherever there is a `use System.Drawing` statement (implemented by a UseDirective node), there is a reference to that namespace. Now it can be used when symbols are looked up in later phases. Neat!

How was the namespace found? Wow, you just won't quit with the questions. The compiler has a global namespace accessed via .globalNS. It's like The Ring in LOTR: "One Ring to rule them all, One Ring to find them, One Ring to bring them all and in the darkness bind them." The global namespace brings *everything* together. If you are a global declaration including a top level namespace, .globalNS has you.

But not modules. They are an abstraction for the compiler and do not represent anything that programmers explicitly declare except for the fact that programmers put their code into files. In Cobra, as in the CLR, namespaces are independent of files. Having come from Python I didn't initially dig it, but after spending some serious time with CLR, I dig it.

### Bind inheritance phase

Just before this phase, the compiler has tokenized, parsed, produced nodes and connected the `use` directives to their respective namespaces.

TODO

### Bind interface phase

TODO

### Bind implementation phase

TODO

During this phase, avoid storing "sharp" values such `someType.sharpRef`. While C# is Cobra's current backend, there will likely be other backends in the future including CIL, JVM, Objective-C, etc. Instead store a reference to the type or other object and then access its "sharp" properties during C# code gen (typically in a .writeSharpDef method implementation).

### Code generation phase

TODO

The code generation is currently C#.

When a statement needs to create a private local variable to help with its implementation, there are some conventions to follow which prevent problems. First, start them with an underscore (_) which will eliminate any possibility of collision with explicit locals (Cobra does not allow locals to start with an undescore). Also, use "_lh_" for local helper which reduces chance collisions with object variables. Also, add the serial number of the current node to further prevent collisions including with other local helper variables. For example:

    def writeSharpDef(sw as SharpWriter)
        # ...
        localName = '_lh_event_[.serialNum]'

.writeSharpDef means "write C# definition" and is the main "driver" of spewing out the C# source code. The various properties such as .sharpRef are used, potentially from other places like when a .writeSharpDef implementation invokes _expr.type.sharpRef.

An implementation of .writeSharpDef will invoke that same method name on its subobjects.

.sharpRef means "a C# reference to the receiver" which often means a qualified name (Foo.Bar.Baz), possibly even prefixed with "global::", so C# doesn't get confused.

Other .sharpFoo properties have various purposes and you'll have to poke around to see how they're used.

If you have to refer to something in the generated code (as opposed to writing out its definition), default to .sharpRef unless you have an explicit reason not to.

### Run phase

TODO

## More key classes

### Container

I could have introduced containers earlier, but I wanted the term "box" to have a chance to settle in. Boxes aren't the only nodes that contain other nodes. NameSpaces and EnumDecls also contain other stuff. So to capture this idea that nodes contain other nodes and the code they must consequently have in common, there is Container. It is the parent class of Box, NameSpace and EnumDecl. It is the grandparent of Box's subclasses, Class, Interface and Struct.

TODO: describe Container being generic

### Stmt

TODO

### Expr

TODO

### Misc Topics

### Finding Members

Box.declForName does not follow inheritance. It does not trigger any "late initialization" code.

Box.memberForName does follow inheritance. It *does* trigger additional initialization (when its required).

"decl" stands for "declaration" and it means that only what's actually declared in the Box should be examined.

"member" means any kind of member from any source.

Both methods return nil if they find nothing.

### Overloads

When members for a box are first being accumulated either during parsing or scanning a DLL, you will see that MemberOverloads are being created to handle the name collisions. For example, there may be two methods named `compute` and these have to be rolled into a single MemberOverload because Boxes store a dictionary that maps names to members.

You might also notice that .declForName is used to find members that have the same name as the currently parsed/scanned member even though (a) it doesn't follow inheritance relationships and (b) overloads cross the inheritance boundary. But that's okay: Overloads are "finished up" at a later time in Box._finishOverloads. Also, you generally want to avoid the inheritance-savvy .memberForName in early stages because it triggers other things.

### Type identifiers, proxies and actual types

- Interface ITypeProxy is small. Go check it out now.
- The parser creates "type identifiers"
  - That is, the parser creates objects that inherit from AbstractTypeIdentifier.
  - These have a token and will resolve themselves to a type, or generate a compilation error.
- Sometimes the compiler needs to create type proxies outside of parsing.
  - There are type proxy classes for this.
  - They have no token (so no source code information).
- Then there are actual types. And these implement ITypeProxy and simply return `this` for .realType.
- Do not mix types with type proxies though.
  - For example, a "wrapper type" like VariType or NilableType should not have a reference to a type proxy, only a type.
  - So if you have a type proxy and you need to build it into something nilable
    - Use NilableTypeIdentifier if you have a real token,
    - or use NilableProxy if you don't.
- IType and ITypeProxy are used to refer to these things at a very high level.
  - For example, Expr has `get type as IType?`
  - And Param has, among other intializers, a `def init(token as IToken, typeProxy as ITypeProxy)`

## It's tricky (to rock a rhyme)

Here I cover things than I found tricky or that others may have expressed uncertainty about. Of course, you might skip this material until you need it one day (when enhancing the compiler), or you might read it because you are interested in tricky stuff.

### Namespaces

Namespaces can span modules. You can even put your own declarations in the System namespace without having access to the system implementation source code:

    namespace System
    
        class Foo
            def bar
                print 'bar'

You might do this to simulate having a class that you know will be present in a future version of the CLR. But most often you will use namespaces because you don't want to put every single class/interface/whatever in one file, so you have multiple files, each with a "namespace MyCompany.MyProject".

It gets trickier. The `use` directive can be put inside a namespace like so:

    namespace MyCompany.MyProject
    
        use MyCompany.Utils
        
        ...

*But* this `use` directive only affects the source code that follows it in this file. It does not affect all the other "MyCompany.MyProject" namespaces in the other files. That is unique because everything else about a namespace *is* merged together. For example, if file A.cobra has namespace MyCompany.MyProject containing class A, and file B.cobra has namespace MyCompany.MyProject containing class B, then when compiling "cobra A.cobra B.cobra" the namespace will have two classes, A and B. However, a `use MyCompany.Utils` in A.cobra will not affect symbol lookup in B.cobra.

How does Cobra keep this all straight?

- A namespace has declarations like any Container descendant does,
- but UseDirectives are kept separate as they are not really declarations.
- Each namespace encountered in a file results in instantiating a NameSpace with that name.
- A corresponding "unified" NameSpace with the same name is created or located in the global namespace of the compiler.
- The UseDirectives are located in the non-unified NameSpace.
- All other declarations are added to both NameSpaces.
- Finally, symbol lookup uses the unified namespace first and then falls back to the use directives of the non-unified namespace if further searching is required.

Declarations can reside in a file with no declared namespace. This is implicitly the global namespace. Here's how it works: Each module has a .topNameSpace which may have `use` directives and declarations. Its .unifiedNameSpace is .compiler.globalNS. The machinery described above then simply works. And that is why modules don't directly track their declarations--they are all contained in the module.topNameSpace which is the only "AST node" the module directly owns.

Why not axe modules and just use namespaces directly? Because there are different types of modules such as SharpModule and AssemblyModule. However, it's notable that if SharpModule got the axe it might be feasible to go directly to NameSpace or to at least have CobraModule and AssemblyModule *be* a NameSpace (inherit) instead of *have* one.

### Generics

Generics are a great feature. They offer numerous benefits and they're fairly easy to use.

Implementing them is a pain in the ass.

Of course, implementing anything is generally harder than using it, but generics really stand out as a form of torture for compiler writers.

In particular, when implementing generics...

What I mean is that...

I've upset myself now and cannot continue.

### Overloads

Not as bad as generics, but they've been good for hiding bugs for months at a time only to interrupt compiler work later on.

TODO

### Self-hosting

The Cobra compiler is written in Cobra. Which came first? The chicken or the egg?

I like the term "self-implemented" for this phenomena, but in reading I most often see "self hosting". There's even a Wikipedia page at http://en.wikipedia.org/wiki/Self-hosting

I find self hosting pretty straightforward, but sometimes developers freeze up on me when I mention it. The key is that you get to self-hosting over time by taking steps. In the case of Cobra, the very first step was to write a Cobra compiler in Python (the CPython flavor as IronPython was unfortunately not mature yet). It sucked up Cobra source code, analyzed it (much as described above) and generated C#.

I then wrote the compiler a second time in Cobra often returning to the original Python-based compiler to add features and fix bugs as this task was a serious test of the language design and compiler.

Eventually, the Cobra-based compiler could pass all the unit tests just like the Python-based one had. Yay!

But it couldn't compile itself! And despite being a compiler of Cobra source code and written in Cobra! Hiss! The attempt to self-compile met with false error messages, interal exceptions and/or improper code generation that C# choked on. What this revealed was that the unit tests were incomplete. Consequently, more unit tests were added and bugs fixed until all those problems went away and the compiler could compile itself. The first time this happened was very emotional for me (not joking here).

But it was short-lived as a special moment. While everything appeared fine due to having produced another cobra.exe, _that_ cobra.exe would fail on the Cobra compiler source code. I never expected that and fixing it _was_ tricky. Some investigation and really deep thought eventually cracked the problem. Essentially, the code generation of the new compiler was incorrect in certain cases that were not covered by the unit tests, but did come up in the compiler itself. And it was really bad: The incorrect code _did_ compile and execute but did not behave correctly at run-time. Nor did it throw an exception at run-time so I could diagnose the problem right where it occurred. This bug just caused bad behavior in select cases and required a serious stare down to crack.

These days it's hard to get into that situation because the unit tests are over 480 and rising. When all the unit tests pass, you are usually in good shape. To get into bad shape, add a new feature to Cobra and then start using it in the compiler before its been thoroughly tested.

So at any point in time now, there is a cobra.exe living in CobraWorkspace/Source/Snapshot which is used to compile CobraWorkspace/Source/*.cobra to produce a cobra.exe in CobraWorkspace/Source that is then tested and debugged. At various points in time when there is a nifty new feature or an important bug is fixed, a new Snapshot is made by copying cobra.exe (and related files such as CobraLang.cobra) into the Snapshot directory. A little script called "make-snapshot" is used for this.

The Snapshot directory is under Subversion control and the new snapshot is checked in. Consequently, when you check out a fresh Cobra workspace, there is a stable Snapshot/cobra.exe ready for your use! (Well, provided you have installed Microsoft .NET or Novell Mono so that you can run it.) In fact, you will often need to use it because it doesn't take long after a release until a new feature or bugfix is taken advantage of in the source code thereby rendering it uncompilable by the _last public release_.

Now to avoid the occasional and subtle problem of a bad compiler produced by the compiler creeping into the Snapshot/ directory, some precautions are taken. First, the new cobra.exe (having passed all unit tests) is copied to cobra-trial.exe in the same directory. Then cobra-trial.exe is applied to the Cobra compiler source code. This is a "mega test" after the hundreds of unit tests. When there is a problem, it is recreated in a small unit test and then fixed. After there are no problems, the new cobra.exe produced by cobra-trial.exe is applied to all unit tests. Then I sometimes repeat the process just one more time (copy cobra.exe to cobra-trial.exe, compile the compiler source, run all unit tests). Finally, I run 'make-snapshot', work with that for awhile and then check it in.

## Misc Notes

### Build the standard library, AKA Cobra.Lang.dll

To make Cobra.Lang.dll:

    cobra -d -build-standard-library

### The Ultimate Test Case

(Ug, I already described this above. Well I'll merge later.)

After the compiler passes the test suite which consists of hundreds of test cases, it is now ready for the ultimate test case: the compiler itself.

At the command line, I enter "make-trial" which is just this little script:

    cp cobra.exe cobra-trial.exe
    echo 'You can now run comp-trial to compile the compiler source code with a trial snapshot of the new compiler.'

Then I enter "comp-trial" (as opposed to the usual "comp") which invokes the fresh cobra-trial.exe instead of Snapshot/cobra.exe.

If this fails then not only has a new bug been introduced to the compiler, but a weakness has been discovered in the test suite. Isolate the problem down to a test case, fix the problem, put the test case in with the other test cases and try again (testify; make-trial; comp-trial).

Upon success, run "testify" again to see that a compiler created with the new compiler still passes the test suite. On some occasions this will fail! Such a failure is probably due to changes in code generation which then affect run-time behavior in subtle ways. Take two aspirin and review your diffs--you're about to have loads of fun.

## Debugging the compiler

### Get a detailed stack trace

Given that Cobra is implemented in Cobra, you can use its own tools on itself. For example, if you're having trouble diagnosing an uncaught exception then recompile the compiler with the -detailed-stack-trace (or -dst) option and try again. The exception report will now contain details for each stack frame including links to the various objects' details. You'll be able to see what source code Cobra choked on by looking at the .token property of these objects.

### Selective debugging output

You might sprinkle some `trace` or `print` statements in the compiler to see what's going on, only to get way too much output to digest. Often the output is only needed for a certain line or lines in a Cobra test program you've written. In that case, you can trim down the output by looking at the token like so:

.code
        v = .token.fileName.contains('foo') and .token.lineNum == 38
        if v
            print
            trace this, x, y
        # ...
        if v
            trace result

The .token property is available in all source-driven nodes including all expressions, statements, boxes and so on.

###  Examine and experiment with generated code

There is an option to "keep the intermediate files" which means "don't delete the generated C# code":

    cobra -keep-intermediate-files -c foo.cobra

Now you can look at "foo.cobra.cs".

And if you crank up verbosity to level 2, Cobra will show the C# compilation command it uses. Also, you can abbreviate that other option, so now we have:

    cobra -kif -c -v=2 foo.cobra

Now you can copy the C# compilation command, *edit* "foo.cobra.cs" and paste the C# compilation command back to the command line. This is useful on occasion for experimenting with the generated code to solve a problem or explore how to implement something new.

### Turning on "debug"

You can "turn on debug" in cobra with -debug, but comp already does this by default:

    comp

But you also have to tell 'mono' to turn on debugging like so:

    mono --debug cobra.exe foo

And now if cobra.exe throws an uncaught exception, you will have line numbers in the CLI stack trace. Without the "mono --debug" an .exe will not include the line numbers in the stack trace.

You may also want a Cobra detailed stack trace and/or to keep the intermediate C# files around:

    comp -dst -kif

Now nothing can stop you from fixing the compiler! Congratulations!

### Parsing problems

If the parser generates a false error and you don't know why, crank up verbosity to -v:3. If that's not enough information, uncomment the "assert false" in .throwError to see the stack at the point of the error.

### Interactive GUI Exploration of Compiler Internals

You can create a version of the Cobra compiler that displays the GUI after compilation for interactive browsing of the compiler objects:

    win-comp
    cobra-win foo bar

The window that pops up contains a treeview that lets you explore the command line object, compiler object, namespaces, modules and messages from the compiler. By drilling down in the treeview, you can get to anything. The properties of the objects such as .name, .type, .args, etc. are what you drill down through. And if there was an uncaught exception, that will also be browsable. This approach can be *much* easier than working with `trace` statements.

Also, if there is a particular object that you're interested in, you can augment the compiler code first, like so:

    if someCondition
        assert false, objectOfInterest

The AssertException instance will show up in the tree view and its `info` property will contain the `objectOfInterest` for your examination.

If you would like to include an object explorer in your own programs, take a look at Source/ObjectExplorer-WinForms.cobra.

## Warts

If you're thinking of fixing a wart, talk about it first on the discussion forums. Someone could already be in the middle of it.

* Container implements IType which is wrong given that NameSpace inherits Container and is not a type. Also, I don't know of anything in the Container that conceptually or technically requires it to be a type.

* IMember has too many methods and properties especially given that classes like CobraType implement it and then end up having to respond to queries that only an IBoxMember, for example, should have to respond to.

* The implementations of _bindImp in DotExpr, CallExpr and MemberExpr are likely in need of some refactoring and clarification.

* Just what is 'isTracked' and why is it needed?

* The exception report writer methods in CobraCore should be broken out to a separate class both for organization and also for the future possibility of optionally stripping this out when compiling.

* Any code that typecasts to passthrough ("to passthrough") or uses $sharp/sharp'' is working around a limitation in the compiler.

* CC: comments stand for "code cleanup" and these are places where I wanted to write the code one way, but had to write it another way because the compiler just isn't there yet. Although it's not uncommon to discover that some of these can already be retired.

* @@ is used in just a few places as a TODO:. These should be resolved or converted to TODO.

* There are some test cases still marked .skip. that need to be reviewed. I suppose these are more TODO items than coding warts.
