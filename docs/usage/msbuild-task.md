# MSBuild Task

The MSBuild Task for GitVersion — **GitVersionTask** — is a simple solution if
you want to version your assemblies without writing any command line scripts or
modifying your build process.

## TL;DR

### Install the MSTask targets

Add the [GitVersionTask](https://www.nuget.org/packages/GitVersionTask/) NuGet
Package into the project you want to be versioned by GitVersion.

From the Package Manager Console:
```shell
Install-Package GitVersionTask
```

### Remove AssemblyInfo attributes

The next thing you need to do is to remove the `Assembly*Version` attributes from
your `Properties\AssemblyInfo.cs` files. This puts GitVersionTask in charge of
versioning your assemblies.

### Done!

The setup process is now complete and GitVersionTask should be working its magic,
versioning your assemblies like a champ. However, more can be done to further 
customize the build process. Keep reading to find out how the version variables 
are set and how you can use them in MSBuild tasks.

## How does it work?

After being installed into a project, the MSBuild task will wire GitVersion into
the MSBuild pipeline and will then perform several actions. These actions are
described below.

### Inject version metadata into the assembly

The sub-task named `GitVersionTask.UpdateAssemblyInfo` will inject version
metadata into the assembly where GitVersionTask has been added to. For each assembly
you want GitVersion to handle versioning, you will need to install 
[GitVersionTask](https://www.nuget.org/packages/GitVersionTask/) into the corresponding 
project via NuGet.

#### AssemblyInfo Attributes

A temporary `AssemblyInfo.cs` will be created at build time. That file will contain the
appropriate SemVer information. This will be included in the build pipeline.

Default sample:
```csharp
[assembly: AssemblyVersion("1.0.0.0")]
[assembly: AssemblyFileVersion("1.0.0.0")]
[assembly: AssemblyInformationalVersion("1.1.0+Branch.master.Sha.722aad3217bd49a6576b6f82f60884e612f9ba58")]
```

Now, when you build:
* `AssemblyVersion` will be set to the `AssemblySemVer` variable.
* `AssemblyFileVersion` will be set to the `MajorMinorPatch` variable with `.0` appended to it.
* `AssemblyInformationalVersion` will be set to the `InformationalVersion` variable.

#### Other injected Variables

All other [variables](../more-info/variables.md) will be injected into an
internal static class:

```csharp
namespace AssemblyName
{
    [CompilerGenerated]
    internal static class GitVersionInformation
    {
        public static string Major = "1";
        public static string Minor = "1";
        public static string Patch = "0";
        ...All other variables
    }
}
```

### Accessing injected Variables

##### All variables

```csharp
var assemblyName = assembly.GetName().Name;
var gitVersionInformationType = assembly.GetType(assemblyName + ".GitVersionInformation");
var fields = gitVersionInformationType.GetFields();

foreach (var field in fields)
{
    Trace.WriteLine(string.Format("{0}: {1}", field.Name, field.GetValue(null)));
}
```

##### Specific variable

```csharp
var assemblyName = assembly.GetName().Name;
var gitVersionInformationType = assembly.GetType(assemblyName + ".GitVersionInformation");
var versionField = gitVersionInformationType.GetField("Major");
Trace.WriteLine(versionField.GetValue(null));
```

### Populate some MSBuild properties with version metadata

The sub-task `GitVersionTask.GetVersion` will write all the derived
[variables](../more-info/variables.md) to MSBuild properties so the information
can be used by other tooling in the build pipeline.

The class for `GitVersionTask.GetVersion` has a property for each variable.
However at MSBuild time these properties are mapped to MSBuild properties that
are prefixed with `GitVersion_`. This prevents conflicts with other properties
in the pipeline.

#### Accessing variables in MSBuild

Once `GitVersionTask.GetVersion` has been executed, the MSBuild properties can be
used in the standard way. For example:

```xml
<Message Text="GitVersion_InformationalVersion: $(GitVersion_InformationalVersion)"/>
```

### Communicate variables to current Build Server

The sub-task `GitVersionTask.WriteVersionInfoToBuildLog` will attempt to write
the version information to the current Build Server log.

If, at build time, it is detected that the build is occurring inside a Build
Server then the [variables](../more-info/variables.md) will be written to the
Build Server log in a format that the current Build Server can consume. See
[Build Server Support](../build-server-support/build-server-support.md).

## Conditional control tasks

Properties `WriteVersionInfoToBuildLog`, `UpdateAssemblyInfo` and `GetVersion`
are checked before running these tasks.

You can disable `GitVersionTask.UpdateAssemblyInfo` by setting
`UpdateAssemblyInfo` to `false` in your MSBuild script, like
this:

```xml
<PropertyGroup>
  ...
  <UpdateAssemblyInfo>false</UpdateAssemblyInfo>
  ...
</PropertyGroup>
```

## My Git repository requires authentication. What should I do?

Set the environment variables `GITVERSION_REMOTE_USERNAME` and
`GITVERSION_REMOTE_PASSWORD` before the build is initiated.