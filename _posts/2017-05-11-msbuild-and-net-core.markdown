---
layout: post
title: "MSBuild and .NET Core"
date: 2017-05-11 12:00:00 +0200
categories: coding
tags: msbuild .net
---

This post (contrary to the previous one) was written for a blog so it should be much easier to read. Nevertheless I encourage you to read the [first one]({% post_url 2017-04-29-msbuild-basics %}) - I'm going to assume that you understand the topics introduced there.

Let's look at the new stuff - MSBuild 15. Together with the last .NET Core release Microsoft made version 15 of MSBuild available. With this `*.csproj` (and `*.fsproj` and `*.vbproj`) files are *the* preferred way to manage projects. Some think this is a step back, others (me) consider this to be a thought out decision, but that's not what I want to talk about. I want the details, preferably technical.

# Target framework
The first and most important change is the technology underneath - as of this version MSBuild is built on top .NET Core. Thanks to that it can run on all platforms supported by .NET Core (Linux, Mac, Windows). This causes some complications when writing "general use" custom tasks, but this might be a topic for another time.

# Save your keystrokes
One of the changes inside the project files is how default `*.props` and `*.targets` files are referenced. This is how it was done so far:

```xml
<Project xmlns="…">
  <Import Project="$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props" />
  <!-- … -->
  <Import Project="$(MSBuildToolsPath)\Microsoft.CSharp.targets" />
</Project>
```

To make this more typing friendly the new format introduces an attribute on the `<Project>` node:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <!-- … -->
</Project>
```

**Caution:** right now these 2 methods do **NOT** give the same effect. The SDK is designed to aim at a few different targets with emphasis on .NET Core and .NET Standard (eg. `netstandard1.3`, `netcoreapp1.0`, `net46` but limited), whereas explicit imports work just like they did in the past (full framework, Xamarin, PCL, WiX etc.). Because of that we'll be seeing solutions with mixed project structures until SDKs get all the scenarios covered. You could have a few library projects targeting .NET Standard which use the SDK structure and a full framework project with explicit imports that uses those libraries.

# Are your defaults sane?
There's a very apparent difference in project file length. `File->New->Project->Class Library` generates a `csproj` with roughly 55 lines. `dotnet new classlib` generates 7 lines 2 of which are empty. So where did it all go? It has to be defined somewhere, right? Well, it was moved to defaults that are a part of the SDK. Many of those settings weren't ever changed so a good default value is more than welcome, but if you ever need to do that you still can. In case of properties you just need to overwrite them with your own value:

```xml
<PropertyGroup>
  <DefineConstants>TRACE;ENABLE_MY_HIDDEN_FEATURE</DefineConstants>
</PropertyGroup>
```

There's more to these defaults than just properties - there are also some items defined with wildcards. What does it mean? It means that when using SDK we don't have to specify even a single code file explicitly. It's enough that the file has the right extension and is placed under the project folder structure. That's a big improvement comared to the "single line per code file" policy we had so far, especially when considering editors that don't manage `csproj` files (I'm looking at you, VSCode). But what if I need precise control over the files included? Simple - disable the default items and define your own:

```xml
<PropertyGroup>
  <EnableDefaultCompileItems>false</EnableDefaultCompileItems>
</PropertyGroup>
<ItemGroup>
  <Compile Include="Program.cs" />
</ItemGroup>
```

Alternatively you could just remove the ones added by default and add only those you want:

```xml
<ItemGroup>
  <Compile Remove="@(Compile)" />
  <Compile Include="Program.cs" />
</ItemGroup>
```

**Caution:** `Compile` items are not the only ones defined by default. You can find out more about this here: <https://docs.microsoft.com/en-us/dotnet/articles/core/tools/csproj#default-compilation-includes-in-net-core-projects>

# Enable NuGet package restore
NuGet support has been built in into MSBuild. Yeah, you heard me right. From now on package refences can be kept in the `csproj` and there's no need to have `nuget.exe` in the repo or to manually download it on CI machines. That's how you use it:

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" Version="10.0.2" />
</ItemGroup>
```

Additionally if the project is supposed to be packaged and sent to NuGet.org (or any other package gallery) you can specify package parameters in the `csproj`:

```xml
<PropertyGroup>
  <PackageVersion>1.2.3</PackageVersion>
  <PackageId>My.Awesome.Package</PackageId>
  <Title>My Awesome Package</Title>
  <!-- … -->
</PropertyGroup>
```

# The End
The changes are *good*, but there are still some full framework scenarios that aren't covered (like XAML preprocessing), so use with care :) If you want to know about the changes in more detail read up here: <https://docs.microsoft.com/en-us/dotnet/articles/core/tools/csproj>

I'll be writing about my approach to versioning assemblies next (when I "find time"). This topic is much more practical, which automatically makes it better.