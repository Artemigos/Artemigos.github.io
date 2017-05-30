---
layout: post
title: "Assembly versioning with MSBuild"
# date: 2017-05-26 12:00:00 +0200
categories: coding
tags: msbuild .net
---

Ah, the problem of assembly versioning - always there to cause headaches. Maybe it's time to solve it once and for all?

# Requirements
This is what **I** want from a versioning system:

- vanilla MSBuild only - I don't want to depend on any external stuff for every single project
- partial SemVer is possible but not required - store a pre-release label in assembly attribute
- easily automated versioning - it should be as easy as passing an argument to MSBuild, which would enable versioning debug builds and automated releases
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
Now that we're set up let's actually figure out how to achieve what we want. I'll spare you the process of googling - this is what we'll do:

- collect the version information through properties - separate property for version number and prerelease suffix, which can be overwritten by MSBuild command line parameters
- output code with assembly version attributes using the `WriteCodeFragment` task: <https://docs.microsoft.com/en-us/visualstudio/msbuild/writecodefragment-task>
- create the code file in the project's `obj\` directory with the use of the `$(IntermediateOutputPath)` property - this file will not get commited to source control
- use the attributes in a way that somewhat supports the SemVer standard - `AssemblyVersionAttribute` and `AssemblyFileVersionAttribute` will hold the numbers and `AssemblyInformationalVersionAttribute` will hold full version with the prerelease label

# Setting up properties
TODO: properties themselves, making sure they have defaults and can be overwritten

# Preparing items for the code task
TODO: description of what is needed, actual code and maybe experimentation with what is being generated

# Running the task
TODO: writing the target, hooking up with build, handling the first build case

# Extra: auto-importing
TODO: show the new Directory.Build files from MSBuild 15.0
