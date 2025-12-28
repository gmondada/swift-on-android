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

```shell
./gradlew :hello-swift-raw-jni:assembleDebug
```

This step can actually be skipped, as the command in the next section will build the APK anyway.

## Install the APK

```shell
./gradlew :hello-swift-raw-jni:installDebug
```

NOTE:

* This command also builds the APK if it has not already been built.
* Android Studio uses the `-Pandroid.injected.testOnly=true` option to ensure the APK never gets distributed. We omit it for simplicity.
* Depending on how the APK is built, a bug in lldb-server prevents some libraries from being debugged. See the "Issues" section below.

## Start the APK and lldb-server on the target

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-lldb-server). The script requires the app ID as an argument:

```shell
./apk-lldb-server org.example.helloswift -D
```

The `-D` option makes the app display the "Waiting for Debugger" popup. In this mode, the app does not fully start, giving the debugger a chance to connect and set all breakpoints before the app runs.

The script will block the terminal. Let it run, and open a new terminal for further commands.

To stop `lldb-server`, press **CTRL-C**. To kill the app, use:

```shell
adb shell am force-stop org.example.helloswift
```

NOTES:

* The `apk-lldb-server` script creates the `~/.lldb/android/last-start-commands` file containing the commands that `lldb` needs to connect to `lldb-server` and attach to the app process. It also stores the app ID and PID in `~/.lldb/android/last-env`.
* The script runs `lldb-server` in the app sandbox, to mimic what Android Studio does.
* The script also cleans up what left behind by previous debug sessions. In a context where lldb crashes frequently, it's important.
* The script outputs the `lldb-server` log on screen.

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

* With the version of LLDB currently shipped with the Swift toolchain, you will probably never reach this point. In practice, many bugs cause LLDB to hang or crash. These issues are listed in the “Issues” section below. To fix or work around them, you need to build LLDB from source.
* `lldb` stores data in the `~/.lldb/module_cache` directory. If you want to investigate the issues mentioned above, it is often useful to clear this cache and start `lldb` without any dependencies on previous runs.
* I personally use my own script to start LLDB, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-swift-lldb). The script is trivial: it clears the cache and starts LLDB.

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

In this early startup phase, the app is also listening for JDWP connections on a Unix‑domain socket. This is the socket used by the Java debugger to control the JVM. To dismiss the popup and continue execution, somebody needs to connect to this socket. We do this with a shell script, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/dismiss-wfd), which uses `jdb` (from the Java SDK) behind the scenes.

```shell
./dismiss-wfd org.example.helloswift
```

Another, more hacky, way to dismiss this popup is to search for `VMDebug_isDebuggerConnected()`, which is the C++ implementation of `Debug.isDebuggerConnected()`. By stopping just before the `ret` instruction and setting the `w0` register to `1`, we force the function to return true.

TODO: explore the feasibility of loading an agent which, like the JDWP agent, overrides `Debug.isDebuggerConnected()`.

### Use alternative versions of LLDB

The Android NDK comes with LLDB. [Here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-ndk-lldb) is the script I use to start it.

This version has no Swift support, but allows you to list modules and symbols, disassembly code, etc. I sometimes use it to determine whether an issue comes from the LLVM LLDB or the Swift LLDB. Unfortunately, it has similar bugs as the Swift's one. In fact, the bugs mentioned above are often not due to Swift, but to Android, JVM and shared object support in lldb.

Xcode also comes with it own `lldb`. You can start it with the command:

```shell
xcrun lldb -o "command source -s 0 -e 0 '~/.lldb/android/last-start-commands'"
```

## Issues

Work to fix the issues listed below is ongoing. Many fixes can be found at: https://github.com/gmondada/llvm-project/tree/gab/swift-lldb-patches.

### Invalid URL

When running the `platform connect...` command in `lldb`, we get this error:

```console
error: Invalid URL: connect://[127.0.0.1]gdbserver.3312f5
```

This seems to come from `lldb-server` which, on connection, reports having a process ready for debugging on that GDB socket (on qQueryGDBServer), which is not true. We use the `lldb-server` provided by the NDK. Moving to a more recent NDK does not change the behavior. Same with `lldb-server` embedded in Android Studio.

### Crash when loading modules from the target (solved)

When running  `process attach...`,  `lldb` often crashes. This happens when pulling modules from the target via ADB. This problem has already been fixed in the llvm.org project, and the related commits are in the Swift project, on branch `next`. To apply them, go in `llvm-project` and cherry-pick the following commits:

```console
git cherry-pick -x bcb48aa5b2cfc75967c734a97201e0c91273169d
git cherry-pick -x 91418ecbdef0e259f83e6ddac5ddfc22a8b6eced
git cherry-pick -x 223cfa8018595ff2a809b4e10701bfea884af709
git cherry-pick -x a19c9a8ba1b01f324f893481d825a375a5a68bc6
git cherry-pick -x 55b0d143d654d9f6c0bc515eaf5a66980a151a4d
git cherry-pick -x f8cb6cd989c8159ede0b454a433dd2b5632c1cb6
```

### Attach by name (solved)

Attaching a process by name does no work, at least not with the `lldb` provides in the NDK or in the swift toolchain. A fix exists in the llvm.org project. It's part of the commits listed above.

NOTE: On Android, attaching by name is equivalent to attaching by app ID, since processes spawned by Zygote use the app ID as their process name.

### Deadlock when loading modules (solved)

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

This is no specific to Android, it happens on macOS as well. To reproduce the problem, create a `test` program:

```swift
struct JNINativeInterface {}
typealias JNIEnv = UnsafePointer<JNINativeInterface>

func MainActivity_stringFromSwift(env: UnsafeMutablePointer<JNIEnv?>) {
    print("JNIEnv pointer: \(env)")  // <= put a breakpoint here (line 5)
}

@main
struct comparative_example {
    static func main() {
        var env: JNIEnv? = nil
        MainActivity_stringFromSwift(env: &env)
    }
}

```

Compile with 6.3-snapshot-2025-12-18 or main-snapshot-2025-12-19 on Mac, then run

```shell
swiftly run lldb test -o "breakpoint set -f test.swift -l 5" -o run
```

### Printing strings in `lldb` does not work (solved)

When paused in a function, printing local variables of type String results in errors:

```lldb
(lldb) print i
(Int) 19
(lldb) print hello
(String) <cannot decode string: unexpected discriminator>
(lldb) print version
(String) <cannot decode string: memory read failed for 0xffffff8000000000>
```

On AArch64, a `String` is a 16-byte structure containing the string length, some flags, and a pointer to the string data. Swift uses pointer tagging to store certain flags in the high bits of this pointer.

Android also uses pointer tagging and reserves the 8 most significant bits of pointers. As a result, on Android, Swift cannot store its flags in the same location as on other platforms. Unfortunately, LLDB is unaware of this Android-specific behavior and therefore looks for the flags in the wrong place.

### Printing arrays in lldb does not always work (solved)

As for strings, arrays in Swift make use of tagged pointers. Android has specific constraints on pointer tagging and lldb is unaware of this.

### Stop on __jit_debug_register_code (solved)

Normally, when we hit a breakpoint, the execution stops and VS Code highlights the corresponding line of code. Sometimes, however, it's like we hit two breakpoints at the same time, and the second breakpoint is on `__jit_debug_register_code`. A that moment, VS Code highlights the code (in this case the desassembled code) of `__jit_debug_register_code`.

I'm not sure if hitting two breakpoints at the same time is something possible by design or just a incorrectly reported state. But the breakpoint on `__jit_debug_register_code` is an internal trick to get notified when the JIT compiler has generated new code dynamically (see https://llvm.org/docs/DebuggingJITedCode.html). `lldb-dap` should not make this visible to VS Code, at least not as a user breakpoint.

```console
(lldb) thread list
Process 31012 stopped
  thread #1: tid = 31012, 0x00000078ad768c1c libc.so`syscall + 28, name = 'mple.helloswift'
  thread #2: tid = 31014, 0x00000078ad7a7548 libc.so`__rt_sigtimedwait + 8, name = 'Signal Catcher'
  thread #3: tid = 31015, 0x00000078ad7a5b04 libc.so`read + 4, name = 'perfetto_hprof_'
  thread #4: tid = 31016, 0x00000078ad7a83c4 libc.so`__ppoll + 4, name = 'ADB-JDWP Connec'
  thread #5: tid = 31017, 0x00000076071d5774 libart.so`__jit_debug_register_code, name = 'Jit thread pool', stop reason = jit-debug-register
  thread #6: tid = 31018, 0x00000078ad768c1c libc.so`syscall + 28, name = 'HeapTaskDaemon'
  thread #7: tid = 31019, 0x00000078ad768c1c libc.so`syscall + 28, name = 'ReferenceQueueD'
  thread #8: tid = 31020, 0x00000078ad768c1c libc.so`syscall + 28, name = 'FinalizerDaemon'
  thread #9: tid = 31021, 0x00000078ad768c1c libc.so`syscall + 28, name = 'FinalizerWatchd'
  thread #10: tid = 31022, 0x00000078ad7a6148 libc.so`__ioctl + 8, name = 'binder:31012_1'
  thread #11: tid = 31023, 0x00000078ad7a6148 libc.so`__ioctl + 8, name = 'binder:31012_2'
  thread #12: tid = 31027, 0x00000078ad7a6148 libc.so`__ioctl + 8, name = 'binder:31012_3'
  thread #13: tid = 31039, 0x00000078ad768c1c libc.so`syscall + 28, name = 'Profile Saver'
  thread #15: tid = 31057, 0x00000078ad768c1c libc.so`syscall + 28, name = 'JDWP Event Help'
  thread #17: tid = 31059, 0x00000078ad7a7944 libc.so`__recvmsg + 4, name = 'JDWP Transport '
  thread #18: tid = 31060, 0x00000078ad7a6148 libc.so`__ioctl + 8, name = 'binder:31012_4'
  thread #19: tid = 31061, 0x00000078ad7a6148 libc.so`__ioctl + 8, name = 'binder:31012_5'
  thread #20: tid = 31062, 0x00000078ad7a8188 libc.so`__epoll_pwait + 8, name = 'RenderThread'
* thread #22: tid = 31065, 0x0000007560cf3aa4 libhelloswift.so`generateSomeText(wordCount=17) at helloswift.swift:36:23, name = 'FreshNewStack', stop reason = breakpoint 1.1
```

### JVM code in the backtrace

As soon as we debug code called from Java, we face strange behaviours and crashes.

TODO: be more specific...

