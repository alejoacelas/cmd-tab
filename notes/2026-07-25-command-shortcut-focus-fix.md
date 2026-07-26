# Command shortcut focus fix

## Symptom

After using Cmd-Tab to switch applications, subsequent Command shortcuts such as Command-K, Command-W, or Command-Q would intermittently behave as if the destination app or window did not have keyboard focus. A common manifestation was the macOS menu bar highlighting a menu whose title began with the typed letter.

## Investigation

The switcher health log consistently showed the expected selection being committed:

```text
received Command-Tab
showing 2 apps
focusing Google Chrome
```

The existing commit path hid the nonactivating panel, unhid the selected application, and called `NSRunningApplication.activate(options: [.activateAllWindows])`.

A direct local activation probe switched from super.engineering to Google Chrome and observed all of the following:

```text
accepted=true
after=Google Chrome
targetActive=true
ownsMenuBar=true
```

This ruled out a consistently rejected activation request as the immediate cause. Apple documents that `activate(options:)` attempts activation and returns whether the request was sent successfully, while `activateAllWindows` only changes which application windows are brought forward. Activation is advisory on current macOS, but it was accepted in the observed failure environment:

- https://developer.apple.com/documentation/appkit/nsrunningapplication/activate(options:)
- https://developer.apple.com/documentation/appkit/nsapplication/activationoptions/activateallwindows

AltTab uses private WindowServer APIs plus Accessibility window raising for exact cross-application, cross-Space window focus. That implementation is substantially more invasive and brittle than needed for this app-level bug:

- https://github.com/lwouis/alt-tab-macos/blob/master/src/switcher/state/Window.swift

## Root cause

The event tap passed the initial Command modifier-down event to the active source application. It intercepted the Tab key to run the switcher. When Command was released, the event tap scheduled `commitSelection()` but returned `true`, deleting the Command modifier-up event.

This created an unbalanced event stream for the source application:

```text
Command down → delivered
Tab down → consumed by Cmd-Tab
Command up → consumed by Cmd-Tab
```

The source app could therefore retain stale AppKit modifier and menu-key-equivalent state. When that app was later reactivated, a subsequent shortcut could be interpreted as menu navigation or as an unmodified letter. The intermittent nature depended on which app had retained the unmatched Command-down state and which app was selected next.

## Fix

Modifier changes now produce an explicit `ModifierChangeDecision`.

When an active switch session sees Command released, it still schedules selection commit, but `shouldConsumeEvent` is `false`. The Command-up event continues downstream before the asynchronously scheduled application switch, allowing the source app to balance the Command-down it already received.

Other modifier changes also pass through without committing.

## Verification

- Added regression coverage that an active-session Command release commits selection without consuming the event.
- Added coverage that Command-down and inactive-session modifier changes pass through without committing.
- `swift test`: six tests passed.
- `swift build -c release`: passed.
- `git diff --check`: passed.
- Rebuilt with the existing stable local signing identity.
- `codesign --verify --deep --strict`: passed for both the build artifact and `/Applications/CmdTab.app`.
- Replaced the installed bundle and relaunched it.
- Health log confirmed `Accessibility trusted=true` and `event tap installed; CmdTab is ready`.

Computer Use could not attach to the desktop because its Apple-event bridge timed out, so the final experiential test of repeated switches followed by Command shortcuts remains manual.

## If the symptom persists

Add post-activation logging for `NSWorkspace.shared.frontmostApplication`, `NSRunningApplication.isActive`, and `ownsMenuBar` after a short run-loop delay. Only if that shows a separate intermittent activation failure should the app adopt a stronger front-process/window-raise path. Avoid private WindowServer APIs until evidence shows the public application activation path is the remaining failure.
