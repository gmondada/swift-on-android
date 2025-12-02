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

NOTE: The `apk-lldb-server` script creates the `/tmp/lldb-commands` file containing the commands that `lldb` needs to connect to `lldb-server` and attach to the app process. It also stores the app ID and PID in `/tmp/lldb-target.env`.

## Start the debugging session

There are a few options:

1. Use the LLDB from the Android NDK. This allows listing symbols, disassembly code, etc but not provide Swift-specific information. This is sometimes useful to determine whether an issue comes from the LLVM LLDB or the Swift LLDB.
2. Use the LLDB from the Swift toolchain. It has full Swift support.
3. Start a debugging session in VS Code, based on lldb-dap.

### Use the LLDB from the Android NDK

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-ndk-lldb). We pass the app ID as an argument, but this is optional. When omitted, the script just uses the data stored by  `apk-lldb-server` in the `/tmp` folder.

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
```

Most of the time, the `Invalid URL` error appears. This looks like a bug, but it's not critical, it just prevents the next commands in the script from executing automatically. To work around this, there are the aliases `a1`, `a2`, `a3`, and `a4` to quickly run these commands manually:

```console
(lldb) a1
Process 8847 stopped
* thread #1, name = 'mple.helloswift', stop reason = signal SIGSTOP
    frame #0: 0x0000007b000ce35c libc.so`syscall + 28
libc.so`syscall:
->  0x7b000ce35c <+28>: svc    #0
    0x7b000ce360 <+32>: cmn    x0, #0x1, lsl #12 ; =0x1000 
    0x7b000ce364 <+36>: cneg   x0, x0, hi
    0x7b000ce368 <+40>: b.hi   0x7b0011ee58   ; __set_errno_internal
...
Architecture set to: aarch64-unknown-linux-android.
(lldb) a2
...
(lldb) a3
...
(lldb) a4
...
```

You can now set a breakpoint and continue execution:

```console
(lldb) b Java_org_example_helloswift_MainActivity_stringFromSwift
Breakpoint 1: where = libhelloswift.so`Java_org_example_helloswift_MainActivity_stringFromSwift, address = 0x00000077d6cdf368
(lldb) cont
Process 8847 resuming
```

If you choose the `-D` option above, you need to dismiss the "Waiting For Debugger" popup. There is a dedicated section about this later in the document.

### Use the LLDB from the Swift toolchain

We use a shell script for this, available [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-swift-lldb). We pass the app ID as an argument, but this is optional. When omitted, the script just uses the data stored by `apk-lldb-server` in the `/tmp` folder.

```console
$ scripts/apk-swift-lldb org.example.helloswift
App ID: org.example.helloswift
Process ID: 8847
Swift Toolchain: /Users/gabriele/Library/Developer/Toolchains/swift-DEVELOPMENT-SNAPSHOT-2025-10-16-a.xctoolchain
(lldb) command source -s 0 '/tmp/lldb-commands'
Executing commands in '/tmp/lldb-commands'.
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
```

Most of the time, the `Invalid URL` error appears. This looks like a bug, but it's not critical, it just prevents the next commands in the script from executing automatically. To work around this, there are the aliases `a1`, `a2`, `a3`, and `a4` to quickly run these commands manually:

```console
(lldb) a1
warning: (aarch64) /Users/gabriele/.lldb/module_cache/remote-android/.cache/4ECB2C57-D666-7E4A-94A7-3568CE392D22/app_process64 No LZMA support found for reading .gnu_debugdata section
...
Process 10971 stopped
* thread #1, name = 'mple.helloswift', stop reason = signal SIGSTOP
    frame #0: 0x0000007b000ce35c libc.so`syscall + 28
libc.so`syscall:
->  0x7b000ce35c <+28>: svc    #0
    0x7b000ce360 <+32>: cmn    x0, #0x1, lsl #12 ; =0x1000 
    0x7b000ce364 <+36>: cneg   x0, x0, hi
    0x7b000ce368 <+40>: b.hi   0x7b0011ee58   ; __set_errno_internal
...
Target 0: (app_process64) stopped.
Executable binary set to "/Users/gabriele/.lldb/module_cache/remote-android/.cache/4ECB2C57-D666-7E4A-94A7-3568CE392D22/app_process64".
Architecture set to: aarch64-unknown-linux-android.
(lldb) a2
(lldb) a3
(lldb) a4
```

Here we often hit another issue: the `a1` command (equivalent to `process attach --pid 8847`) often crashes or hangs `lldb`. Some of these bugs have been corrected in the llvm project (see below).

TODO: explain workaround

TODO: some more lldb commands

If you choose the `-D` option above, you need to dismiss the "Waiting For Debugger" popup. There is a dedicated section about this later in the document.

### Start a debugging session in VS Code

TODO

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

This seems to come from `lldb-server` which, on connection, reports having a process ready for debugging on that GDB socket (on qQueryGDBServer), which is not true. We use the lldb-server provided by the NDK. Moving to a more recent NDK does not change the behavior.

TODO: verify if Andorid Studio comes with its own version of lldb

### Crash when loading modules from the target

When running  `process attach...`,  `lldb` often crashes. This happens when pulling modules from the target via ADB. This problem has already been fixed in the llvm.org project, and the related commits are in the Swift project, on branch `next`. To apply them, go in `llvm-project` and cherry-pick the following commits:

```console
git cherry-pick bcb48aa5b2cfc75967c734a97201e0c91273169d
git cherry-pick 91418ecbdef0e259f83e6ddac5ddfc22a8b6eced
git cherry-pick 223cfa8018595ff2a809b4e10701bfea884af709
git cherry-pick a19c9a8ba1b01f324f893481d825a375a5a68bc6
git cherry-pick 55b0d143d654d9f6c0bc515eaf5a66980a151a4d
git cherry-pick f8cb6cd989c8159ede0b454a433dd2b5632c1cb6
```

### Attach by name

Attaching a process by name does no work, at least not with the `lldb` provides in the NDK or in the swift toolchain. A fix exists in the llvm.org project. It's part of the commits listed above.

### Deadlock when loading modules

The swift toolchain `lldb` hangs on `process attach...`, when loading modules.

TODO: investigate