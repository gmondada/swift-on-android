# Debugging a Swift executable on Android

This is a simple exploratory work to experiment with Swift debugging on Android, see what works and what doesn’t, and understand how difficult the available tools are to use. It’s just an initial step toward the ultimate goal, which is to integrate these tools in a way that provides a smooth and intuitive user experience for anyone trying or adopting Swift on Android.

## Background

* Based on [work by Jason Foreman](https://github.com/threeve/swift-android-sdk/wiki/Remote-Debugging-with-LLDB)
* Focused on debugging a simple executable built with Swift, not an Android app that includes Swift code as a library
* Using VS Code and `lldb-dap`, aiming to stay as close as possible to how the Swift VS Code extension works today
* Toolchain: `main-snapshot-2025-10-16`
* Android SDK: `swift-DEVELOPMENT-SNAPSHOT-2025-10-16-a-android-0.1`

## Starting `lldb-server` on the target

We assume that:

* The Android emulator is running. Therefore, `adb devices` should list a single device, and `adb shell uname -m` should display `aarch64`.
* `ANDROID_NDK_HOME` is set correctly, as required when installing the Android SDK (here using `android-ndk-r27d`).
* The Swift toolchain and the Swift Android SDK are installed properly.
* A Swift package with a single executable target named **hello** is available. This is the program we’ll run and debug on Android.

In the root directory of the **hello** package, execute the following commands:

```bash
# build the executable
swiftly run swift build --swift-sdk aarch64-unknown-linux-android29  --static-swift-stdlib

# push the executable to the target
adb push .build/aarch64-unknown-linux-android29/debug/hello /data/local/tmp

# push libc++ to the target
adb push ${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/darwin-x86_64/sysroot/usr/lib/aarch64-linux-android/libc++_shared.so /data/local/tmp

# push lldb-server to the target
adb push ${ANDROID_NDK_HOME}/toolchains/llvm/prebuilt/darwin-x86_64/lib/clang/18/lib/linux/aarch64/lldb-server /data/local/tmp

# start lldb-server on the target
adb shell cd /data/local/tmp \; ./lldb-server platform --server --listen unix-abstract:///data/local/tmp/debug.socket
```

At this stage, `lldb-server` is running, but the `hello` program will only start when the debug session is launched in VS Code.

## Starting the debug session in VS Code

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
            "name": "hello on Android",
            "launchCommands": [
                "!platform select remote-android",
                "!platform connect unix-abstract-connect:///data/local/tmp/debug.socket",
                "!env LD_LIBRARY_PATH=/data/local/tmp",
                "!file /data/local/tmp/hello",
                "!b main",
                "!run"
            ],
        }
    ]
}
```

In VS Code, open the **Run & Debug** tab on the left, select "hello on Android" in the dropdown menu, and press the green ▶️ button. The debug session should start as expected.

## Notes

* `lldb` automatically manages adb port forwarding 💪
* In this setup, we artificially set a breakpoint at `main`, otherwise the program starts before any other breakpoint has a chance to be set. I currently don't see how to configure the `lldb-dap` session to run the program after having set all breakpoints.

## Bugs

The only bug I'm facing is a crash in `lldb-dap` (or `lldb`) when trying to display variables or expressions which do not exist. This can be easily reproduced by the lldb command `print abc`, assuming you do NOT have an `abc` variable.

The crash is actually an assertion:

```
Assertion failed: (m_initialized_search_path_options && m_initialized_clang_importer_options && "search path options must be initialized before ClangImporter"), function GetASTContext, file SwiftASTContext.cpp, line 3607.
```

This problem has already been noticed by Jason Foreman few months ago. In his case, however, there was an error message, not a crash:

```
error: Error while searching for Xcode SDK: Unrecognized SDK type: Android.sdk
error: could not create a Swift AST context
```

By using the version of `lldb` embedded in Xcode, `llbd` does not crash and displays this error message:

```
error: could not initialize Swift compiler
```

By enabling logs in lldb (with the lldb command `log enable lldb all`), we see "Failed to compute triple" just before the crash. However, the same message appears with the Xcode version. Therefore, the problem is not in the triple or in creating the `SwiftASTContext` object, but probably in how this object is used later.

## Links

* https://github.com/swiftlang/llvm-project/issues/10831

* https://github.com/swiftlang/llvm-project/blob/next/lldb/source/Plugins/TypeSystem/Swift/SwiftASTContext.cpp
