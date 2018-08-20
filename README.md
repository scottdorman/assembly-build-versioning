# assembly-build-versioning

Visual Studio has always allowed you to specify a version number for your .NET projects using the `[assembly:AssemblyVersionAttribute("2.0.1")]` syntax, usually in an `AssemblyInfo.cs` file. You can even use some special syntax to generate the build or revision number automatically.

> You can specify all the values or you can accept the default build number, revision number, or both by using an asterisk (\*). For example, [assembly:AssemblyVersion("2.3.25.1")] indicates 2 as the major version, 3 as the minor version, 25 as the build number, and 1 as the revision number. A version number such as [assembly:AssemblyVersion("1.2.\*")] specifies 1 as the major version, 2 as the minor version, and accepts the default build and revision numbers. A version number such as [assembly:AssemblyVersion("1.2.15.\*")] specifies 1 as the major version, 2 as the minor version, 15 as the build number, and accepts the default revision number. The default build number increments daily. The default revision number is the number of seconds since midnight local time (without taking into account time zone adjustments for daylight saving time), divided by 2. *(See the [AssemblyVersionAttribute Class](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.assemblyversionattribute?view=netframework-4.7.1) documentation.)*

While this approach is convenient, it's not very flexible and doesn't provide you with a meaningful version number. Over the years, there have been numerous attempts at solving the problem of automatically versioning .NET assemblies. While these approaches work, and some are very flexible, they aren't always a simple answer.

When .NET Core was introduced, it brought along some much-needed improvements to MSBuild and the project file format. We can now leverage these improvements to create a relatively simple way to achieve a consistent version number across all projects in a solution, even if your solution has a mixture of .NET Core and regular projects.

I use this solution in two of my production projects at the moment and am in the process of updating the remainder of my projects to use it as well.

All of the files mentioned in this post are available at https://github.com/scottdorman/assembly-build-versioning, so I'm only going to show relevant portions of some of them here.

## Getting started

To get started using this process, head over to the GitHub [repository](https://github.com/scottdorman/assembly-build-versioning) and copy the `build` folder, the `Directory.Build.props`, `common.props`, and the `ReleaseNotes.xml` files into your solution folder.

<div class="alert alert-info">
If you already have a <code>Directory.build.props</code> file or a <code>common.props</code> file, you'll want to merge the contents together or take other steps to prevent your files from being overwritten while still including the files from this process.
</div>

![project tree layout](/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/project-tree.png) 

Next, update the relevant properties in the `common.props` file.

A typical `common.props` file might look like 

```xml
<Project>
  <PropertyGroup>
    <Product>WebApp</Product>
    <Company></Company>
    <Copyright>Copyright (c) 2018</Copyright>
  </PropertyGroup>

  <PropertyGroup>
    <VersionMajor>1</VersionMajor>
    <VersionMinor>1</VersionMinor>
  </PropertyGroup>

  <PropertyGroup>
    <VersionPrefix>$(VersionMajor).$(VersionMinor)</VersionPrefix>
    <VersionSuffix>preview</VersionSuffix>
  </PropertyGroup>

  <PropertyGroup>
    <GenerateReleaseNotes Condition="'$(GenerateReleaseNotes)' == ''">false</GenerateReleaseNotes>
    <GenerateAssemblyBuildDateAttribute Condition="'$(GenerateAssemblyBuildDateAttribute)' == ''">true</GenerateAssemblyBuildDateAttribute>
  </PropertyGroup>
</Project>
``` 
Finally, add the `build\VersionUpdate.csproj` project to your solution.

> **Note:** This project needs to be added to your solution for the project references to work properly. Visual Studio automatically builds a `project.assets.json` file, and without it, the references don't seem to work reliably. *This is an issue with the MSBuild project references themselves, and not a limitation inherent in this process.*

If there are projects you want to exclude from being automatically versioned, you can add the following property group to the project file:

```xml
<PropertyGroup>
    <IgnoreVersionUpdate>true</IgnoreVersionUpdate>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>
```

If you have older projects that you want to include in this process, you just need to add the following import to the project file:
```xml
<Import Project="$(BuildDir)\GenerateAssemblyInfo.targets" Condition="Exists('$(BuildDir)\GenerateAssemblyInfo.targets')" /> 
```

### Updating release notes

*This section is somewhat opinionated on the format of the release notes and the file type and is designed to work with an XML file for the release notes. If you want to change this, you would need to update the `AddReleaseNotesRootEntry` task and most likely the `UpdateReleaseNotes` target in `VersionUpdate.csproj`.*


If you want to maintain a release notes file by hand, typically to display to the end user, it can be helpful to include both the version number and build date at the start of each section. The default process uses a release notes XML file which looks like:

```xml
<?xml version="1.0" encoding="utf-8"?>
<notes>
  <!--
  <entry version="" build-date="">
    <content>
      <ul>
        <li></li>
      </ul>
    </content>
  </entry>
  -->
</notes>
```

When the `UpdateReleaseNotes` target runs, it looks for the first `<entry>` element (that isn't in a comment) where the `version` attribute is blank (or is the value `vNext`) and fills in values for both the `version` and the `build-date` attributes.

The rest of the content is up to you.

## How it works

If you want to know the details of how this approach works, this section is for you.

### MSBuild props and targets files

It all starts with the `Directory.build.props` file. Starting with MSBuild 15, you can use this file to add new properties and item groups to every project. For our purposes, we want this file to be in the root solution folder. *(For more information, see [Customize your build](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build))*

This file defines some common properties, imports the other properties files, and adds the `VersionUpdate` project to all of the other projects in the solution. There isn't much reason to change this file.

The `common.props` file defines common/shared assembly and NuGet package metadata. Here is where you can define the standard assembly and NuGet package properties, like the product, company, copyright, and authors. You can also define the major and minor version number, the version prefix and version suffix properties.

By default, this approach uses the `build` folder for the `build.props`, `GenerateAssemblyInfo.targets`, `VersionUpdate.csproj`, and optionally the `version.props` files. If you want to change it to something else, be sure to change the `BuildDir` property in `Directory.build.props`, although I recommend leaving it as the default.

This process automatically updates the `build.props` file, which contains the build date, build, and revision numbers used by the final version information. A continuous integration (CI) build process might insert the CI build number into this file as well. 

A typical `build.props` file would look like

```xml
<!-- This file may be overwritten by automation. -->
<Project>
  <PropertyGroup>
    <BuildDate>8/16/2018 2:02:35 PM</BuildDate>
    <VersionBuild Condition="'$(VersionBuild)' == ''">18228</VersionBuild>
    <VersionRevision Condition="'$(VersionRevision)' == ''">25277</VersionRevision>
    <CI_BUILD_NUMBER>171</CI_BUILD_NUMBER>
  </PropertyGroup>
</Project>
```

By default, this approach assumes a property named `CI_BUILD_NUMBER`. If your CI system sets a different MSBuild property, you can either update `Directory.build.props` and change the property name or simply set the `CI_BUILD_NUMBER` property value to the property used by your CI system.

> For example, if you use [AppVeyor](https://www.appveyor.com/) as your CI system, it uses `APPVEYOR_BUILD_NUMBER`, so you can change the `CI_BUILD_NUMBER` property to be 
>
> ```xml
> <CI_BUILD_NUMBER>$(APPVEYOR_BUILD_NUMBER)</CI_BUILD_NUMBER>
> ```

If instead, you want to write the version information into a separate file, you can use a `version.props` file, which if it exists, is automatically imported. 

The `GenerateAssemblyInfo.targets` file, used only by older project files, is mostly a copy of the targets file used when compiling a .NET Core project and is responsible for generating the `SolutionInfo.cs` file that gets included into your project at compile time.

### The `VersionUpdate` project
The `VersionUpdate` project contains the `UpdateAssemblyVersionInfo` and `UpdateReleaseNotes` targets and also the associated tasks used by those targets.

> This project uses the `CodeTaskFactory` so the tasks can be defined inside the project and not require a separate assembly. If you're building with the .NET CLI or on a non-Windows platform, this project won't build (see this [comment](https://github.com/Microsoft/msbuild/issues/616#issuecomment-219258405)). You can install the [RoslynCodeTaskFactory ](https://github.com/jeffkl/RoslynCodeTaskFactory) or upgrade to Visual Studio 15.8 if you need that support. Doing so would change the `UsingTask` elements from
>
> ```xml
> <UsingTask TaskName="AddReleaseNotesRootEntry" 
>    TaskFactory="CodeTaskFactory" 
>    AssemblyFile="$(MSBuildToolsPath)\Microsoft.Build.Tasks.v4.0.dll">
> ```
> to
> ```xml
> <UsingTask TaskName="AddReleaseNotesRootEntry"  
>    TaskFactory="CodeTaskFactory"  
>    AssemblyFile="$(RoslynCodeTaskFactory)"
>    Condition=" '$(RoslynCodeTaskFactory)' != '' ">
> ```

> **Note:** This project does need to be added to your solution for the project references to work properly. Visual Studio automatically builds a `project.assets.json` file, and without it, the references don't seem to work reliably. *This is an issue with the MSBuild project references themselves, and not a limitation inherent in this process.*
