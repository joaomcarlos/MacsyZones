# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MacsyZones is a native macOS window management utility (FancyZones equivalent) that allows users to create custom layouts and snap windows into predefined zones. Built with Swift, SwiftUI, and AppKit, it leverages macOS Accessibility APIs extensively for window manipulation.

**License:** GPL-3.0
**Minimum macOS:** 12.0+
**Installation:** `brew install --cask macsyzones`

## Build & Development Commands

### Building
```bash
# Open in Xcode
open MacsyZones.xcodeproj

# Build from command line
xcodebuild -project MacsyZones.xcodeproj -scheme MacsyZones -configuration Debug build
xcodebuild -project MacsyZones.xcodeproj -scheme MacsyZones -configuration Release build
```

### Running
- Build and run in Xcode (Cmd+R)
- The app runs as a menu bar utility (no Dock icon due to `.prohibited` activation policy)
- Requires Accessibility permissions to function

### Testing
- No automated test suite currently exists
- Manual testing required for all use cases (per CONTRIBUTING.md)
- Test across multiple displays and macOS Spaces/Desktops

## High-Level Architecture

### Core Component Relationships

```
AppDelegate (App.swift)
    ↓ initializes
    ├─→ Macsy.swift (core window management runtime)
    ├─→ Layout system (SectionWindow, LayoutWindow overlays)
    ├─→ Global hotkeys (HotKey.swift)
    ├─→ Observers (AX window movement, space changes)
    └─→ Menu bar popover (TrayPopupView)

Macsy.swift
    ↓ coordinates
    ├─→ Window snapping logic (snap-to-zone, shake-to-snap)
    ├─→ PlacedWindows state (tracks window→zone mappings)
    ├─→ OriginalWindowProperties (for window restoration)
    └─→ Zone cycling & navigation

UserData (persistence layer)
    ↓ saves to ~/Library/Application Support/MacsyZones/
    ├─→ AppSettings.json
    ├─→ SpaceLayoutPreferences.json
    └─→ UserLayout files
```

### Global State Machine

Core state flags in `Macsy.swift` control app behavior:
- `isFitting`: Zone overlay is visible
- `isEditing`: User is editing layout
- `isQuickSnapping`: QuickSnapper panel active
- `isSnapResizing`: Window is being snap-resized
- `isMovingAWindow`: Window drag in progress
- `isZoneNavigating`: Keyboard zone navigation active

**Critical:** Always check these flags before introducing new behaviors that could conflict.

### Data Flow for Window Snapping

```
1. User Input (modifier key hold / snap key / shake gesture)
   ↓
2. Show overlay (userLayouts.currentLayout.layoutWindow.show())
   ↓
3. Mouse movement → hover detection (getHoveredSectionWindow)
   ↓
4. Mouse up → snapWindowToZone()
   ↓
5. resizeAndMoveWindow() (with retry logic for Electron/webview drift)
   ↓
6. Update PlacedWindows state
   ↓
7. Save to UserData
```

### Multi-Screen & Multi-Space Architecture

- `SpaceLayoutPreferences` maps (screen, space) pairs → layout names
- Layout switching happens automatically on space changes when `selectPerDesktopLayout` is enabled
- Focused screen detection uses mouse cursor position
- Zone coordinates stored as percentages (screen-agnostic) and converted to absolute on draw/snap

### Accessibility API Integration

- `AXObserver` monitors window movement (`kAXWindowMovedNotification`)
- `AXUIElement` used for window position/size manipulation
- Bridging header exposes private APIs:
  - `_AXUIElementGetWindow` - Get CGWindowID from AXUIElement
  - `CGSCopyManagedDisplaySpaces` - Access workspace/space info
  - `CGSSetConnectionProperty` - Window server connections

### Feature Modules (`features/` directory)

Self-contained features with scoped state:

1. **WindowZoneAssociation.swift**
   - Auto-associates windows to zones based on geometry (6px edge tolerance)
   - Runs on app load, layout switch, QuickSnapper activation

2. **ZoneNavigation.swift**
   - Keyboard-driven window movement between adjacent zones
   - Direction-based logic (up/down/left/right)
   - Uses `isZoneNavigating` flag to prevent conflicts

## Key Implementation Patterns

### Adding a New Setting

1. Add `@Published var newSetting: Type = default` in `AppSettings` (Settings.swift)
2. Add optional `newSetting: Type?` in `AppSettingsData` struct
3. On load: `self.newSetting = settings.newSetting ?? newSetting`
4. On save: Include in `AppSettingsData` constructor
5. Reference via `appSettings.newSetting` (never store separately)

### Adding a New Global Hotkey

1. Create `GlobalHotkey` instance in `AppDelegate`
2. Register after `GlobalHotkey.setup()` in `applicationDidFinishLaunching`
3. Store shortcut string in `AppSettings` for persistence
4. Action closure must return `noErr`
5. Early-return if blocking state active (`isEditing`, etc.)

### Working with Layouts and Zones

- **Zone identity:** Integer `sectionConfig.number` (sequential 1, 2, 3...)
- **Normalization:** Call `UserLayout.reArrange()` after adding/removing zones
- **Coordinates:** Always use percentage properties; convert to absolute only for drawing/snapping via `sectionConfig.getAXRect(on:)`
- **Window placement:** Call `OriginalWindowProperties.update(windowID:)` before first snap to enable restoration

### Persistence Pattern

```swift
// UserData subclass pattern
class MySettings: UserData {
    @Published var setting: Type = default

    override func load() {
        super.load()
        if let decoded = try? JSONDecoder().decode(MySettingsData.self, from: data) {
            self.setting = decoded.setting ?? setting
        }
    }

    override func save() {
        data = (try? JSONEncoder().encode(MySettingsData(setting: setting))) ?? Data()
        super.save()
    }
}
```

**Never** write files directly; always assign to `data` then call `save()`.

### Window Manipulation Best Practices

- **Reuse helpers:** `resizeAndMoveWindow()`, `moveWindowToMatch()`, `snapWindowToZone()`
- **Retry logic:** `resizeAndMoveWindow` includes corrective micro-retries for Electron/webview drift
- **Multi-display bug:** Size workaround already implemented in resize helper
- **Window validation:** Check `kAXStandardWindowSubrole` and `kAXWindowRole` before acting
- **Permissions:** Degrade gracefully if `hasAccessibilityPermission` is false

## File Organization

```
MacsyZones/
├── App.swift              # Entry point, AppDelegate, initialization
├── Macsy.swift           # Core window management runtime & state
├── Layout.swift          # Zone overlay UI (SectionWindow, LayoutWindow)
├── UserData.swift        # Persistence base class
├── Settings.swift        # AppSettings model
├── States.swift          # PlacedWindows, OriginalWindowProperties
├── HotKey.swift          # Carbon-based global hotkey system
├── Popover.swift         # Menu bar popover UI
├── QuickSnapper.swift    # Quick snap panel
├── Preferences.swift     # SpaceLayoutPreferences (per-desktop layouts)
├── Screen.swift          # Screen/focus utilities
├── Updater.swift         # Auto-update from GitHub
├── features/             # Self-contained feature modules
│   ├── WindowZoneAssociation.swift
│   └── ZoneNavigation.swift
└── Utils.swift           # Debug logging (debugLog)
```

## Development Guidelines

### State Management
- Respect blocking flags before introducing new behaviors
- Use feature-scoped state flags (see `ZoneNavigation.swift` pattern)
- Minimize new global flags; document necessity with comments
- Always flip `isFitting` consistently when showing/hiding overlays

### Threading
- AX operations can block; offload scanning to background threads
- Dispatch UI-critical work to main: `Task { @MainActor in ... }`
- Don't block main thread with loops over AX windows

### Debugging
- Use `debugLog(...)` instead of `print()` (defined in Utils.swift)
- Format: short context prefix + dynamic values

### Safety
- Check `hasAccessibilityPermission` before invasive AX operations
- Ensure standard window validation before manipulation
- Test across multiple displays and Spaces

### Code Style
- Follow existing Swift conventions in codebase
- No formal linter configured
- Keep feature modules self-contained in `features/` directory
- Reuse existing helpers instead of duplicating logic

## Contributing Feature Modules

Preferred structure for new features:

```swift
// features/NewFeature.swift

// Feature-scoped state flag
var isNewFeatureActive = false

func performNewFeature() {
    guard !isEditing, !isSnapResizing else { return }

    isNewFeatureActive = true
    defer { isNewFeatureActive = false }

    // Feature logic...
    // Reuse existing helpers from Macsy.swift
}
```

Integrate via:
- Global hotkey registration in `App.swift`
- Event handler in `Macsy.swift`
- Setting in `AppSettings` if user-configurable

## Known Architectural Details

### PlacedWindows Schema
Tracks window placements as:
```
windowID → (screenID, spaceID, layoutName, sectionNumber)
```

### OriginalWindowProperties
Stores pre-snap window dimensions for fallback/restoration:
```
windowID → (width, height, x, y)
```

### Snap Triggers
1. Modifier key hold (Control/Command/Option based on settings)
2. Snap key press
3. Right-click (if enabled)
4. Shake-to-snap gesture (velocity & acceleration thresholds)

### Window Cycling
`cycleWindowsInZone(forward:)` cycles through windows in same zone on focused screen.

### Zone Navigation
`moveWindowToAdjacentZone(direction:)` moves focused window to adjacent zone (up/down/left/right) using edge proximity calculations.

## Resources

- **Website:** https://macsyzones.com
- **Discord:** https://discord.gg/C4axTA6rpn
- **Issues:** https://github.com/rohanrhu/MacsyZones/issues
- **Copilot Instructions:** `.github/copilot-instructions.md` (detailed patterns)
