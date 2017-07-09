---
layout: post
title: "Assembly versioning with MSBuild"
# date: 2017-05-26 12:00:00 +0200
categories: coding
tags: msbuild .net
---

Ah, the problem of assembly versioning - always there to cause headaches. Maybe it's time to solve it once and for all? Well... we can at least try 😃

# Requirements
This is what **I** want from a versioning system:

- vanilla MSBuild only - I don't want to depend on any external stuff for every single project
- partial SemVer is possible but not required - store a pre-release label in assembly attribute
- easily automated versioning - it should be as easy as passing an argument to MSBuild, which would enable versioning CI builds and automated releases
- no changes in the code repository - when I run a build locally I don't want any new files or modifications to existing ones
- extra: no need to modify every project in solution

# Preparations
For the sake of simplicity I'll assume the default folder structure for .NET solutions. This is how it looks:

```
MyProj
|
+ MyProj.Executable
|\
| | MyProj.Executable.csproj
|
+ MyProj.Library
|\
| | MyProj.Library.csproj
|
| MyProj.sln
```

To keep the versioning targets separate from any single project we will keep them in separate files.
A common practice is to have `Name.props` and `Name.targets` files so we will do that.
This means we have to add `Versioning.props` and `Versioning.targets` on the solution level. Let's add the default content to them right away:

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">

</Project>
```

We can also create a target for applying the versioning, since we're gonna need it anyway. Let's add it to the targets file:

```xml
<Target Name="ApplyVersioning">

</Target>
```

At this point we might as well import our versioning files into the `csproj`s:

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Import Project="..\Versioning.props" />
    <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" Condition="Exists('$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props')" />

    <!-- ... -->

    <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
    <Import Project="..\Versioning.targets" />
</Project>
```

It's important to note that the props file is imported before the `Microsoft.Common.props` and the targets file is imported after the `Microsoft.CSharp.targets`.
**If we were to create a NuGet package with those files and then install it, this is where NuGet would place the imports.**
Having them in this order is helpful when planning to package them and upload to a NuGet gallery.

# The *how*
Now that we're set up let's actually figure out how to achieve what we want. I'll spare you the process of searching - this is what we'll do:

- collect the version information through properties - separate property for version number and prerelease suffix, which can be overwritten by MSBuild command line parameters
- output code with assembly version attributes using the `WriteCodeFragment` task: <https://docs.microsoft.com/en-us/visualstudio/msbuild/writecodefragment-task>
- create the code file in the project's `obj\` directory with the use of the `$(IntermediateOutputPath)` property - this file will not get commited to source control
- use the attributes in a way that somewhat supports the SemVer standard - `AssemblyVersionAttribute` and `AssemblyFileVersionAttribute` will hold the numbers and `AssemblyInformationalVersionAttribute` will hold full version with the prerelease label

# Setting up properties
First we will deal with data that can be evaluated right away after build starts - the version properties. We will put them in the `Versioning.props` file.
The properties that will usually have values provided from command line are `AssemblyVersion`, `AssemblyVersionSuffix` and `Author`.
The last two can be empty, but `AssemblyVersion` always needs a value. Let's define a default value:

```xml
<PropertyGroup>
    <AssemblyVersion>0.0.1</AssemblyVersion>
</PropertyGroup>
```

We have a default value, but can it be changed through command line? The answer is yes. The way MSBuild evaluates properties makes command line properties "global" which are always preferred over properties defined in the xml.

Let's prepare the other values we need. For partial semver support we will need a `version-suffix` value, but in case there's no suffix defined the value should hold only the version number. Let's create a `AssemblyVersionWithSuffix` property for that:

```xml
<AssemblyVersionWithSuffix>$(AssemblyVersion)</AssemblyVersionWithSuffix>
<AssemblyVersionWithSuffix Condition="'$(AssemblyVersionSuffix)' != ''">$(AssemblyVersion)-$(AssemblyVersionSuffix)</AssemblyVersionWithSuffix>
```

What's happening here is that we assign the version-only value to the property and if suffix is defined we overwrite it right away with the version-suffix variant.

Last but not least let's prepare a copyright string:

```xml
<AssemblyCopyright>Copyright © $(Author) $([System.DateTime]::Now.Year)</AssemblyCopyright>
```

# Preparing items for the code task
At this point we're pretty much done with the props file. We'll now need to put those values into a code file and include it in the compilation process.
The `WriteCodeFragment` task is perfect for that - it allows us to write a series of assembly attributes to a code file, which can be parametrized through special items. According to docs the items should have the following structure:

```xml
<ItemName Include="AssemblyAttributeName">
    <_Parameter1>AttributeConstructorParameter</_Parameter1>
</ItemName>
```

Since we have our requirements defined and values prepared let's turn them into items:

```xml
<ItemGroup>
    <Attr Include="AssemblyVersion">
        <_Parameter1>$(AssemblyVersion)</_Parameter1>
    </Attr>
    <Attr Include="AssemblyFileVersion">
        <_Parameter1>$(AssemblyVersion)</_Parameter1>
    </Attr>
    <Attr Include="AssemblyInformationalVersion">
        <_Parameter1>$(AssemblyVersionWithSuffix)</_Parameter1>
    </Attr>
    <Attr Include="AssemblyCopyright">
        <_Parameter1>$(AssemblyCopyright)</_Parameter1>
    </Attr>
</ItemGroup>
```

One last thing we need to do before writing the target is to set up the path to the versioning code file and enrol it for compilation:

```xml
<PropertyGroup>
    <VersionsFile>$(IntermediateOutputPath)_ver.cs</VersionsFile>
</PropertyGroup>
<ItemGroup>
    <Compile Include="$(VersionsFile)" />
</ItemGroup>
```

We want it in the intermediate output directory, because it's just a byproduct that shouldn't be tracked by source control.

# Running the task
Everything's ready, so we can generate the code. We will do that in the `ApplyVersioning` target we prepared earlier.
The first thing we need to do is create the directory for the code file. Since this is the `obj/` directory it will be there most of the time, but if this is the first build it might not exist yet. Let's take care of that:

```xml
<MakeDir Directories="$([System.IO.Path]::GetDirectoryName('$(VersionsFile)'))" />
```

Now we can finally generate the code:

```xml
<WriteCodeFragment AssemblyAttributes="@(Attr)" Language="C#" OutputFile="$(VersionsFile)" />
```

Easy enough. Let's add some debug info and the target should be ready:

```xml
<Message Text="Applied versioning attributes:" />
<Message Text="  %(Attr.Identity) = %(Attr._Parameter1)" />
```

At this point the only thing left to do is to make sure that the target actually runs before every build.
To do that we'll modify the built-in `BuildDependsOn` property:

```xml
<PropertyGroup>
    <BuildDependsOn>ApplyVersioning;$(BuildDependsOn)</BuildDependsOn>
</PropertyGroup>
```

With this the targets file is finished and we can apply a version during build with command line arguments:

`msbuild /p:AssemblyVersion=0.10.3 /p:AssemblyVersionSuffix=dev /p:Author=Me`

# Extra: auto-importing
We managed to build a solution that satisfies all the requirements but there's one more thing that bugs me.
Every time a new project is added to the solution you need to remember to add the imports at the beginning and the end.
Wouldn't it be great if it happened automatically? For that MSBuild 15 comes to the rescue.

It turns out that in that version every project will automatically look for and import `Directory.Build.props` and `Directory.Build.targets` files in the directories above them. This is perfect for us - all we have to do is change the file names of `Versioning.props` and `Versioning.targets`. Alternatively if you need them to be separate, you can create simple `Directory.Build` files beside the sln file with just imports.

*Directory.Build.props*
```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Import Project="Versioning.props" />
</Project>
```

*Directory.Build.targets*
```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
    <Import Project="Versioning.targets" />
</Project>
```

This way we won't ever have to worry about it again. *(One can hope at least...)*
