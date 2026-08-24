# cinnamon-closetab

Windows-11-style close buttons on Cinnamon's Alt+Tab switcher. Hover a preview,
click the `×` or press q and that window closes, without switching to it and without
dismissing the switcher.

![Alt+Tab with a close button on the hovered preview](docs/screenshot.png)

> **Linux Mint 22.3 (Cinnamon 6.6) only.** It hooks directly into Cinnamon's
> stock switcher internals, which move between releases. Other versions are
> untested and will likely break.

## Quickstart

```bash
git clone https://github.com/maxwellmollersten/cinnamon-closetab.git
cp -r cinnamon-closetab/windows-alt-tab@local ~/.local/share/cinnamon/extensions/
gsettings set org.cinnamon enabled-extensions "['windows-alt-tab@local']"
```

No restart or logout needed. Then hold **Alt**, press **Tab**, and hover a preview.

| | |
| --- | --- |
| Click a preview | switch to that window, as always |
| Click the `×` | close that window, switcher stays open |
| **Q** | close the selected window |
| Middle-click | close the hovered window |

Closing uses `Meta.Window.delete()` — the same polite request as Alt+F4 — so an
app with unsaved work still gets to show its "Save changes?" dialog. Nothing is
ever killed.

## Uninstall

```bash
gsettings set org.cinnamon enabled-extensions "[]"
rm -rf ~/.local/share/cinnamon/extensions/windows-alt-tab@local
```

Disabling restores every Cinnamon function the extension patched; nothing under
`/usr/share/cinnamon/` is ever touched.

## More

Behaviour toggles (always-show the `×`, disable middle-click, debug logging) are
constants at the top of
[`extension.js`](windows-alt-tab@local/extension.js). See
[NOTES.md](NOTES.md) for how it hooks into Cinnamon and why.
