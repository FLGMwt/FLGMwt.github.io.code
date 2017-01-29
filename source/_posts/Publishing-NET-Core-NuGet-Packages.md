---
title: Publishing .NET Core NuGet Packages
date: 2017-01-29 15:32:36
tags: csharp dotnet dotnetcore nuget
---

*For a condensed version of this topic, head over to the [tl;dr version of this post](/2017/01/29/Publishing-NET-Core-NuGet-Packages-tl-dr/)*

So you've got some code you think would be helpful to your community. Or maybe you just want to save yourself the hassle of managing bits of code that you frequently use in your projects.

Thankfully, you have available to you the same mechanism that all your favorite dependencies use to distribute and retrieve packages: **NuGet**.

## About NuGet

NuGet is a package manager which is used for various things such as [Windows software](https://chocolatey.org/) and [deployment packaging](https://octopus.com/), but is primarily used by .NET developers for packaging and managing code libraries.

Developers have the ability to turn their software (typically class libraries) into NuGet packages (`*.nupkg`), publish those packages to a NuGet repository (https://www.nuget.org/ or their own private repositories), and have other developers reference the packages in their own projects.

With the recent changes to the .NET tooling and build infrastructure, there's been a lack of clarity about what this process is going to look like going forward. Since I recently decided to publish a package, I'll share the steps I took to create a package on nuget.org with the latest tools (specifically, `1.0.0-rc4-004706` and later).

## What you'll need

- [A nuget.org account](https://www.nuget.org/users/account/LogOn)
- [The NuGet Command Line Tools](https://docs.microsoft.com/en-us/nuget/guides/install-nuget#nuget-cli)
- [.NET Core SDK & CLI tools](https://github.com/dotnet/cli/#installers-and-binaries)

It's pretty important following along that you have this exact version of the tools. These are changing *very* quickly and this guide may not be entirely correct or relevant for other versions.

If your version is lower than the following, try reinstalling the dotnet tools from the site above.

``` PowerShell
dotnet --version
1.0.0-rc4-004706
```

## Creating a Class Library

You can create NuGet packages out of your existing class libraries, but for this guide, I'll be starting from scratch with a new project, using the `dotnet` cli:

``` PowerShell
mkdir MyFancyPackage
cd MyFancyPackage
dotnet new -t Lib # creates a new .NET project using the Class Library template
```

You should now have a `Class1.cs` and `MyFancyPackage.csproj`. That's right, a `csproj`. You may have heard of or fiddled with .NET Core in the past and been exposed to files such as `project.json` and `*.xproj`, but the path forward for .NET is `csproj` files and MSBuild.

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
- No build tools versions
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
dotnet build
```

This builds our project in the default configuration (`Debug`), and you should have a `.dll` in `bin/Debug/netstandard1.4/`.

## NuGet Project Properties

Previously, NuGet configuration was done with a separate `.nuspec` file which allowed setting package name, owner, version, and the like. With the recent releases of .NET Core SDK, that is consolidated as well into the project `.csproj`.

Edit your `csproj` to read as follows (replacing the `{}` blocks with your own info):

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

These properties in the `csproj` set the metadata which will be associated with your packagage when it is published to nuget.org. I suggest using your GitHub handle so that you don't have to worry about package collision.

## Creating the NuGet Package

Before the .NET Core SDK, the `nuget` command line tool was used for both creating and publishing packages. Because NuGet is becoming more integrated with the .NET Core workflow, we'll use the `dotnet` cli tool to build and bundle our package.

Run the following in the project root:

``` PowerShell
dotnet pack -c Release
```

Notice that it created a package using the `AssemblyName` and version information you supplied in the `csproj`. The `-c` or `--configuration` flag sets your build configuration. This also prescribes the output folder (`bin/Release` in this case).

## Publishing Your Package

Remember creating that nuget.org account? Well take a look at your [account page](https://www.nuget.org/account). At the bottom of the page, you'll see an "API Key" section. This API key is used to identify you as a package publisher and is required to create and manage your packages.

Let's try publishing your package now (substituing your own handle and API key):

``` PowerShell
nuget push bin/Release/<YOUR_HANDLE>.MyFancyPackage.0.0.1.nupkg <API_KEY> -Source https://www.nuget.org/api/v2/package
```

You should get a success message, and if you navigate back to your nuget.org account page and click "Manage My Packages", you'll see your new public package! Congratulations!

Right after you publish your package, you'll see a message that says "This package has not been indexed yet." This means that your package won't show up in search or be available to install just yet. For me, this only took about five minutes, but may be quicker or slower. Once this disappears though, you should be able to pull in your new package to any project you like!

# Testing Your New Package

Sure, we've put up some bytes on nuget.org, but do they work? We can build a small consuming app to try it out.

## Modifying the Library

Let's make the package a little more interesting so we can prove that we're using the new package.

Modify `Class1.cs` to read as follows:

``` cs
namespace MyFancyPackage
{
    public static class Class1
    {
        public static readonly string Message = "Hello world, from nupkg!";
    }
}
```

Now we'll build and publish it again to reflect our changes:

``` PowerShell
dotnet pack -c Release
nuget push bin/Release/<YOUR_HANDLE>.MyFancyPackage.0.0.1.nupkg <API_KEY> -Source https://www.nuget.org/api/v2/package
```

Uh oh, looks like that failed. That's because we already published a 0.0.1 version of our package and nuget.org does not allow modifying or deleting packages once they're published. You wouldn't want a dependency to break from under you, [would you](http://www.haneycodes.net/npm-left-pad-have-we-forgotten-how-to-program/)?

We'll need to bump our version back in the `csproj`:

``` xml
...
<Version>0.0.2</Version>
...
```

And try again:

``` PowerShell
dotnet pack -c Release
nuget push bin/Release/<YOUR_HANDLE>.MyFancyPackage.0.0.1.nupkg <API_KEY> -Source https://www.nuget.org/api/v2/package
```

All better! While that's indexing, let's get our testing project up and running.

## Test Project

Since our test project is going to have a `csproj` of its own, we'll want it in a different directory. Let's pop out and create it:

``` PowerShell
cd ..
mkdir MyConsumingApp
cd MyConsumingApp
dotnet new
```

This time, we used the default project template which is a console app. Let's give it a trial run:

``` PowerShell
dotnet restore
dotnet run
-> Hello world!
```

Nice! By now, our package should be ready, so let's pop open `MyConsumingApp.csproj` and add a reference to the package:

``` xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>netcoreapp1.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="{YourGitHubHandle}.MyFancyPackage" Version="0.0.2" />
  </ItemGroup>
</Project>
```

We've added something new: a `PackageReference`. These are the types of things you would have seen in traditional `packages.config` files or even in the more recent `project.json`. Once you add a project reference and run `dotnet restore`, you'll be able to use the third party code you depend on. Let's do that now:

``` PowerShell
dotnet restore
```

Everything should be in place. Now we can start using our package! Change `Program.cs` to look like this:

``` cs
using System;
using {YourGitHubHandle}.MyFancyPackage;

namespace ConsoleApp
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine(Class1.Message);
        }
    }
}
```

Moment of truth! Let's run it.

``` PowerShell
dotnet run
-> Hello world, from nupkg!
```

Well done! You're a published (NuGet) author!

If you have any questions about these steps, hit me up at [@ryanastelly](https://twitter.com/ryanastelly) and I'll do my best to help out. Cheers!