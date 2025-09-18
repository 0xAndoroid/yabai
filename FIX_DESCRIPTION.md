# Arc Browser Fullscreen Fix for macOS Sequoia/Tahoe

## Problem Description

Arc browser on macOS Sequoia (15.x) and Tahoe (25.x) has a critical window management bug where windows become unmanaged after exiting fullscreen mode. The window incorrectly maintains fullscreen-related accessibility attributes even after returning to normal mode, preventing yabai from properly managing it.

### Symptoms
- Arc windows become unmanaged (not tiled) after exiting fullscreen
- Windows report `is-native-fullscreen: true` even when not in fullscreen
- Windows report `can-move: false` preventing them from being tiled
- The issue occurs consistently on macOS Sequoia/Tahoe but not on earlier versions

## Root Cause

Arc browser does not properly update its accessibility attributes when transitioning out of fullscreen mode. Specifically:

1. When entering fullscreen, Arc reports `is_fullscreen=1` while still on user space (before transitioning to fullscreen space)
2. When exiting fullscreen, Arc continues to report incorrect accessibility states
3. The standard `WINDOW_RESIZED` event does not fire reliably when Arc exits fullscreen
4. Arc windows lose their managed state and are not automatically re-added to tiling

## Solution Implementation

The fix consists of two main components:

### 1. Fullscreen Space Tracking

A lightweight state tracker monitors Arc windows as they enter fullscreen spaces:

```c
// Track Arc fullscreen transitions on macOS Sequoia/Tahoe
typedef struct {
    uint32_t window_id;
    uint64_t last_fullscreen_space;
} arc_window_state;
```

When Arc enters a fullscreen space (detected in `WINDOW_RESIZED`), we record the space ID. This allows us to identify when Arc has exited fullscreen.

### 2. Exit Detection and Correction

The fix is applied in the `WINDOW_MOVED` event handler, which reliably fires when Arc exits fullscreen:

```c
// Arc exited fullscreen: on user space, not fullscreen, not managed
if (is_user_space && !is_fullscreen && !is_managed) {
    // Check if we just came from a fullscreen space
    if (arc_state->last_fullscreen_space != 0) {
        // Ensure window has correct flags
        window_set_flag(window, WINDOW_MOVABLE);
        window_set_flag(window, WINDOW_RESIZABLE);
        window_clear_flag(window, WINDOW_FULLSCREEN);
        
        // Re-add to management
        struct view *view = space_manager_tile_window_on_space(&g_space_manager, window, current_space);
        window_manager_add_managed_window(&g_window_manager, window, view);
        
        // Clear the fullscreen space tracking
        arc_state->last_fullscreen_space = 0;
    }
}
```

## Technical Details

### Event Flow

1. **Entering Fullscreen:**
   - `WINDOW_RESIZED`: Arc reports `is_fullscreen=1` while on user space
   - `WINDOW_RESIZED`: Arc moves to fullscreen space (e.g., space 475)
   - We record the fullscreen space ID

2. **Exiting Fullscreen:**
   - `WINDOW_MOVED`: Arc returns to user space, reports `is_fullscreen=0`, `is_managed=0`
   - Fix detects the transition and re-adds the window to management
   - Window is properly tiled again

### Why WINDOW_MOVED?

The `WINDOW_MOVED` event is used because:
- `WINDOW_RESIZED` doesn't fire reliably when Arc exits fullscreen on Tahoe
- `SPACE_CHANGED` events are not triggered during Arc's fullscreen transitions
- `WINDOW_MOVED` consistently fires and provides accurate window state

### Memory Management

The fix tracks up to 10 Arc windows simultaneously using a static array. When full, it reuses the oldest slot. This is sufficient for typical usage patterns and has minimal memory overhead.

## Testing

The fix has been tested on:
- macOS Sequoia 15.x
- macOS Tahoe 25.x (internal Apple build)
- Arc browser (latest version as of 2025)

### Test Procedure

1. Open Arc browser window
2. Enter fullscreen (⌃⌘F or green button)
3. Exit fullscreen (Esc or ⌃⌘F)
4. Verify window is properly tiled

The fix works consistently across multiple fullscreen enter/exit cycles.

## Future Considerations

This workaround is specific to Arc browser and should be removed once Arc fixes their accessibility API implementation. The fix is designed to have no impact on other applications or Arc's behavior on older macOS versions.

## Related Issues

- Original issue: #2576
- Arc browser's incorrect accessibility attribute reporting
- macOS Sequoia/Tahoe specific behavior changes
