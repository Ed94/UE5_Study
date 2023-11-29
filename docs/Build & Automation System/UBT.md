# Unreal Build Tool

Located in `Engine\Source\Programs\UnrealBuildTool\`
C# solution.

## Main Loop

Located in `UnrealBuildTool.cs`

1. Start peformance info capture
2. Parse command line arguments
3. Parse global options
4. Logging & UBT assembly setup
5. Set working directory to Engine/Source
6. Setup build mode
7. Setup tool mode options
8. Get engine directory contents
9. Read XML Configurataion
10. Create UBT run file
11. "Lock Branch"
12. Register build platforms
13. Create ToolMode which will handle the rest
14. Close out

## Global Options

Also located in same file.

Offical inline documentation:
> Global options for UBT (any modes)

Commandline options are deffined using Epic's `CommandLineAttribute`.

## ToolMode

Offical inline documentation:
> Base class for standalone UBT modes. Different modes can be invoked using the -Mode=[Name] argument on the command line, where [Name] is determined by the ToolModeAttribute on a ToolMode derived class. The log system will be initialized before calling the mode, but little else.

-Mode is defined in GlobalOptions definition.

Has a single function:

```csharp
public abstract int Execute(CommandLineArguments Arguments, ILogger Logger);
```

## BuildMode

Derived from ToolMode, used to build a *target*.

`Execute` procedure Flow:

1. Output arguments & setup the logger
2. Read the xml configuration files
3. Apply architecture configs (platform specific configs for a target)
4. More logging setup
5. Create build configuraiton object
6. Parse and build targets
    - Pase all target descriptors
    - Clean all targets with `CleanMode` (a tool mode) that have `bRebuild` flagged.
    - Handle remote builds (Seems to be excluisvely Mac)
    - Handle local builds
       - Get all project directories & build options
       - For each project: create a `SourceFileWorkingSet` object and `Build`.
7. Process the dumps to see if anything failed.
8. Save Caches

`BuildConfiguration` Contains target agonstic settings.

## QueueProjectDirectory

Used when retreving all project directories.

Will Enqueue a `FileMetadataPrefetch.ScanProjectDirectory` call for the directory with the prefecter.

### ScanProjectDirectory

1. Get all extension directoires for the target project directory
2. For each directory
    - Enqueue a scan of the plugin directory (`ScanPluginFolder`, the `Plugins` folder)
    - Enqueue a scan of the project directory (`ScanDirectoryTree`, the `Source` folder)

`Unreal.GetExtensionDirs` will do an initial scan for Platform, Restricted, and BaseDirectories, and remove them from CachedDirectories based if any of options for doing so are set.

### ScanPluginFolder

Similar to ScanProject directory, recursively scans for plugin directories.

- For all subdirectories :
   1. Check if any of the files has a .uplugin extension
      - If it does enqueue a `ScanDirectoryTree` for the `Source`  directory.
   2. Enqueue a `ScanPluginFolder`  for the sub directory.

### ScanDirectoryTree

> Scan an arbitrary directory tree.

#### DirectoryItem

This is the main container being populated as it gets recursively iterated upn by QueueProjectDirectory and its helper functions.

This is done mainly by `CacheDirectories()` which is wrapped by `EnumerateDirectories()` (called on the iteration foreach for the above queue & scanning functions).

## BuildOptions

Not much, just an enum that lets the user specify:

- Skip Build
- XGE Export
- No Engine Changes

## SourceFileWorkingSet & ISourceFileWorkingSet

> Defines an interface which allows querying the working set. Files which are in the working set are excluded from unity builds, to improve iterative compile times.

By default Unreal provides the following implmentations of the interface:

- None `EmptySourceFileWorkingSet`
- Default
- Git `GitSourceFileWorkingSet`
- Perforce `PerforceSourceFileWorkingSet`

There is no refrences to where Default is used.

If the project's provider is set to None or the flag `bGenerateProjectFiles` is set, it will use the EmptySourceFileWorkingSet (essentially disable any segmentation of the working set from the unity builds).

It picks the others based on whats within the BuildConfiguration.xml located within:
[UBT Configuration Paths from offical docs](https://docs.unrealengine.com/5.3/en-US/build-configuration-for-unreal-engine/)

## BuildAsync

BuildMode's async task generator.

The first `BuildAsync` called within `ExecuteAysnc` is a prepper for the actual `BuildAsync`

1. For all target descriptors provied:
   1. Create & execute init scripts
   2. Create async a `TargetMakeFile`.
   3. If it should also do PreBuildTargets:
      1. Add to `TargetDescriptors`

It prepares the `TargetMakeFiles` and `TargetDescriptors` for the actual procedure

The Actual `BuildAsync`:

1. Check the ActionGraph has no conflicts for the current for the current build's make files.
2. Make sure no paths exceed the max path length.
3. Make a `LinkedAction` for each `TargetMakeFile`:
   1. Convert each `TargetMakeFile` to `LinkedAction` with its `TargetDescriptor`.
   2. If a specific `TargetDescriptor` for the make file is setup for `SpecificFilesToCompile`:
      1. Check for `bSingleFileBuildDependents` and get all the related sources & headers. (To add to `FilesToBuild`)
      2. Generated a list of `ProducedItems` (`FileItem` IO action element), these are added to the `MergedOutputItems` (`Hashset` of `ProducedItems`)
4. Cleanup previous & setup current `HotReload` for a `TargetMakeFile`
5. If there are more than one `TargetDescriptor` for the build, merged them into one list of LinkedActions.
6. Call `GatherOutputItems` for all `TargetDescriptors`
   > Determines all the actions that should be executed for a target (filtering for single module/file, etc..)
7. `ActionGraph.Link` : Link all the `MergedActions` together:
   1. Build a map from item to its action.
   2. Check for cyclic-dependencies, if any are found an exection will be thrown.
   3. Setup prerequisite action for each action.
   4. Sort the action graph.
   > For improved parallelism with local execution.
8. Generate Prerequisite Actions based on what was setup with `MergedActions`
9. Generate the `ActionHistory`
10. Generate CppDependenciesCache from the TargetDescriptors
11. Create the artifact cache if one doesn't already exist. (Place to store compiled binaries, intermediate files, and other output content). Directory is specified by the `BuildConfiguration.ArtifactDirectory`
12. Generate a list of ModuleDependencyActions that track depedency files:
    1. From the ActionGraph, get all outdated actions for the cpp dependencies that need to be preprocesed.
    2. Wait for all `PreprocessActions` to execute on these dependents.
13. Determine which actions have outdated cached artifacts and need to be built: (Stored in `ActionToOutatedFlag`)
    1. Check al the link actions to see if its outdated and needs to be built.
    2. Propcate outdated state for a file to its dependents. (`ProducedModules`/`ModuleOutputs`)
    3. Check to se if external dependencies were updated. (`ImportedModules`/`ModuleImports`)
    4. Setup `PrerequisiteActions` futher.
    5. `ActionGraph.GatherAllOutdatadActions`:
    > Plan the actions to execute for the build. For single file compiles, always rebuild the source file regardless of whether it's out of date.
14. Generate a list of `LinkedAction` (`MergedActionsToExecute`) using `ActionToOutdatedFlag` and Link them to the `ActionGraph`.
15. Foreach TargetDescriptor, check to see if a hot reload patch should be applied.
    - Only one target can be hot-reloaded if more than one is attempted, an exception will be thrown.
    - If There are any non-hot reload actions to execute `HotReload.CheckForLiveCodingSessionActive` will check to prevent the build from continuing if a Live coding sessions is detected. As the user needs to shutdown the editor/engine in order for this to occur.
16. If a hot-reload target is detected, the `MergedActionsToExecute` is overwritten with a `LiveCodingActionsToExecute`.
17. If the BuildOptions has `NoEngineChanges` set, throw an exception if any engine file modifcations are detected.
18. Make sure build configuration can use the executor if for the `BuildPlatform` set for the `TargetDescriptors`
19. Delete from the `ActionGraph` any outdated `ProducedItem`s (from `MergedActionsToExecute`) and save the `ActionHistory` 
(`History`)
20. Create directories for the outdated items.
21. Time to execute actions:
    1. If just exporting XGE, export the actions to an XML file.
    2. If `WriteOutdatedActionsFile` was specified, write the actions to a JSON output.
    3. Actually execute the actions:
        1. If there are no `MergedActionsToExcecute` just state to the user that the target/s are up to date.
        2. Log targets to build and toolchain info.
        3. Set the memory per action via the `ActionExecuter`.
        4. Wait for `actionArtifactCache` to be ready.
        5. Execute each action with `ActionGraph.ExecuteActionsAsync`
        6. Flush the `actionArtifactCache` of any changes.
    4. For all the make files: Deploy the receipt of the actions via the build platform.
