# Automation Tool Driver

For a general overview of the Automation Tool see [AutomationTool](AutomationTool.md)

A simple C# program to help a user script unattended processes the engine can complete.  
The entire tool is implemented within `Program.cs` and has all its implmentationw within the `Program` class.  
The following headers will be a breakdown of each of its members.

## Main

1. Sets up logger
2. Call `ParseCommandLine` (populates `AutomationToolCommandLine` and ``CommandsToExecute`)
3. Waits for debugger attachment if `AutomationCommandLine` detects `-WaitForDebugger` is set.
4. If MacOS or Linux setup Ctrl+C to be handled correctly so there is no zombie process situation.
5. Set the working directory for the Environment (to the Engine's root directory)
6. Various contextual logging ( operating env, whether launched from launcher, app version...)
7. Run `MainProc` using `ProcessSingleton.RunSingleInstanceAsync`

## AutomationToolCommandLine

A static data member. Is of type `System.ParsedCommandline`. and popualated by `Main`.

| Flag                         | Summary                                                                                             |
|------------------------------|-----------------------------------------------------------------------------------------------------|
| `-Verbose`                   | Enables verbose logging.                                                                            |
| `-VeryVerbose`               | Enables very verbose logging.                                                                       |
| `-TimeStamps`                | Adds timestamps to log entries (no specific description provided).                                  |
| `-Submit`                    | Allows UAT command to submit changes.                                                               |
| `-NoSubmit`                  | Prevents any submit attempts.                                                                       |
| `-NoP4`                      | Disables Perforce functionality (default if not run on a build machine).                            |
| `-P4`                        | Enables Perforce functionality (default if run on a build machine).                                 |
| `-IgnoreDependencies`        | Ignores script dependencies (no specific description provided).                                     |
| `-Help`                      | Displays help information.                                                                          |
| `-List`                      | Lists all available commands.                                                                       |
| `-NoKill`                    | Does not kill any spawned processes on exit.                                                        |
| `-UTF8Output`                | Outputs logs in UTF-8 encoding (no specific description provided).                                  |
| `-AllowStdOutLogVerbosity`   | Allows standard output log verbosity settings (no specific description provided).                   |
| `-NoAutoSDK`                 | Disables automatic SDK setup (no specific description provided).                                    |
| `-Compile`                   | Force all script modules to be compiled.                                                            |
| `-NoCompile`                 | Do not compile any script modules, run with whatever is up to date.                                 |
| `-IgnoreBuildRecords`        | Ignore build records in determining if script modules are up to date.                               |
| `-UseLocalBuildStorage`      | Use local storage for root build storage dir, changing default to `Engine\Saved\LocalBuilds`.       |
| `-WaitForDebugger`           | Waits for a debugger to be attached, and breaks once debugger successfully attached.                |
| `-BuildMachine`              | Indicates the program is running on a build machine (no specific description provided).             |
| `-WaitForUATMutex`           | Waits for the Unreal Automation Tool mutex (no specific description provided).                      |
| `-msbuild-verbose`           | Increases verbosity for MSBuild operations.                                                         |
| `-NoCompileUAT`              | Disables compilation of the Unreal Automation Tool (UAT).                                           |

## ParseCommandLine

Populates `AutomationToolCommandLine` and `CommandsToExecute` (which is within the former and is of type `List<CommandInfo>`).

It will throw execptions if it finds anything wrong with the commands passed.

## ParseProfile

Used by `ParseCommandLine`. Used if `-Profile` is passed to the automation tool.  
Handles profile files which are used to configure how the AutomationTool executes. These profile files can contain a variety of settings and scripts that need to be processed.

1. **Profile File Identification**: Identifies if a `-profile` argument is present in the command line and extracts the profile file path.
2. **Arguments Initialization**: Initializes a list to store the arguments extracted from the profile file.
3. **Iterating Command Line Arguments**: Iterates through command line arguments, treating `-profile=` as the profile file path, and adds other arguments to the Arguments list.
4. **Profile File Existence Check**: Verifies if the specified profile file actually exists.
5. **Reading and Parsing Profile File**: Reads the content of the profile file and parses it as JSON to extract a dictionary of settings.
6. **Processing Scripts in Profile**: Extracts and processes scripts from the `scripts` section in the JSON:
    - Extracts the name of each script and adds it to the Arguments list if not already present.
    - Removes the 'script' key from the script's dictionary.
    - Calls `ParseDictionary` on the dictionary to parse and add these settings to the Arguments list.
7. **Updating Command Line Arguments**: Replaces the original command line arguments with the updated Arguments list containing information processed from the profile file.

## ParseParam

Used by `ParseCommandLine`.

1. **Checks for Ignored Parameters**: Identifies and skips parameters that are not relevant for processing, ensuring efficiency in parsing.
2. **Global Parameter Setting**: Sets global parameters that affect the overall operation of the tool, based on the identified command-line arguments.
3. **Handling Specific Options**: Specifically processes options like `-ScriptsForProject`, `-ScriptDir`, and `-Telemetry`, adjusting the tool's behavior accordingly.
4. **Managing Unknown Parameters and Environment Variables**: Handles parameters that are unrecognized or not standard, and also manages the influence of environment variables on the tool.
5. **Setting Command-line Arguments in `AutomationToolCommandLine`**: Populates the `AutomationToolCommandLine` with the parsed parameters, which is a key resource used throughout the tool.
6. **Exception Handling**: Implements error handling to manage potential issues during the parsing process, ensuring stability and user feedback.

## ParseDictionary

1. **Parameter Verification**: Checks if the dictionary parameter is not null, ensuring it's suitable for processing.
2. **Dictionary Iteration**: Iterates through each key-value pair in the dictionary parameter.
3. **Key Processing**: Processes the keys of the dictionary, which could involve validation, formatting, or conversion to ensure they are in the correct format for further operations.
4. **Value Processing**: Examines and processes the values associated with each key. This step may involve additional parsing logic, especially if the values are complex types (like nested dictionaries or lists).
5. **Recursive Handling**: In cases where values are collections (like lists or dictionaries), `ParseDictionary` may recursively call itself or other parsing methods (like `ParseList`) to ensure all elements are properly processed.
6. **Constructing Processed Dictionary**: Builds a new dictionary with the processed keys and values, which is then ready for use in the tool's operations.

Uses the following helper functions:

### ParseString

1. **Parameter Verification**: Checks if the string parameter is null or empty to ensure only valid strings are processed.
2. **Trimming Whitespace**: Removes leading or trailing whitespace from the parameter string for uniform processing.
3. **String Processing**: Applies specific processing to the cleaned string parameter, such as format conversion or additional parsing logic.

### ParseList

1. **Parameter Verification**: Ensures the list parameter is not null for safe processing.
2. **List Iteration**: Goes through each element in the list parameter.
3. **Element Processing**: Trims whitespace, validates, and converts each list element to the required format or type.
4. **Compiling Processed Elements**: Creates a new list or array from the processed elements, prepared for further use in the tool's workflow.

## MainProc

Core of the AutomationTool's operation, orchestrating the parsing of command-line arguments and executing the necessary commands.

1. **Logger Initialization**: Sets up a logger for information logging.
2. **Initialization Message**: Logs the initialization of script modules.
3. **Start Time Recording**: Records the start time for performance tracking.
4. **Command Line Argument Processing**: Retrieves and processes various command-line arguments related to scripts, compilation, and execution modes (e.g., `-ScriptsForProject`, `-ScriptDir`, `-Compile`, `-NoCompile`, `-IgnoreBuildRecords`).
5. **Command Preparation**: Prepares the list of commands to be executed based on the command-line arguments.
6. **Script Module Compilation**: Initializes and compiles script modules, handling any build processes required and logging related information.
7. **Build Success Check**: Checks if the script module build was successful. If not, returns an error exit code.
8. **Script Module Validation**: Ensures that at least one script module (like AutomationUtils) is available after compilation.
9. **Loading AutomationUtils Assembly**: Loads the `AutomationUtils.Automation.dll` assembly, essential for further processing.
10. **Automation Process Invocation**: Invokes the `ProcessAsync` method from the `AutomationTool.Automation` class, which drives the execution of the tool's functionality based on the prepared script modules and command-line arguments.
11. **Execution Time Logging**: Logs the total time taken for script module initialization.
12. **Awaiting Process Completion**: Awaits the completion of the `ProcessAsync` method and returns its result as an exit code.

## StartupListener

> Captures all log output during startup until a log file writer has been created
