# LibTooling

https://kevinaboos.wordpress.com/2013/07/23/clang-tutorial-part-ii-libtooling-example/
http://clang.llvm.org/docs/RAVFrontendAction.html

*LibTooling* is the most powerful way to create your own tools with clang. It is a library and associated infrastructure that lets you write standalone executables with full control over all stages of file and AST processing.

We start out with the main file of a hypothetical libTooling executable:

```cpp
#include "clang/Tooling/CommonOptionsParser.h"
#include "clang/Tooling/Tooling.h"

#include "llvm/Support/CommandLine.h"

// Set up the command line interface
namespace {
llvm::cl::OptionCategory ToolCategory("MyTool");

llvm::cl::extrahelp
    CommonHelp(clang::tooling::CommonOptionsParser::HelpMessage);

llvm::cl::extrahelp MoreHelp("My funny tool")
}  // namespace

auto main(int argc, const char* argv[]) -> int {
  using namespace clang::tooling;

  CommonOptionsParser OptionsParser(argc, argv, ToolCategory);
  ClangTool Tool(OptionsParser.getCompilations(), // compilation database
                 OptionsParser.getSourcePathList());

  auto action = newFrontendActionFactory<MyTool::Action>();
  return Tool.run(action.get());
}
```

As you can see, we first need to setup the basic command line interface. Inside
`main()` we can then use LLVM and clang's command line facilities to find out
about all source files to compile. These are used to construct a `ClangTool`,
which is the driver for our own check (here called `MyTool`). We use the
`newFrontendActionFactory` function template to instantiate our `MyTool::Action`
frontend action, which we will look into next.

The basic infrastructure (boilerplate) needed to make a tool are the following
components:

1. A `FrontendAction`, which is a certain piece of code we want to run on the frontend.
2. An `ASTConsumer`, which lets us perform actions on the AST.
3. A `RecursiveASTVisitor` or `ast_matchers::MatchFinder::MatchResult` to do our node matching and eventual transformations.

These components will be discussed further below.

## `FrontendAction`

In clang's infrastructure, a `FrontendAction` is any operation that can be
performed by the frontend. Such a class will usually derive from
`ASTFrontendAction` (since `FrontendAction` is an abstract base class) and can
then override a number of virtual functions that provide access to the
compilation process at various stages. For example, we can override the
`BeginSourceFileAction` method, which is invoked for each translation unit
supplied to the tool. This will be where instantiate the subsequent components
in our pipeline. Besides this method, others include for example
`EndSourceFileAction`, which is invoked after the processing of a TU and gives
us the ability to output our transformed code, for example. Also, inside
`BeginInvocation`, we can modify the `CompilerInstance` even before the
processing stage. The compiler instance class holds contextual information about
the compilation process and environment, such as language options (C++ standard)
or configuration of the diagnostics engine. Also, we can specify if the action
is to use the preprocessor only, in which case no AST consumer will be created.

Our main focus right now is the consumption and processing of the AST, so we will be looking at the `CreateASTConsumer` method, which is supposed to return a `Consumer` for the next stage (see below). This looks like so:

```cpp
class Action : public clang::ASTFrontendAction {
public:
  using ASTConsumerPointer = std::unique_ptr<clang::ASTConsumer>;

  ASTConsumerPointer CreateASTConsumer(clang::CompilerInstance& Compiler,
                                       llvm::StringRef) override;

  bool BeginSourceFileAction(clang::CompilerInstance& Compiler,
                             llvm::StringRef Filename) override;

  void EndSourceFileAction() override;
};
```

For reference, we'll also log something before and after processing a file:

```cpp
Action::ASTConsumerPointer
Action::CreateASTConsumer(clang::CompilerInstance& Compiler, llvm::StringRef) {
  return std::make_unique<Consumer>();
}

bool Action::BeginSourceFileAction(clang::CompilerInstance& Compiler,
                                   llvm::StringRef Filename) {
  ASTFrontendAction::BeginSourceFileAction(Compiler, Filename);
  llvm::outs() << "Processing file '" << Filename << "' ...\n";

  return true;
}

void Action::EndSourceFileAction() {
  ASTFrontendAction::EndSourceFileAction();
  llvm::outs() << "Finished processing file ...\n";
}
```

## `ASTConsumer`

The next step in the pipeline is the consumer we created inside the
`CreateASTConsumer` method. The consumer itself does not really have that much
functionality. Its main job is to dispatch the AST visitor, which will either be
a `RecursiveASTVisitor` or match callback when using the `ASTMatcher` library.
The `ASTConsumer` again provides a number of hooks that we can override to
process the AST at various stages or for various kinds of nodes. For example,
the `HandleTranslationUnitDecl` method is the main entry point to a translation
unit -- it is inside this function that we will instantiate our visitor.
Additional overrideable methods include `HandleVTable`, which is called with a
`CXXRecordDecl*` and lets us know that a vtable is required for the given class
(this is more an implementation detail of the clang frontend). On the other
hand, `HandleInlineFunctionDefinition()` is invoked with a `FunctionDecl*` every
time the definition of an inline function was completed. It is used by the code
generator (`CodeGen/CodeGenAction`), for example.

Note that `CreateASTConsumer` inside the frontend action is called as part of
`BeginSourceFile` (not `BeginSourceFileAction`), which is a non-virtual (thus
non-overrideable) function. Note that this means a consumer will persist for the
lifetime of a *translation unit*. Any state that you want to keep for longer
than that will have to be kept in the frontend action or be made a static
member.

`consumer.hpp`
```cpp
class Consumer : public clang::ASTConsumer {
public:
  void HandleTranslationUnit(clang::ASTContext &Context) override;
};

```

`consumer.cpp`
```cpp
void Consumer::HandleTranslationUnit(clang::ASTContext &Context) {
  Visitor()->TraverseDecl(Context.getTranslationUnitDecl());
}

```

## Visitation

The visitation of the AST nodes is the final and most interesting part of the
process. This is where we can do our actual analysis on particular declarations,
types, statements and so on. The two main ways of doing this right now is to
write a `RecursiveASTVisitor` subclass or use the `ASTMatcher` interface.

### `RecursiveASTVisitor`

We begin by using the `RecursiveASTVisitor` method to traverse the AST and visit
particular nodes of interest. The `RecursiveASTVisitor` is truly powerful and
allows us to visit all basic kinds of AST node (leaving us to do further
filtering and processing of nodes). This is again done by overriding methods
that are invoked with various kinds of nodes. These include:

* `TraverseDecl (Decl *D)`, to visit declarations,
* `TraverseAttr (Attr *At)`, to visit C++11 attributes,
* `TraverseTemplateName (TemplateName Template)`, to traverse template names (e.g. `X` in `X<int> var;`),
* `TraverseLambdaCapture (LambdaExpr *LE, const LambdaCapture *C, Expr *Init)` to visit an expression inside a lambda capture, where `Init` is the actual expression,
* `TraverseLambdaBody (LambdaExpr *LE, DataRecursionQueue *Queue=nullptr)`, to visit the bodies of a lambda.

Each of these traversals return a boolean which should be `true` if the
traversal is to continue or `false` if it is to stop after returning from this
node. Note that you can also configure whether the class is to do post-order or pre-order traversal via the `shouldTraversePostOrder()` function, which you can override. By default, it returns false (i.e. does pre-order traversal.)

The `RecursiveASTVisitor` can actually do three things (violating design
principles):

1. Traverse the AST with the `Traverse*` methods. This does some more advanced
stuff to enable the actual traversal and should thus seldomly be overriden,
2. For each node, beginning at its dynamic type (e.g. `FunctionDecl`), begins
walking up the clang class hierarchy and visits each node. For example, there
will be a step from `FunctionDecl` to `Decl`, each time calling `VisitDecl` with
a different dynamic type. This is done by the `WalkUpFrom*` methods,
3. Visit nodes via the `Visit*` functions, which are the main interface to users
are supposed to do the actual processing.

`visitor.hpp`
```cpp
class Visitor : public RecursiveASTVisitor<Visitor> {
public:
    virtual bool VisitDecl(Decl* Declaration);
    virtual bool VisitStmt(Stmt* Statement);
};
```

`visitor.cpp`
```cpp

bool Visitor::VisitDecl(Decl* Declaration) {
  Declaration->dump(llvm::outs());
}

bool Visitor::VisitStmt(Stmt* Statement) {
Statement->dump(llvm::outs());
}
```

### `ASTMatcher`

The second option we have for node visitation is to use the ASTMatcher library.
This is actually a much more powerful method when looking for very specific
kinds of nodes. An example is given below. We first have to change what we do
inside the consumer:

`consumer.hpp`
```cpp
class Consumer : public clang::ASTConsumer {
public:
  void HandleTranslationUnit(clang::ASTContext &Context) override;
};
```

`consumer.cpp`
```cpp
void Consumer::HandleTranslationUnit(clang::ASTContext &Context) {
  using namespace clang::ast_matchers;
  MatchFinder MatchFinder;
  Handler Handler;
  MatchFinder.addMatcher(functionDecl(), &Handler);
  MatchFinder.matchAST(Context);
}
```

and then write our simple visitor class (now as a match callback):

`visitor.hpp`
```cpp
class Handler : public clang::ast_matchers::MatchFinder::MatchCallback {
public:
  using MatchResult = clang::ast_matchers::MatchFinder::MatchResult;
  void run(const MatchResult& Result) override;
};
```

`visitor.cpp`
```cpp
void Handler::run(const MatchResult& Result) {
  const auto* Function = Result.Nodes.getNodeAs<clang::FunctionDecl>("root");
  Function->dump(llvm::outs());
}
```
