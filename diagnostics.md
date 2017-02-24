# Diagnostics

One of the core strong points of the clang compiler project, in comparison to
GCC, is its user-friendly error messages. Clang tries [very
hard](https://clang.llvm.org/diagnostics.html) to emit correct, precise and
helpful warnings and errors in all kinds of situations. One nice thing is that
next to warnings and errors (generally *diagnostics*), clang also has the
ability to provide correction hints with a very low false positive rate. Such
corrections are termed *FixItHints*.

Of course, since clang is built as a library, we also have access to these
diagnostics and FixItHints facilities. There is little official documentation on
this, but we can gain some interesting insight into how diagnostics work inside
clang by looking at the source code directly.

Most of the diagnostics code can be found in `clang/Basic/Diagnostic.h`, which
(unfortunately) contains many classes for FixItHints and diagnostics reporting.

## Classes

### DiagnosticsEngine

http://clang.llvm.org/doxygen/classclang_1_1DiagnosticsEngine.html

The `DiagnosticsEngine` class is the main actor in the clang's
diagnostic-reporting architecture. It mainly handles "meta" aspects of the
diagnostics mechanism, such as whether warnings should be reported as errors,
what the limit on number of template instantiations in a diagnostic is, whether
errors should be printed as a tree, whether colors should be used, to what
extent the compiler should do spell-checking and other options. Besides, the
`DiagnosticsEngine` combines two further parts of the diagnostics:
`DiagnosticsConsumer`s and `DiagnosticBuilder`s. The former are arbitrary
consumers of diagnostic output which are derived from `DiagnosticConsumer`. The
latter is a lightweight class to build error messages. This process can be
initiated by using `DiagnosticsEngine::Report`, which allows you to report an
error or warning and also attach a `FixItHint`. The `DiagnosticEngine` also
gives you access to the number of warnings and errors reported so far.

An important `enum` defined inside the `DiagnosticEngine` is the diagnostic
*level* (`Level`). Every diagnostic that is emitted is associated with a certain
level, such as `Warning`, `Error` or `Fatal`. On the one hand, this level
determines how the consumer will ultimately output or format the diagnostic. On
the other hand, it also controls whether or not the diagnostic is emitted at
all, since the `DiagnosticsEngine` can be configured to ignore all warnings, for
example.

### Diagnostic

The `Diagnostic` class encapsulates a shared pointer to a `DiagnosticEngine` and
delegates calls to it. It is especially useful to store information about the
currently "in-flight" error or warning (i.e. the one currently being built while
the `DiagnosticBuilder` hasn't been destructed yet). It is primarily used by
`DiagnosticConsumer`s, as they have a method `HandleDiagnostic()`', which takes
a `Diagnostic` object along with the level at which to emit that diagnostic.

In general, a diagnostic consist of the following pieces of information:

1. A unique ID, so that you can query the `DiagnosticEngine` for more information associated with that ID. For even more information about a diagnostic ID, you can use the `DiagnosticIDs` class.
2. The level at which the diagnostic will be emitted (`DiagnosticIDs::Level`).
3. A location in the source code at which the error occurred (`FullSourceLoc`). This is also where the caret will ultimately show up.
4. A message, simply as a `std::string`.
5. A vector of `CharSourceRange`s, that the consumer can make use of.
6. A vector of `FixItHint`s, to recommend corrections.

### DiagnosticBuilder

The `DiagnosticBuilder` is a very lightweight class that is part of the
error-emitting protocol. Basically, when you want to emit an error, you will
first request a unique diagnostic ID from the diagnostics engine. You create
this ID by calling `DiagnosticEngine::createCustomDiagID` and passing a message
and a level. Within the message you pass, you can write placeholders like `%0`
or `%1` (like `{0}` or `{1}` in Python), which you can then format using the
`Builder`. Moreover, you can pass along FixIt hints and source ranges, to be
highlighted by the final consumer (renderer). An example in code would look like
this:

```cpp
void EmitWarning(DiagnosticsEngine& DE, const Decl& MyDecl) {
  auto ID = DE.getCustomDiagID(DiagnosticsEngine::Warning,
                               "variable %0 is bad");

  DE.Report(MyDecl->getLocStart(), ID)
    .AddString(MyDecl->getNameAsString());

  // or ...

  auto builder = DE.Report(MyDecl->getLocStart(), ID);

  builder << MyDecl->getNameAsString();

  // or ...

  builder.AddTaggedVal(MyDecl->getNameAsString(),       
                       DiagnosticsEngine::ArgumentKind::ak_std_string);

  // Emit the diagnostic regardless of suppression settings
  // builder.setForceEmit();
}
```

Note that the builder will emit the diagnostic (internally, send it to the
diagnostics engine) *in the destructor*. Note that there also exist a set of
predefined diagnostic types with associated ID and (sometimes parameterized)
messages. You can find these in the `*.td` files under `include/clang/Basic/`.
Some examples include:

* `def err_fe_pch_file_overridden : Error<
    "file '%0' from the precompiled header has been overridden">;` (`DiagnosticSerializationKinds.td`)
* `def note_constexpr_no_return : Note<
  "control reached end of constexpr function">;` (`DiagnosticASTKinds.td`)
* `def note_constexpr_lshift_of_negative : Note<"left shift of negative value %0">;` (`DiagnosticASTKinds.td`)
* `def warn_nested_block_comment : Warning<"'/*' within block comment">,
  InGroup<Comment>;`

With these, you can make really short error reports like so:

```cpp
DE.Report(Location, diag::note_fixit_applied);
```

By the way, inside this folder you also find `DiagnosticsGroup.td`, which
includes the definitions of all warning groups. Here you will find out, for
example, that `-Wall` does not actually enable all warnings, but only a
relatively small subset.

### FixItHints

A *FixItHint* is a suggestion to the user about a particular region of source
code that could be inserted, deleted or modified to definitely fix a compiler
error. The false positive rate for such errors should be very small, i.e. the
hint should almost always be correct for the given situation. We can create our
own FixIt hints very easily and add them to a `DiagnosticBuilder` when we are
emitting a diagnostic.

Internally, a FixIt hint consists of:

* A `RemoveRange`, which is a `CharSourceRange` of source code that we want to recommend for deletion
* A `CharSourceRange` of existing source code that should be inserted at a particular insertion location. This is useful for modification.
* A (possibly empty) `std::string` of code to insert.
* A boolean flag, indicating whether or not the current insertion should be inserted before (true) previous insertions within that range (or else precisely at the specified location?).

To actually create a `FixItHint`, you use various factory functions. These are:

* `CreateInsertion`: Given a `SourceLocation` where to insert, a `StringRef` of code and the `BeforePreviousInsertions` flag, create a FixIt hint for a pure insertion;
* `CreateInsertionFromRange`: Given a `SourceLocation` where to insert, a `CharSourceRange` of existing source code to insert from and the `BeforePreviousInsertions` flag, suggest an insertion of that code range at the location;
* `CreateRemoval`: Given only a `CharSourceRange` (or, in an overload, a `SourceRange`), suggest the removal of that range;
* `CreateReplacement`: Given a `CharSourceRange` (or `SourceRange`) and a `StringRef` of code, suggest the replacement of the range with the string.
