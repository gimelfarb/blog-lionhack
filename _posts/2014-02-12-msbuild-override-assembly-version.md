# MSBuild: Override Assembly Version

Common requirement when creating a build for your .NET project is to stamp assemblies with some build version number. Most likely you'll be utilizing some sort of Continuous Integration system, such as TFS Build, and want the produced assemblies to have the matching build number.

## Common technique - modify AssemblyInfo file

A [common technique][updVer1] is to locate and modify the AssemblyInfo source file, which contains the assembly-level `AssemblyVersion` attribute. This has been widely documented on the net - either [through TFS Activities][updVer2], or [through MSBuild][updVer3].

> The **problem** with this approach is that it modifies the source tree, i.e. it leaves `AssemblyInfo.cs` (or `.vb`) in a modified state!

This is _clumsy_ for several reasons:

- If testing MSBuild scripts locally, it modifies local files for each project, which you then have to manually revert each time
- If doing it through TFS Activity, then it is hard to test assembly versioning locally (on developer's machine)
- When checking out source from certain SCM systems (e.g. TFS), files are read-only by default, meaning we have to reset the read-only flag 

[updVer1]: http://www.codeproject.com/Articles/705482/Updating-Assembly-Versions-During-TFS-Builds
[updVer2]: http://www.ewaldhofman.nl/post/2010/05/13/Customize-Team-Build-2010-e28093-Part-5-Increase-AssemblyVersion.aspx
[updVer3]: http://weblogs.asp.net/srkirkland/archive/2010/12/07/simple-msbuild-configuration-updating-assemblies-with-a-version-number.aspx

## Better technique - override during build

A better technique is to override the `AssemblyVersion` attribute during the build process, without touching original source files.

The idea is simple - instead of modifying original `AssemblyInfo.cs` (or `.vb`) file, we create a temporary copy of it during the build process, and pass it to the compiler. Generated assembly then has the modified version number, and original source file is left untouched!

This is ideally included in some `BuildCommon.targets` file that you are already importing into every `.csproj` (or `.vbproj`). This way you can share the customization:

    <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
    <Import Project="$(SolutionDir)\BuildCommon.targets" />

Make sure to import it **AFTER** the common targets (e.g. "Microsoft.CSharp.targets"), because we will be overriding some of the default targets.

Create **BuildCommon.targets**:

    <?xml version="1.0" encoding="utf-8"?>
    <Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
        <!-- Below snippets go here -->
    </Project>

Inject a new target into the default build process:

    <!--
        Defining custom Targets to execute before project compilation starts.
    -->
    <PropertyGroup>
        <CompileDependsOn>
            CommonBuildDefineModifiedAssemblyVersion;
            $(CompileDependsOn);
        </CompileDependsOn>
    </PropertyGroup>

`$(CompileDependsOn)` will be used by the `Compile` task, and the above snippet injects a target called "CommonBuildDefineModifiedAssemblyVersion" into the default process. Lets's define this target:

    <!--
        Creates modified version of AssemblyInfo.cs, replaces [AssemblyVersion] attribute with the one 
        specifying actual build version (from MSBuild properties), and includes that file instead of the original AssemblyInfo.cs in the compilation.
        
        Works with both, .cs and .vb version of the AssemblyInfo file, meaning it supports C# and VB.Net projects simultaneously.
    -->
    <Target Name="CommonBuildDefineModifiedAssemblyVersion" Condition="'$(VersionAssembly)' != ''">
        <!-- Find AssemblyInfo.cs or AssemblyInfo.vb in the "Compile" Items. Remove it from "Compile"
             Items because we will use a modified version instead. -->
        <ItemGroup>
            <OriginalAssemblyInfo Include="@(Compile)" Condition="%(Filename) == 'AssemblyInfo' And (%(Extension) == '.vb' Or %(Extension) == '.cs')" />
            <Compile Remove="**/AssemblyInfo.vb" />
            <Compile Remove="**/AssemblyInfo.cs" />
        </ItemGroup>
        <!-- Copy the original AssemblyInfo.cs/.vb to obj\ folder, i.e. $(IntermediateOutputPath). The
             copied filepath is saved into @(ModifiedAssemblyInfo) Item. -->
        <Copy SourceFiles="@(OriginalAssemblyInfo)"
              DestinationFiles="@(OriginalAssemblyInfo->'$(IntermediateOutputPath)%(Identity)')">
            <Output TaskParameter="DestinationFiles" ItemName="ModifiedAssemblyInfo"/>
        </Copy>
        <!-- Replace the version bit (in AssemblyVersion and AssemblyFileVersion attributes) using regular expression. Use the defined property: $(VersionAssembly). -->
        <Message Text="Setting AssemblyVersion to $(VersionAssembly)" />
        <RegexUpdateFile Files="@(ModifiedAssemblyInfo)"
                    Regex="Version\(&quot;(\d+)\.(\d+)(\.(\d+)\.(\d+)|\.*)&quot;\)"
                    ReplacementText="Version(&quot;$(VersionAssembly)&quot;)"
                    />
        <!-- Include the modified AssemblyInfo.cs/.vb file in "Compile" items (instead of the original). -->
        <ItemGroup>
            <Compile Include="@(ModifiedAssemblyInfo)" />
        </ItemGroup>
    </Target>

Logic goes like this:

- Find `AssemblyInfo.cs` (or `.vb`) in the `@(Compile)` item, which represents all of the source files selected for compilation
- Remove it from `@(Compile)` item, as we will replace it with a modified copy
- Create a copy inside the `obj\` folder (`$(IntermediateOutputPath)`)
- Replace the version string inside `AssemblyVersion` and `AssemblyFileVersion` attributes
- Add the modified file to the `@(Compile)` item to be passed to the compiler

The only missing thing is the referenced `RegexUpdateFile` task, which is actually a custom task, because MSBuild does not have regex update capabilities. There is a FileUpdate task in [MSBuild Community Tasks project][msbuildtasks], which you can use if you want to import it. 

I don't want to import a whole lot of MSBuild Community Tasks just for this, so I define it quickly inside our **BuildCommon.targets**:

    <UsingTask TaskName="RegexUpdateFile" TaskFactory="CodeTaskFactory" AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.v4.0.dll">
        <ParameterGroup>
            <Files ParameterType="Microsoft.Build.Framework.ITaskItem[]" Required="true" />
            <Regex ParameterType="System.String" Required="true" />
            <ReplacementText ParameterType="System.String" Required="true" />
        </ParameterGroup>
        <Task>
            <Reference Include="System.Core" />
            <Using Namespace="System" />
            <Using Namespace="System.IO" />
            <Using Namespace="System.Text.RegularExpressions" />
            <Using Namespace="Microsoft.Build.Framework" />
            <Using Namespace="Microsoft.Build.Utilities" />
            <Code Type="Fragment" Language="cs">
                <![CDATA[
                try {
                    var rx = new System.Text.RegularExpressions.Regex(this.Regex);
                    for (int i = 0; i < Files.Length; ++i)
                    {
                        var path = Files[i].GetMetadata("FullPath");
                        if (!File.Exists(path)) continue;
                        
                        var txt = File.ReadAllText(path);
                        txt = rx.Replace(txt, this.ReplacementText);
                        File.WriteAllText(path, txt);
                    }
                    return true;
                }
                catch (Exception ex) {
                    Log.LogErrorFromException(ex);
                    return false;
                }
            ]]>
            </Code>
        </Task>
    </UsingTask>

That's it - now you can override the version on **ALL** assemblies produced by the MSBuild running against your solution (i.e. `.sln`). The only required setting is the `$(VersionAssembly)` property that needs to be supplied, in the form of `"major.minor.build.revision"`.

How to define this property is up to you - you can pass it externally to MSBuild via arguments, or have a process that reads the version number from somewhere, etc. In a future post I will talk about various suggested techniques for defining and managing a build version number.

[msbuildtasks]: https://github.com/loresoft/msbuildtasks

---

For now, just enjoy the new trick of how to set an output assembly version number without modifying your source tree!
