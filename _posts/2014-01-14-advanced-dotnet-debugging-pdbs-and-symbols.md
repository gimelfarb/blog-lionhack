# Advanced .NET Debugging - PDBs and Symbol Stores

Debugging is the epitome of software development. Knowing about the available tools and how to use them effectively is [paramount to being a productive developer](http://www.wintellect.com/blogs/jrobbins/pdb-files-what-every-developer-must-know). In this installment we'll look at debugging symbols and how to use them like a pro.

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

It is important to remember, that in Release/Production, when JIT optimization is enabled, it may be difficult for CLR to [map optimized code back to IL instructions correctly](http://msdn.microsoft.com/en-us/library/ms231459(v=vs.110).aspx). Be aware when debugging production code that has been optimized - some variables won't evaluate, and debugger might show a wrong source code line.

---
**Experiment**:

  - Using the same `Dia2Dump.exe` tool from previous section, dump info inside a .NET PDB file

  You will notice it contains a lot less information than the native one. The important sections are the ones mapping function offsets to source code files and line numbers. This is how debugger can translate instruction pointer to a line number. And also how .NET's `Exception.ToString` can generate a stack trace with corresponding line numbers.

---

## How Debugger finds PDBs

Every binary (compiled to include debug info) includes a pointer to its symbols file. This is a property of [Microsoft PE (Portal Executable) format](http://msdn.microsoft.com/en-us/library/windows/desktop/ms680547(v=vs.85).aspx). One of the sections in the file is called "Debug Directory", and contains information about debugging symbols.

---
**Experiment**:

  - Compile any .NET application or a library (or take an existing one)
  - Run "Developer Command Prompt for VS"
  - On a Command Prompt use `dumpbin` utility to print header information in the assembly .EXE/.DLL

  `> dumpbin /headers MyAssembly.dll`
    
  The output should contain an info similar to the following:

![DUMPBIN Debug Info](http://static.lionhack.com/images/2014-01-14-advanced-dotnet-debugging-pdbs-and-symbols/DumpBin_DebugInfo.png)

  **NOTE:** This is not specific to .NET assemblies - this works for all native Windows binaries as well, since .NET uses the same common PE format.

---

The "Debug Directory" information is just a pointer to the PDB file. Note, however, that there are two more pieces of information:

  * GUID - uniquely identifying the PDB file beyond just the filename
  * Integer (called Age) - which is incremented on every build, if contents doesn't change

**GUID & Age from the binary MUST match the GUID & Age stored inside PDB file**. That is Visual Studio Debugger will refuse to load a PDB which does not match this signature, even if filename matches. This is a leading cause of confusion among the developers - _"Why is it not loading the freakin' PDB I am providing!?"_.

It makes sense - PDB is intricately linked to the offsets and layout of the binary for which it was produces. Any re-compilation of the binary invalidates that, and a new PDB must be generated.

> "PDB files are as important as source code!" -- [John Robbins' Blog](http://www.wintellect.com/blogs/jrobbins/pdb-files-what-every-developer-must-know)

Thus it is important for Debugger to find a PDB file that matches the GUID & Age parameters for a particular binary.

Visual Studio Debugger searches the following places for a matching PDB file:

  * The exact path embedded in "Debug Directory" block of the assembly (this works well when debugging locally an assembly you've just built - the path matches exactly)
  * In the base directory of the running application
  * In the directory from which assembly was loaded
  * In the Symbol Store cache
  * In the Symbol Store paths (_we talk about Symbol Store further below_)

Once PDB is found, debugger can load it, and be able to map memory offsets to original source code locations, enabling the familiar debugging experience of stepping through the code.

---
**Experiment**:

  - Start a .NET application in Debug mode (F5)
  - Open Modules windows from Debug > Windows > Modules menu item.

![Visual Studio Modules View](http://static.lionhack.com/images/2014-01-14-advanced-dotnet-debugging-pdbs-and-symbols/Debugging_Modules_View.png)

Modules view shows all the modules (assemblies) loaded by the current application, and whether or not a corresponding symbols file (PDB) was found. If a symbols file cannot be found, then you won't be able to step through its source code and will have difficulty setting Breakpoints effectively.

Now you know how to check if you'll be able to Debug an assembly or not. Check the Modules window!

  - Check where symbols file was loaded from, or which locations were searched to find it, by right-clicking and choosing "Symbol Load Information"

![Visual Studio Symbol Load Information](http://static.lionhack.com/images/2014-01-14-advanced-dotnet-debugging-pdbs-and-symbols/Symbol_Load_Info.png)

This is valuable to diagnose why a particular PDB file is not loading. (You did ensure that GUID & Age matches, right?)

---

## Do I need to deploy PDBs together with binaries?

One of the hotly debated issues about PDBs is whether they should be deployed together with binaries/assemblies. Let's discuss the various sides of this issue.

  * Firstly note that PDBs tend **get very large** - as your source code grows, so will PDBs, as they have to maintain a mapping to every source line of codes, and every variable declared. Although for .NET assemblies it is not as big as it is for native code (which also stores all the type information).
  * For **packaged software** that gets distributed to customers it is definitely not a good idea - not only does it increase the size of distributed package, PDB files contain extra information to help with reverse engineering the code, which a lot of software companies want to avoid to protect intellectual property.
  * For **internal software** it just increases the size of the software when deployed, and has some affect on memory usage and performance, as CLR has to load PDB files when Exception stack trace is generated.
  * By far the biggest argument for including PDBs is that it helps produce **exception stack traces** which contain a mapping to original source line numbers

For the issue of **exception stack traces** see the last section in this article - there are ways to work around it. For everything else there is a [Symbol Store](http://msdn.microsoft.com/en-us/library/windows/desktop/ms680693\(v=vs.85\).aspx) that we cover next.

Using Symbol Store/Server will not only help you debug your own applications, but inherently move you to the next level of being able to debug others' code, including .NET Framework itself.

> Consider this - Microsoft does not distribute PDB files for the Windows Operating system or Microsoft Office, and yet they are perfectly able to diagnose and debug issues in-house based on field reports. The trick is in the Symbol Store.

## Debugging Tools for Windows

Before going further it is useful to download and install [**Debugging Tools for Windows**](http://msdn.microsoft.com/en-us/windows/hardware/hh852365.aspx), which is a part of Windows DDK. Scroll down the page and there is a standalone installer (no need for a full DDK for our purposes).

Debugging Tools for Windows contains important tools and libraries (such as SYMSRV and SRCSRV) that are mentioned below.

## Symbol Stores - A wealth of information

Symbol Store is essentially an indexed database of symbol files (i.e. PDB files). Microsoft's [SYMSRV](http://msdn.microsoft.com/en-us/library/windows/desktop/ms681416\(v=vs.85\).aspx) is both a protocol for retrieving symbol files and a structure for storing them. "`symsrv.dll`" from Debugging Tools for Windows is the client DLL that encapsulates this logic.

PDB files inside Symbol Store are indexed by name and GUID, which means multiple versions of symbols for the same assembly can be stored and archived side-by-side. Symbol Store can be as simple as a shared directory or a file share on the network. Additionally, for public symbols, and HTTP protocol is also supported.

Microsoft, for example, [publishes symbols for all public releases of Windows Operation System](http://support.microsoft.com/kb/311503) at [http://msdl.microsoft.com/download/symbols](). Enabling "Microsoft Symbol Servers" under Tools > Options > Debugging > Symbols will mean that native stack traces will now show full internal function names, rather than cryptic memory addresses.

For .NET Framework, Microsoft publishes Reference Source Symbols at [http://referencesource.microsoft.com/](). This means having an ability to step through internal .NET Framework code when debugging, rather than be presented with a "Dissassembly" window. This also utilizes a SYMSRV extension to link symbols to remote source files (see next section).

Developers setup their Visual Studio debugger to make use of Symbol Servers, by configuring a list of them to try (under menu Tools > Options > Debugging > Symbols). Debugger will try to download symbols from the specified Symbol Servers, if it cannot find them locally or in the cache.

> If you don't have the Microsoft symbol servers listed above enabled yet, do so now. The debug experience with no symbols vs when appropriate symbols are loaded is night and day!

Symbol Store technology is also open to anyone to use, allowing developers to publish symbols for their own software, and aiding in debugging it a later date. It is a shame to see that many developers still don't use it at their workplaces. 

This workflow allows developers who did not compile the original code to be able to debug it, without having to distribute PDBs together with the application! If they point to the right Symbol Store, their debugger will simply download the matching PDB file (using Filename + GUID + Age as the search criteria). This is perfect for shared libraries, or debugging application running on the test server!

---
**Experiment**:

  - Make sure `C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x64` is on your PATH
  - Add a PDB file to local Symbol Store by running following:

  `> symstore add /f <path to PDB> /s C:\symbols /t "My Product"`

  This will create a new Symbol Store inside `C:\symbols` folder, and add a PDB file into it. Luckily you won't have to do this manually if you use TFS Build system, which includes a built-in Symbol Publishing (see "Debugging across enterprise" below).

--- 

## SYMSRV extension & PDB Indexing

A very useful extension to Symbol Server client library (symsrv.dll) is so called "Source Indexing" (or SRCSRV). [Source Indexing](http://msdn.microsoft.com/en-us/library/windows/hardware/ff556898\(v=vs.85\).aspx) is a process of appending additional information to PDB symbols file (an alternate SrcSrv stream) that contains a mapping of source file paths to where they can be retrieved from. 

Consider having to debug a 3rd party library for which symbols have been published. After symbols are loaded in the debugger, for a particular code location inside the library it will attempt to show the original source file. However the path to original file is local to the machine that compiled the original source code, and is not likely present on your machine.

Of course, it is possible to share source code by other means, and then direct debugger to a location of downloaded source code, when it pops up a dialog box asking for it. But it is inefficient at best. You also have to know what version the source code was at when it was compiled, and retrieve that version specifically.

**Source Indexing** solves all of that in a very clever manner. Essentially it creates a map between source file paths and a script to retrieve these files from some central location, i.e. source control system. Source Indexing process embeds a script inside PDB, which debugger knows how to execute (via SRCSRV extension).

TFS Builds come with a [built-in Source Indexing feature](http://msdn.microsoft.com/en-us/library/hh190722.aspx) that is enabled by default in your Build Definition. What's more is that the embedded script will reference the exact version of the source file from TFS Source Control! I think **that is a killer feature** of TFS Build system.

When debugger needs to display the source file (e.g. to show a location on the stack), it will execute the script, which will in turn **download matching source file** from Source Control automatically. You've only seen science fiction like that in the movies!

Debugging Tools for Windows actually comes with pre-built scripts to support other version control systems: Perforce, VSS, CVS and SVN. But you'll have to include the Source Indexing step manually in your build process by invoking [`Ssindex.cmd` script](http://msdn.microsoft.com/en-us/library/windows/hardware/ff558878\(v=vs.85\).aspx). 

In the next section we look at how to use this for an efficient developer debugging workflow.

> **NOTE:** Because SRCSRV executes a potentially arbitrary script, be careful about the source of your PDB files, and make sure you trust it.

---
**Experiment**:

  - Build a project using TFS Build system (ensure Index Source is "True" in Build Definition)
  - Take one of the PDB files from the build Drop Folder
  - Use `pdbstr` tool from Debugging Tools for Windows (under "C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x64\srcsrv")

  `> pdbstr -r p:<pdbfile> -i:out.txt -s:srcsrv`

  This will write the contents of SRCSRV stream to `out.txt` file where you can examine it. Notice the mapping for all the source files to a set of parameters to pass to a script. The script itself is at the top of the file.

---

## Debugging across enterprise

Armed with all of this knowledge on debugging symbols you can go forth and make debugging life that much easier across your entire organization. Debugging life as you know it will never be the same!

First step - setup a **Symbol Store** to be shared by all the teams that share functionality. Most often that would be the entire IT department, since, in the ideal world you are efficient and have enabled common library sharing across the company. At the very least set it up to be shared by all the teams you normally co-operate with.

Symbol Store can be as simple as a shared folder on the internal network that's visible and readable by all the developers, e.g. `\\fancy-dev-server\symbols`. You can restrict write access to the automated build service account only (e.g. for your TFS Builds).

Second step - enable **Source Indexing** in [your TFS Build definition](http://msdn.microsoft.com/en-us/library/hh190722.aspx) to embed versioned source paths into generated PDB files. This allows SRCSRV to retrieve file directly from source control to display during debugging! Set to publish symbols to the Symbol Store network share setup earlier.

![TFS Build Source Indexing Options](http://static.lionhack.com/images/2014-01-14-advanced-dotnet-debugging-pdbs-and-symbols/TFSBuild_SourceIndexing.png)

Third step - get every developer to setup **Symbol Server Paths** on their machines, via Visual Studio
under Tools > Options > Debugging > Symbols (add the `\\fancy-dev-server\symbols` share setup earlier).

![Visual Studio Debugging Symbols Paths](http://static.lionhack.com/images/2014-01-14-advanced-dotnet-debugging-pdbs-and-symbols/Debugging_Options_Symbols.png)

Now anyone debugging applications (in any environment - be it Production or Test) can get correct symbols loaded, together with auto-magical retrieval of matching source code directly from TFS version control!

Put on some black shades - you're officially cool!.

> **NOTE:** TFS 2013 introduced [Git source control](http://blogs.msdn.com/b/mvpawardprogram/archive/2013/11/13/git-for-tfs-2013.aspx) option, in addition to original TFVC. Source Indexing, at the moment, is [not supported for TFS Builds running against Git source control](http://social.msdn.microsoft.com/Forums/en-US/0a9fe95b-7acc-4a09-830f-f10d21f051cb/tfs-2013-preview-git-build-source-indexing).
>
> If you are brave, you can do this world a favor by hacking it together following the same pattern as the scripts in "C:\Program Files (x86)\Windows Kits\8.1\Debuggers\x64\srcsrv", which show support for CVS, Perforce, TFVC & VSS. Or ... you can wait until Microsoft updates Source Indexing support for Git.

## Debugging across open source

If you are working on an open source project, and/or developers using your binaries are not on the same network, you want to slightly modify the steps above, which are applicable to enterprise environments. Mainly you don't have the luxury of a network share that everyone can access.

Luckily there happen to be online services that would let you share symbol files across the open Internet. [SymbolSource.org](http://www.symbolsource.org/Public) allows you to push source code and symbol files for free, and exposes them in a format suitable for [SYMSRV and SRCSRV integration with Visual Studio Debugger](http://www.symbolsource.org/Public/Wiki/Using).

In essence, developers add `http://srv.symbolsource.org/pdb/Public` to their symbol paths, instead of the network share.

[SymbolSource.org](http://www.symbolsource.org/Public) is geared towards [NuGet](http://nuget.org)-based development. NuGet has a built-in integration with SymbolSource, and will generate `.symbols.nupkg` automatically, which can then be [pushed to SymbolSource.org](http://www.symbolsource.org/Public/Wiki/Publishing).

    > nuget pack -symbols <nuspec-or-project-file>
    > nuget push <package-file>

This auto-magically handles Source Indexing as well, so SRCSRV can automatically retrieve your source code from SymbolSource.org when the code is being debugged. Users of your open-source library will sure compose a ballad in your name!

## Stack trace & Line numbers

There can be an argument made about a special case that benefits from deploying PDBs together with the application (which you don't really want to do, having read this article). 

> When generating an exception stack trace, if PDB symbols are not present, **mapping to source file and line number is not available**.

This makes sense - CLR, running your .NET code, does not know about IL offset mapping back to your source code, and so stack trace will only display function names, but not the corresponding line number. Needless to say - this can **seriously hamper your experience in supporting a production/live system**.

What to do? On one hand, deploying PDBs together with binaries is a practice frowned upon, especially by those who understand the Symbol/Source Server concepts outlined above. It has the side-effect of slowing down your exception stack trace generation too, since CLR has to load all the relevant PDB files while doing so, plus use up RAM to hold them.

On the other hand, if you don't deploy them, you stack trace will be rather bland, and contain less info for pinpointing the issue. A typical approach in most organizations, with a _"just slap it together - we got bigger problems"_ attitude, would be to just deploy those PDBs and forget about it. While not a big deal in itself, it's one of the steps down a [slippery path of software entropy](http://pragprog.com/the-pragmatic-programmer/extracts/software-entropy). I's a valid approach, but one that would compromise all of the goodness we discussed so far, such as Source Indexing. And could lead to a performance degradation, if a lot of exceptions are being thrown.

**There must be a better way!** And there is. It involves writing enough information in your stack trace to re-create source file and line number mapping during diagnosis. But this article is already too long... This trick deserves its own post (coming soon).

---
---

Congratulation is you made it this far and actually read the whole thing! I hope this article will improve lives of many .NET developers, uninitiated in the art of effortless Debugging. Knowing your tools is the first step to enlightenment!