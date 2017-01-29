---
title: Publishing .NET Core NuGet Packages
date: 2017-01-28 15:32:36
tags: csharp dotnet dotnetcore nuget
---

##### *NOTE: This post is only getting written because I couldn't find a reliable and up to date guide for creating public NuGet packages for the current state of .NET Core tooling. I'll do my best to either update this or delete it when it becomes irrelevant in order to prevent the confusion that made this a little difficult for me.*

So you've got some code you think would be helpful to your community. Or maybe you just want to save yourself the hassle of managing bits of code that you frequently use in your projects.

Thankfully, you have available to you the same mechanism that all your favorite dependencies use to distribute and retrieve packages: **NuGet**.

## About NuGet

NuGet is a package manager which is used for various things such as [Windows software](https://chocolatey.org/) and [deployment packaging](https://octopus.com/), but is primarily used by .NET developers for packaging and managing code libraries.

Developers have the ability to turn their software (typically Class Libraries) into NuGet packages (*.nupkg), publish those packages to a NuGet repository (https://www.nuget.org/ or their own private repositories), and have other developers reference the packages in their own projects.

With the recent changes to the .NET tooling and build infrastructure, there's been a lack of clarity about what this process is going to look like going forward. Since I recently decided to publish a package, I'll share the steps I took to create a package on nuget.org with the latest tools (specifically, `1.0.0-rc3-004530`).

## What you'll need

- [A nuget.org account](https://www.nuget.org/users/account/LogOn)
- [.NET Core SDK & CLI tools](https://github.com/dotnet/core/blob/master/release-notes/rc3-download.md)

It's pretty important following along that you have this exact version of the tools. These are changing *very* quickly and this guide may not be entirely correct or relevant for other versions.

If your version is different than the following, try reinstalling the dotnet tools.

``` PowerShell
>dotnet --version
1.0.0-rc3-004530
```

## Creating a Class Library

You can create NuGet packages out of your existing class libraries, but for this guide, I'll be starting from scratch with a new project, using the `dotnet` cli:

``` PowerShell
>mkdir FancyPackage
>cd FancyPackage
>dotnet new -t Lib # creates a new .NET project using the Class Library template
```

You should now have a `Class1.cs` and `FancyPackage.csproj`. That's right, a `csproj`. You may have heard of or fiddled with .NET Core in the past and been exposed to files such as `project.json` and `*.xproj`, but the path forward for .NET is `csproj` files and MSBuild.

Don't fret though! This isn't the `csproj` you're used to/annoyed with/cripplingly terrified of. Here's what a basic class library looks like:

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard1.4</TargetFramework>
  </PropertyGroup>
</Project>
```

Some things to note:

- 5 lines instead of the 47 lines from a VS 2015 Class Library template
- No build tools versiosn
- No build targets paths
- No build configurations
- No file includes
- No external references

The .NET Core tooling team has brought together familiar parts from the `csproj` xml you know and love, melded with the readability and convention-over-configuration tendencies of the popular `project.json`.

How does `Class1.cs` make its way into the compiled project, you ask? Projects using the .NET Core SDK have an implicit **opt-out** strategy for project files. This means that, by default, everything in the project directory is included in the build. This *also* probably means, at least for me, less CI build failures because I added or renamed a file, but forgot to Save All in VS before committing (blasted `csproj` changes!).

Also of note, the [NETStandard.Library](https://www.nuget.org/packages/NETStandard.Library/) NuGet package is implicitly included. This package can roughly be considered the alternative to the `dll`s that make up the .NET Framework SDK install.

## Building the Starter Class Library

Building our newly generated project is as straightforward as running:

``` PowerShell
>dotnet build
```