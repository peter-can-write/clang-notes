# Choosing the Right Tool for the Job

There are three main options to build a tool using the clang library
infrastructure: *libClang*, *plugins* and *libTooling*. These are discussed in
more detail in the following paragraphs. First a description of each option is
provided, followed by a listing of pros and cons.

## LibClang

One of the core elements of the LLVM development philosophy is to continuously
push for progress and don't shy away from breaking the eggs. If widespread
changes to the clang AST or the LLVM IR are deemed necessary to move the project
forward, then those changes are made and clients are forced to react. However,
to nevertheless be able to provide a stable API to clients who need it and
especially those clients who don't need all the nitty gritty details or
intricacies of the clang internals, the project maintains high level C APIs
which have very stable and dependable APIs. C is presumably chosen because it is
a simple language and especially because bindings to other languages like Python
are easily developed. *libClang* is one example of such a stable, high level C
API. It has a relatively abstract and simple interface to traverse the AST and
query it for basic information like the names or types of a node.

libClang is especially well suited to editor integration, as it also has an API
for simple code completion.

+ easy to use from other languages,
+ very stable and backwards compatible API,
+ very high level and easy to use,

- does not provide full control over the AST.

## Clang Plugins

*Clang Plugins* are more sophisticated C++ tools written with the goal of
compiling them into shared libraries that can be linked into the compilation
process dynamically. Clang plugins are well suited to simple linters or other tools that do not need full context over the entire build process. The disadvantage is that they are run over each translation unit individually and thus cannot maintain state across TUs (code bases).

+ runs whenever any dependency changes,
+ can make or break a build,
+ gives you complete control over the AST.

- does not have full context over the entire build process
- does not get access to any of the infrastructure around loading a file, so therefore also cannot emit transformed source after processing a file (like you would want to do with a S2S-tool like clang-format).

## LibTooling

While clang plugins can only run over individual files, *LibTooling* tools have
access to the entire compilation process and are thus the most powerful option
among the three. They can be used for static analysis, refactoring and
source-to-source transformation purposes.

+ get full context about the build process and can therefore decide what files to run on,
+ gives you full control over the AST.

- unstable API (!)
- relatively low level (less so with ASTMatchers)

Note that the main difference between LibTooling and plugins is that the former
produce standalone tools, while the latter would have to be invoked as part of
your build process and thus have to rerun on every (dependency) change.

## ClangTidy

...
