# Test Plan for Tab/Pane Closing Freeze Fix

## Summary of Changes

Fixed WezTerm freezing issue when closing tabs/panes after long sessions by:

1. **Moved tab/pane removal operations to background threads** 
   - Modified `close_current_pane()`, `close_current_tab()`, and `close_specific_tab()` in `wezterm-gui/src/termwindow/mod.rs`
   - Changed from synchronous `mux.remove_pane()` and `mux.remove_tab()` calls to asynchronous `spawn_into_new_thread()`

2. **Updated confirmation dialogs to use background threads**
   - Modified `confirm_close_pane()`, `confirm_close_tab()`, and `confirm_close_window()` in `wezterm-gui/src/overlay/confirm_close_pane.rs`
   - Changed from `spawn_into_main_thread()` to `spawn_into_new_thread()` for removal operations

3. **Optimized scrollback buffer clearing**
   - Modified `erase_scrollback()` in `term/src/screen.rs`
   - Replaced O(n) individual `pop_front()` operations with efficient `drain()` for bulk removal

4. **Made Windows process termination asynchronous**
   - Modified `do_kill()` and `ChildKiller::kill()` in `pty/src/win/mod.rs`
   - Spawned `TerminateProcess()` calls in separate threads to prevent blocking

## Testing Scenarios

### Scenario 1: Large Scrollback Buffer
1. Open WezTerm
2. Run a command that generates lots of output: `for i in {1..100000}; do echo "Line $i"; done`
3. Let it accumulate for a while
4. Close the tab with Ctrl+Shift+W
5. **Expected**: Tab closes immediately without freezing

### Scenario 2: Long-Running Process
1. Open WezTerm
2. Start a long-running process: `npm run watch` or similar
3. Let it run for several hours/days
4. Close the tab with Ctrl+D or Ctrl+Shift+W
5. **Expected**: Tab closes without freezing the entire window

### Scenario 3: Multiple Panes with Complex Layout
1. Open WezTerm
2. Create multiple panes (split horizontally and vertically)
3. Run different processes in each pane
4. Close individual panes and tabs
5. **Expected**: Each close operation completes quickly without blocking

### Scenario 4: Windows-Specific Process Tree
1. On Windows, start a Node.js build watcher
2. Let it spawn multiple child processes
3. Close the tab containing the process tree
4. **Expected**: Tab closes without waiting for stubborn processes

## Verification Steps

1. Build WezTerm with the changes
2. Run each test scenario
3. Monitor for GUI responsiveness during close operations
4. Check logs for any errors or warnings
5. Verify processes are properly terminated in Task Manager

## Known Limitations

- The fix makes close operations asynchronous, so there might be a slight delay before resources are fully released
- Process termination on Windows is now "fire and forget" - the GUI won't wait for confirmation
- Users should ensure they save any important work before closing tabs/panes