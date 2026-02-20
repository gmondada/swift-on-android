# Debugging Swift on Android

Swift relies on LLDB for debugging. Swift and LLDB both come with a VS Code extension, making VS Code a good place to experiment with them. Making these tools working for debugging Swift on Android is an ongoing process, tracked [here](https://github.com/swiftlang/llvm-project/issues/10831). The ultimate goal is to integrate these tools in a way that provides a smooth and intuitive user experience for anyone trying or adopting Swift on Android.

I'm documenting here some of the ongoing work.

## Exploratory work

* [Debugging a Swift executable on Android](exploring-lldb-dap.md)
* [Debugging an APK containing Swift code](exploring-apk-debugging.md)

## Issues and related PRs

- Deadlock when loading modules
  - Solved by cherry-picking from branch `next`
  - PR: https://github.com/swiftlang/llvm-project/pull/12049
  - Merged in `swiftlang:stable/21.x`
- Crash when loading modules from the target + Attach processes by name
  - Solved by cherry-picking from branch `next`
  - PR: https://github.com/swiftlang/llvm-project/pull/12042
  - Merged in `swiftlang:stable/21.x`
- Do not expose internal breakpoints to DAP clients
  - PR: https://github.com/llvm/llvm-project/pull/173848
  - Merged in `llvm:main`
  - PR: https://github.com/swiftlang/llvm-project/pull/12263
  - Merged in `swiftlang:stable/21.x`
- Management of Android specific pointer tagging in LLDB
  - PR: https://github.com/swiftlang/llvm-project/pull/12050
  - Merged in `swiftlang:stable/21.x`
- lldb-server ignoring last entry in zip files
  - PR: https://github.com/llvm/llvm-project/pull/173966
  - In review, discussing tests
- Assertion on UnsafeMutablePointer<JNIEnv?>
  - Issue: https://github.com/swiftlang/llvm-project/issues/12053
  - PR: https://github.com/swiftlang/llvm-project/pull/12295
  - Merged in `swiftlang:swift/release/6.3`
- Correctly format LLDB warnings in the debug console (cosmetic issue)
  - PR: https://github.com/llvm/llvm-project/pull/173852
  - In review, discussing tests
