# Expo Xcode 27 Repro

Minimal Expo SDK 56 project for Xcode 27 beta issues.

This repository was created with:

```sh
bun create expo-app expo-xcode27-repro --template blank-typescript --yes --no-agents-md
cd expo-xcode27-repro
bun x expo prebuild --platform ios --no-install
```

The generated `ios/` directory is intentionally committed so maintainers can inspect and run the native output without regenerating it.

## Environment

Observed locally with:

```text
expo: ~56.0.9
react-native: 0.85.3
react: 19.2.3
Xcode: 27.0 beta / Build 27A5194q
devicectl: 629.3
devicectl JSON version: 4
iOS device/runtime: iOS 27.0
```

## Issue 1: UIScene Lifecycle Runtime Crash

When this unmodified Expo SDK 56 app is built with Xcode 27 beta, the app installs but fails to launch immediately.

Runtime issue shown by Xcode:

```text
failure in void _UIApplicationEvaluateRuntimeIssueForNoSceneLifecycleAdoption(void)_block_invoke (UIApplication_RuntimeIssues.m:106):
Application failed to launch: UIScene life cycle is required for apps built with this SDK. See Technote TN3187 for more information on migration.
```

The generated native project does not include `UIApplicationSceneManifest`:

```sh
plutil -extract UIApplicationSceneManifest raw -o - ios/expoxcode27repro/Info.plist
# UIApplicationSceneManifest: missing
```

The generated `AppDelegate.swift` still owns the main `UIWindow` in `application(_:didFinishLaunchingWithOptions:)`:

```swift
@main
class AppDelegate: ExpoAppDelegate {
  var window: UIWindow?

  public override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
  ) -> Bool {
    let delegate = ReactNativeDelegate()
    let factory = ExpoReactNativeFactory(delegate: delegate)
    delegate.dependencyProvider = RCTAppDependencyProvider()

    reactNativeDelegate = delegate
    reactNativeFactory = factory

    window = UIWindow(frame: UIScreen.main.bounds)
    factory.startReactNative(
      withModuleName: "main",
      in: window,
      launchOptions: launchOptions)

    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

### Reproduction

```sh
bun install
bun run ios
```

Or open `ios/expoxcode27repro.xcodeproj` in Xcode 27 beta and run the app on an iOS 27 simulator or device.

Expected: the app launches.

Actual: UIKit stops the app at launch because the scene lifecycle is required for apps built with the Xcode 27 SDK.

## Issue 2: Expo CLI Xcode 27 Device Hub/devicectl

Xcode 27 beta appears to have renamed or moved the simulator/device management app expected by Expo CLI.

With Xcode 27 selected:

```sh
xcode-select -p
# /Applications/Xcode-beta.app/Contents/Developer
```

`simctl` exists and works:

```sh
xcrun -f simctl
# /Applications/Xcode-beta.app/Contents/Developer/usr/bin/simctl
```

But the app path currently checked by Expo CLI does not exist:

```sh
ls /Applications/Xcode-beta.app/Contents/Developer/Applications/Simulator.app
# No such file or directory
```

Xcode 27 beta has Device Hub instead:

```sh
ls /Applications/Xcode-beta.app/Contents/Applications/DeviceHub.app
# /Applications/Xcode-beta.app/Contents/Applications/DeviceHub.app

/usr/libexec/PlistBuddy -c 'Print :CFBundleIdentifier' \
  /Applications/Xcode-beta.app/Contents/Applications/DeviceHub.app/Contents/Info.plist
# com.apple.dt.Devices
```

`expo run:ios --device` also warns about the `devicectl` JSON version:

```text
Unexpected devicectl JSON version output from devicectl. Connecting to physical Apple devices may not work as expected.
```

Local `devicectl` output:

```sh
xcrun devicectl list devices --json-output /tmp/devices.json --timeout 5
python3 - <<'PY'
import json
print(json.load(open('/tmp/devices.json'))['info']['jsonVersion'])
PY
# 4
```

Expected: Expo CLI recognizes Xcode 27's Device Hub setup and accepts `devicectl` JSON version 4.
