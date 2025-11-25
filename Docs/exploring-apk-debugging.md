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

Open a terminal in the app’s root folder and run:

```console
$ ./gradlew :hello-swift-raw-jni:assembleDebug
```

This step can actually be skipped, as the command in the next section will build the APK anyway.

## Install the APK

```console
$ ./gradlew :hello-swift-raw-jni:installDebug
```

## Start the APK and lldb-server on the target

We use a shell script for this, which can be downloaded from [here](https://github.com/gmondada/swift-on-android/blob/main/Scripts/apk-lldb-server).

The script requires the app ID as an argument:

```console
$ scripts/apk-lldb-server org.example.helloswift -D
```

The `-D` option makes the app display the "Waiting for Debugger" popup. In this mode, the app does not fully start, giving the debugger a chance to connect and set all breakpoints before the app runs.

The script will block the terminal. Let it run, and open a new terminal for further commands.

To stop `lldb-server`, press **CTRL-C**. To kill the app, use:

```console
$ adb shell am force-stop org.example.helloswift
```

## Start the debugging session

There are a few options:

1. Use the LLDB from the Android NDK. This allows listing symbols, disassembly code, etc but not provide Swift-specific information. This is sometimes useful to determine whether an issue comes from the LLVM LLDB or the Swift LLDB.
2. Use the LLDB from the Swift Android SDK. It has full Swift support.
3. Start a debugging session in VS Code, based on lldb-dap.

