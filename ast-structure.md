# AST Structure

## Understanding the Clang AST

https://jonasdevlieghere.com/understanding-the-clang-ast/

`clang` is the compiler frontend for the C language family. In general, a
compiler frontend is responsible for the lexing and parsing steps, as well as
syntactic and semantic analysis. During this step, the frontend may create a
symbol table and verify the correctness of the AST. The result of the frontend
stage is code in an *intermediate representation*, that may be handed on to the
compiler backend and, optionally, before that, an optimizer.

The `ASTContext` class stores information about the AST that can not found from
the tree alone. Also, it takes on the role of an "AST manager", providing an
entry point to the AST via the `getTranslationUnitDecl()` function, which
provides the AST of the whole translation tree under inspection. The ASTContext
also stores the identifier table, the source code manager and a list of declared
types. Lastly, it also has some glue methods to find the parent of a node and
provides global access to the language options specific constructs like the
standard `size_t` type node.

The three most important classes in the clang AST are `Decl` (declarations),
`Stmt` (statements) and `Type` (types). These classes form the base of many
further subclasses (such as all kinds of declarations, like `functionDecl`).
However, it's important to note that these nodes do not share a common base
class. As such there is no interface for traversing arbitrary nodes in the AST.
Rather you most likely will be working with visitors, with dedicated functions
to traverse particular kinds of nodes.

For a given C/C++ file, we can use `clang -Xclang -ast-dump -fsyntax-only <file>` to see a dump of the AST structure.

For example, an if-statement consists of an `IfSmt`, with appropriate glue methods `getCond()`, `getThen()` and `getElse()` to further traverse the AST.

Every token in a C/C++ input stream is identified by a `SourceLocation`. Every
node in the AST contains at least such `SourceLocation`s, so they are kept small
by using IDs that can be used to lookup further information in the
`SourceManager`. Otherwise you would have to store row, column and source-file
information for every single node.

We can use `RecursiveASTVistor` or `ASTMatcher`s to traverse the AST to look for
points of interest. Both of these approaches are embedded into LibTooling, which
is nice as they save you the effort of creating the AST yourself -- you only
have to operate on it. The only downside is that this API is quite instable.
Another option is to use libClang, which is the stable, high-level C API
provided by clang.

The first option we have is to use the `RecursiveASTVistor` technique. For this
method, we subclass the `RecursiveASTVistor` class, which declares overridable
methods that we can redefine for custom behavior. Among others, the class
provides methods to stop the recursive descent of the AST for `Decl`s
(`visitDecl`), `Type`s (`visitType`) and more. Every such function should return
`true` or `false` to tell the traversal engine to continue or stop traversing.
This can lead to the following code:

```cpp
class FindNamedCallVisitor : public RecursiveASTVisitor<FindNamedCallVisitor> {
 public:
  explicit FindNamedCallVisitor(ASTContext* Context, std::string fName)
      : Context(Context), fName(fName) {};

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
  callExpr(callee(functionDecl(hasName(".doSomething()")).bind("call")));

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
