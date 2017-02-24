# Videos

## Clang Tooling at Euro-LLVM 2013

https://www.youtube.com/watch?v=VqCkCDFLSsc&t=2322s

The clang AST is very rich in semantic information, it is fully type-resolved and it has quite a huge source tree (> 100K LOC).

One class you encounter very often when dealing with clang is the `ASTContext`, which stores information about an AST. More precisely, it has an *identifier table* (symbol table) and source (code) manager.

The core classes in the clang AST are:

1. `Decl`, with further subclasses, such as:
  1. `CXXRecordDecl`
  2. `VarDecl`
  3. `UnresolvedUsingTypenameDeclaration`
2. `Stmt`
  1. `ComponoundStmt`
  2. `CXXTryStmt`
  3. `BinaryOperator`
  Note that in clang, expressions (like operators) are the same thing as statments.
3. `Type`
  1. `PointerType`

Nodes in the AST have *pointer identity*.

There are also *glue classes*, which combine these three basic types of classes.
The `DeclContext` is information a `Decl` may have when it consists of further
Decls. `TemplateArgument`, which store both a statement and a type.

Classes also have *glue methods*, which allow you to further traverse the graph,
e.g. via the `getElse()` statement of an `IfStmt`.

How are types modeled in clang? Via `QualTypes` and `Types`. A `QualType` is a
type with qualifications such as `const` or `volatile`.

Locations are very important in clang. A `SourceLocation` is usually represented
by an ID that is encoded efficiently, as to not have to store row and column
each time. A location also always represents a location, as used by the lexer.

`SourceLocations` always point to tokens.

You can use `clang -Xclang -ast-dump -fsyntax-only <file.cpp>` to show you a
representation of the AST of your code.

## A Brief Introduction to LLVM

https://www.youtube.com/watch?v=a5-WaD8VV38

Compilers used to be monolithic monsters, where each step of the compilation
process was unflexibly engrained in the whole system. LLVM turned this around by
developing a compiler that was composed of many, modular and independent
*libraries*, that could have many different use cases. More precisely, the steps
that the LLVM compiler toolchain provides are

1. Lexing,
2. Parsing,
3. Syntactic/Semantic Analysis,
4. Intermediate Representation,
5. Optimization and
6. Code Generation.

Intermediate representation, for which LLVM is famous, can be implemented in various ways, including:

* Structured format (graph or tree),
* Flat, tuple-based (three-argument-format: opcode + two arguments)
* Flat, stack-based

LLVM IR is a low-level programming language similar to a RISC assembly language,
but *strongly typed*. Moreover, it has an *infinite set of registers*, which are
*immutable* (i.e. variables are not mutable, so that it's easier to reason about
them).

A lot of parts of GCC are mediocre and it is generally monolithic.

## Using Clang for the Chromium Project

https://www.youtube.com/watch?v=n9aSa9XVuiQ

Clang is extensible and built as a set of libraries.

Other cool clang tools:

* ThreadSanitizer, checks for race conditions;
* AddressSanitizer, checks for memory leaks / out of bounds access;
* UndefinedBehaviorSanitizer, checks for UB.

## Refactoring with C++

https://www.youtube.com/watch?v=U98rhV6wONo

The interfaces to clang:

* Clang Plugins: run as part of the compilation, very tightly coupled. Can break build when they fail (because you check something).
* libClang: high-level C API, gives you things like cursors or references to code.
* libTooling: More powerful than libClang, less coupled than plugins.

libTooling can run over a string, or run over multiple files in a project.

Before matchers, matching nodes in the AST was more complicated.

## Optimizing LLVM for GPGPU

https://www.youtube.com/watch?v=JHfb8z-iSYk

CPUs are very different from GPUs. Initially, LLVM was almost 10 times slower than NVCC.

* CPU is optimized for high throughput and few cores.
* GPU is:
  + optimized for graph rendering.
  + very light weight threads with branch prediction.
  + Can deploy tens of thousands of cores.
  + ISA is different and instructions are different.

Optimization is about changing and reordering code, without changing the
ultimate result of the program, but at the same time tuning the code so that it
can be translated into the most efficient instructions on a particular ISA.
