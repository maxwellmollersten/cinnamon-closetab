# Implementation notes

Background for anyone reading or modifying `extension.js`.
For installation and usage see [README.md](README.md).

## How it works

Cinnamon's switcher lives in `/usr/share/cinnamon/js/ui/appSwitcher/`:

| file | role |
| --- | --- |
| `appSwitcher.js` | `AppSwitcher` — modal grab, window list, key handling, and `_removeDestroyedWindow()` bookkeeping |
| `classicSwitcher.js` | `ClassicSwitcher` → `AppList` → `AppIcon`; `SwitcherList.addItem()` wraps each `AppIcon` in an `St.Button` with style class `item-box` — that button *is* the clickable preview |
| `appSwitcher3D.js`, `coverflowSwitcher.js`, `timelineSwitcher.js` | the 3D styles |

Cinnamon exposes no hook for preview creation, and `windowManager.js`
destructures the `ClassicSwitcher` constructor at import time, so subclassing
would never be picked up. The extension therefore monkey-patches six prototype
methods, saving each original so `disable()` can put it back:

| patched | why |
| --- | --- |
| `AppIcon.prototype._init` | wraps the stock preview in an `St.Widget` + `Clutter.BinLayout` and overlays the close button in the top-right corner |
| `AppList.prototype._addIcon` | wires hover visibility and middle-click onto the `item-box` button that was just created |
| `ClassicSwitcher.prototype._updateList` | re-syncs close-button visibility after the list is rebuilt under a stationary pointer |
| `ClassicSwitcher.prototype._onDestroy` | drops the idle callback the above may have queued |
| `AppSwitcher.prototype._keyPressEvent` | **Q** closes the selected window |
| `AppSwitcher.prototype._init` | replaces the `'map'` handler (see below) |

Everything else is left to Cinnamon. In particular
`AppSwitcher._removeDestroyedWindow()` already reacts to the window manager's
`'destroy'` signal: it splices the window out, fixes up `_currentIndex`,
rebuilds the list, re-selects — and destroys the switcher when the last window
goes. Nothing here rebuilds the switcher by hand.

Two details worth calling out:

* **Why the click does not activate the window.** `St.Button` claims button 1
  on *press*, so the event never reaches the `item-box` `St.Button` underneath
  and `_onItemClicked()` is never called. Middle clicks are different — `St.Button`
  ignores button 2, so it would otherwise bubble all the way to the switcher's
  root actor, which dismisses the switcher; the `_addIcon` patch stops it.
* **Why `_init` is patched.** Stock Cinnamon dismisses the switcher (and
  activates the selected window) as soon as *any* window maps. That fires the
  instant an application pops up its close-confirmation dialog, stealing focus
  from the very dialog we caused. The replacement handler ignores a map from a
  process we asked to close a window in within the last few seconds.

## Tunables

Constants near the top of `extension.js`:

| constant | default | meaning |
| --- | --- | --- |
| `SHOW_ON_HOVER_ONLY` | `true` | set `false` to always show the `×` |
| `MIDDLE_CLICK_CLOSES` | `true` | middle-click a preview to close it |
| `CLOSE_COOLDOWN_MS` | `250` | ignore a second close request within this window, so an accidental double click cannot close whichever preview slid under the pointer |
| `CLOSE_CONFIRM_GRACE_MS` | `4000` | how long a map from a closing process counts as its confirmation dialog |
| `FADE_TIME` | `100` | fade duration for the `×` |
| `DEBUG` | `false` | verbose logging |

## Limitations

* Close buttons only appear for the **classic** switcher styles
  (`icons`, `thumbnails`, `preview` and their combinations — the Cinnamon
  default is `icons+thumbnails`). The `coverflow` and `timeline` styles draw
  3D-transformed window clones through a different class hierarchy and are not
  decorated; **Q**-to-close still works there.
* Disabling the extension while the switcher is on screen leaves that one
  switcher instance decorated. It is torn down as soon as you release Alt.
  Its close buttons keep working until then, but nothing of the extension
  outlives it: `disable()` cancels any idle still queued, and every signal and
  actor belongs to the switcher and dies with it.

## Cleanup

The only external resource this extension holds is a one-shot idle source,
queued by `_updateList` to re-sync close-button visibility after the preview
list is rebuilt under a stationary pointer.

Live source IDs are tracked in a module-level `pendingSyncIds` set. A switcher
normally cancels its own in `_onDestroy`, but when the extension is disabled
while a switcher is open that patch has already been reverted, so `disable()`
sweeps the set. `cancelSync()` checks membership before removing, so a source
`disable()` already swept is never removed twice — which would otherwise warn
about an unknown source ID if the extension were re-enabled and that switcher
then destroyed.

Everything else is owned by a switcher instance: the wrapper actor, the close
button, and the `notify::hover` / `button-press-event` / `button-release-event`
connections all live on actors that Cinnamon destroys when the switcher closes,
which drops their signal connections with them.

## Not implemented (deliberately)

Settings GUI, custom animations, configurable shortcuts, a full Windows 11
visual theme, and any replacement of Cinnamon's window enumeration.
