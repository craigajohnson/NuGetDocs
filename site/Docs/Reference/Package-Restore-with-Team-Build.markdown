# Package Restore with Team Foundation Build

This section will provide a detailed walkthrough on how to restore packages as part of the [Team Foundation Build](http://msdn.microsoft.com/en-us/library/ms181710(v=VS.90).aspx) both, for [git](http://en.wikipedia.org/wiki/Git_(software)) as well as [TF Version Control](http://msdn.microsoft.com/en-us/library/ms181237(v=vs.120).aspx).

Although this walkthrough is specific for the scenario of using [Team Foundation Service](http://tfs.visualstudio.com/), the concepts also apply to other version control- and build systems.

## The General Approach

An advantage of using NuGet is that you can use it to avoid checking in binaries to your version control system.

This is especially interesting if you are using a [distributed version control](http://en.wikipedia.org/wiki/Distributed_revision_control) system like git because developers need to clone the entire repository, including the full history, before they can start working locally. Checking in binaries can cause significant repository bloat as binary files are typically stored without delta compression.

NuGet has supported [restoring packages](http://docs.nuget.org/docs/reference/package-restore) as part of the build for a long time now. The previous implementation had a chicken-and-egg problem for packages that want to extend the build process because NuGet restored packages while building the project. However, MSBuild doesn't allow extending the build during the build; one could argue that this an issue in MSBuild but I would argue that this a inherent problem. Depending on which aspect you need to extend it might be too late to register by the time your package is restored.

The cure to this problem is making sure that packages are restored as the first step in the build process. NuGet 2.7 makes this super easy via a [simplified command line](http://docs.nuget.org/docs/release-notes/nuget-2.7#Simplified_Package_Restore_from_the_Command-Line). In order to restore packages for an entire solution all you need is to execute a command line like this:

	nuget.exe restore path\to\solution.sln

When your build process restores packages before building the code, you don't need to check-in **.targets** files 

<p class="info">
<strong>Note:</strong> Packages must be authored to allow loading in Visual Studio. Otherwise, you may still want to check in <strong>.targets</strong> files so that other developers can simply open the solution without having to restore packages first. 
</p> 

The following demo project shows how to set up the build in such a way that the `packages` folders and **.targets** files don't need to be checked-in. Finally, I'll show how you can setup an automated build on the [Team Foundation Service] for this sample project.

## Repository Structure

Our demo project is a simple command line tool that uses the command line argument to query Bing. It targets the .NET Framework 4 and uses many of the [BCL packages](http://www.nuget.org/profiles/dotnetframework/) ([Microsoft.Net.Http](http://www.nuget.org/packages/Microsoft.Net.Http), [Microsoft.Bcl](http://www.nuget.org/packages/Microsoft.Bcl), [Microsoft.Bcl.Async](http://www.nuget.org/packages/Microsoft.Bcl.Async), and [Microsoft.Bcl.Build](http://www.nuget.org/packages/Microsoft.Bcl.Build)).

The structure of the repository looks as follows:

	<Project>
	    │   .gitignore
	    │   .tfignore
	    │   build.proj
	    │
	    ├───src
	    │   │   BingSearcher.sln
	    │   │
	    │   └───BingSearcher
	    │       │   App.config
	    │       │   BingSearcher.csproj
	    │       │   packages.config
	    │       │   Program.cs
	    │       │
	    │       └───Properties
	    │               AssemblyInfo.cs
	    │
	    └───tools
	        └───NuGet
	                NuGet.exe

You can see that we haven't checked-in the `packages` folder nor any **.targets** files.

We have, however, checked-in the `nuget.exe` as it's needed during the build. Following widely used conventions we've checked it in under a shared `tools` folder.

The source code is under the `src` folder. Although our demo only uses a single solution, you can easily imagine that this folder contains more than one solution.

In order to communicate to the version control that we don’t intent to check-in the **packages** folders, we've also added ignore files for both git (`.gitignore`) as well as TF version control (`.tfignore`). These files describes patterns of files you don't want to check-in.

The `.gitignore` file looks as follows:

	syntax: glob
	*.user
	*.suo
	bin
	obj
	packages

The `.gitignore` file is [quite powerful](https://www.kernel.org/pub/software/scm/git/docs/gitignore.html). For example, if you want to generally not check-in the contents of the `packages` folder but want to go with previous guidance of checking in the **.targets** files you could have the following rule instead:

	packages
	!packages/**/*.targets

This will exclude all `packages` folders but will re-include all contained **.targets** files. By the way, you can find a template for `.gitignore` files that is specifically tailored for the needs of Visual Studio developers [here](https://github.com/github/gitignore/blob/master/VisualStudio.gitignore).

TF version control supports a very similar mechanism via the [.tfignore](http://msdn.microsoft.com/en-us/library/ms245454.aspx) file. The syntax is virtually the same:

	*.user
	*.suo
	bin
	obj
	packages

## build.proj
 
For our demo, we keep the build process fairly simple. We'll create an MSBuild project that builds all solutions while making sure that packages are restored before building the solutions.

This project will have the three conventional targets `Clean`, `Build` and `Rebuild` as well as a new target `RestorePackages`.

- The `Build` and `Rebuild` targets both depend on `RestorePackages`. This makes sure that you can both run `Build` and `Rebuild` and rely on packages being restored.
- `Clean`, `Build` and `Rebuild` invoke the corresponding MSBuild target on all solution files.
- The `RestorePackages` target invokes `nuget.exe` for each solution file. This is accomplished by using [MSBuild's batching functionality](http://msdn.microsoft.com/en-us/library/ms171473.aspx).

The result looks as follows:

	<?xml version="1.0" encoding="utf-8"?>
	<Project ToolsVersion="4.0"
	         DefaultTargets="Build" 
	         xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
	
	  <PropertyGroup>
	    <OutDir>$(MSBuildThisFileDirectory)bin</OutDir>
	    <Configuration>Release</Configuration>
	    <ProjectProperties>
	        OutDir=$(OutDir);
	        Configuration=$(Configuration);
	    </ProjectProperties>
	  </PropertyGroup>
	
	  <ItemGroup>
	    <Solution Include="$(MSBuildThisFileDirectory)src\*.sln" />
	  </ItemGroup>
	
	  <Target Name="RestorePackages">
	    <Exec Command="&quot;$(MSBuildThisFileDirectory)tools\NuGet\NuGet.exe&quot; restore &quot;%(Solution.Identity)&quot;" />
	  </Target>
	
	  <Target Name="Clean">
	    <MSBuild Targets="Clean"
	             Projects="@(Solution)"
	             Properties="$(ProjectProperties)" />
	  </Target>
	
	  <Target Name="Build" DependsOnTargets="RestorePackages">
	    <MSBuild Targets="Build"
	             Projects="@(Solution)"
	             Properties="$(ProjectProperties)" />
	  </Target>
	
	  <Target Name="Rebuild" DependsOnTargets="RestorePackages">
	    <MSBuild Targets="Rebuild"
	             Projects="@(Solution)"
	             Properties="$(ProjectProperties)" />
	  </Target>
	
	</Project>

## Configuring Team Build

Team Build offers various process templates. For this demonstration, we're using the [Team Foundation Service]. On premise installations of TFS will be very similar though.

Git and TF Version Control have different Team Build templates, so the following steps will vary depending on which version control system you are using. In both cases, all you need is selecting the build.proj as the project you want to build.

First, let's look at the process template for git. In the git based template the build is selected via the property `1. Solution to build`:

![Build Process for git](images/PackageRestoreTeamBuildGit.png)

Please note that this property is a location in your repository. Since our `build.proj` is in the root, we simply used `build.proj`. If you place the build file under a folder called `tools`, the value would be `tools\build.proj`.

In the TF version control template the project is selected via the property `1. Projects`:

![Build Process for TFVC](images/PackageRestoreTeamBuildTFVC.png)

In contrast to the git based template the TF version control supports pickers (the button on the right hand side with the three dots). So in order to avoid any typing errors we suggest you use them to select the project. 

## Modifying a Team Foundation Build Template To Invoke NuGet Restore

Another approach to invoking NuGet restore is to modify the Team Foundation Build Template to call NuGet.exe automatically for each solution to be built. This makes it possible to use Automatic Package Restore during development while ensuring that all packages are restored prior to the MSBuild step of the Team Foundation Build process.

For this example, we'll modify the DefaultTemplate included with TFS.

1. To begin, copy NuGet.exe to a known location in your build environment. This can be copied either to the same location on each build server or to a shared network location.
1. Checkout the `DefaultTemplate.xx.y.xaml` file located in the `BuildProcessTemplates` folder in your TFS project. (The actual name will vary based on which version of TFS is installed).
1. Double-click on the `DefaultTemplate.xx.y.xaml` file to edit.
1. In the workflow, locate each instance of `Run MSBuild for Project` activity.
1. Just prior to each MSBuild activity, insert an `InvokeProcess` activity from `Team Foundation Build Activities` in the toolbox. We'll be modifying this build activity to run NuGet.exe
1. 






