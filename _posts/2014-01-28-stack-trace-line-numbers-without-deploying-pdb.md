# Stack Trace line numbers without deploying PDBs

In a [previous post][1] I talked at length about PDB debugging symbols, how they enhance our life as developers, how they are used to map generated compiled binaries back to original source code, and how a Symbol Server is a must for everyone and that it eliminates the need to deploy PDBs on the target system (or distribute them with your shiny cool library).

One problem remained - .NET default `Exception` stack trace output **does not contain original source line number mapping**, if PDB symbols are not present at run-time. 

That's a bit of a let-down, since more often than not we are diagnosing a problem **AFTER** it had occurred, and don't always have a way to recreate the same conditions while under a Debugger. **Source line mapping in the logged Exception stack trace is a very useful bit of information**! (Especially if the code is a little hairy, with methods as long as ballads, and classes that used to rule the planet during [Jurassic](http://en.wikipedia.org/wiki/Dinosaur))

So will this be the undoing of the beautiful scheme of not having to deploy PDBs? There must be something we can do! And there is - read on.

> If you are after a quick solution, you can jump directly to a [NuGet package][2] that implements the solution described below. Open-sourced and completed with [usage instructions][3].

[1]: http://www.lionhack.com/2014/01/14/advanced-dotnet-debugging-pdbs-and-symbols/
[2]: https://www.nuget.org/packages/ProductionStackTrace
[3]: https://github.com/gimelfarb/ProductionStackTrace#productionstacktrace
[4]: https://github.com/gimelfarb/ProductionStackTrace
[5]: http://www.lionhack.com/2014/01/14/advanced-dotnet-debugging-pdbs-and-symbols/#doineedtodeploypdbstogetherwithbinaries

## Why avoid deploying PDBs

First, let's address the begging question: why would you want to avoid deploying PDBs in the first place?

I wrote about it already in the [previous blog post][5]. But here's the summary again:

  - **PDBs are large** - usually much larger than the compiled assemblies themselves
    - Your distribution size is unnecessarily big due the size of PDBs (more than doubled)
    - PDBs are loaded into memory by CLR to generate original source mapping in stack traces, so the memory is wasted too
  - **Symbol lookup takes times** - Exception stack trace generation with loaded symbols is a magnitude slower than without
  
There is also the point about **obfuscation** - if you wanted to prevent reverse engineering, the PDB files contain extra info that you wouldn't want to share around.

## Solution overview

We already know from the [previous blog post][5] that assembly binary DLL/EXE is stored in [Microsoft Portable Executable (PE) format][PE], where one of the headers contains a reference to the matching PDB symbols file. Reference contains GUID and Age parameters that have to match the PDB symbol file, in addition to filename, to be loaded by the Debugger.

**PDB reference parameters** have to be known by whoever analyzes the stack trace to be able to find and load matching symbols info from the [Symbol Store][SymStore].

For any `Exception` we can get a `StackTrace`, where we would know the IL Offset of every stack frame ([`frame.GetILOffset()`][msdn1]). Without loaded PDB symbols we wouldn't get the original source file mapping, but we don't really need it right there and then! (Plus we know that getting that mapping during runtime slows things down)

All we need is to save this **IL Offset**. Then during log analysis we can map it to the original source line numbers using matching PDB symbols from the [Symbol Store][SymStore]. Basically doing the same thing that .NET CLR would do for us - but instead of doing it at runtime, we do it offline, back in the lab!

Sounds like a plan.

[PE]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms680547(v=vs.85).aspx
[SymStore]: http://www.lionhack.com/2014/01/14/advanced-dotnet-debugging-pdbs-and-symbols/#symbolstoresawealthofinformation
[msdn1]: http://msdn.microsoft.com/en-us/library/system.diagnostics.stackframe.getiloffset(v=vs.110).aspx

## Extracting PDB reference from Assembly

PDB reference is required if we want to find the matching PDB later during analysis. Finding this info requires looking at the native PE binary that houses the Assembly, i.e. the DLL/EXE file.

PE format is [well-known and documented][PE]. Don't be afraid to parse it - it's just a file, with a specific binary format. Once you look into it and look under the hood, it really isn't that scary.

Although it would be nice to avoid having to read the file from disk, since we know the binary is already loaded into memory (it's executing when you get the exception stack trace, so, of course, it's loaded!). If we could only just find where it is ...

Here's where knowing underlying native Windows API is helpful. Since assemblies are just Windows modules, they are subject to the [same rules][msdn2]. If we know [HINSTANCE][msdn3] of a native module, then we know it's base address in memory. And if we know the base address, and we know it starts with the PE header information, we can easily parse it!

.NET's [`Marshal.GetHINSTANCE(Module)`][msdn4] does exactly that - it returns us the HINSTANCE handle for the loaded assembly module, which is the unmanaged memory address pointer to the start of the PE-format binary.

Adapting [some code already written][PEcode] to parse PE headers in .NET, we read the info directly from unmanaged memory using `Marshal.PtrToStructure` and `Marshal.ReadXXX` methods.

TODO: insert code

[msdn2]: http://msdn.microsoft.com/en-us/magazine/cc301805.aspx 
[msdn3]: http://msdn.microsoft.com/en-us/library/windows/desktop/aa383751(v=vs.85).aspx
[msdn4]: http://msdn.microsoft.com/en-us/library/system.runtime.interopservices.marshal.gethinstance(v=vs.110).aspx
[PEcode]: http://code.cheesydesign.com/?p=572

## Generating extended Stack Trace

## Restoring line numbers afterwards


## NuGet package

All of the above has been implemented and put together in an easy to use [NuGet package][2]. Simply install the package in your projects:

    PM> Install-Package ProductionStackTrace

And then use in the code to produce the full info stack trace:

    Console.WriteLine(ExceptionReporting.GetExceptionReport(ex));

See [Usage Instructions][3] for more info, and how to integrate with popular logging libraries. Source code is also [open-sourced on GitHub][4].

---

There you have it. You can [have your cake and eat it too](http://en.wikipedia.org/wiki/You_can't_have_your_cake_and_eat_it) after all. You can **avoid deploying PDB symbols with your binaries**, and still be able to get exception stack traces that you can analyze quickly to get original source line mapping.

Go on now - impress your friends! 