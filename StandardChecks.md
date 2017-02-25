# Standard checks and updates 

These are standard checks, low-hanging fruit that I perform when picking up a .Net project that hasn't been maintained in a while. They are roughly ordered from simple and easy to more complex, far-reaching and tricky.

### Is there anyone in there?

See if it builds. See if the test pass. See if it runs. You will need these to validate further changes.

### Do not check in binary packages

Remove binary nuget packages from the repository. 

If there are any folders in the repo under `\packages\`, delete them. I don't really mind if you keep the file `\packages\repositories.config` or not - it's small and doesn't change frequently.

On the git command line this is `git rm -r --cached ./packages/*`. 

Commit this change. Verify that in your upstream repository, none of these packages are present.

To prevent packages being added in future, ignore them: in the `.gitignore` file, add a line: `**/packages/*/**`. 

Why do this? Storing the binaries was useful in the case that the nuget.org server isn't working. But this is now rare - we can now rely on the nuget.org servers. Source code repositories don't work well with large binary files. These files are immutable anyway (e.g. `Newtonsoft.Json.9.0.1` always has the same contents across all package sources), so restoring them upon build always has the same results. 

What invariably happens when packages are stored is that one or more packages are updated in the project config, but the files stored in git under `\packages\` are not updated. The "packages that I keep" and "packages that I need" become increasingly disjoint sets over time. Then you have the worst of both worlds: large binaries in git, and reliance on on the nuget.org server for packages. Simplify by deleting them.

### Remove the nuget binary

Remove the `\.nuget` folder, the binaries in it, and remove references to them from `.csproj` files. These are no longer needed, are again unnecessary binary files in the source code repository, and it store a specific version of `nuget.exe`, usually an obsolete one.

If you use custom package sources and want this metadata to live with the project, then you need the `nuget.config` file here. But that's the only file. 


### EditorConfig

Add a standard `.EditorConfig` file in the root folder. [EditorConfig](http://editorconfig.org/) means that some annoying formatting variations across checkins just don't happen any more.

For Visual Studio 2015, ensure that the Visual Studio add-in that reads this is installed. The good news is that you don't need the addin for Visual Studio 2017.

It might be necessary to do one last formatting-only commit, where every file standardised. Otherwise this will happen as you work, interwoven with code changes.

### Warnings as errors

If possible, turn on warnings as errors. If there are warning that prevent this, try to fix them: many common warning are trivial to fix.

### .Net version

Update to a current and consistent framework version.
Currently 4.6.2 for deployable projects and tests, and 4.5.0 for libs that are expected to be used from mono.

Before you do this, you might have to verify that the suitable pieces are installed on production servers, test servers and build machines. 

### Unused assembly references

look for unused references to assemblies in projects. The default reference to `System.Xml.Linq` is seldom used, the default reference to `System.Data.DataSetExtensions` is almost always unused and is a good marker that these references have never been uncluttered. If the project does not do any data access either directly or indirectly, you can remove `System.Data`. 

Sometimes packages are not referenced in code, but are in fact used at runtime, so be prepared to revert if a test or a deploy fails after removing a reference.

### Nuget package consistency

Right-click the solution, select "Manage nuget packages for solution...", click on the "Consolidate" tab. 

At minimum, use the same version of a given nuget package throughout the solution. Otherwise behaviour may differ between e.g. tests and deployed code. 


### Nuget package updates

There is lots to do  here, each package is unique.

Managing packages dependencies and updates is an often underrated skill - it's not coding and it doesn't get taught but it is important. After all, what's the point of writing code and packaging it the new code never gets used? It's sometimes hard to get right, and the commits are not very easy to read - all that the commit shows is that some version numbers change, and others do not. 

Ease of doing it depends on the package, and understanding them comes with experience. [In general, minor version increments are safe, but major versions may make breaking changes](http://semver.org/). 

If there are many packages to update, don't try to update all the packages at once, but do it in stages. Look for groups (e.g. all parts of the [AWS SDK](https://www.nuget.org/packages/AWSSDK.Core/) can and should be updated together)

Be prepared to revert combinations that don't work, and learn what they are for future reference.

Many packages have to be updated together. 
We currently use `Newtonsoft.Json.9.0.1` throughout, so updating packages that depend upon Newtonsoft.Json means updating it as well. 

Remember that when there are updates to Assembly Binding Redirects in the `.config` files, these need to go in the config templates as well.

### Remove superfluous packages

I tend to the view that each package is a dependency, and so comes with a cost of using and updating it. If it's trivial to use your own code instead, then do that.

Also look for packages that do the same thing.  Everyone has [a favourite mocking framework](https://www.nuget.org/packages/Moq/), [Ioc container](https://www.nuget.org/packages/structuremap/) or [unit testing library](https://www.nuget.org/packages/NUnit/) but the only hard and fast rule that I have is that two of them is one too many. For instance, if there is [Moq](https://www.nuget.org/packages/Moq/) and [NSubstitute](https://www.nuget.org/packages/NSubstitute/) then migrate the code and remove one of them.

### Do async well

See here for more details on this topic: [avoiding async basic mistakes](./AsyncBasicMistakes).


### Reuse the HttpClient instance

 See [here](http://codereview.stackexchange.com/a/69954) and links from there. 
 
For a given endpoint you pre-configure  a `HttpClient` with request headers, base address and so on, then re-use it. "it will help reuse TCP connections where possible which will in general lead to better performance", thought his may only show up under high load.

### Prefer DateTimeOffset to DateTimeOffset

The docs say ["DateTimeOffset should be considered the default date and time type for application development"](https://msdn.microsoft.com/en-us/library/bb384267.aspx). 
If there is any possibility of your code processing values from multiple time zones or experiencing daylight saving time changes, be explicit about the time zone of each value with a `DateTimeOffset` not a `DateTime`. 
(./iso8601)[It's good to be explicit].

###  Logging and metrics

Look at the logging and stating to ensure that errors are being recorded in a suitable structured format, and that important operations have metrics and timers.

