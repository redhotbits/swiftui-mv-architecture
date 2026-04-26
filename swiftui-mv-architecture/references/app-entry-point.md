# App entry point

The SwiftUI `App` protocol replaces both `AppDelegate` and `SceneDelegate`. Do not generate either file unless the project has a concrete need that the framework cannot satisfy on its own.

## Default — App only, no delegates

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

No `AppDelegate.swift`, no `SceneDelegate.swift`. This is the correct starting point for every new SwiftUI project.

## When UIKit lifecycle hooks are genuinely required

If a third-party SDK or system API requires `UIApplicationDelegate` callbacks (push notification registration, background fetch configuration, etc.), wire it in with `@UIApplicationDelegateAdaptor` — do not make the AppDelegate the `@main` entry point.

```swift
@main
struct MyApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

Only add the `AppDelegate` file at that point, and only implement the specific callbacks the SDK requires. Leave every other method out — don't copy the full Xcode template boilerplate.

## SceneDelegate

Almost never needed in SwiftUI. The `App`/`Scene` protocol handles multi-window support natively. Add a `SceneDelegate` only if you are bridging a UIKit scene lifecycle that cannot be expressed in SwiftUI's `Scene` API.

## What to delete from Xcode-generated projects

When Xcode generates a new project with UIKit lifecycle, clean up before writing any SwiftUI:

1. Delete `AppDelegate.swift` and `SceneDelegate.swift`
2. Delete `Main.storyboard` and remove `UIMainStoryboardFile` from build settings
3. Clear `UIApplicationSceneManifest` from `Info.plist`
4. Create `@main struct MyApp: App`
