# Debugging an APK containing Swift code

## Introduction

This document is part of an effort to enable Swift debugging on Android (https://github.com/swiftlang/llvm-project/issues/10831). It explores the various steps needed to set up a debugging session.

Here, we focus on debugging Swift code embedded in an Android app. The app may be written in Kotlin, with the Swift code compiled into a shared library accessed through JNI.

We will go through these steps:

* Compile and build an APK for debugging
* Install the APK on the target (on the Android emulator)
* Start the APK on the target
* Start lldb-server on the target
* Start the lldb debugging session

## Background

We need the following tools: swiftly, the swift compiler, the Swift SDK for Android, the Android NDK, the Android emulator and adb.

The Android emulator must be running, and it must be the only device visible to `adb`.

Let's check the environment:

```console
$ swiftly use
6.3-snapshot-2026-02-27 (default)

$ swift sdk list
swift-6.3-DEVELOPMENT-SNAPSHOT-2026-02-27-a_android

$ echo $ANDROID_NDK_HOME
/Users/gabriele/Library/Android/sdk/ndk/android-ndk-r27d

$ adb --version
Android Debug Bridge version 1.0.41
Version 36.0.0-13206524

$ adb devices
List of devices attached
emulator-5554	device
```

We also need an app to debug. In this example, we use the `hello-swift-raw-jni` app from the [`swift-android-examples`](https://github.com/swiftlang/swift-android-examples) repository.

## Build an APK for debugging

Open a terminal in the `swift-android-examples` root folder and run:

```shell
./gradlew :hello-swift-raw-jni:assembleDebug
```

Depending on when you checked out the examples, you may need to update the  `swift-android.gradle.kts` in this way:

```kotlin
var swiftVersion: String = "6.3-snapshot-2026-02-27", // Swift version
var androidSdkVersion: String = "6.3-DEVELOPMENT-SNAPSHOT-2026-02-27-a_android" // SDK 
```

NOTE: This build phase can actually be skipped, as the command in the next section will build the APK anyway.

## Install the APK

```shell
./gradlew :hello-swift-raw-jni:assembleDebug
./gradlew :hello-swift-raw-jni:installDebug -Pandroid.injected.testOnly=true
```

In theory, the first command is not needed, and the `-Pandroid.injected.testOnly=true` parameter is not needed too. However, we do this intentionnaly, to work around a bug in lldb-server (see https://github.com/llvm/llvm-project/pull/173966). The goal is to avoid having a shared library as the last entry in the APK. The second command modifies the manifest, which then automatically becomes the last entry.

## Start the APK and lldb-server on the target

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-lldb-server). The script requires the app ID as an argument:

```shell
./apk-lldb-server org.example.helloswift -D
```

The `-D` option makes the app display the "Waiting for Debugger" popup. In this mode, the app does not fully start, giving the debugger a chance to connect and set all breakpoints before the app runs.

The script will block the terminal. Let it run, and open a new terminal for further commands.

NOTES:

* The `apk-lldb-server` script creates the `~/.lldb/android/last-start-commands` file containing the commands that `lldb` needs to connect to `lldb-server` and attach to the app process. It also stores the app ID and PID in `~/.lldb/android/last-env`.
* The script runs `lldb-server` in the app sandbox, to mimic what Android Studio does.
* The script also cleans up what left behind by previous debug sessions. In a context where lldb crashes frequently, it's important.
* To stop the app, use the command `adb shell am force-stop org.example.helloswift`

## Start the debugging session

We have two options:

1. Use LLDB from the command line
2. Start a debugging session in VS Code, based on lldb-dap

### Use LLDB from the command line

To start `lldb` just open a terminal and run:

```shell
swiftly run lldb -o "command source -s 0 -e 0 '~/.lldb/android/last-start-commands'"
```

You should see something like this:

```console
$ swiftly run lldb -o "command source -s 0 -e 0 '~/.lldb/android/last-start-commands'"
(lldb) command source -s 0 -e 0 '~/.lldb/android/last-start-commands'
Executing commands in '/Users/gabriele/.lldb/android/last-start-commands'.
(lldb) platform select remote-android
  Platform: remote-android
 Connected: no
(lldb) platform connect unix-abstract-connect:///org.example.helloswift/lldb-platform.sock
  Platform: remote-android
    Triple: aarch64-unknown-linux-android
OS Version: 31 (5.10.110-android12-9-00004-gb92ac325368e-ab8731800)
  Hostname: localhost
 Connected: yes
WorkingDir: /data/user/0/org.example.helloswift
    Kernel: #1 SMP PREEMPT Tue Jun 14 13:40:53 UTC 2022
error: Invalid URL: connect://[127.0.0.1]gdbserver.3312f5
(lldb) process attach --name org.example.helloswift
...
```

You can now set a breakpoint and continue execution:

```console
(lldb) breakpoint set -f helloswift.swift -l 17
Breakpoint 1: no locations (pending).
(lldb) cont
Process 8847 resuming
```

If you choose the `-D` option above, you need to dismiss the "Waiting For Debugger" popup. There is a dedicated section about this later in the document.

When the breakpoint is hit, the execution stops.

```console
Process 8847 stopped
* thread #1, name = 'mple.helloswift', stop reason = breakpoint 1.1
```

NOTES:

* `lldb` stores data in the `~/.lldb/module_cache` directory. When facing issues, it's often useful to clear this cache and start `lldb` without any dependencies on previous runs.
* If you see the error `Invalid URL: connect://[127.0.0.1]gdbserver.e286f6` just ignore it. It's a bug in lldb-server which does not cause any real problem.

### Start a debugging session in VS Code

We assume VS Code has the official Swift extensions installed, including "Swift" (from swift.org) and "LLDB DAP" (from llvm.org).

Create or edit `.vscode/settings.json` and add:

```json
{
    "lldb-dap.executable-path": "xxx/usr/bin/lldb-dap"
}
```

Replace `xxx` with the path to your toolchain. If you’re unsure where it is, run the following command:

```shell
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

In this early startup phase, the app is also listening for JDWP connections on a Unix‑domain socket. This is the socket used by the Java debugger to control the JVM. To dismiss the popup and continue execution, someone needs to connect to this socket. We do this with a shell script, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/dismiss-wfd), which uses `jdb` (from the Java SDK).

```shell
./dismiss-wfd
```

Another, more hacky, way to dismiss this popup is to search for `VMDebug_isDebuggerConnected()`, which is the C++ implementation of `Debug.isDebuggerConnected()`. By stopping just before the `ret` instruction and setting the `w0` register to `1` (on AArch64), we force the function to return true.
