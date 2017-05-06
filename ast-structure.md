# AST Structure

## Understanding the Clang AST

https://jonasdevlieghere.com/understanding-the-clang-ast/

`clang` is the compiler frontend for the C language family. In general, a
compiler frontend is responsible for the lexing and parsing steps, the result of
which is an abstract syntax tree (AST), as well as syntactic and semantic
analysis. During these steps, the frontend may create a symbol table and verify
the correctness of the AST. The result of the frontend stage is code in an
*intermediate representation*, that may be handed on to the compiler backend
and, optionally, before that, an optimizer. In the case of clang and LLVM, there
is an optimizer of course (the LLVM `opt` optimizer).

The three most important classes in the clang AST are `Decl` (declarations),
`Stmt` (statements) and `Type` (types). These classes form the base of many
further subclasses (such as all kinds of declarations, like `FunctionDecl`).
However, it's important to note that these nodes do not share a common base
class. As such there is no interface for traversing arbitrary nodes in the AST,
for example via a pointer to a generic `ASTNode`. One reason for this design
decision could be that even if you had a common base class, this class would not
be very rich, as currently the APIs of `Decl`, `Stmt` and `Type` are all very
different (i.e. the intersection of methods that would form the interface of an
abstract `ASTNode` would be very small). Of course, runtime polymorphism can be
replaced by compile-time polymorphism using templates, but because of this small
intersection of APIs, this would not be so useful. Rather you most likely will
be working with the *visitor pattern*, where you define or override dedicated
functions for each of the three broad classes of nodes (and often more) and then
let clang traverse the AST for you, calling your visitor function for whatever
nodes it encounters. Note that this is slightly different from the C and Python
APIs, which does actually have means of generically pointing to a node of any
type in the AST. As such, the traversal APIs in these languages are sometimes
more flexible and rely less heavily on visitation.

The `ASTContext` class stores information about the AST that can not be found
from the tree alone. Also, it takes on the role of an "AST manager", providing
an entry point to the AST via the `getTranslationUnitDecl()` function, which
provides the AST of the whole translation tree under inspection. The ASTContext
also stores the identifier table, the source code manager and a list of declared
types. It also has some glue methods to find the parents of a node
(`getParents`) and provides global access to the language options
(`getLangOpts`) specific constructs like the standard `size_t` type node.

For a given C/C++ file, we can use `clang -Xclang -ast-dump -fsyntax-only
<file>` to see a dump of the AST structure. For example, the following C++
program:

```cpp
int f(int x, int y) {
  return x + y;
}

auto main() -> int {
  return f(1, 2);
}
```

gives the following dump (`clang -Xclang -ast-dump -fsyntax-only -std=c++14 file.cpp`):

```cpp
|-TypedefDecl 0x7fe591818858 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
| `-BuiltinType 0x7fe591818540 '__int128'
|-TypedefDecl 0x7fe5918188b8 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
| `-BuiltinType 0x7fe591818560 'unsigned __int128'
|-TypedefDecl 0x7fe591818bf8 <<invalid sloc>> <invalid sloc> implicit __NSConstantString 'struct __NSConstantString_tag'
| `-RecordType 0x7fe5918189a0 'struct __NSConstantString_tag'
|   `-CXXRecord 0x7fe591818908 '__NSConstantString_tag'
|-TypedefDecl 0x7fe591818c88 <<invalid sloc>> <invalid sloc> implicit __builtin_ms_va_list 'char *'
| `-PointerType 0x7fe591818c50 'char *'
|   `-BuiltinType 0x7fe591818360 'char'
|-TypedefDecl 0x7fe591832000 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list 'struct __va_list_tag [1]'
| `-ConstantArrayType 0x7fe591818f60 'struct __va_list_tag [1]' 1
|   `-RecordType 0x7fe591818d70 'struct __va_list_tag'
|     `-CXXRecord 0x7fe591818cd8 '__va_list_tag'
// -----------------------------------------------------------------------------
|-FunctionDecl 0x7fe5918321a0 <file.cpp:1:1, line:3:1> line:1:5 used f 'int (int, int)'
| |-ParmVarDecl 0x7fe591832060 <col:7, col:11> col:11 used x 'int'
| |-ParmVarDecl 0x7fe5918320d0 <col:14, col:18> col:18 used y 'int'
| `-CompoundStmt 0x7fe591832358 <col:21, line:3:1>
|   `-ReturnStmt 0x7fe591832340 <line:2:3, col:14>
|     `-BinaryOperator 0x7fe591832318 <col:10, col:14> 'int' '+'
|       |-ImplicitCastExpr 0x7fe5918322e8 <col:10> 'int' <LValueToRValue>
|       | `-DeclRefExpr 0x7fe591832298 <col:10> 'int' lvalue ParmVar 0x7fe591832060 'x' 'int'
|       `-ImplicitCastExpr 0x7fe591832300 <col:14> 'int' <LValueToRValue>
|         `-DeclRefExpr 0x7fe5918322c0 <col:14> 'int' lvalue ParmVar 0x7fe5918320d0 'y' 'int'
`-FunctionDecl 0x7fe591832430 <line:5:1, line:7:1> line:5:6 main 'auto (void) -> int'
  `-CompoundStmt 0x7fe591832660 <col:20, line:7:1>
    `-ReturnStmt 0x7fe591832648 <line:6:3, col:16>
      `-CallExpr 0x7fe591832610 <col:10, col:16> 'int'
        |-ImplicitCastExpr 0x7fe5918325f8 <col:10> 'int (*)(int, int)' <FunctionToPointerDecay>
        | `-DeclRefExpr 0x7fe5918325a0 <col:10> 'int (int, int)' lvalue Function 0x7fe5918321a0 'f' 'int (int, int)'
        |-IntegerLiteral 0x7fe591832560 <col:12> 'int' 1
        `-IntegerLiteral 0x7fe591832580 <col:15> 'int' 2
```

Note where I inserted the comment line (`// ---...`). Up to this line, all C++
programs emit the same AST, simply because of my particular system environment
(note the `NS` stuff from macOS). Everything after that is the clang AST output
for the actual program

Next to a rich API to access information about each particular kind of AST node,
some of the most important methods one deals with when interacting with the
clang AST in C++ are nodes' so-called *glue methods*. Glue methods are methods
that allow you to actually traverse the AST, basically bridging the gap between
the various parts of the AST. For example, an If-statement consists of an
`IfSmt`, with appropriate glue methods `getCond()`, `getThen()` and `getElse()`
to access the related parts of the AST.

Every token in a C/C++ input stream is identified by a `SourceLocation`. Here, a
token is the most basic entity of a program, of which all higher level semantic
concepts like expressions, classes or functions are composed (tokens can be
classified as keywords, identifiers, literals, punctuation or comments -- see
`Basic/TokenKinds.def`). Every node in the AST contains at least such
`SourceLocation`s, so they are kept small by using IDs that can be used to
lookup further information in the `SourceManager`. Otherwise you would have to
store row, column and source-file information for every single node.

To traverse the AST and to look for points of interest, we can either use the
`RecursiveASTVistor` or `ASTMatcher` classes. Both of these approaches are
embedded into LibTooling, which is nice as they save you the effort of
traversing the AST yourself -- you only have to operate on it. The only downside
is that this API is quite unstable. The alternative is to use libClang, which is
the stable, high-level C API provided by clang.

The first option we have is to use the `RecursiveASTVistor` technique. For this
method, we subclass the `RecursiveASTVistor` class, which declares overridable
methods that we can redefine for custom behavior. Among others, the class
provides methods to stop the recursive descent of the AST for `Decl`s
(`visitDecl`), `Type`s (`visitType`) and more. Every such function should return
`true` or `false` to tell the traversal engine to continue or stop traversing.
This could look like so:

```cpp
class FindNamedCallVisitor : public RecursiveASTVisitor<FindNamedCallVisitor> {
 public:
  explicit FindNamedCallVisitor(ASTContext* Context, std::string fName)
      : Context(Context), fName(fName) {}

  // We want to stop the traversal for call expressions.
  bool VisitCallExpr(CallExpr* CallExpression) {
    // Grab the function type
    QualType q = CallExpression->getType();
    const Type* t = q.getTypePtrOrNull();

    if (t != NULL) {
      FunctionDecl* func = CallExpression->getDirectCallee();
      const std::string funcName = func->getNameInfo().getAsString();

      // If this is the function we are looking for
      if (fName == funcName) {
        // Grab the source location (FullSourceLoc = SourceLocation + SourceManager)
        FullSourceLoc FullLocation =
            Context->getFullLoc(CallExpression->getLocStart());

        if (FullLocation.isValid())
          llvm::outs() << "Found call at "
                       << FullLocation.getSpellingLineNumber() << ":"
                       << FullLocation.getSpellingColumnNumber() << "\n";
      }
    }

    return true;
  }

 private:
  ASTContext *Context;
  std::string fName;
};
```

We would then furthermore require a `Consumer` and `FrontendAction`. The second
option is to use `ASTMatcher`s, whose API provides a sweet DSL for querying the
AST and matching nodes of particular qualities. For this, you would first define
a query and then subclass `MatchFinder::MatchCallback`, which lets you override
its `run()` method, to which LibTooling will then pass every matched node in the
AST. The above example would be implemented like so with ASTMatchers:

```cpp

// Look for all function calls to `func` and bind the result to the name `call`
// so that we can later reference this part of the match.
StatementMatcher CallMatcher =
  callExpr(callee(functionDecl(hasName("func")).bind("call")));

// ...

class MyCallback : public MatchFinder::MatchCallback {
 public:
  virtual void run(const MatchFinder::MatchResult &Result) {
    ASTContext* Context = Result.Context;
    if (const CallExpr* E =
            Result.Nodes.getNodeAs<clang::CallExpr>("call")) {
      FullSourceLoc FullLocation = Context->getFullLoc(E->getLocStart());
      if (FullLocation.isValid()) {
        llvm::outs() << "Found call at " << FullLocation.getSpellingLineNumber()
                     << ":" << FullLocation.getSpellingColumnNumber() << "\n";
      };
    }
  }
};
```

Note that we can generally only bind names to nodes that sound like nouns, e.g.
`decl`, but not verbs like `hasName`, since those are just attributes. Also note
that one of the nice things about having this interface work with functors is
that we can maintain state in the matcher.

The last option we have to traverse the clang AST is libClang, the stable C
interface to clang. There are two central concepts w.r.t. AST traversal in
libClang.  The first is *cursors*, which are basically pointers to nodes in the
AST. When traversing the AST, we will deal with and operate on cursors. Using
these cursors, we can then get the spelling (name) of a node or its source
location. Next to cursors, the other important concept is the visitation
pattern, which you can use to implement operations over nodes.

Basically, the steps to traverse the AST with libClang are the following:

1. Create an *index* with `clang_CreateIndex()`. An index represents a group of translation units that we are interested in.
2. Create a `CXTranslationUnit` using `clang_parseTranslationUnit` to load the AST of a file into your program.
3. Define a visitor returning a `CXChildVisitResult` and taking a `current` and `parent` cursor and optionally any additional `CXClientData` (`void*`) user data.

The first two steps would look something like this:

```cpp
#include <clang-c/Index.h>
#include <iostream>

auto main(int argc, const char* argv[]) -> int {
  if (argc < 2) {
    std::cout << "Invalid number of arguments!" << std::endl;
  }

  // excludeDeclsFromPCH = 1, displayDiagnostics = 1
  CXIndex Index = clang_createIndex(1, 1);

  // Expected arguments:
  // 1) The index to add the translation unit to,
  // 2) The name of the file to parse,
  // 3) A pointer to strings of further command line arguments to compile,
  // 4) The number of further arguments to compile,
  // 5) A pointer to a an array of CXUnsavedFiles structs,
  // 6) The number of CXUnsavedFiles (buffers of unsaved files) in the array,
  // 7) A bitmask of options.
  CXTranslationUnit TU = clang_parseTranslationUnit(
    Index,
    argv[1],
    argv + 2,
    argc - 2,
    nullptr,
    0,
    CXTranslationUnit_SkipFunctionBodies);

    // RAII?
    clang_disposeTranslationUnit(TU);
    clang_disposeIndex(Index);
}
```

After parsing the TU, we get an AST. We then grab the cursor to the root of the AST, which is the `TranslationUnitDecl`:

```cpp
CXCursor cursor = clang_getTranslationUnitCursor(translationUnit);
```

And define and visit a function:

```cpp
CXChildVisitResult visit(CXCursor cursor, CXCursor, CXClientData) {
}

// ...

clang_visitChildren(cursor, visit, nullptr);
```

Finally we can define the following `visit` function to look for all functions
with the name `foo` in them:

```cpp
CXChildVisitResult visit(CXCursor cursor, CXCursor, CXClientData) {
  CXCursorKind kind = clang_getCursorKind(cursor);

  // We are looking for functions or methods with the name 'foo' in them.
  if (kind == CXCursorKind::CXCursor_FunctionDecl ||
      kind == CXCursorKind::CXCursor_CXXMethod) {
        // The display name is sometimes more descriptive than the spelling name
        // (which is just the source code).
        auto cursorName = clang_getCursorDisplayName(cursor);

        auto cursorNameString = std::string(clang_getCString(cursorName));
        if (cursorNameString.find("foo") != std::string::npos) {
          // Grab the source range, i.e. (start, end) SourceLocation pair.
          CXSourceRange range = clang_getCursorExtent(cursor);

          // Grab the start of the range.
          CXSourceLocation location = clang_getRangeStart(range);

          // Decompose the SourceLocation into a location in a file.
          CXFile file;
          unsigned int line;
          unsigned int column;
          clang_getFileLocation(location, &file, &line, &column, nullptr);

          // Get the name of the file.
          auto fileName = clang_getFileName(file);

          std::cout << "Found function"
                    << " in " << clang_getCString(fileName)
                    << " at " << line
                    << ":" << column
                    << std::endl;

          // Manual cleanup!
          clang_disposeString(fileName);
        }

        // Manual cleanup!
        clang_disposeString(cursorName);
    }

    return CXChildVisit_Recurse;
}
```
