# LibClang

## Parsing C++ in Python with Clang

http://eli.thegreenplace.net/2011/07/03/parsing-c-in-python-with-clang

[LibClang](https://clang.llvm.org/doxygen/group__CINDEX.html) is a high-level C
API to clang. It allows us to do many things we could also do with LibTooling,
but provides a much lighter interface. Most importantly, because this interface
is so high-level, it is guaranteed to be (quite) stable. This is especially
useful for editors or other applications that want to use clang's power for
syntax-highlighting, code-completion or refactoring, but don't want to be at the
mercy of the clang upstream.

More precisely, libClang is a shared library that packages clang into a
high-level API for AST-traversal. What is nice is that next to the core C API,
there are also Python bindings (and [Go](https://github.com/go-clang/v3.9)).

libClang's Python bindings work on translation units. We would then usually
create an *index* on those translation units, so we can accessed the parsed AST.
An index is actually a group of translation units that can be compiled and
linked together. This function enables cross-TU referencing. To create an index
on a set of TUs in Python, we use the following code:

```python
index = clang.cindex.Index.create()
tu = index.parse(sys.argv[])
```

where `clang.cindex` is the Python binding module. `Index.create()` calls the
function `clang_createIndex()` in C. The method `index.parse()` then calls
`clang_parseTranslationUnit`, which is the main entry point to process a TU with
libClang. The result of `index.parse()` is a `CXTranslationUnit` in C and a
`TranslationUnit` in Python. This `TranslationUnit` has many exciting properties
that we can query and manipulate.

The most important attribute of the `TranslationUnit` is its `cursor`. A cursor
is basically a pointer/iterator that points to some node in a parsed AST. It
basically abstracts away the differences between entities in the AST (`Decl` vs
`Expr` vs `Type`) and provides a unified interface to manipulating the AST.

The most interesting attributes of a cursor are:

* `kind`: specifying the kind of node the cursor is currently pointing at,
* `spelling`: the name of the node (e.g. the name of the class or variable),
* `location`: the location of the node in the source code (row/column),
* `get_children`: the children of the node, to allow for further traversal.

For `get_children`, the Python and C APIs diverge slightly. The C APIs work with
visitation functions, whereas the Python API lets us do the traversal ourself. For example, given the following C/C++ code:

```cpp
int main() { int x; }
```

We can use the following recursive function to traverse the AST and print the kinds of every node we find:

```python
def visit(cursor):
  print(cursor.kind)
  for child in cursor.get_children():
    visit(child)
```

The Python bindings use [ctypes](https://docs.python.org/3.6/library/ctypes.html) to call the *libclang* shared library.

# Baby steps with libClang

An *index* groups multiple translation units together. Interestingly, with libclang, we need not necessarily parse the code ourselves, in the program. Rather, we can export an AST file with clang like so:

```shell
$ clang -emit-ast <file>
```

This will yield a file called `<file>.ast`, which we can then load into libclang with the following program:

```cpp
#include <clang-c/Index.h>

int main(int argc, char const* argv[]) {
  if (argc != 2) return 1;

  CXIndex index = clang_createIndex(0, 1);
  CXTranslationUnit tu = clang_createTranslationUnit(index, argv[1]);

  if (tu == nullptr) return 1;

  CXCursor root = clang_getTranslationUnitCursor(tu);

  clang_disposeTranslationUnit(tu);
  clang_disposeIndex(index);
  return 0;
}
```

libclang makes heavy use of the *visitor pattern*. As such, when we want to
traverse the AST (via cursors), we need to define a visitor function. It must
have the following signature:

```cpp
CXChildVisitResult visitor(CXCursor cursor, CXCursor parent, CXClientData clientData);
```

The relevant parts here are:

* `CXChildVisitResult`: A structure that tells libclang whether to continue visiting child nodes after a visitation. There are constants defined that you can return as valid values for this: `CXChildVisit_Break` to stop traversal, `CXChildVisit_Continue` to continue traversing the *siblings* of the current node and `CXChildVisit_Recurse` to continue recursing through the children of the current node;
* The `cursor`, which points to the node currently being traversed;
* The `parent`, which points to the parent of the node (or `NULL`, if none exists),
* Some `clientData`, which is a `void*` to any data you want to pass along during the traversal.

Once you have your visitation function defined, you can use
`clang_visitChildren` to traverse the tree. You pass it the root cursor from
which you want to start recursing, the visitor function/callback as well as
optionally any user data you want to take with you during the traversal. This allows us to write something like this:

```cpp
#include <iostream>
#include <string>
#include <clang-c/Index.h>

std::string getCursorKindName(CXCursorKind cursorKind) {
  CXString kindName = clang_getCursorKindSpelling(cursorKind);
  std::string result = clang_getCString(kindName);

  clang_disposeString(kindName);
  return result;
}

std::string getCursorSpelling(CXCursor cursor) {
  CXString cursorSpelling = clang_getCursorSpelling(cursor);
  std::string result = clang_getCString(cursorSpelling);

  clang_disposeString(cursorSpelling);
  return result;
}

CXChildVisitResult visit(CXCursor cursor, CXCursor, CXClientData data) {
  CXSourceLocation location = clang_getCursorLocation(cursor);
  if (clang_Location_isFromMainFile(location) == 0) {
    return CXChildVisit_Continue;
  }

  CXCursorKind cursorKind = clang_getCursorKind(cursor);

  const auto currentLevel = *(reinterpret_cast<unsigned int*>(data));

  std::cout << std::string(currentLevel, '-') << " "
            << getCursorKindName(cursorKind)
            << " (" << getCursorSpelling(cursor) << ")"
            << "\n";

  auto nextLevel = currentLevel + 1;

  clang_visitChildren(cursor, visit, &nextLevel);

  return CXChildVisit_Continue;
}

int main(int argc, char const* argv[]) {
  if (argc != 2) {
    std::cout << "Wrong number of arguments" << std::endl;
    return 1;
  }

  CXIndex index = clang_createIndex(1, 1);
  CXTranslationUnit tu;

  CXErrorCode rc = clang_createTranslationUnit2(index, argv[1], &tu);

  if (!tu) {
    std::cout << "TU is null" << std::endl;
    return 1;
  }

  CXCursor root = clang_getTranslationUnitCursor(tu);

  unsigned int level = 0;
  clang_visitChildren(root, visit, &level);

  std::cout << std::flush;

  clang_disposeTranslationUnit(tu);
  clang_disposeIndex(index);
  return 0;
}
```

You would pass to this an AST or PCH (pre-compiled header) file. The former can be created using `clang++ -emit-ast <file> -- <options>`.

https://clang.llvm.org/doxygen/group__CINDEX.html#ga51eb9b38c18743bf2d824c6230e61f93
http://bastian.rieck.ru/blog/posts/2015/baby_steps_libclang_ast/

## Exploring the Source

Note that libclang has good support for C++ (like templates):

https://clang.llvm.org/doxygen/group__CINDEX__CPP.html#gafe1f32ddd935c20f0f455d47c05ec5ab

And it is perfect for code completion libraries, as it has built-in support for
it (very high-level, literally `completeCodeAt`!):

https://clang.llvm.org/doxygen/group__CINDEX__CODE__COMPLET.html
