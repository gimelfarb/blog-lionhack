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

<pre>
<code class="language-csharp">
Assembly assembly;
var memPtr = Marshal.GetHINSTANCE(assembly.ManifestModule);
</code>
</pre>

Adapting [some code already written][PEcode] to parse PE headers in .NET, we read the info directly from unmanaged memory using `Marshal.PtrToStructure` and `Marshal.ReadXXX` methods.

<pre>
<code class="language-csharp">
private static T FromMemoryPtr<T>(IntPtr memPtr, ref long index)
{
    var obj = (T)Marshal.PtrToStructure(new IntPtr(memPtr.ToInt64() + index), typeof(T));
    index += Marshal.SizeOf(typeof(T));
    return obj;
}
</code>
</pre>

First is the [MS-DOS header and stub program][msdn2], which then points to beginning of PE image (`IMAGE_FILE_HEADER`):

<pre>
<code class="language-csharp">
var index = startOffset;
var dosHeader = FromMemoryPtr&lt;IMAGE_DOS_HEADER&gt;(memPtr, ref index);
index = startOffset + dosHeader.e_lfanew + 4;
var fileHeader = FromMemoryPtr&lt;IMAGE_FILE_HEADER&gt;(memPtr, ref index);
</code>
</pre>

Depending on whether this is 32-bit or 64-bit image, we read [`IMAGE_OPTIONAL_HEADER32`][msdnStructs] or [`IMAGE_OPTIONAL_HEADER64`][msdnStructs]:

<pre>
<code class="language-csharp">
var optMagic = Marshal.ReadInt16(new IntPtr(memPtr.ToInt64() + index));
var is32bit = (optMagic != IMAGE_NT_OPTIONAL_HDR64_MAGIC); /* 0x20b */
if (is32bit)
    optionalHeader32 = FromMemoryPtr&lt;IMAGE_OPTIONAL_HEADER32&gt;(memPtr, ref index);
else
    optionalHeader64 = FromMemoryPtr&lt;IMAGE_OPTIONAL_HEADER64&gt;(memPtr, ref index);
</code>
</pre>

PE's [`IMAGE_OPTIONAL_HEADER`][msdnStructs] contains a [data directory][msdnDataDir] pointing to various informational sections. One of those is the [`IMAGE_DEBUG_DIRECTORY`][msdnStructs], which contains the debug info we are after.

<pre>
<code class="language-csharp">
// Find offset of the debug directory
index += 6 * Marshal.SizeOf(typeof(IMAGE_DATA_DIRECTORY));
var debugDirRef = FromMemoryPtr&lt;IMAGE_DATA_DIRECTORY&gt;(memPtr, ref index);

// Navigating to the DEBUG_DIRECTORY portion, which holds the debug information
index = debugDirRef.VirtualAddress;
var debugDir = FromMemoryPtr&lt;IMAGE_DEBUG_DIRECTORY&gt;(memPtr, ref index);
</code>
</pre>

Now that we are inside the "debug info" section, we can interpret it. For PDB symbols the type will always be `IMAGE_DEBUG_TYPE_CODEVIEW`, which has been used by Microsoft since early 1990s. The format of the actual data block is in the undocumented "RSDS" format, which is [pretty easy to decode][rsds].

<pre>
<code class="language-csharp">
private const UInt32 RSDS_SIGNATURE = 0x53445352; /* "RSDS" */

// http://www.godevtool.com/Other/pdb.htm
[StructLayout(LayoutKind.Sequential, Pack = 1, CharSet = CharSet.Ansi)]
private struct RSDS_DEBUG_FORMAT
{
    public UInt32 Signature; // "RSDS"
    [MarshalAs(UnmanagedType.ByValArray, SizeConst=16)]
    public Byte[] Guid;
    public UInt32 Age;
    // Follwed by zero-terminated PDB file path string
}
</code>
</pre> 

It just contains the full path of the original PDB file (produced when compiling), the PDB GUID and Age parameters. Exactly the info we are after!

<pre>
<code class="language-csharp">
index = debugDir.AddressOfRawData;
var rsdsInfo = FromMemoryPtr&lt;RSDS_DEBUG_FORMAT&gt;(memPtr, ref index);
if (rsdsInfo != RSDS_SIGNATURE) return null;

var path = Marshal.PtrToStringAnsi(new IntPtr(memPtr.ToInt64() + index));

return new AssemblyDebugInfo()
{
    Guid = new Guid(rsdsInfo.Guid),
    Age = rsdsInfo.Age,
    Path = path
};
</code>
</pre>

Wasn't so hard now, was it? You can see [full source code on GitHub][4].

[msdn2]: http://msdn.microsoft.com/en-us/magazine/cc301805.aspx 
[msdn3]: http://msdn.microsoft.com/en-us/library/windows/desktop/aa383751(v=vs.85).aspx
[msdn4]: http://msdn.microsoft.com/en-us/library/system.runtime.interopservices.marshal.gethinstance(v=vs.110).aspx
[msdnStructs]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms680198(v=vs.85).aspx
[msdnDataDir]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms680305(v=vs.85).aspx
[PEcode]: http://code.cheesydesign.com/?p=572
[rsds]: http://www.godevtool.com/Other/pdb.htm

## Generating extended Stack Trace

We now have to override how the Exception stack trace is rendered to include the extra information about the corresponding PDB symbols, and include info to be able to interpret stack frame locations later on.

StackFrame already provides us with **IL Offset** through [`frame.GetILOffset()`][msdn1] method. In order to be able to convert this later to a source file & line number we have to locate this offset in the PDB file. IL Offset is relative to the method, and is not absolute - therefore we must somehow identify the method to find it inside PDB.

Method's name (even fully qualified) is not reliable. There could be multiple overloads with the same name, and PDB does not store argument information for the methods to be able to distinguish them. But PDB does have a unique identifier for each symbol - **"metadata token"**. Luckily we can retrieve that easily using .NET Reflection via [MemberInfo.MetadataToken][mdtoken] property for each `MethodInfo`.

So we modify stack trace output generation to include these extra parameters:

<pre>
System.Exception: Test exception
   at <b>ProductionStackTrace.Test!0x0600000f!</b>ProductionStackTrace.Test.TestExceptionReporting.TestSimpleException() <b>+0xc</b>
==========
MODULE: ProductionStackTrace.Test => ProductionStackTrace.Test, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null; G:4e6f400982514fc29d72d9928819aac0; A:6
</pre>

The extra information includes:

  - Assembly name at the start of each line (e.g. "ProductionStackTrace.Test!")
    - This allows locating the matching assembly which contains the method for this stack frame
  - Method's **"metadata token"** to be able to uniquely identify it inside PDB later
  - **IL Offset** of the stack frame relative to the method as a hex number (e.g "+0xc")
  - Module mapping section at the bottom
    - This maps the assembly name to a fully qualified one, plus the extracted PDB GUID & Age parameters, necessary to locate the PDB later in the Symbol Store

The code that implements this formatting logic [is on GitHub](https://github.com/gimelfarb/ProductionStackTrace/blob/master/ProductionStackTrace/ExceptionReporting.cs).

With stack trace in this format we have enough information to be able to restore original source code line mapping at a later date, on a developer machine with access to the Symbol Store.

[mdtoken]: http://msdn.microsoft.com/en-us/library/system.reflection.memberinfo.metadatatoken(v=vs.110).aspx

## Restoring line numbers afterwards

Back in the lab we can use the extra information from the stack trace to find the matching PDB file in the Symbol Store, and lookup mapping of IL Offsets to original source line numbers.

I'll avoid spelling out the boring bit of parsing the information from the stack trace - you can see how that's implemented by looking at the [source on GitHub][4].

To find matching PDB file we use functionally already exported by [DbgHelp native library][dbghelp] (dbghelp.dll), which is a shared library commonly used by Windows debuggers and debugging tools. It exposes a number of `SymXXX` prefixed functions, which basically call through to [symsrv.dll][symsrv], implementing the Symbol Server protocol.

<pre>
<code class="language-csharp">
// Declarations to call native dbghelp.dll exports
private static class DbgHelp
{
    [DllImport("dbghelp.dll", CharSet = CharSet.Unicode)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static extern bool SymInitialize(
        IntPtr hProcess,
        [MarshalAs(UnmanagedType.LPTStr)] string UserSearchPath,
        [MarshalAs(UnmanagedType.Bool)] bool fInvadeProcess);

    [DllImport("dbghelp.dll", CharSet = CharSet.Unicode)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static extern bool SymCleanup(
        IntPtr hProcess);

    [DllImport("dbghelp.dll", CharSet = CharSet.Unicode)]
    [return: MarshalAs(UnmanagedType.Bool)]
    public static extern bool SymFindFileInPath(
      IntPtr hProcess,
      [MarshalAs(UnmanagedType.LPTStr)] string SearchPath,
      [MarshalAs(UnmanagedType.LPTStr)] string FileName,
      IntPtr id,
      uint two,
      uint three,
      uint flags,
      StringBuilder FilePath,
      SymFindFileInPathProc callback,
      IntPtr context
    );

    [return: MarshalAs(UnmanagedType.Bool)]
    public delegate bool SymFindFileInPathProc([MarshalAs(UnmanagedType.LPTStr)] string fileName, IntPtr context);

    public const uint SSRVOPT_GUIDPTR = 0x0008;
}
</code>
</pre>

Essentially the process boils down to calling `SymFindFileInPath` with `SSRVOPT_GUIDPTR` option to execute the search using Symbol Store semantics of finding PDB via filename, GUID and age parameters that were extracted from the binary PE header (during runtime):

<pre>
<code class="language-csharp">
// Search path is semicolon-separated list of SYMSRV paths (i.e. same
// paths you configure in VS under Tools > Options > Debugger > Symbols)
DbgHelp.SymInitialize(hProcess, searchPath, false);

var filePath = new StringBuilder(256);
var guidHandle = GCHandle.Alloc(guid, GCHandleType.Pinned);
try
{
    if (!DbgHelp.SymFindFileInPath(hProcess, null, pdbFileName,
        guidHandle.AddrOfPinnedObject(), (uint)age, 0,
        DbgHelp.SSRVOPT_GUIDPTR, filePath, null, IntPtr.Zero))
        return null;
}
finally
{
    guidHandle.Free();
    DbgHelp.SymCleanup(hProcess);
}
</code>
</pre>

Now that we have the matching PDB file path, we can load it using the [DIA (Debugger Interface Access) COM API][diasdk], included with Visual Studio. We use .NET's COM-interop to interact with it:

<pre>
<code class="language-csharp">
IDiaSession session;
IDiaDataSource src = new DiaSource();
src.loadDataFromPdb(filePath);
src.openSession(out session);
</code>
</pre>

To find the original source file number we have to locate the method, which contained the stack frame, inside the PDB file. Most unambiguous way to do that is to use the "metadata token", which is emitted for every symbol referenced by the PDB (and stored inside .NET runtime type reflection metadata):

<pre>
<code class="language-csharp">
IDiaSymbol symMethod;
_session.findSymbolByToken((uint)methodMetadataToken, SymTagEnum.SymTagFunction, out symMethod);

var rvaMethod = symMethod.relativeVirtualAddress;
rvaMethod += (uint)ilOffset;

IDiaEnumLineNumbers lineNumbers;
_session.findLinesByRVA(rvaMethod, 1, out lineNumbers);

foreach (IDiaLineNumber ln in lineNumbers)
{
    var sourceFile = ln.sourceFile;
    return new SourceLocation() { LineNumber = ln.lineNumber, SourceFile = (sourceFile == null) ? null : sourceFile.fileName };
}

return null;
</code>
</pre>

Note that _Relative Virtual Address (RVA)_ is the offset from the start of the module when loaded in memory. IL Offset is relative to the method in which the stack frame was captured, so we have to add it to the base relative virtual address of the method.

So here we are - with an **original source file and line number**, mapped manually by finding and interrogating the matching PDB symbols file with the available DIA SDK and DbgHelp libraries. Was that awesome or what!?

See full [implementation on GitHub][4], in ProductionStackTrace.Analyze project.

[dbghelp]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms679291(v=vs.85).aspx
[symsrv]: http://msdn.microsoft.com/en-us/library/windows/desktop/ms681416(v=vs.85).aspx
[diasdk]: http://msdn.microsoft.com/en-us/library/x93ctkx8.aspx

## NuGet package

And just so you don't complain that this is [too hard][2hard] - all of the above has been implemented and put together in an easy to use [NuGet package][2]. Simply install the package in your projects:

    PM> Install-Package ProductionStackTrace

And then use in the code to produce the full info stack trace:

    Console.WriteLine(ExceptionReporting.GetExceptionReport(ex));

See [Usage Instructions][3] for more info, and how to integrate with popular logging libraries. Source code is also [open-sourced on GitHub][4].

Analyzing the logs is also a breeze - install the matching "ProductionStackTrace.Analyze.Console" package and it will add a command to NuGet Package Manager Console right inside Visual Studio. See [Usage Instructions][3] for a quick info on how to get started.

[2hard]: http://www.urbandictionary.com/define.php?term=Too%20hard%20basket

---

There you have it. Apparently you can [have your cake and eat it too](http://en.wikipedia.org/wiki/You_can't_have_your_cake_and_eat_it) after all. You can **avoid deploying PDB symbols with your binaries**, and still be able to get exception stack traces that you can analyze quickly to get original source line mapping.

Go on now - impress your friends! 