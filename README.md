# ARCHIVED - moved to my dotfiles(https://github.com/DhanvanthR23/dotfiles)

# qs-rice

A Quickshell (noctalia-qs) shell for Niri. Floating pill bar, with every popup growing out of a pill.

```
[launcher][workspaces]   clock   [cpu+ram][control]
```

- **Clock pill** (center): hover opens a week-strip calendar with the media row; notifications, track changes and OSDs appear as toasts on the pill; a dot shows pending updates.
- **Control pill** (right): wifi icon inside a battery progress ring. Opens the control center: Wi-Fi, Bluetooth, Power profile, Night light, Updates and Keep awake tiles, volume and brightness sliders (monitor picker on the brightness icon), and the notification list.
- **Launcher**: apps plus a calculator, usage-ranked, fuzzy matching.
- **Overlays that grow out of the clock pill**: clipboard history, theme and wallpaper picker, session menu.
- **Idle handling** is done by the shell itself (screen off, lock, lock before sleep), no hypridle.

## Requirements

Shell and compositor: `quickshell` (noctalia-qs fork), `niri`.

Used by services: `foot`, `brightnessctl`, `ddcutil` (external monitors), `wlsunset`, `cliphist`, `wl-clipboard`, `playerctl`, `awww`, `imagemagick`, `ripgrep`, `fish`, `pacman-contrib` (`checkupdates`), `paru` or `yay`, `wlctl`, `bluetui`, `dbus` (`dbus-monitor`), `systemd` (`systemd-inhibit`).

System services: PipeWire, NetworkManager, BlueZ, UPower, power-profiles-daemon.

Fonts: JetBrainsMono Nerd Font (icons), Google Sans Flex (text).

The shell is the notification server, so no other notification daemon (mako, dunst) may run.

Niri starts it with `spawn-at-startup "qs"`. For lower memory use `QT_QUICK_BACKEND=software` in niri's `environment` block.

## IPC

All handlers live in `services/Ipc.qml`. List them with `qs ipc show`.

| Target | Functions | Bound to |
|---|---|---|
| `launcher` | `toggle` | Mod+D |
| `control` | `toggle` | Mod+A |
| `clipboard` | `toggle`, `show`, `hide` | Mod+V |
| `picker` | `toggle`, `show`, `hide` | Mod+Ctrl+T |
| `session` | `toggle`, `show`, `hide` | Mod+Escape, Ctrl+Alt+Delete |
| `idle` | `toggle` | Mod+H |
| `notifs` | `clear` | Mod+Shift+N |
| `updates` | `run`, `check` | Mod+Shift+U |
| `osd` | `volume`, `brightness`, `caps`, `num` | media, brightness and lock keys |
| `theme` | `reload` | called by `set-theme.fish` |

`launcher` and `control` act on the focused output (`Picker.focusedOutput()`), so they work on multiple monitors.

Add a new target: add an `IpcHandler` in `Ipc.qml`. Functions need the `: void` return type. `qmlformat` and `qmllint` cannot parse that annotation, which is why all handlers sit in this one file.

## Niri binds

```kdl
Mod+D { spawn "qs" "ipc" "call" "launcher" "toggle"; }
Mod+A { spawn "qs" "ipc" "call" "control" "toggle"; }
Mod+V { spawn "qs" "ipc" "call" "clipboard" "toggle"; }
Mod+Ctrl+T { spawn "qs" "ipc" "call" "picker" "toggle"; }
Mod+Escape { spawn "qs" "ipc" "call" "session" "toggle"; }
Ctrl+Alt+Delete { spawn "qs" "ipc" "call" "session" "toggle"; }
Mod+H { spawn "qs" "ipc" "call" "idle" "toggle"; }
Mod+Shift+N { spawn "qs" "ipc" "call" "notifs" "clear"; }
Mod+Shift+U { spawn "qs" "ipc" "call" "updates" "run"; }

XF86AudioRaiseVolume allow-when-locked=true { spawn-sh "wpctl set-volume -l 1 @DEFAULT_AUDIO_SINK@ 5%+; qs ipc call osd volume"; }
XF86AudioLowerVolume allow-when-locked=true { spawn-sh "wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-; qs ipc call osd volume"; }
XF86AudioMute allow-when-locked=true { spawn-sh "wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle; qs ipc call osd volume"; }
XF86MonBrightnessUp allow-when-locked=true { spawn-sh "brightnessctl -q -d intel_backlight set 5%+; qs ipc call osd brightness"; }
XF86MonBrightnessDown allow-when-locked=true { spawn-sh "brightnessctl -q -d intel_backlight set 5%-; qs ipc call osd brightness"; }
Caps_Lock allow-when-locked=true { spawn-sh "qs ipc call osd caps"; }
Num_Lock allow-when-locked=true { spawn-sh "qs ipc call osd num"; }
```

The keys change the value themselves and only call IPC to show the toast, so volume and brightness keep working if the shell crashes.

## Layout

```
shell.qml            entry point; references Ipc so its handlers register at startup
config/              Theme (colors, sizes, fonts), Settings (idle times, location, terminal)
services/            singletons: Time Cpu Ram Battery Net Bt Audio Brightness Power NightLight
                     Notifs Updates Idle Media Launcher Clip Themes Picker Session Osd Ipc
components/          Pill, Tile, SliderRow, ArrowButton, StatItem, NotifAvatar, UpdateDot
modules/bar/         Bar, ClockPill, LauncherPill, Workspaces, SystemPill, ControlPill
modules/calendar/    CalendarPopup (week strip + media row)
modules/control/     ControlPopup, ControlBackdrop, BrightnessRow, NotifList
modules/launcher/    LauncherPopup, LauncherOverlay
modules/clipboard/   ClipPopup, ClipOverlay
modules/picker/      PickerPopup, PickerOverlay
modules/session/     SessionPopup, SessionOverlay
scripts/             update-count.sh
```

Rules for editing:
- Use relative imports everywhere (`import "../../config"`). `import qs.…` does not work here.
- A new singleton needs a `singleton Name 1.0 Name.qml` line in `services/qmldir` (or `config/qmldir`).
- Singletons load lazily. One that must exist at startup, such as one owning an `IpcHandler`, has to be referenced somewhere.
- Popup overlays follow one pattern: a `Scope` holding a `progress` value plus a `LazyLoader`, and a full-screen `PanelWindow` containing the card. Copy `session/` to start a new one.

## Theming

Themes are `KEY=hex` files in `~/.config/colors/themes/`. Applying one copies it over `~/.config/colors/colors.conf`, runs `generate-colors.sh` (niri, foot, fuzzel, zathura, hyprlock, Qt and GTK colors), then calls `qs ipc call theme reload` so the shell recolors live. `set-theme.fish` also picks a random wallpaper from the theme's folder in `~/Pictures/wallpapers`; `wall-dir.fish` holds the theme-to-folder mapping.

Helper scripts live in `~/.config/niri/scripts/`: `set-theme.fish`, `wall-list.fish`, `wall-thumbs.fish`, `wall-dir.fish`, `generate-colors.sh`, `lock.sh`.

## Files outside the repo

| Path | Content |
|---|---|
| `~/.local/state/qs-test/theme` | current theme slug |
| `~/.local/state/qs-test/wallpaper` | current wallpaper path (restored on login) |
| `~/.local/state/qs-test/launches.json` | launcher usage counts |
| `~/.cache/qs-test/update-count` | cached update count (25 min TTL) |
| `~/.cache/qs-test/thumbs/` | wallpaper picker thumbnails |

## Settings

`config/Settings.qml`: night light temperature, latitude and longitude (for the auto schedule), idle screen-off and lock timeouts (seconds), terminal.

## Known quirks (WIP)

- The brightness OSD shows the built-in panel only, as a linear percentage; the control center slider uses an exponential curve.
- Caps and Num Lock OSDs read `/sys/class/leds/input3::*` after a 200 ms delay; on another machine the LED name may differ.
- There is no system tray, no Do Not Disturb, and only one toast is shown at a time (the control center list keeps all notifications).
- Wi-Fi and Bluetooth management open `wlctl` and `bluetui` in foot on right-click; Quickshell has no NetworkManager password agent.
- Idle memory is about 90 MB Pss with the software renderer; after opening every popup it settles around 85 MB anon and stays flat.
