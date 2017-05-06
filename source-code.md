# Source Code

## AST/ASTContext.hpp

The ASTContext provides information about an AST as well as additional utilities
that are required any time one is dealing with the AST. It is basically a
"namespace of functions", "instantiated" for the current AST. As such it
provides functions to modify a type (e.g. to add qualifications to it). It also
gives access to the top-level translation unit, the source manager and the
identifier table.

## Basic/SourceLocation

This file declares and defines various classes used to represent locations in
source code. Here, a *location* is really any $(\mathtt{row, column})$ pair in
code. The main `SourceLocation` class is a lightweight wrapper over an encoded
`unsigned int` that refers to such a location. The `SourceManager` can later
resolve the location to the actual row and column. The MSB (bit 31) of the
underlying SourceLocation ID stores whether or not the source location is inside
a macro *expansion*, or in actual (not macro-expanded) code. The ID must be
greater zero to be valid.

The file also holds the `SourceRange` class, which simply contains two
`SourceLocation`s. Furthermore, the `CharSourceRange` class can represent either
a range of the first and last *character* in a range, or the first character of
the first and last *token* in the range. For this, the class also stores a
boolean to indicate if it is a *token range* (i.e. the range points to tokens,
not characters).

Next to `SourceLocation`, there exists `FullSourceLoc`, which aggregates a
`SourceLocation` with the reference to a `SourceManager`, allowing access to
spelling/expansion column and line number, for example.

## Basic/SourceManager

The `SourceManager` is a huge class (and associated source files) that handles
loading, storing and providing access to all files in a translation unit. It is
also the main interface for getting more information about a `SourceLocation`.
For example, `getSpellingLoc(SourceLocation)` will get you the actual source
definition of a token, which will be different in the case of a macro expansion.
Also, `getFileID(SourceLocation)` gets you the `FileID` for a token (a `FileID`
can be either an `#included` file or a macro). The `SourceManager` also gives you `SourceLocation`s for the beginning and end of a `FileID`.

The method `getSpellingLoc` looks like this:

```cpp
SourceLocation getSpellingLoc(SourceLocation Loc) const {
  // Handle the non-mapped case inline, defer to out of line code to handle
  // expansions.
  if (Loc.isFileID()) return Loc;
  return getSpellingLocSlowCase(Loc);
}
```

Basically, it checks if the source location already is in source code, rather
than in a macro expansion. If so, it just returns the location. Else, it calls
following function:

```cpp
SourceLocation SourceManager::getSpellingLocSlowCase(SourceLocation Loc) const {
  do {
    std::pair<FileID, unsigned> LocInfo = getDecomposedLoc(Loc);
    Loc = getSLocEntry(LocInfo.first).getExpansion().getSpellingLoc();
    Loc = Loc.getLocWithOffset(LocInfo.second);
  } while (!Loc.isFileID());
  return Loc;
}
```

This will first decompose the source location into its `FileID` and offset into
that file. It will then get the `SourceLoc` for the `Loc`. A `SourceLoc` is a
union of a `FileInfo`, holding information about an `#include`-ed file, and a
`ExpansionInfo`, which encodes the start and end of a macro expansion (the name
of the macro invocation, up to the closing `)` or the end of the macro name, if
it is not a function) as well as the spelling location of the macro. We thus
grab the spelling location of the macro and move to the offset of the original
`Loc` within that macro (which is the same stream of tokens (arguments are not
expanded in this representation?)). If this is now real code, we can stop. If it
is yet another macro expansion, we have to do this again.

## AST/ASTMatcherMacros

This file holds the infrastructure for the matchers macro. Basically, it defines
a set of macros that allow you to easily create new macros for the matching DSL. One of the most important macros is the following for `AST_MATCHER`:

```cpp
#define AST_MATCHER(Type, DefineMatcher)                                       \
  namespace internal {                                                         \
  class matcher_#DefineMatcher#Matcher                                       \
      : public ::clang::ast_matchers::internal::MatcherInterface<Type> {       \
  public:                                                                      \
    explicit matcher_#DefineMatcher#Matcher() = default;                     \
    bool matches(const Type &Node,                                             \
                 ::clang::ast_matchers::internal::ASTMatchFinder* Finder,      \
                 ::clang::ast_matchers::internal::BoundNodesTreeBuilder        \
                    * Builder) const override;                                 \
  };                                                                           \
  }                                                                            \
  inline ::clang::ast_matchers::internal::Matcher<Type> DefineMatcher() {      \
    return ::clang::ast_matchers::internal::makeMatcher(                       \
        new internal::matcher_#DefineMatcher#Matcher());                     \
  }                                                                            \
  inline bool internal::matcher_#DefineMatcher#Matcher::matches(             \
      const Type &Node,                                                        \
      ::clang::ast_matchers::internal::ASTMatchFinder* Finder,                 \
      ::clang::ast_matchers::internal::BoundNodesTreeBuilder* Builder) const
```

This macro allows for defining a new matcher with the name `DefineMatcher` that
returns a `Matcher<Type>`. We can see this macro has three main parts:

1. A class called `matcher_<DefineMatcher>` that declares the matcher interface.
We can also see that each matcher takes a Node of the type we want to match
(e.g. `Decl`), a `ASTMatchFinder`, which is the interface to the entire matching
process (like a "matching context") and lastly a `BoundNodesTreeBuilder`, which
is necessary to bind names to nodes in the AST (this happens in the `MatcherInterface` inside ASTMatcherInternal.h);
2. The function that you end up calling is defined (`DefineMatcher`). It calls `makeMatcher` in the internal namespace, which just does type deduction and returns a `Matcher<Type>`;
3. Lastly the beginning of the definition the method we define when we use the macro (first declared in the class).

As such, a simple matcher would look something like this:

```cpp
AST_MATCHER(QualType, isInteger) {
    return Node->isIntegerType();
}
```

this matcher would match for all `QualType` nodes. It defines a `matches` method
that returns a `Matcher<QualType>`. The matcher itself takes no arguments, as we
can see. For more complex queries, we'll want `AST_MATCHER_P`, which allows us
to take one parameter (there exist variants for two, also for overloads):

```cpp
AST_MATCHER_P(BinaryOperator, hasLHS, internal::Matcher<Expr>, InnerMatcher) {
  const auto* LeftHandSide = Node.getLHS();
  if (LeftHandSide == nullptr) return;
  return InnerMatcher.matches(* LeftHAndSide, Finder, Builder);
}
```

Another point of interest in this file is the way matchers are declared that allow more than one nested matcher in an implicit `allOf()`. These kinds of matchers are basically most `Decl`s  and `Stmt`s. They are declared like this:


```cpp
const internal::VariadicDynCastAllOfMatcher<
  Decl,
  CXXRecordDecl> cxxRecordDecl;
```
