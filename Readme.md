This repository demonstrates an issue with the fast up-to-date checks in Visual Studio 2017 15.9.10 for projects which have `CopyToOutputDirectory` items but no project references.

⚠ Note that this demonstration pertains only to modern, SDK-style C# projects. A similar issue happens with legacy projects, but the root cause seems to be different (the presence of project references doesn't matter) and the workarounds do not apply.

The solution is set up as follows:

* There are four projects:
    * `TestProgram` - a .NET Core 2.2 app
    * `LibraryA` - a .NET Standard 2.0 library
    * `LibraryB` - a .NET Standard 2.0 library
    * `Dummy` - a .NET Standard 2.0 library
* `TestProgram` references `LibraryA`
* `LibraryA` references `LibraryB`
* `Dummy` is only present to demonstrate a workaround for the issue.
* Both `LibraryA` and `LibraryB` include a text file which is copied to the output directory with `<CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>`, the contents of these files are printed when `TestProgram` is run.

# Reproducing the problem

(Note: The following assumes you have Visual Studio configured to build out of date projects when you run.)

1. Open the solution in Visual Studio.
2. Select `TestProgram` as the startup project.
3. Press F5 to start the program. The output is as follows:
    ```
    Hello World!
    Hello from ClassA!
    Text file A reporting for duty!
    Hello from ClassB!
    Text file B reporting for duty!
    Done.
    ```
4. Close the program and add a message `TextFileA.txt` and press F5 to start the program again. The output includes your message:
    ```
    Hello World!
    Hello from ClassA!
    Text file A reporting for duty! Hello, world!
    Hello from ClassB!
    Text file B reporting for duty!
    Done.
    ```
5. Close the program and add a message to `TextFileB.txt` instead, press F5 to start the program again. **The output will not include your message:**
    ```
    Hello World!
    Hello from ClassA!
    Text file A reporting for duty! Hello, world!
    Hello from ClassB!
    Text file B reporting for duty!
    Done.
    ```
6. Observe that the file is updated in `LibraryB`'s output (`LibraryB\bin\Debug\netstandard2.0\TextFileB.txt`), but not in `TestProgram`'s output (`TestProgram\bin\Debug\netcoreapp2.2\TextFileB.txt`).

# The source of the problem

The problem is caused by the fast up-to-date checks. The fast up-to-date checks do not consider whether or not `CopyToOutputDirectory` items from referenced projects are up-to-date or not. Instead it is relying on the reference copy marker. This can be observed if you enable verbose logging for up-to-date checks.

In the case of editing `TextFileA.txt`, `TestProgram` is updated because the `LibraryA.csproj.CopyComplete` file is newer than `TestProgram`'s output marker:

```
4>FastUpToDate: Adding input reference copy markers: (TestProgram)
4>FastUpToDate:     'C:\Development\Playground\UsingTransitiveCopiedContent\LibraryA\obj\Debug\netstandard2.0\LibraryA.csproj.CopyComplete' (TestProgram)
4>FastUpToDate:     'C:\Development\Playground\UsingTransitiveCopiedContent\LibraryB\obj\Debug\netstandard2.0\LibraryB.csproj.CopyComplete' (TestProgram)
4>FastUpToDate: Adding output reference copy marker: (TestProgram)
4>FastUpToDate:     'C:\Development\Playground\UsingTransitiveCopiedContent\TestProgram\obj\Debug\netcoreapp2.2\TestProgram.csproj.CopyComplete' (TestProgram)
4>FastUpToDate: Latest write timestamp on input marker is 3/27/2019 8:44:47 PM on 'C:\Development\Playground\UsingTransitiveCopiedContent\LibraryA\obj\Debug\netstandard2.0\LibraryA.csproj.CopyComplete'. (TestProgram)
4>FastUpToDate: Write timestamp on output marker is 3/27/2019 8:39:52 PM on 'C:\Development\Playground\UsingTransitiveCopiedContent\TestProgram\obj\Debug\netcoreapp2.2\TestProgram.csproj.CopyComplete'. (TestProgram)
4>FastUpToDate: Input marker is newer than output marker, not up to date. (TestProgram)
4>FastUpToDate: Project is not up to date. (TestProgram)
```

(The above output is printed by Visual Studio when the up-to-date checks logging is set to verbose in Tools > Options > Project and Solutions > .NET Core.)

When you edit only `TextFileB.txt`, `TestProgram` is not updated because the `LibraryB.csproj.CopyComplete` marker doesn't exist:

```
4>FastUpToDate: Adding input reference copy markers: (TestProgram)
4>FastUpToDate:     'C:\Development\Playground\UsingTransitiveCopiedContent\LibraryA\obj\Debug\netstandard2.0\LibraryA.csproj.CopyComplete' (TestProgram)
4>FastUpToDate:     'C:\Development\Playground\UsingTransitiveCopiedContent\LibraryB\obj\Debug\netstandard2.0\LibraryB.csproj.CopyComplete' (TestProgram)
4>FastUpToDate: Adding output reference copy marker: (TestProgram)
4>FastUpToDate:     'C:\Development\Playground\UsingTransitiveCopiedContent\TestProgram\obj\Debug\netcoreapp2.2\TestProgram.csproj.CopyComplete' (TestProgram)
4>FastUpToDate: Latest write timestamp on input marker is 3/27/2019 8:45:36 PM on 'C:\Development\Playground\UsingTransitiveCopiedContent\LibraryA\obj\Debug\netstandard2.0\LibraryA.csproj.CopyComplete'. (TestProgram)
4>FastUpToDate: Write timestamp on output marker is 3/27/2019 8:45:38 PM on 'C:\Development\Playground\UsingTransitiveCopiedContent\TestProgram\obj\Debug\netcoreapp2.2\TestProgram.csproj.CopyComplete'. (TestProgram)
4>FastUpToDate: Project is up to date. (TestProgram)
```

(If a file is missing, [`BuildUpToDateCheck` will not consider it](https://github.com/dotnet/project-system/blob/a0f6882dd8cd38631ff2b012d71c0e3e6199123b/src/Microsoft.VisualStudio.ProjectSystem.Managed/ProjectSystem/UpToDate/BuildUpToDateCheck.cs#L313-L316). You can see the file is completely missing rather than stale by checking in the intermediate directory for `LibraryB`.)

The reason the `CopyComplete` marker doesn't exist is because [MSBuild doesn't create it if there's no references copied to the output](https://github.com/Microsoft/msbuild/blob/adb180d394176f36aca1cc2eac4455fef564739f/src/Tasks/Microsoft.Common.CurrentVersion.targets#L4348-L4393).

# Workarounds

I've found two workarounds to this issue.

## Add a dummy reference to `LibraryB`

One workaround is to add a dummy reference to `LibraryB`, which is what the `Dummy` project is for. There's already a commented-out `ProjectReference` in `LibraryB.csproj` for this purpose.

The downside to this approach is that there will be a useless `Dummy.dll` assembly floating around.

## Create the copy marker yourself

The other workaround is to create the `CopyComplete` file yourself when MSBuild doesn't.

I've added a small target to `LibraryB` that does this:

```xml
<Target Name="MissingCopyMarkerWorkaround" AfterTargets="CopyFilesToOutputDirectory" Condition="'@(ReferenceCopyLocalPaths)' == '' or '@(ReferencesCopiedInThisBuild)' == ''">
  <Touch Files="@(CopyUpToDateMarker)" AlwaysCreate="true">
    <Output TaskParameter="TouchedFiles" ItemName="FileWrites" />
  </Touch>
</Target>
```

Basically this just touches the `CopyComplete` file in the opposite condition that MSBuild is doing so already.

This avoids the `Dummy.dll` issue, but it is unclear if this has unintended consequences since skipping the marker is presumably done for a reason.

# Misc notes

Note: This repository demonstrates the issue with a reference of a reference, but that is done only because:
* I thought the initial issue only happened with references of rererences.
* I wanted to demonstrate when the issue *doesn't* happen.

If you remove the reference to `LibraryB` from `LibraryA`, the issue will start happening to `TextFileA.txt` too.
