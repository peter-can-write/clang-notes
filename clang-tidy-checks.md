# Clang Tidy Checks

http://bbannier.github.io/blog/2015/05/02/Writing-a-basic-clang-static-analysis-check.html

Clang Tidy is a linting and static analysis tool built on top of the LibTooling
infrastructure. It also provides the ability to very easily add a check of your
own, which you can then run using your own (patched) build of clang-tidy, or
attempt to merge upstream.

You can find the clang-tidy source `tools/clang/tools/extra/clang-tidy`. This is
where you'll also see a file `add_new_check.py`, which you can pass a category
and the name of your check (using kebab-case). The script will then generate all
the necessary testing and plugin boilerplate for you to get working on the
plugin right away.

For example, if we run `python add_new_check.py misc virtual-shadowing`, this
will generate `VirtualShadowingCheck.h` and `VirtualShadowingCheck.cpp` in the
clang tidy source tree as your header and implementation files. If you rebuild
LLVM/clang, you can then already run your check with `clang-tidy
-checks='-*,misc-virtual-shadowing' <file>`. The boilerplate for the check will look something like this:

```cpp
class VirtualShadowingCheck : public ClangTidyCheck {
public:
  VirtualShadowingCheck(StringRef Name, ClangTidyContext* Context)
      : ClangTidyCheck(Name, Context) {}
  void registerMatchers(ast_matchers::MatchFinder* Finder) override;
  void check(const ast_matchers::MatchFinder::MatchResult &Result) override;
};
```

The two methods `registerMatchers()` and `check()` you see here are all we need
to define our own check. The former is where you register the `ASTMatchers` you
want to match on and the latter is the callback clang-tidy will call for every
matched AST node. If we were looking for virtual functions which shadow
non-virtual functions of the same name in the parent, we could write something
like this for a matcher inside `registerMatchers` in
`VirtualShadowingCheck.cpp`:

```cpp
void VirtualShadowingCheck::registerMatchers(MatchFinder* Finder) {
  // Find all virtual methods
  Finder->addMatcher(cxxMethodDecl(isVirtual()).bind("method"), this);
}
```

For the `check()`, we will first want a test case. `add_new_check.py` already
generated the test boilerplate for us under
`tools/clang/tools/extra/test/clang-tidy/misc-virtual-shadowing.cpp`:

```cpp
// RUN: %check_clang_tidy %s misc-virtual-shadowing %t

// FIXME: Add something that triggers the check here.
void f();
// CHECK-MESSAGES: :[[@LINE-1]]:6: warning: function 'f' is insufficiently awesome [misc-virtual-shadowing]

// FIXME: Verify the applied fix.
//   * Make the CHECK patterns specific enough and try to make verified lines
//     unique to avoid incorrect matches.
//   * Use {{}} for regular expressions.
// CHECK-FIXES: {{^}}void awesome_f();{{$}}

// FIXME: Add something that doesn't trigger the check here.
void awesome_f2();
```

We can see that the LLVM/clang test driver uses comments in the source code to
drive its test expectations. We use the `CHECK-MESSAGES` directive to set
expectations for diagnostics and `CHECK-FIXES` to set expectations for fixes. At
the top, the `RUN` directive tells the driver how to execute the test. For our
purposes, we'll change this to the following tests:

```cpp
// RUN: $(dirname %s)/check_clang_tidy.sh %s misc-virtual-shadowing %t
// REQUIRES: shell

struct A {
  void f() {}
};

struct B : public A {
  // CHECK-MESSAGES: :[[@LINE+1]]:3: warning: method hides non-virtual method from a base class [misc-virtual-shadowing]
  virtual void f() {}  // problematic
};

struct C {
  virtual void f() {}  // OK(1)
};

struct D : public C {
  virtual void f() {}  // OK(2)
};
```

Here you can see code examples with inline test directives. Basically, clang's
test harness will check that the specified diagnostics are emitted at the
specified line and column (`@LINE+1`:3). Note also how we've used `OK` to denote
negative examples, i.e. cases where the check should not trigger.

We can now write the body of our `check`:

```cpp
void VirtualShadowingCheck::check(
    const ast_matchers::MatchFinder::MatchResult &Result) {

  // Returns a CXXMethodDecl* which will be alive as long as the translation
  // unit is loaded, which means we can keep store it.
  const auto* Method = Result.Nodes.getNodeAs<CXXMethodDecl>("method");
  const auto* ClassDecl = Method->getParent();

  if (ClassDecl->getNumBases() == 0)
    return;

  bool AnyBaseMatches = ClassDecl->forallBases([TargetName = Method->getName()](
      const CXXRecordDecl* Base) {
    for (const auto* BaseMethod : Base->methods()) {
      if (BaseMethod->getName() == TargetName) {
        return !BaseMethod->isVirtual();
      }
    }
    return false;
  });

  if (AnyBaseMatches) {
    diag(Method->getLocStart(),
         "method hides non-virtual method from a base class");
  }
}
```

The method basically grabs the class declaration of each matched method and then
traverses all direct and indirect base classes of that class, looking for one
with a method that has the same name but is not declared virtual.
