About two years ago I [talked](https://scottdorman.blog/2018/08/20/net-core-project-versioning/) about a way to create consistent version numbers across .NET Core and .NET Framework projects. At the time, I thought the approach I'd come up with was fairly easy to use and didn't require much modification to your projects.

At the end of March 2020, GitHub announced [public availability of GitHub Actions](https://github.blog/changelog/2020-03-24-github-actions-api-is-now-generally-available/). While it isn't as full-featured as other continuous integration/continuous-delivery (CI/CD) tools out there, like AppVeyor and Jenkins, it doesn't require any additional accounts to set up, does the job adequately well for some of the more common scenarios, and seems to steadily be getting better and more full-featured.

I've setup build actions for two private repositories I work in, and after understanding how the build environments work and playing around with some of the projects in one of the repositories, I decided to revisit my assembly versioning setup.

This time around, it takes advantage of the actions workflow and requires even less changes to your projects.

One of the main changes is that the code tasks have now been moved into a compiled assembly that will become part of my Cadru Framework. This allows it to fully run from the command line through calling the `dotnet` command line tooling. The second major change is that it no longer requires you to add a project to your solution to try and hook into the build process. Since there wasn't a reliable way to insert the target into the process of building the solution, it ended up running for each project. While this generally didn't cause much issue, it would sometimes result in slightly different version numbers. For those who need, or want to see, the source code for the `Cadru.Build.Tasks` project, it's [available](https://github.com/scottdorman/cadru/tree/master/src/Cadru.Build.Tasks) on GitHub as well.

Just like with the earlier implementation, head over to the GitHub [repository](https://github.com/scottdorman/assembly-build-versioning) and copy the contents of `src` folder (not the folder itself, just what's in it) into your solution folder. If you don't want the task to update a release notes XML file, you can delete the `ReleaseNotes.xml` file. Although the files have changed slightly from their earlier versions, they still work pretty much the same way.

> The original implementation is still there, but it's under a `v1.0` branch. The `master` branch is the latest version, 2.0.

The `Directory.Build.props` file adds new properties and item groups to every project, and defines some of the common properties used. It also imports the other properties files. There really isn't a need to change this file.

The `common.props` file is where you can define the standard assembly and NuGet package properties, like product, company, copyright, and authors. You can also define the version number properties here as well.

Everything else about the process still lives in the `build` folder, by default. (You can change it, but doing so will require you to change some of the core targets files, so I'd suggest just leaving it where it is.) This is also where things start to change with the new implementation.

The first change is that `build.props` has been renamed to `version.props`. I think this name makes more sense. The name of the properties in it has also changed. A typical file looks like

```XML
<Project>
  <PropertyGroup>
    <BuildDate>4/17/2020 8:09:02 AM</BuildDate>
    <VersionPatch>20217</VersionPatch>
    <VersionRevision>36371</VersionRevision>
  </PropertyGroup>
</Project>
```

This file is updated once by calling `dotnet build /build/Cadru.VersionUpdate.targets -t:UpdateAssemblyVersionInfo`. That task sets the build date and computes the patch and revision numbers. Since this file is imported before `common.props`, if you set one of those properties there, the value here will be overridden.

> The version numbering scheme is loosely based on what's done in [Arcade](https://github.com/dotnet/arcade) and is described in the [.NET Core Ecosystem v2 - Versioning](https://github.com/dotnet/arcade/blob/master/Documentation/CorePackages/Versioning.md) document.

Since this file is updated "out of band" from compiling the rest of the code, it ensures that all of the projects will pick up the same version information when they're compiled.

To update a release notes file, you just need to set the `GenerateReleaseNotes` property in `common.props` to `true` and then uncomment and provide values for the item group. Remember, it's still somewhat opinionated on the format and is designed to work with an XML file. The version related properties are available to you in MSBuild, though, so if you wanted to write your own target to update a release notes file, you can look at the `UpdateReleaseNotes` in `Vadru.VersionUpdate.Targets` and update it as necessary.

If you only have .NET Core projects, you're done. If you have older .NET Framework projects, you need to add the following somewhere in the project file. (I'd suggest just adding it at the end.)

```XML
 <PropertyGroup>
   <GenerateAssemblyInfo>true</GenerateAssemblyInfo>
 </PropertyGroup>
 <Import Project="$(BuildDir)\GenerateAssemblyInfo.targets"> Condition="Exists('$(BuildDir)\GenerateAssemblyInfo.targets')" />
```

If there are .NET Core projects you don't want automatically versioned, you can opt them out by adding the following property group to the project file.

```XML
<PropertyGroup>
    <IgnoreVersionUpdate>true</IgnoreVersionUpdate>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
  </PropertyGroup>
```

## Planned improvements
Now that I've simplified how things work, there are definitely some improvements I want to make.
- Add additional versioning strategies. Right now, there are two, both of which are date-based.
- Add more flexibility in the creating a variant of the `version.props` file in different formats. I can see plain text and JSON formatted right now, possibly some other formats later on.
- Improve task logging and output
- Add more flexibility for updating a release notes document, possibly different formats as well.
