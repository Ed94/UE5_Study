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
