# AST Matchers

## Official Documentation

http://clang.llvm.org/docs/LibASTMatchers.html

The `ASTMatcher` library, also called `LibASTMatcher`, provides a set of
functions, classes and macros to form a domain-specific-language (DSL) that
allows for efficient expression of points of interest (nodes) in the AST and
subsequent matching via callbacks and built-in traversal.

Say, for example, we wanted to match all classes or unions in a C++ source file.
Both of these are referred to as a *record* in clang. Therefore, our base
*matcher* will be `recordDecl()`. We can then add further matchers within this
matcher to narrow our search. If we want all records with the name `Foo`, we can
use a `hasName()` matcher inside the `recordDecl`: `recordDecl(hasName("foo"))`.
Next to these predicate matchers, there exist also "quantifier" matchers like
`allOf()` or `anyOf`. All of these quantifiers take one or more
matchers and return a boolean. What's important to know is that all matchers
that can accept multiple inner matchers actually have an implicit `allOf()`
clause, so that you can simply write `matcher(<matcher1>, <matcher2>, ...)`
instead of having to wrap the inner matchers into an `allOf`. For example, we
could write `recordDecl(hasName("Foo"), isDerivedFrom("Bar"))` to look for
classes or unions that are called `Foo` and are derived from `Bar`.

There are more than a thousand classes in the AST. To figure out how to write a
matcher for a particular kind of node, use this recipe:

1. Take some source code containing an example of what you want to match,
2. Dump the AST (for that region) and see how the statement is built-up,
3. Use the [ASTMatchers reference](http://clang.llvm.org/docs/LibASTMatchersReference.html) to look for a matcher that fits the *outer-most* kind of node you're interested in, or at least narrows down the search,
4. Use `clang-query` to verify that your matcher matches the example.
5. Examine the subsequent inner nodes in the AST-dump of your example.
6. Repeat for the next inner/child node.

It is also possible to bind certain nodes in your match expression to names, so
that you can reference them later. You can do this by adding a `.bind("<name>")`
to any noun-matcher in your expression. With "noun-matcher"I mean any matcher
that sounds like a noun (`Decl`, `Stmt`, `Type` etc.) rather than a
verb/attribute (e.g. `hasName` or `isInteger`). Once you've bound a node, you
can retrieve it later inside your callback:

```cpp
void run(const MatchFinder::MatchResult& Result) const {
  // Will be null if the result is not actually a `CallExpr`
  const CallExpr* E = Result.Nodes.getNodeAs<clang::CallExpr>("call");
}
```

The `MatchResult` object here is a lightweight struct that provides all
necessary (and available) information about a particular match: the matched
nodes in the `Nodes` member, the `Context` and the `SourceManager`. The `Nodes`
member is of type `clang::ast_matchers::BoundNodes` and provides the important
`getNodeAS()` method to retrieve bound nodes, as well as a `getMap()` member
which returns a map from IDs to `DynTypedNode`s, which are type erased AST
nodes.

It is also possible to create your own matchers. For this there are two main
possibilities. The first is the `VariadicDynCastAllOfMatcher` type. Matchers
declared with this type form the backbone of a typical matcher hierarchy. They
are declared as `const internal::VariadicDynCastAllOfMatcher<Base, Derived>
<name>` and can contain any number of matchers of type `Base`, as long as the
types of those matchers *could* be cat to `Derived`. For example, `recordDecl`s
are defined as `const internal::VariadicDynCastAllOfMatcher<Decl, CXXRecordDecl>
cxxRecordDecl;`. Note that, to be more precise about binding, you can only bind
nodes that are declared with this `VariadicDynCastAllOfMatcher` type.

The second possibility for defining your own matchers are the `AST_MATCHER*`
macros. An example would be following use, which matches all integer types:

```cpp
AST_MATCHER(Type, isInteger) {
  // We have access to the Node, which is of <Type>, in here.
  return Node.isInteger();
}
```

## ASTMatcher Reference

http://clang.llvm.org/docs/LibASTMatchersReference.html

There are three basic kinds of matchers we can use:

1. *Node Matchers* match a specific type of AST node (they are nouns),
2. *Narrowing Matchers* match attributes of AST nodes,
3. *Traversal Matchers* allow traversing the AST to other nodes.

All node matchers allow any number of arguments and implicitly wrap them in an `anyOf()` clause. Also, you may only `bind` a name to a node matcher.

Some examples for each node type follow below. The foramt is `name(param)`.

### Node Matchers

The parameters of all of these matchers are further matchers, since they are nodes and have an implicit `allOf()` clause inside of them.

* `classTemplateDecl`, matches a C++ template declaration, like `template<class T> class Z {};`.
* `decl` matches any `Decl`.
* `functionDecl` matches any function declaration, like `void f();`.
* `qualType`: matches qualified types, e.g. `const int`.
* `breakStmt`: matches all `break` statements.
* `cxxCatchStmt`: matches all `catch` blocks.

Note that, in general, C++-specific nodes have a `cxx` prefix.

### Narrowing Matchers

* `allOf(Matchers...)`: the node must match all inner matchers.
* `anyOf(Matchers...)`: the node must match at least one inner matcher.
* `anything`: the node can match anything.
* `equals(bool|char|integer)`: the node match the given value.
* `isConst()`: the method or function must be const.
* `isLambda()`: the C++ record must be a lambda expression.
* `hasAttr(AttrKind)`: the declaration must have the given attribute enum value.
* `isNoThrow()`: the function declaration must be declared with `noexcept` or `throw()`.
* `isArrow()`: the *member expression* must use an arrow (`->`) as opposed to a dot.
* `hasName(std::string)`: the `NameDecl` must have the given name.
* `matchesName(std::string)`: the `NameDecl` must match the given name regex.
* `isInteger()`: the given `QualType` must be an integer.

### Traversal Matchers

* `eachOf(Matchers...)`, matches if any of its submatchers match, but unlike `anyOf`, generates a binding result for each submatch, rather than the first one found.
* `forEachDescendant(Matcher)`: matches if any direct or indirect descendant of the node matches the property and creates a binding result for each such matching descendant.
* `hasDescendant(Matcher)`: like `forEachDescendant`, but generates at most one binding result for a suitable descendant, even if multiple descendants match the matcher.
* `hasAncestor(Matcher)`: matches nodes that have an ancestor that satisfy the given matcher.
* `has(Matcher)`: matches if any child (i.e. direct descendant) satisfies the given matcher.
* `hasParent(Matcher)`: matches if the node has a parent that satisfies the given matcher.
* `hasIndex(Matcher)`: matches the index expression of a subscript operator.
* `hasBase(Matcher)`: matches an array subscript expression.
* `hasLHS(Matcher)`: matches the given matcher against the left hand side of a binary operator or array subscript expression.
* `pointee(Matcher<Type>)`: matches if the type pointed to by a pointer or reference satisfies the given type-matcher.
* `pointsTo(Matcher)`: matches if a `QualType` points to something that satisfies the given matcher.
* `hasBody(Matcher<Type>)`: matches if the body of a function or a for, while or do statement matches the given matchers.

## Examples

Match all pointer variables:

```cpp
pointerType()
```

Match all Const Lambdas that taken Auto variable, are Noexcept and contain a Goto statement (clang):

```cpp
varDecl(
  hasType(isConstQualified()),
  hasInitializer(
    hasType(cxxRecordDecl(
      isLambda(),
      has(functionTemplateDecl(
        has(cxxMethodDecl(
          isNoThrow(),
          hasBody(compoundStmt(hasDescendant(gotoStmt())))))))))))
```
