# Oculus
## Current Android SDK Project setup
| Name                                  | UE 4.25+ | UE 5.0+ | UE 5.1+ | UE 5.2+ |
| --------------------------------------|----------|---------|---------|---------|
| __Android Studio__                    | 4.0 (from May 28, 2020) | - | -  | - |
| __Android SDK Version__               | *SetupAndroid.bat* | - | - | - |
| __Android SDK Platform Tools__        | Latest | - | - | - |
| __Andorid Command Line Tools__        | 8.0 | 8.0 | 8.0 | 9.0 |
| __NDK Version__                       | r21b | r21e | r25b | r25b |
| __AGDE__                              | - | - | v22.2.69+ | v23.1.82+ |
| __SDK API Level__                     | Latest | - | - | - |
| __NDK API Level__                     | android-21 | - | - | - |
| __Android Minimum SDK Version__       | 29 | - | - | - |
| __Android Target SDK Version__        | 29 | - | - | - |
| __Enable FullScreen Immersive on...__ | true | - | - | - |
| __Oculus Plugin Name__                | OculusVR | OculusVR | MetaXR | MetaXR |
| __Support OpenGL ES3.2__              | true | false | false | false |
| __Support Vulkan__                    | false | true | true | true |

## Random Notes
- Presence of MetaXR plugin in the `plugin` folder during compilation with Oculus fork of *Unreal Engine* causes issues and crashes.
- __MSAA__ is the only way. Preferably 4x.
- Always disable __Mobile HDR__.
- __Mobile Multi-View__ works only in certain versions (looks like in 5.1.1 and above it works well).
- For debugging always *Package data inside APK*.
- For *UE 5.+* disable to __Android File Server__ Plugin.
- __OpenXR__ is the only API supported further.
- __PhysX__ is getting slowly deprecated.
- __Nanite__ is not yet supported on *Oculus Quest* devices.
- Performance regression from UE4 to UE5 is possible.
- Ensure to enable __AndroidDeviceProfileSelector__ plugin as it might hang the apps on start otherwise (UE5).
- CPU __Occlusion Culling__ is not supported by UE5 and the new GPU system has couple of downsides.
    - There is a one frame delay.
    - More work for the GPU.
- Use *Android Studio* to solve __gradle__ errors. You can re-package the project without having to build and cook everything and even clean and package again.

## Debugging with AGDE
This is still a mistery to me. I was not able to make it work just yet.

## SetupAndroid.bat troubleshooting
The script will fail to find the __sdkmanager.bat__ for certain versions of *Unreal Engine*. If you find yourself in such situation either install older version of the __Android SDK Platform Tools__ or just edit the script and direct the towards the right path of your __sdkmanager.bat__. I would recommend the latter.

# OculusDebugger Extension
As far as I understand what is hidden behind the courtains of the *Visual Studio Code* extension is a wrapper around *lldb* 12.0 with deployment scripts for Standalone devices *(Quest 1, Quest 2, Quest Pro)*. There is very little to be found on the internet about how to setup and troubleshoot the system and the sources are not available. In my desperation I even decompiled the __debugcito__.

Anyway, there are a couple of pieces to the whole puzzle.
- __debugcito__
    - Vanilla lldb 12.0 (this is where the lldb-server for your arm64 comes from)
    - Android SDK Tools (in case you don't have platform-tools in your path already)
- __fb-lldb 12.0__ (supposedly modded version built specifically for meta standalone devices)

## Unreal Engine Integration
__UnrealBuildTool__ from the Oculus fork of *Unreal Engine* is able to produce the *Visual Studio Code* workspace that contains additional targets and *tasks* for the *OculusDebugger Extension*.

To generate the workspace you have to either switch your code editor of choice to *Visual Studio Code* in the *Unreal Engine Editor* or call the __UnrealBuildTool__ or one of the helper scripts with __CLI__ parameters `\UnrealEngine\Engine\Build\BatchFiles\GenerateProjectFiles.bat <absolute_path_to_your_uproject_file> -game -vscode`. This will produce the workspace in your projects root directory.

After opening the workspace you should find a couple of additional options in the __Run and Debug__ section:
- __<ProjectName> Oculus (Debug) Launch__
- __<ProjectName> Oculus (DebugGame) Launch__

- __<ProjectName> Oculus (Development) Launch__
- __<ProjectName> Oculus (Development) Attach__
- __<ProjectName> Oculus (Shipping) Launch__

Before running the those you have to first fully __build and cook__ the project for *Android* as all of the skip the _cooking_ step by default. If you don't see any of the options above in your workspace make sure you are running __UBT__ from the Oculus fork of the *Unreal Engine* and that it does not produce any errors during the process.

> *Tip: You can Build and Cook right from the workspace without running the editor (CTRL+SHIFT+P -> Run Task -> <ProjectName> Android Development Deploy/Build/Cook/Rebuild/Clean*

## The debugcito process
This is how the process runs when your device is clean and you are running the debugger for the first time.

1. The lldb-vscode.exe gets started from the fb-lldb directory with your project parameters. You can override couple of the settings at this point. Like ADB and JRE directories.
2. Checks `adb devices`
3. Checks `adb shell getprop.ro.debuggable`
4. Kills the lldb-server `adb shell killall lldb-server`
5. Really kills the lldb-server `adb shell su root killall lldb-server`
6. Clears the `<TEMP>/lldbinit` file
7. Kills the target APP `adb shell am force-stop com.<company>.<app>`
8. Another kill attempt at the lldb-server via `adb shell run-as com.<company>.<app> killall lldb-server`
9. Kills some kind of oculus service? `adb shell am force-stop com.oculus.os.clearactivity`
10. Starts the App `adb shell am start -D -a android.intent.action.Main -n com.oculus.HandsTrainSample/com.epicgames.ue4.GameActivity`
11. Sets the App up for debug `shell am set-debug-app com.oculus.HandsTrainSample`
12. Checks for presense of lldb-server in app folder `/data/data/com.<company>.<app>/lldb-server`
13. Checks for presense of lldb-server in tmp folder `/data/local/tmp/lldb-server-arm64/lldb-server`
14. Copies over the lldb-server from PC to `data/local/tmp/lldb-server-arm64`
15. Copies the lldb-server from tmp to the app data/data folder and chmods the execution flag
16. Runs the lldb-server `shell run-as com.<company>.<app>  debugcito/lldb-server p --server --listen unix-abstract:///com.<company>.<app>-debugcito/debugcito-6738677603.sock </dev/null  >/dev/null 2>&1 &`
17. Checks for the lldb-server actually running
18. Writes the lldb config into the `<TEMP>/lldbinit` file. (Useful for manual debugging)
19. Attempts to attach the running lldb-vscode.exe to the running lldb-server on your device

## Troubleshooting

### Clean the Extension data
Use the `Clean LLDB caches and Oculus runtime downloads` command from the `CTRL+SHIFT+P` Menu

If for any reason you don't have access to the command just delete the folder `~/.oculus_debugger_runtime`, `~/.lldb`, `~/AppData/Roaming/Code`

### Hanging Extension download
If the Extension ends up hanging on any of the download steps shutdown the VSCode and clean the downloaded folders and try again. Resuming you are risking more problems down the line.

### "Unable to establish connection with debug server on the device, please start over to retry." Error
This message pops-up and you should be able to find it in the Oculus Debugger log as well. Prefixed by couple of error messages attempting to start the lldb-server on your device this signifies that you are either running into issues with permissions or the lldb-server binary is actually crashing. The latter is more probable.

Try to run `adb shell run-as com.<company>.<app> debugcito/lldb-server` and if this yeilds a segfault then one of the copy steps corrupted the binary or your download didn't work out just right. If you end up with a permission error you might have a bad project setup and you should check the android project settings and regenerate and recook the project.

To fix the corrupted binary you have to first `Clean the Extension data` and make sure the extension downloads properly. Then you have to clean up the corrupted binaries from your device as the `debugcito` caches them.

`adb shell run-as com.<company>.<app> rm -rf debugcito` and `adb shell rm -rf /data/local/tmp/lldb-server-arm64` should take care of it.

### lldb python crash when loading scripts
If you encounter a problem during your experiments where the lldb crashes after loading the formatter scripts `UnrealEngine/Engine/Extras/LLDBDataFormatters/UEDataFormatters_2ByteChars.py`, or running python in general `lldb script`, try settings the `PYTHONPATH` env variable to `~/.oculus_debugger_runtime/lldb/2021.02.10/lldb/PythonLibs`. If this does not help your lldb is probably corrupted and you should consider redownloading it clean.

Note that the VSCode extension uses the lldb-vscode.exe and sets the `PYTHONPATH` at start-up. This should not be the core of your issues but might help you investigate the setup further.

### Logs are your only hope
The oculus debugger extension and the underlyin `debugcito` is closed source and I wasn't able to find anything useful even from disassembling it. So logs are your only hope and it is a real shame that many of the return values from the device are discarded in the process. 

The best thing I can recommend is to try to replay the same setup the `debugcito` does under the hood yourself from the steps you see in the logs and find fishy. Fortunatelly it does show the full ADB commands and file paths.

### "attachCommands failed to attach to process" Error
This is a common error message that means that something went wrong during the initial setup of lldb's connection, that you can replay from `%TMP%/lldbinit`, and it has probably even crashed. This failure might be completelly silent and without a trace of an error in the logs.

If you are running Oculus fork of Unreal Engine 5 it's worth checking the generated .vscode project files for bad path formats. Something happened when the team migrated the fork over to UE5 and now the projects contain more escapes in the debugger settings than neccessarry. Last time I checked the issue was present for all the tag `oculus-5.0.0-release-1.71.0-v39.0` and __above__. The `lldb` does not like escaped paths so if you see `\\\"` in your `launch.json` project file you are in trouble and this will silently fail the `lldb` process.

To fix this you'll have to edit the engine sources at `UnrealEngine/Engine/Source/Programs/UnrealBuildTool/ProjectFiles/VisualStudioCode/VSCodeProjectFileGenerator.cs:WriteNativeLaunchConfigAndroidOculus(...)` and remove the extra escapes in this section:

```
...
    OutFile.BeginObject("lldbConfig");
    {
        OutFile.BeginArray("librarySearchPaths");
        OutFile.AddUnnamedField("\\\"" + SymbolPathArm64.ToNormalizedPath() + "\\\""); // << REMOVE HERE
        OutFile.EndArray();

        OutFile.BeginArray("lldbPreTargetCreateCommands");
        FileReference DataFormatters = FileReference.Combine(ProjectRoot, "Engine", "Extras", "LLDBDataFormatters", "UEDataFormatters_2ByteChars.py");
        OutFile.AddUnnamedField("command script import \\\"" + DataFormatters.FullName.Replace("\\", "/") + "\\\""); // << REMOVE HERE
        OutFile.EndArray();

        OutFile.BeginArray("lldbPostTargetCreateCommands");
        //on Oculus devices, we use SIGILL for input redirection, so the debugger shouldn't catch it.
        OutFile.AddUnnamedField("process handle --pass true --stop false --notify true SIGILL");
        OutFile.EndArray();
    }
...
```
> *I'm not sharing exact line numbers or patch since the sources might change and tracking down the exact spot would be more difficult.*
