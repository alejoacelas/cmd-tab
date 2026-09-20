# Cmd-Tab decisions

## Core decisions

### Scope

- [Reproduce the selected AltTab behavior without importing its unused surface](#decision-1).

### Keyboard correctness

- [Pass Command release through before committing the switch](#decision-2).

### macOS identity

- [Keep stable signing and verify the actual focus boundary](#decision-3).

## Details

<a id="decision-1"></a>

### Reproduce the selected AltTab behavior without importing its unused surface

Keep one main eligible window per application, the documented exclusions, ordering, shortcuts and icon-based compact display. configuration.md is the target contract; settings, alternate shortcuts, telemetry and updater features are outside this small replacement. See [configuration.md](configuration.md).

<a id="decision-2"></a>

### Pass Command release through before committing the switch

The old event tap swallowed modifier-up after delivering modifier-down, leaving stale shortcut/menu state. Preserve ModifierChangeDecision’s non-consuming modifier events; do not add private WindowServer activation machinery unless new evidence isolates a separate activation failure. See [Sources/CmdTab/CmdTabModel.swift](Sources/CmdTab/CmdTabModel.swift).

<a id="decision-3"></a>

### Keep stable signing and verify the actual focus boundary

Stable signing preserves Accessibility and Screen Recording grants across rebuilds. The modifier regression had unit/build coverage but the final repeated-switch experiential check was blocked; do not present that as completed live verification. See [notes/2026-07-25-command-shortcut-focus-fix.md](notes/2026-07-25-command-shortcut-focus-fix.md). History inspected: [beb7198](https://github.com/alejoacelas/cmd-tab/commit/beb7198), [59fac32](https://github.com/alejoacelas/cmd-tab/commit/59fac32).
