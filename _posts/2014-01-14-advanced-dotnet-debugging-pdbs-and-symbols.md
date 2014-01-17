# Advanced .NET Debugging - PDBs and Symbol Stores

Debugging is the epitome of software development. Knowing about the available tools and how to use them effectively is paramount to being a productive developer. In this installment we'll look at debugging symbols and how to use them like a pro.

## PDB - What's inside

I will assume you know the basics already. You already know about those `<AssemblyName>.pdb` files that get generated when you compile your .NET application. You also know that these files are used by Visual Studio debugger to allow you to step through a running code, set breakpoints, examine variables and memory data structures, and navigate the thread stacks.

PDB stands for [Program Database](http://msdn.microsoft.com/en-us/library/ff558825\(v=vs.85\).aspx), and it's been in use since [Visual C++ 1.0 debuted in 1993](https://support.microsoft.com/kb/121366) as a container of debug information for all Windows application. PDB is also called "**debugging symbols**". Symbols have been used for debugging since [way back in the mainframe era](http://en.wikipedia.org/wiki/Debug_symbol), so it's nothing new.

Since executables contain machine code, and humans are [notoriously bad at reading machine code](http://programmers.stackexchange.com/questions/126095/resources-on-learning-to-program-in-machine-code), symbol files (e.g. PDBs) contain a mapping between the high-level language, such as C++, that was used to write the source code and the compiler-generated native code. This includes:

  * Original compilation settings and compiler options
  * List of all the source modules and source files compiled
  * List of all functions, with their names and arguments
  * List of all the local and global variables, and their memory offsets
  * Mapping of each machine code memory offset to a source file and line number
  * Mapping of custom types (classes/structs) to their flat memory representation

Using this information, a debugger can _interpret_, at any given time, how memory pointers on the stack correspond to original source code lines, and how memory addresses map to variables and complex structure types.

---
**Experiment**:
  
  - Go to `C:\Program Files (x86)\Microsoft Visual Studio <version>\DIA SDK\Samples\DIA2Dump`
  - Open `dia2dump.sln` and build it
  - Go to generated `Debug` folder on Command Prompt
  - Run `dia2dump.exe dia2dump.pdb >dia2dumpout.txt`

    Dia2Dump is a little sample application that uses [DIA SDK (Debug Information Access)](http://msdn.microsoft.com/en-us/library/x93ctkx8.aspx) to parse PDB files and print out information inside them. Check out generated `dia2dumpout.txt` file to get an idea what is contained inside a typical symbol file for a native application.

---

## PDB in .NET

.NET also adopted the PDB format for storing debug information for managed applications (a testament to its flexibility). Although for a .NET assembly a PDB file contains far less information than it does for native code. It simply does not need to - managed code already contains the ["type metadata"](http://msdn.microsoft.com/en-us/library/4y7k7c6k\(v=vs.90\).aspx) as part of its [Reflection capability](http://msdn.microsoft.com/en-us/library/f7ykdhsy\(v=vs.110\).aspx), and information on all the method names and arguments in the [expressive IL form](http://en.wikipedia.org/wiki/Common_Intermediate_Language).

For a .NET assembly a symbol file contains only:

  * Mapping of each **IL offset** to a source file and line number
  * Names of the local variables in each function block, and their memory offset

Armed with this information, and in conjunction with type metadata from the assembly itself, a debugger can figure out, at any given time, where each stack frame maps to in the original source code, and where all the variables are located in memory (together with their typed representation).

Note, that for managed code the mapping is between IL offsets and original source code. .NET CLR converts IL to native instructions before it runs (Just-in-Time compilation), and provides an attached debugger with a [mapping between native code and IL](http://msdn.microsoft.com/en-us/library/bb384548\(v=vs.110\).aspx), which can then be mapped to source code using PDB info.

It is important to remember, that in Release/Production, when JIT optimization is enabled, it may be difficult for CLR to [map optimized code back to IL instructions correctly](http://msdn.microsoft.com/en-us/library/ms231459(v=vs.110).aspx). Be aware when debugging production code that has been optimized.

---
**Experiment**:

  - Using the same `Dia2Dump.exe` tool from previous section, dump info inside a .NET PDB file

  You will notice it contains a lot less information than the native one. The important sections are the ones mapping function offsets to source code files and line numbers. This is how debugger can translate instruction pointer to a line number. And also how .NET's `Exception.ToString` can generate a stack trace with corresponding line numbers.

---

## Debugging Tools for Windows
## Symbol Stores - A wealth of information
## SYMSRV extension & PDB Indexing
## Debugging across enterprise
## Debugging across open source
## Stack trace & Line numbers