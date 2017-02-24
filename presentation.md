# clang

clang all the things

clang stands for the "Klang" of metal.

Chandler Carruth's home directory is under extra/home/chandlerc.

Mike calls me up and wants a tool. Looked at code, fortran looked better, please write me a tool.

LangOptions is a cool file. Don't like signed integer overflow? just change it. Get rid of GC. Also get rid of MSVC.

Show design of RewriteRope.h. So you're writing a compiler and you wanna support rewriting.

1. You have a long string and you want to efficiently insert and erase ranges
into/from that string. How do you represent the string? Answer: List of
Ropes/StringViews.

2. You have a list of string views that are ordered. You want to (1) be able to
search in that string and (2) insert into that list at arbitrary locations. What
data structure do you use? Answer: BTree.

Show some source code
Show LLVM IR

http://www.aosabook.org/en/llvm.html

GCC is monolithic, LLVM is modular on the macro and micro (e.g. optimizer) level.

Find clang in the slides. Give chocolate. If you do not like chocolate, it will
be donated to me.
