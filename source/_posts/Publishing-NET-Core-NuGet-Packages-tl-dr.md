---
title: Publishing .NET Core NuGet Packages (tl;dr)
date: 2017-01-29 11:39:31
tags: csharp dotnet dotnetcore nuget tl;dr
---

*If you want a thorough exploration of this topic, head over to the [detailed version of this post](/2017/01/28/Publishing-NET-Core-NuGet-Packages/)*

## Prerequisites

- [A nuget.org account](https://www.nuget.org/users/account/LogOn)
- [The NuGet Command Line Tools](https://docs.microsoft.com/en-us/nuget/guides/install-nuget#nuget-cli)
- [.NET Core SDK & CLI tools](https://github.com/dotnet/cli/#installers-and-binaries)

## Creating a Class Library

``` PowerShell
mkdir MyFancyPackage
cd MyFancyPackage
dotnet new -t Lib # creates class library project in current dir
dotnet restore # restores NuGet packages
dotnet build # builds library
```

## Adding Package Metadata

Modify your `csproj` to look like this, replacing any `{}` placeholders:

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard1.4</TargetFramework>
    <Version>0.0.1</Version>
    <AssemblyName>{YourGitHubHandle}.MyFancyPackage</AssemblyName>
    <PackageId>{YourGitHubHandle}.MyFancyPackage</PackageId>
    <Description>Sample package for .NET Core NuGets</Description>
    <Authors>{yourName}</Authors>
    <PackageProjectUrl>{yourGithubRepoUrl}</PackageProjectUrl>
    <PackageTags>sample dotnetcore nuget dotnetstandard</PackageTags>
  </PropertyGroup>
</Project>
```

## Build a Package From Your Library

``` PowerShell
dotnet pack -c Release # bundles your library as a .nupkg in bin/release
```

## Publish Your Package

Replace <API_KEY> with the API key on your [nuget.org account page](https://www.nuget.org/account):

``` PowerShell
nuget push bin/Release/<YOUR_HANDLE>.MyFancyPackage.0.0.1.nupkg <API_KEY> -Source https://www.nuget.org/api/v2/package
```

Done!

*If you want to see your package in action, follow [these steps](/2017/01/28/Publishing-NET-Core-NuGet-Packages/#Testing-Your-New-Package) in the detailed post.*