---
name: quickshell-omarchy-plugin
description: Use when creating, installing or debugging an Omarchy/Quickshell bar plugin - manifest.json, Panel.qml widget, install.sh, uninstall, systemd units, theme colors, packaging a release tarball.
---

# Omarchy/Quickshell bar plugin

Verified in pricewatch, 80procent, lidswitch, news-colector, local_news.

## Layout (plugin root)

`manifest.json`, `Panel.qml`, `install.sh` (executable), `bin/` (CLI the QML
shells out to), optional `src/`, `systemd/`, `test/`, `dist.sh`, `README.md`,
`LICENSE`, `sources.json` (user-editable config beside the widget). lidswitch
variant: widget sources live in `omarchy-plugin/<id>/`; install copies that subdir only.

## manifest.json

Fields in every project: `schemaVersion: 1`, `id`, `name`, `version`, `author`,
`license`, `description`, `kinds: ["bar-widget"]`,
`entryPoints: { "barWidget": "Panel.qml" }`, plus:

```json
"barWidget": {
  "displayName": "Pricewatch",
  "description": "long UX prose: clicks, keys, data location",
  "category": "Utilities",
  "aliases": ["pricewatch", "prijs"],
  "allowMultiple": false,
  "defaultSection": "right",
  "defaults": { "interval": 60 },
  "schema": [{ "key": "interval", "type": "number", "label": "...", "description": "..." }]
}
```

`defaultSection`: `"left"` (80procent, lidswitch), `"center"` (news), `"right"`
(pricewatch). Categories: `Systeem`, `Internet`, `Utilities`. `defaults`/`schema`
may be `{}`/`[]`; only `"number"` schema type occurs.

## Panel.qml

Imports (all five identical): `QtQuick`, `QtQuick.Controls`, `Quickshell`,
`Quickshell.Io`, `qs.Ui`, `qs.Commons`.

```qml
Panel {
  id: root
  moduleName: "<manifest id>"
  ipcTarget: "<manifest id>"
  implicitWidth: button.implicitWidth
  implicitHeight: bar ? bar.barSize : 26
  readonly property string scriptPath:
    Quickshell.env("HOME") + "/.config/omarchy/plugins/<id>/bin/<script>"
  readonly property int bodyFont: Style.font.body
  readonly property int captionFont: Style.font.caption
  readonly property int subtitleFont: Style.font.subtitle
}
```

- Manifest settings reach QML via `setting("key", default)`, clamped
  (`Math.max(15, Number(setting("interval", 60)))`).
- Never read state files from QML: run `bin/` scripts with `Process { stdout:
  StdioCollector { waitForEnd: true; onStreamFinished: ... } }` and
  `JSON.parse` (`status --json`, `sources`, `set ...`). State lives in
  `~/.local/share/<id>/`, `~/.local/state/<id>/`, `~/.config/<id>/`; tests
  redirect with env vars (`PRICEWATCH_HOME`, `LIDSWITCH_HOME`).
- Bar button: `WidgetButton { id: button; anchors.fill: parent; bar: root.bar;
  text: root.barText; fontFamily: root.bar ? root.bar.fontFamily : Style.font.family;
  fontSize: Style.font.body; foreground: root.barForeground; active: ...;
  activeColor: Color.accent; tooltipText: ...; onPressed: function(b) {...} }`.
  Icon-only variant: `BarIconButton { ... iconComponent: Component { Item {...} } }`
  (custom glyph; text glyphs unreadable at bar height).
- Panel: `KeyboardPanel { anchorItem: button; owner: root; bar: root.bar;
  open: root.opened; focusTarget: keys; contentWidth: Style.space(500);
  contentHeight: Style.space(430); PanelKeyCatcher { ... } }`.
- `Panel.opened` is read-only: `open()`/`close()`/`toggle()`; assigning it logs
  `TypeError: Cannot assign to read-only property` (80procent still does - don't copy).
- QML ids must not share a name with a function; never name a property `data`
  (shadows `Item.data`). Clicks: left = action, right = `toggle()`, middle = refresh.

## Theme

`Color.accent|muted|urgent`, `Color.menu.background|selectedBackground`;
`Style.space(n)`, `Style.cornerRadius`, `Style.spacing.controlGap|controlPaddingY`,
`Style.font.family|body|caption|subtitle`; bar text via host `root.barForeground`
and `root.bar.fontFamily`.

## install.sh

Skeleton: `set -euo pipefail`; `SRC_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"`;
`plugin_id() { jq -r '.id // empty' "$MANIFEST"; }`; require `omarchy` and `jq`. Steps:

1. `omarchy plugin validate "$SRC_DIR"`.
2. Stage: `stage="$HOME/.config/omarchy/plugins/.$id.install.$$"`; `cp -a
   manifest.json Panel.qml bin [src] [systemd] "$stage/"`; strip `__pycache__`;
   `python3 -m py_compile` gate; `chmod +x "$stage"/bin/*`.
3. Backup existing to `~/.config/omarchy/plugin-backups/$id-$(date +%Y%m%d-%H%M%S)`
   (news-colector: preserve user-edited `sources.json` from that backup).
4. `rm -rf "$PLUGINS_DIR/$id"; mv "$stage" "$PLUGINS_DIR/$id"` — atomic replace;
   lidswitch copies to `$WIDGET_DIR.new` then `mv`, because Quickshell hot-reloads
   and a truncating `cp` exposes half a file.
5. `omarchy-shell shell rescanPlugins >/dev/null || true`.
6. If not enabled (poll `omarchy plugin list --json | jq -e --arg id "$id"
   'any(.[]; .id == $id)'` up to 40x `sleep 0.05`):
   `omarchy plugin enable "$id" "$(jq -r '.barWidget.defaultSection // "left"' "$MANIFEST")"`.

Subcommands (`case "${1:-install}"`): `install`, `version` (bundled vs installed
via `jq .version`), `uninstall`, `help`; extras `timer`/`timer-off`, `cli`
(symlink to `~/.local/bin`), `update` (only if newer bundled, `sort -V`),
`helper` (prints a `sudo install -m 4755` line for the human to run).
local_news variant: `rsync -a --exclude sources.json --exclude .git` instead of
stage+mv; backup as tarball; enable without section or polling.

## Uninstall

`omarchy plugin remove "$id"` (official tool, makes its own backup), guarded by
an installed check via `omarchy plugin list --json`. lidswitch has separate
`uninstall.sh`: rm CLI symlinks + widget dir, `systemctl --user disable --now`,
rm unit, `daemon-reload`; leaves `~/.config/<id>` and `~/.local/state/<id>`,
listing them for manual removal.

## systemd

Units live in repo `systemd/`, copied to `~/.config/systemd/user/`, then
`systemctl --user daemon-reload` + `enable --now`. Patterns:
- oneshot: `Type=oneshot`, `ExecStart=%h/.config/omarchy/plugins/<id>/bin/...`,
  `TimeoutStartSec=180`, `WantedBy=default.target`.
- timer: `OnBootSec=4min`, `OnUnitActiveSec=30min`, `AccuracySec=1min`,
  `Persistent=false` (no catch-up after sleep), `WantedBy=timers.target`.
- daemon: `Type=simple`, `Restart=always`, `RestartSec=2`,
  `After=graphical-session.target`, `NoNewPrivileges=yes`.
`%h` works in `ExecStart` but not `StandardOutput=`; logs via
`journalctl --user -u <unit> -n 30`. Privileged one-offs (logind drop-in under
`/etc/systemd/logind.conf.d/`) are printed as a `sudo install -Dm644` line for the
human, never run silently.

## Release packaging

pricewatch `dist.sh`: version via python3 from manifest; stage in `mktemp -d`;
copy `manifest.json Panel.qml install.sh README.md LICENSE bin src systemd`
(exists-guarded); delete `__pycache__`/`*.pyc`; `chmod +x`; `tar -czf
dist/<id>-<version>.tar.gz`; print `sha256sum` + unpack hint
`tar -xzf ... -C ~/.config/omarchy/plugins-staging`.
Variant (80procent, news-colector): `dist/<id>/` mirror of shippable files zipped
as `dist/<id>-omarchy-plugin-<version>.zip`.

## Testing

`test/run-all.sh`: `python3 -m unittest discover -s test`, `bash test/cli.test.sh`,
`qmllint -I /usr/share/omarchy/shell Panel.qml` (syntax only; behaviour only in
the real bar), `omarchy plugin validate .`. CLI tests fake state with env vars
(`LIDSWITCH_ACPI_DIR`, `PRICEWATCH_HOME=/tmp/pw`); never smoke-test against the real state dir.
