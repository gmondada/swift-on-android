# Debugging an APK containing Swift code

## Introduction

This document is part of an exploratory effort to experiment with debugging Swift code on Android.

Here, we focus on debugging Swift code embedded in an Android app. The app may be written in Kotlin, with the Swift code compiled into a shared library accessed through JNI.

We will go through these steps:

* Compile and build an APK for debugging
* Install the APK on the target (on the Android emulator)
* Start the APK on the target
* Start lldb-server on the target
* Start the lldb debugging session
* Eventually, start the Java debugging session

## Background

We need the following tools: swiftly, the swift compiler, the Swift SDK for Android, the Android NDK, the Android emulator and adb.

The Android emulator must be running, and it must be the only device visible to `adb`.

Let's check the environment:

```console
$ swiftly use
main-snapshot-2025-10-16 (default)

$ swift sdk list
swift-DEVELOPMENT-SNAPSHOT-2025-10-16-a-android-0.1

$ echo $ANDROID_NDK_HOME
/Users/gabriele/Library/Android/sdk/ndk/android-ndk-r27d

$ adb --version
Android Debug Bridge version 1.0.41
Version 36.0.0-13206524

$ adb devices
List of devices attached
emulator-5554	device

$ adb shell uname -m
aarch64
```

We also need an app to debug. In this example, we use the `hello-swift-raw-jni` app from the [`swift-android-examples`](https://github.com/swiftlang/swift-android-examples) repository.

## Build an APK for debugging

Open a terminal in the `swift-android-examples` root folder and run:

```console
$ ./gradlew :hello-swift-raw-jni:assembleDebug
```

This step can actually be skipped, as the command in the next section will build the APK anyway.

## Install the APK

```console
$ ./gradlew :hello-swift-raw-jni:installDebug
```

NOTE: Should we add `-Pandroid.injected.testOnly=true` as done by Android Studio?

## Start the APK and lldb-server on the target

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-lldb-server). The script requires the app ID as an argument:

```console
$ ./apk-lldb-server org.example.helloswift -D
```

The `-D` option makes the app display the "Waiting for Debugger" popup. In this mode, the app does not fully start, giving the debugger a chance to connect and set all breakpoints before the app runs.

The script will block the terminal. Let it run, and open a new terminal for further commands.

To stop `lldb-server`, press **CTRL-C**. To kill the app, use:

```console
$ adb shell am force-stop org.example.helloswift
```

NOTE: The `apk-lldb-server` script creates the `~/.lldb/android/last-start-commands` file containing the commands that `lldb` needs to connect to `lldb-server` and attach to the app process. It also stores the app ID and PID in `~/.lldb/android/last-env`.

## Start the debugging session

There are a few options:

1. Use the LLDB from the Android NDK. This allows listing symbols, disassembly code, etc but not provide Swift-specific information. This is sometimes useful to determine whether an issue comes from the LLVM LLDB or the Swift LLDB.
2. Use the LLDB from the Swift toolchain. It has full Swift support.
3. Start a debugging session in VS Code, based on lldb-dap.

### Use the LLDB from the Android NDK

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-ndk-lldb). We pass the app ID as an argument, but this is optional. When omitted, the script just uses the data stored by  `apk-lldb-server` in the `~/.lldb/android` folder.

```console
$ ./apk-ndk-lldb org.example.helloswift
(lldb) platform select remote-android
  Platform: remote-android
 Connected: no
(lldb) platform connect unix-abstract-connect:///org.example.helloswift-0/lldb-platform.sock
  Platform: remote-android
    Triple: aarch64-unknown-linux-android
OS Version: 31 (5.10.110-android12-9-00004-gb92ac325368e-ab8731800)
  Hostname: localhost
 Connected: yes
WorkingDir: /data/user/0/org.example.helloswift
    Kernel: #1 SMP PREEMPT Tue Jun 14 13:40:53 UTC 2022
error: Invalid URL: connect://[127.0.0.1]gdbserver.35df1d
(lldb) process attach --pid 8242
...
```

NOTE: The `Invalid URL` error looks like a bug, but it's not critical. See below.

You can now set a breakpoint and continue execution:

```console
(lldb) b Java_org_example_helloswift_MainActivity_stringFromSwift
Breakpoint 1: where = libhelloswift.so`Java_org_example_helloswift_MainActivity_stringFromSwift, address = 0x00000077d6cdf368
(lldb) cont
Process 8847 resuming
```

If you choose the `-D` option above, you need to dismiss the "Waiting For Debugger" popup. There is a dedicated section about this later in the document.

### Use the LLDB from the Swift toolchain

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-swift-lldb). We pass the app ID as an argument, but this is optional. When omitted, the script just uses the data stored by `apk-lldb-server` in the `~/.lldb/android` folder.

```console
$ scripts/apk-swift-lldb org.example.helloswift
App ID: org.example.helloswift
Process ID: 8847
Swift Toolchain: /Users/gabriele/Library/Developer/Toolchains/swift-DEVELOPMENT-SNAPSHOT-2025-10-16-a.xctoolchain
(lldb) command source -s 0 '~/.lldb/android/last-start-commands'
Executing commands in '~/.lldb/android/last-start-commands'.
(lldb) command alias a1 process attach --pid 8847
(lldb) command alias a2 process handle SIGSEGV -n false -p true -s false
(lldb) command alias a3 process handle SIGBUS -n false -p true -s false
(lldb) command alias a4 process handle SIGCHLD -n false -p true -s false
(lldb) platform select remote-android
  Platform: remote-android
 Connected: no
(lldb) platform connect unix-abstract-connect:///org.example.helloswift-0/lldb-platform.sock
  Platform: remote-android
    Triple: aarch64-unknown-linux-android
OS Version: 31 (5.10.110-android12-9-00004-gb92ac325368e-ab8731800)
  Hostname: localhost
 Connected: yes
WorkingDir: /data/user/0/org.example.helloswift
    Kernel: #1 SMP PREEMPT Tue Jun 14 13:40:53 UTC 2022
error: Invalid URL: connect://[127.0.0.1]gdbserver.3312f5
(lldb) a1
...
```

NOTE: The `Invalid URL` error looks like a bug, but it's not critical. See below.

Here we often have a problem: the `a1` command (equivalent to `process attach --pid xxx`) crashes or hangs `lldb`. Some of these bugs have been corrected in the llvm project (see below).

TODO: explain workaround

TODO: some more lldb commands

TODO: explain why we use a1, a2, ... aliases.

If you choose the `-D` option above, you need to dismiss the "Waiting For Debugger" popup. There is a dedicated section about this later in the document.

### Start a debugging session in VS Code

We assume VS Code has the official Swift extensions installed, including "Swift" (from swift.org) and "LLDB DAP" (from llvm.org).

Create or edit `.vscode/settings.json` and add:

```json
{
    "lldb-dap.executable-path": "xxx/usr/bin/lldb-dap"
}
```

Replace `xxx` with the path to your toolchain. If you’re unsure where it is, run the following command:

```bash
swiftly use --print-location
```

Then, create or edit `.vscode/launch.json` and add:

```json
{
    "configurations": [
        {
            "type": "lldb-dap",
            "request": "launch",
            "name": "Hello Swift",
            "launchCommands": [
                "command source -s 0 -e 0 '~/.lldb/android/last-start-commands'"
            ],
        }
    ]
}
```

In VS Code, open the **Run & Debug** tab on the left, select "Hello Swift" in the dropdown menu, and press the green ▶️ button. The debug session should start as expected.

If you choose the `-D` option above, you need to dismiss the "Waiting For Debugger" popup. There is a dedicated section about this below.

### Dismiss the "Waiting For Debugger" popup

If you use the `-D` option when running `apk-lldb-server`, the app starts by displaying a "Waiting For Debugger" popup. At this stage, the app is in an early startup phase, stuck in a loop polling `Debug.isDebuggerConnected()` until it returns `true`. This allows us to attach the debugger and set breakpoints before the app continues running.

In this early startup phase, the app is also listening for JDWP connections on a Unix‑domain socket. This is the socket used by the Java debugger to control the JVM. To dismiss the popup and continue execution, somebody needs to connect to this socket. We do this with a shell script, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/dismiss-wfd), which uses `jdb` (from the Java SDK) behind the scenes.

```console
$ scripts/dismiss-wfd org.example.helloswift
```

Another, more hacky, way to dismiss this popup is to search for `VMDebug_isDebuggerConnected()`, which is the C++ implementation of `Debug.isDebuggerConnected()`. By stopping just before the `ret` instruction and setting the `w0` register to `1`, we force the function to return true.

TODO: explore the feasibility of loading an agent which, like the JDWP agent, overrides `Debug.isDebuggerConnected()`.

## Issues

### Invalid URL

When running the `platform connect...` command in `lldb`, we get this error:

```console
error: Invalid URL: connect://[127.0.0.1]gdbserver.3312f5
```

This seems to come from `lldb-server` which, on connection, reports having a process ready for debugging on that GDB socket (on qQueryGDBServer), which is not true. We use the `lldb-server` provided by the NDK. Moving to a more recent NDK does not change the behavior. Same with `lldb-server` embedded in Android Studio.

### Crash when loading modules from the target

When running  `process attach...`,  `lldb` often crashes. This happens when pulling modules from the target via ADB. This problem has already been fixed in the llvm.org project, and the related commits are in the Swift project, on branch `next`. To apply them, go in `llvm-project` and cherry-pick the following commits:

```console
git cherry-pick -x bcb48aa5b2cfc75967c734a97201e0c91273169d
git cherry-pick -x 91418ecbdef0e259f83e6ddac5ddfc22a8b6eced
git cherry-pick -x 223cfa8018595ff2a809b4e10701bfea884af709
git cherry-pick -x a19c9a8ba1b01f324f893481d825a375a5a68bc6
git cherry-pick -x 55b0d143d654d9f6c0bc515eaf5a66980a151a4d
git cherry-pick -x f8cb6cd989c8159ede0b454a433dd2b5632c1cb6
```

### Attach by name

Attaching a process by name does no work, at least not with the `lldb` provides in the NDK or in the swift toolchain. A fix exists in the llvm.org project. It's part of the commits listed above.

NOTE: On Android, attaching by name is equivalent to attaching by app ID, since processes spawned by Zygote use the app ID as their process name.

### Deadlock when loading modules

The swift toolchain `lldb` hangs on `process attach...`, when loading modules. A fix is already available in brach `next`. Just cherry-pick it:

```console
git cherry-pick -x 7fb620a5cc02a511a019d6918b0993b4cbdea825
git cherry-pick -x 66d5f6a60550a123638bbdf91ec8cff76cb29c5a
```

### Swift library not visible from lldb

After having changed some Swift code and rebuilt the APK, `lldb` stops making the Swift code debuggable. Breakpoints cannot be set, and the command `image list -b libhelloswift.so` acts as if the image does not exist. However, we know the shared library has been loaded because the Swift code is running.

Analysis:

* `lldb` gets notified when the shared object is loaded (`DynamicLoaderPOSIXDYLD::RefreshModules()` gets called and sees the new module).
* It then asks lldb-server for the `ModuleSpec` through the GDB port, command `qModuleInfo`. It provides the module path, which points to the shared object inside the APK: `/data/app/~~JD12dD8imR6vc8siCWvUtQ==/org.example.helloswift-XNHfP5zPCR2PCBxNvXJErg==/base.apk!/lib/arm64-v8a/libhelloswift.so`.
* `lldb-server` replies with an error, and `lldb` ignores this module.
* When changing some Swift code and rebuilding the APK, the shared library appears as the last element in the APK. This can be seen with shell command `unzip -lv ./hello-swift-raw-jni/build/outputs/apk/debug/hello-swift-raw-jni-debug.apk`. In that case, I always hit the problem. By changing the manifast file or anything else that makes the shared object moving to the second to last or any other position in the APK, the problem disappears.
* I tested with lldb-server from NDK 29.0.14206865 and from Android Studio. Both show the problem.

Workaround:

Rebuild the APK entirerly, or make any change other than the shared object just before installing. Example:

```console 
./gradlew :hello-swift-raw-jni:assembleDebug
./gradlew :hello-swift-raw-jni:installDebug -Pandroid.injected.testOnly=true
```

The `-Pandroid.injected.testOnly=true` option causes a change in the manifest, which ends up in being the last file in the APK.

### Assertion on UnsafeMutablePointer<JNIEnv?> 

When pausing execution in a context involving a UnsafeMutablePointer<JNIEnv?>, `lldb` aborts on this assertion:

```
Assertion failed: (!m_elem_type.GetTypeName().GetStringRef().starts_with("Swift.Optional")), function Update, file SwiftUnsafeTypes.cpp, line 373.
```

Basically, this appends every time we put a breakpoint in a function directly called by JNI.

It happens also when printing a variable of type `UnsafeMutablePointer<JNIEnv?>` with the `print` command in `lldb`.

### Printing strings in `lldb` does not work

When paused in a function, printing local variables of type String results in errors:

```lldb
(lldb) print i
(Int) 19
(lldb) print hello
(String) <cannot decode string: unexpected discriminator>
(lldb) print version
(String) <cannot decode string: memory read failed for 0xffffff8000000000>
```

