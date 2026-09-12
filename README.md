# Handoff

Handoff is a macOS menu-bar app that notices when you are repeating a multi-step task and offers to finish the rest for you. There is nothing to record and nothing to script. You do the task a couple of times the way you normally would; Handoff works out the pattern, shows you what it thinks the next pass looks like, and takes over only after you confirm.

The kind of work it is built for looks like this: copying each assignment from a course page into a calendar, creating a folder for every name on a client list, or looking up a drive time for each address in a spreadsheet. The steps are identical every time and only the data changes.

## How it works

```
watch  ->  normalize  ->  detect  ->  understand  ->  confirm  ->  replay
```

1. **Watch.** A global event tap captures clicks and keystrokes in every app. Each click is hit-tested against the accessibility tree so Handoff knows it landed on "row 4 of the file list", not just on a pixel.
2. **Normalize.** Raw events are folded into human-scale steps: `Copy`, `Switch to Finder`, `Click item 4 of 12`, `Type "..."`.
3. **Detect.** The step stream is scanned for its shortest repeating sequence. Steps whose values changed between passes are marked as the task's parameters, and Handoff infers the rule behind each one (next row, incrementing number, a value read off another window, the clipboard).
4. **Understand.** The pattern is given a plain-language name. A built-in namer handles this offline; optionally, Claude can name it from a structure-only summary.
5. **Confirm.** A small non-activating window appears. You can edit the name and the run count, choose whether to stop before the committing step, and read a dry-run preview of exactly what the next pass will do.
6. **Replay.** Each element is re-found live in the accessibility tree on every pass. Before any committing step, a guard checks that the world still looks the way the recording assumed. Pressing any key aborts the run immediately.

## Why it is not a macro recorder

- **Elements, not coordinates.** A click is stored as a role path and ordinal and re-resolved on every pass, so the task survives windows moving, resizing, or scrolling.
- **Values are inferred, not replayed.** Handoff distinguishes the fixed structure of a task from the parts that change, and computes the next value instead of pasting the old one.
- **Structural identity.** Two tabs titled "Directions to 520 Chestnut St" and "Directions to 88 Oak Ave" are recognised as the same step with a different value. Without this, most real tasks are undetectable because the data lives in the labels.
- **Guarded replay.** An unexpected dialog, a missing target, or a value that does not match the expected shape causes Handoff to flag that row and move on rather than type into the wrong place.
- **Irreversible steps are handed back.** By default Handoff performs the mechanical part and stops before Send, Save, or Post so you can confirm.

## Privacy

Everything runs locally. The pattern library at `~/.handoff/patterns.json` (mode 0600) stores structure only: step kinds, role paths, ordinals, and hashes. It never contains typed text, file names, URLs, or window titles.

The one optional network call is task naming through Claude. It is off unless an API key is configured, and by default it sends step kinds plus the site hostname, never page contents or typed text. A network failure falls back to the built-in namer. Keystrokes typed while macOS secure input is active (password fields) never enter the pipeline.

## Requirements

- macOS 15 or later on Apple Silicon.
- An Xcode toolchain. The build calls the toolchain binaries directly rather than through `xcodebuild`; see `HackCMU/scripts/env.sh` for the reasoning.
- Two permissions, granted once: **Accessibility** and **Input Monitoring**. Reading other apps' UI and observing input both require them.

## Getting started

All commands run from the `HackCMU/` directory.

```sh
cd HackCMU
./handoff cert      # one time: create the "Handoff Dev" signing identity
./handoff run       # build, sign, and launch; a menu-bar icon appears
./handoff doctor    # confirm signing, permissions, and a live event tap
```

The app is always launched through `open` so that macOS attributes the permission grants to Handoff rather than to your terminal. Grant Accessibility and Input Monitoring from the menu-bar panel or from System Settings > Privacy & Security.

To see the suggestion flow without performing a task first:

```sh
./handoff demo
```

### Trying it on a real task

Handoff needs the passes to be consistent. Do the same steps in the same order each time; one stray click per pass is tolerated. The staging script sets up three ready-made tasks with scratch data:

| Command | Task | What it demonstrates |
|---|---|---|
| `./handoff stage 1` | Finder list view: click a row, ⌘I, ⌘W, for each file | Walking a list; the suggestion appears on the third pass |
| `./handoff stage 2` | Copy each name from a Safari page, create a folder for it in Finder | Cross-app data movement; one name contains a `/` that Finder rejects, which the replay guard catches |
| `./handoff stage 3` | Copy each assignment title into a calendar form in another tab and click Save | A committing step; Handoff stops before Save by default |

If nothing appears, open the menu-bar panel. It shows the steps seen, patterns found, and the reason a suggestion was withheld (for example, "this is just typing"). To capture a full trace, run `./handoff trace`, perform the task, and read `/tmp/handoff-trace.txt`.

## Commands

| Command | Description |
|---|---|
| `./handoff run` | Build, sign, and launch |
| `./handoff demo` | Show the suggestion window on a built-in sample |
| `./handoff doctor` | Preflight: signing, permissions, live event tap |
| `./handoff stage {1\|2\|3}` | Set up one of the sample tasks and restart Handoff clean |
| `./handoff test` | Headless self-test suite plus a render of the suggestion window |
| `./handoff trace` | Run with step and web-snapshot tracing to `/tmp/handoff-trace.txt` |
| `./handoff axdump` | Print what the resolver sees in the front Finder and Safari windows (read-only) |
| `./handoff replaytest [--live]` | Exercise replay targeting against the live Finder window; dry by default |
| `./handoff understand` | Print the exact payload that would be sent to Claude, and call it if a key is set |
| `./handoff cert` | Create the "Handoff Dev" signing identity |
| `./handoff dr` | Print the designated requirement (must be certificate-based, not a cdhash) |
| `./handoff reset` | Reset the TCC permission grants |
| `./handoff clean` | Remove build artifacts |

## Architecture

Swift with no external dependencies. Sources live under `HackCMU/Sources/Handoff/`.

| Directory | Responsibility |
|---|---|
| `Capture/` | Global event tap on its own thread, a lock-free ring buffer, and the enrich queue |
| `Detect/` | Normalizes events into steps, finds the repeating period, and infers how each varying value advances |
| `AX/` | Accessibility: hit-test clicks to elements, re-find them by role path, read values from other windows, detect blocking dialogs |
| `Understand/` | Optional task naming through Claude, with a heuristic fallback |
| `Suggest/` | The non-activating panel and the two-stage suggestion and confirmation dialog |
| `Replay/` | Live element resolution, `AXPress` in preference to synthetic clicks, per-step guarding, and instant abort on user input |
| `Library/` | The on-disk, structure-only pattern store and prefix recognition for tasks seen before |

### Capture: keeping the event tap alive

macOS silently disables an event tap whose callback runs too long. The tap callback therefore does one thing: copy a plain struct into a single-producer, single-consumer ring buffer. The write path has no allocation, no syscall, and no lock. On overflow the newest event is dropped and counted, because a truncated history is still measurable while a history with a hole in it is not.

```swift
// Capture/RawEventRing.swift
/// PRODUCER - tap thread only. Wait-free: no allocation, no syscall, no lock.
@inline(__always)
func write(_ e: RawEvent) {
    let h = head.load(ordering: .relaxed)
    let t = tail.load(ordering: .acquiring)
    guard h &- t <= mask else {
        droppedCount.wrappingAdd(1, ordering: .relaxed)
        return
    }
    buf[Int(h & mask)] = e
    head.store(h &+ 1, ordering: .releasing)   // publishes the element
}
```

### Detection: finding the period

The detector looks for the smallest `p` such that each of the last `m` steps matches the step `p` positions before it. Checking small periods first is what stops a three-step loop from being reported as a six-step one. Four scans run in order of evidence strength, so a step whose value changes between passes is not mistaken for noise.

```swift
// Detect/LoopDetector.swift
private func scan() -> LoopCandidate? {
    //   1. exact strict        - identical passes, full identity
    //   2. exact structural    - identical STRUCTURE, values in titles differ
    //   3. tolerant strict     - identical passes bar a stray step
    //   4. tolerant structural - the messiest real case
    return exactScan(structural: false)
        ?? exactScan(structural: true)
        ?? tolerantScan(structural: false)
        ?? tolerantScan(structural: true)
}
```

A candidate fires after two complete passes. Single-step loops must clear a higher count, since one repeated key is easily idle noise.

### Deciding whether a loop is worth offering

Repetition alone is not a task. Typing is repetitive. `TaskValue` scores a candidate on whether it moves data, crosses apps, walks a list, or ends in an action, and vetoes the shapes that are never worth interrupting someone for.

```swift
// Detect/TaskValue.swift
if movesData {
    score += 2.0; reasons.append("moves data from one place to another")
}
if apps.count >= 2 || !switches.isEmpty {
    score += 1.5; reasons.append("crosses between apps")
}
if advances.contains(where: { if case .ordinal = $0.rule { return true }; return false }) {
    score += 1.5; reasons.append("walks through a list")
}
if commitStep != nil {
    score += 0.5; reasons.append("ends in an action")
}

if period.allSatisfy({ $0.kind == .text }) {
    vetoes.append("this is just typing")
}
```

These reasons are what the menu-bar panel shows when it explains why a suggestion did or did not appear.

### Replay: the guard before every step

The judgement about whether it is safe to proceed is pure and synchronous. The engine gathers the accessibility facts; `ReplayGuard` decides, which is what lets it be tested without a screen.

```swift
// Replay/ReplayGuard.swift
static func decide(_ s: State) -> Decision {
    if let modal = s.unexpectedModal {
        return .flagAndSkip("an unexpected dialog appeared (\(modal)) - left this one for you")
    }
    if let shape = s.expectedShape, let value = s.value, !value.isEmpty, !shape.matches(value) {
        return .flagAndSkip("the value here doesn't look right - skipped so it isn't copied in as-is")
    }
    if !s.targetPresent {
        return .flagAndSkip(s.isCommitting
            ? "the field this step fills is not where it was - skipped rather than risk writing to the wrong place"
            : "a step's target is no longer on screen - skipped this one")
    }
    return .proceed
}
```

### Library: what is written to disk

A stored step holds how to find the element again and how to press the key again, and nothing a person did. The container's label is kept only as a hash, which is enough to re-find the right list without revealing what it was called.

```swift
// Library/PatternLibrary.swift
struct StoredStep: Codable, Sendable {
    var key: UInt64
    var kind: UInt8
    var keyCode: UInt16
    var modifiers: UInt64
    var bundleID: String
    var role: String?
    var rolePath: String?
    var containerPath: String?
    var containerTitleHash: UInt64?   // hash only, never the title
    var ordinal: Int?
    var siblingCount: Int?
    // ...
}
```

## Task naming with Claude (optional)

Set `HANDOFF_ANTHROPIC_API_KEY` (or `ANTHROPIC_API_KEY`, or write the key to `~/.handoff/anthropic-key`). Nothing is read from your shell profile. Without a key the feature is off and Handoff never touches the network.

The request goes to `claude-fable-5-1` with `claude-opus-4-8` as the API-side fallback. By default the payload is structure only; `HANDOFF_UNDERSTAND_FULL=1` opts into sending labels and typed text. Run `./handoff understand` to print the exact payload before enabling it.

## Testing

```sh
./handoff test
```

This compiles the detection, guard, and library sources directly with `swiftc` and runs 191 checks headlessly, then renders the suggestion window to `build/selftest/` in light and dark mode. The capture layer needs a live event tap and the UI needs a GUI session, so those are exercised separately: `./handoff doctor` verifies the tap, and `./handoff axdump` and `./handoff replaytest` exercise resolution and replay against live Finder and Safari windows.

The suite is not a SwiftPM test target because `swift test` routes through the same broken `xcodebuild` shims as everything else on the development machine, and a test target cannot import an executable target.

## Limitations

- **macOS only.** Detection watches every app, but replay currently drives Finder and Safari. Chrome exposes only a stub accessibility tree for web content, and Terminal exposes none, so neither is a replay target yet (see `HackCMU/PROBE-FINDINGS.md`).
- **Consistency is required.** A task done differently each pass, or interleaved with unrelated activity, may not form a detectable period.
- **Spreadsheet cells are not enumerable** in the accessibility tree, so Handoff cannot count remaining rows in Google Sheets. Set the run count in the dialog and advance rows with a consistent keystroke.
- **Live end-to-end replay is proven for Finder list walks.** Cross-app and in-browser replay, and the guarded-replay dialog handling, are unit-tested but less exercised against live apps.

## License

MIT. See [LICENSE](LICENSE).
