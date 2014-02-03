# Assembly Multi-Targeting in Visual Studio

Here's a little trick on how to target multiple versions of .NET Framework in your Visual Studio project simultaneously. This is especially useful when creating NuGet packages, which can hold multiple instances of your assembly targeting a different framework version.

## Visual Studio multi-targeting support

Visual Studio does have an advertised ["multi-targeting" support][vsmulti]. But it usually means that you have to pick just one version per project and stick with it.

[vsmulti]: TODO

## Compiling multiple targets simultaneously

1. Right-click project in Visual Studio, and choose "Unload Project"
2. Right-click again, and choose "Edit &lt;ProjectName&gt;.csproj"
3. Remove &lt;OutputPath&gt; property each configuration

    <pre><code>
    <strike>&lt;OutputPath&gt;bin\Debug\&lt;/OutputPath&gt;</strike>
    ...
    <strike>&lt;OutputPath&gt;bin\Release\&lt;/OutputPath&gt;</strike>
    </code></pre>

4. 