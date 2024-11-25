[Original: Gérald Barré,  Getting the date of build of a .NET assembly at runtime](https://www.meziantou.net/getting-the-date-of-build-of-a-dotnet-assembly-at-runtime.htm)


Getting the date of build of a .NET assembly at runtime
=======================================================

*   09/24/2018
*   Gérald Barré

It can be useful to get the compilation date of an assembly. For instance, you can display it in addition to the version number: `v1.2.3 built one day ago`.

[#](#method-1-linker-time)Method 1: Linker timestamp
----------------------------------------------------

You can retrieve the embedded linker timestamp from the `IMAGE_FILE_HEADER` section of the [Portable Executable](https://en.wikipedia.org/wiki/Portable_Executable) header:

    public static DateTime GetLinkerTimestampUtc(Assembly assembly)
    {
        var location = assembly.Location;
        return GetLinkerTimestampUtc(location);
    }
    
    public static DateTime GetLinkerTimestampUtc(string filePath)
    {
        const int peHeaderOffset = 60;
        const int linkerTimestampOffset = 8;
        var bytes = new byte[2048];
    
        using (var file = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.ReadWrite))
        {
            file.Read(bytes, 0, bytes.Length);
        }
    
        var headerPos = BitConverter.ToInt32(bytes, peHeaderOffset);
        var secondsSince1970 = BitConverter.ToInt32(bytes, headerPos + linkerTimestampOffset);
        var dt = new DateTime(1970, 1, 1, 0, 0, 0, DateTimeKind.Utc);
        return dt.AddSeconds(secondsSince1970);
    }

You can then get the build date using:

    GetLinkerTimestampUtc(Assembly.GetExecutingAssembly());

Warning

This method doesn't work if you compile your application with `/deterministic` flag ([documentation](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/compiler-options/code-generation?WT.mc_id=DT-MVP-5003978), [GitHub issue](https://github.com/dotnet/roslyn/issues/372))

[#](#method-2-assemblyinf)Method 2: AssemblyInformationalVersionAttribute
-------------------------------------------------------------------------

Using the .NET SDK, you can use an MSBuild property to define the value of the `AssemblyInformationalVersionAttribute`. So, you can inject the value of `DateTime.UtcNow` in the value of this attribute and then parse it at runtime to get the build date.

Open the csproj and add the following line:

csproj (MSBuild project file)

    <Project Sdk="Microsoft.NET.Sdk.Web">
      ...
      <PropertyGroup>
        <SourceRevisionId>build$([System.DateTime]::UtcNow.ToString("yyyyMMddHHmmss"))</SourceRevisionId>
      </PropertyGroup>
      ...
    </Project>

The value of `SourceRevisionId` is added to the metadata section of the version (after the `+`). The value is of the following form: `1.2.3+build20180101120000`

    private static DateTime GetBuildDate(Assembly assembly)
    {
        const string BuildVersionMetadataPrefix = "+build";
    
        var attribute = assembly.GetCustomAttribute<AssemblyInformationalVersionAttribute>();
        if (attribute?.InformationalVersion != null)
        {
            var value = attribute.InformationalVersion;
            var index = value.IndexOf(BuildVersionMetadataPrefix);
            if (index > 0)
            {
                value = value.Substring(index + BuildVersionMetadataPrefix.Length);
                if (DateTime.TryParseExact(value, "yyyyMMddHHmmss", CultureInfo.InvariantCulture, DateTimeStyles.None, out var result))
                {
                    return result;
                }
            }
        }
    
        return default;
    }

You can then get the build date using:

    GetBuildDate(Assembly.GetExecutingAssembly());

[#](#method-3-using-a-cus)Method 3: Using a custom Attribute
------------------------------------------------------------

By looking at the target file that generates the `AssemblyInfo.cs` (`C:\Program Files\dotnet\sdk\2.1.401\Sdks\Microsoft.NET.Sdk\targets\Microsoft.NET.GenerateAssemblyInfo.targets`), you can see that you can easily add other custom attributes to the file.

Tip

that you can use the [MSBuild binary log](/exploring-the-msbuild-log-using-the-binary-log-viewer.htm "Exploring a MSBuild binary log using the binary log viewer") as described in the previous post and search for `AssemblyInfo.cs` to find such kind of information

The target takes a parameter named `AssemblyAttribute`:

csproj (MSBuild project file)

    <WriteCodeFragment AssemblyAttributes="@(AssemblyAttribute)" Language="$(Language)" OutputFile="$(GeneratedAssemblyInfoFile)">
        <Output TaskParameter="OutputFile" ItemName="Compile" />
        <Output TaskParameter="OutputFile" ItemName="FileWrites" />
    </WriteCodeFragment>

If you look at the file, you can see how they are declared:

csproj (MSBuild project file)

    <Target Name="GetAssemblyAttributes" DependsOnTargets="GetAssemblyVersion;AddSourceRevisionToInformationalVersion">
      <ItemGroup>
        <AssemblyAttribute Include="System.Reflection.AssemblyCompanyAttribute" Condition="'$(Company)' != '' and '$(GenerateAssemblyCompanyAttribute)' == 'true'">
          <_Parameter1>$(Company)</_Parameter1>
        </AssemblyAttribute>
        <AssemblyAttribute Include="System.Reflection.AssemblyConfigurationAttribute" Condition="'$(Configuration)' != '' and '$(GenerateAssemblyConfigurationAttribute)' == 'true'">
          <_Parameter1>$(Configuration)</_Parameter1>
        </AssemblyAttribute>
        ...
      </ItemGroup>
    </Target>

So, you can easily add a custom attribute. First, declare the attribute in your code:

    [AttributeUsage(AttributeTargets.Assembly)]
    internal class BuildDateAttribute : Attribute
    {
        public BuildDateAttribute(string value)
        {
            DateTime = DateTime.ParseExact(value, "yyyyMMddHHmmss", CultureInfo.InvariantCulture, DateTimeStyles.None);
        }
    
        public DateTime DateTime { get; }
    }

Then, add the attribute declaration in the `.csproj`:

csproj (MSBuild project file)

    <Project Sdk="Microsoft.NET.Sdk">
    
      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>netcoreapp2.1</TargetFramework>
      </PropertyGroup>
    
      <ItemGroup>
        <AssemblyAttribute Include="BuildDateAttribute">
          <_Parameter1>$([System.DateTime]::UtcNow.ToString("yyyyMMddHHmmss"))</_Parameter1>
        </AssemblyAttribute>
      </ItemGroup>
    </Project>

It will generate the following file under `obj\Debug\netcoreapp2.1\BuildTime.AssemblyInfo.cs`:

    //------------------------------------------------------------------------------
    // <auto-generated>
    //     This code was generated by a tool.
    //     Runtime Version:4.0.30319.42000
    //
    //     Changes to this file may cause incorrect behavior and will be lost if
    //     the code is regenerated.
    // </auto-generated>
    //------------------------------------------------------------------------------
    
    using System;
    using System.Reflection;
    
    [assembly: BuildDateAttribute("20180901204042")] 👈
    [assembly: System.Reflection.AssemblyCompanyAttribute("Test")]
    [assembly: System.Reflection.AssemblyConfigurationAttribute("Debug")]
    [assembly: System.Reflection.AssemblyFileVersionAttribute("1.0.0.0")]
    [assembly: System.Reflection.AssemblyInformationalVersionAttribute("1.0.0")]
    [assembly: System.Reflection.AssemblyProductAttribute("Test")]
    [assembly: System.Reflection.AssemblyTitleAttribute("Test")]
    [assembly: System.Reflection.AssemblyVersionAttribute("1.0.0.0")]
    
    // Generated by the MSBuild WriteCodeFragment class.

Finally, you can get the build date using:

    private static DateTime GetBuildDate(Assembly assembly)
    {
        var attribute = assembly.GetCustomAttribute<BuildDateAttribute>();
        return attribute != null ? attribute.DateTime : default(DateTime);
    }

[#](#conclusion-08c35f)Conclusion
---------------------------------

MSBuild is very powerful. The [binary log](/exploring-the-msbuild-log-using-the-binary-log-viewer.htm "Exploring a MSBuild binary log using the binary log viewer") allows you to understand how it works and customize the build as you want. So, you can find some easy ways to reach your goal. The latest way is the one I choose for my projects because the `SourceRevisionId` property may be set by other targets such as [SourceLink](/how-to-debug-nuget-packages-using-sourcelink.htm "How to debug NuGet packages using SourceLink").