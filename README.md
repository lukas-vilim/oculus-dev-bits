# OculusDebugger Extension
As far as I understand what is hidden behind the courtains of the VS code extension it is a wrapper around lldb 12.0 widh deployment scripts for Standalone devices. There is very little to be found on the internet about how to setup and troubleshoot the system and the sources are not available. In my desperation I even decompiled the debugcito.

Anyway, there are a couple of pieces to the whole puzzle.
- debugcito
    - Vanilla lldb 12.0 (this is where the lldb-server for your arm64 comes from)
    - Android SDK Tools (in case you don't have platform-tools in your path already)
- fb-lldb 12.0 (supposedly modded version built specifically for meta standalone devices)

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

### attachCommands failed to attach to process
This is a common error message that means that something went wrong during the initial setup of lldb's connection, that you can replay from `%TMP%/lldbinit`, and it has probably even crashed. This failure might be completelly silent and without a trace of an error in the logs.

If you are running Oculus Unreal 5.+ it's worth checking the generated .vscode project files for bad path formats. Something happened when the team migrated the fork over to UE5 and now the projects contain more escapes in the debugger settings than neccessarry. The `lldb` does not like escaped paths so if you see `\\\"` in your `launch.json` project file you are in trouble and this will silently fail the `lldb` process.
