# Deskflow KVM

A software KVM for **one monitor, one keyboard/mouse, two computers**: a Linux machine whose built-in panel has an HDMI input, and a Mac plugged into that input.

One hotkey moves the keyboard, mouse and clipboard to the other computer (via [Deskflow](https://github.com/deskflow/deskflow)), and the monitor switches input to match (via DDC/CI with `ddcutil`).

Built and tested on an HP Envy 34" curved All-in-One (Linux Mint 22.3, Cinnamon on X11) with a Mac mini (Apple silicon, macOS 27) on its HDMI input. It should work with any monitor that supports DDC/CI input switching.

| Hotkey (Linux keyboard) | Action |
|---|---|
| **Ctrl+Shift+F12** | Toggle: keyboard, mouse and clipboard go to the other computer; the screen follows |
| **Ctrl+Shift+F10** | Always return to Linux (safety key) |
| **Super+F11** | Screen-only toggle without Deskflow (Cinnamon shortcut, fallback) |

## How it works

```
 Linux (Deskflow SERVER, real keyboard/mouse)          Mac (Deskflow CLIENT)
 ───────────────────────────────────────────           ──────────────────────
 Ctrl+Shift+F12 ──► Deskflow switches screens ───────► Mac "enter" hook
                                                         └─ ~/bin/deskflow-screen hdmi
 ~/.local/bin/toggle-input-remote  ◄── ssh (restricted key) ──┘
   └─ toggle-input hdmi
        ├─ ddcutil setvcp 60 <HDMI>          (monitor → Mac)
        └─ ssh mac: pmset displaysleepnow;    (retrain HDMI link,
                    caffeinate -u              else panel stays black)

 Ctrl+Shift+F12 again ─────────────────────────────────► Mac "exit" hook
                                                         └─ deskflow-screen native
                                                              └─ ssh → toggle-input native
                                                                   (monitor → Linux, ~8 s)
```

The screen-switch hooks run on the **Mac**, because the Linux Deskflow runs as a Flatpak and its sandbox can't run host commands like `ddcutil`. The Mac's SSH key is locked to a forced command that only accepts `native` or `hdmi`.

## Files

| File | Install to | Purpose |
|---|---|---|
| `linux/toggle-input` | `~/.local/bin/` | Switch panel input; to HDMI also wakes the Mac's display |
| `linux/toggle-input-remote` | `~/.local/bin/` | Forced command for the Mac's restricted SSH key |
| `linux/server-custom.conf` | `~/.var/app/org.deskflow.deskflow/config/Deskflow/` | Deskflow hotkeys and links (no edge links) |
| `linux/deskflow.desktop` | `~/.config/autostart/` | Start the Deskflow server at login |
| `linux/authorized_keys.example` | line for `~/.ssh/authorized_keys` | Restricted key entry for the Mac |
| `mac/deskflow-screen` | `~/bin/` | Hook: ask Linux to switch the panel |
| `mac/Deskflow.conf.example` | keys for `~/Library/Deskflow/Deskflow.conf` | Client settings and enter/exit hooks |
| `mac/local.deskflow.autostart.plist` | `~/Library/LaunchAgents/` | Start Deskflow at login |

### Reference setup (the original install)

| Setting | Value |
|---|---|
| Linux hostname / Deskflow screen name | `jello-HP-ENVY-Curved-All-in-One-PC-34-b0xx` (`.local` via avahi) |
| Linux IP / user | `192.168.1.5` (Wi-Fi) / `jello` |
| Mac mDNS name | `Stacys-Mac-Mini.local` |
| Mac Deskflow screen name | `Mac-mini.localdomain` |
| Mac IP / user | `192.168.1.200` (DHCP reservation) / `jello` |
| Panel DDC | bus `4`, native `0x03`, HDMI-1 `0x11` |
| Deskflow grid | Linux slot 7, Mac slot 9 (gap between them, so no edge links) |
| Versions | Deskflow 1.27.0, Linux Mint 22.3, macOS 27.2 |

Placeholders to replace: `mac-mini.local`, `linux-host.local`, `linux-user`, `LINUX-USER`, `MAC-USER`, `LINUX-SCREEN-NAME`, `MAC-SCREEN-NAME`, and the DDC values at the top of `toggle-input`.

## Setup

### 1. Linux: monitor input switching
```bash
sudo apt install ddcutil
ddcutil detect                         # note the I2C bus (/dev/i2c-N)
ddcutil --bus N capabilities           # Feature 60 lists the input codes
ddcutil --bus N getvcp 60              # the current (native) input
```
On the HP AIO: bus 4, native = `0x03` (reported as "DVI-1"), HDMI-1 = `0x11`. Put your values in the CONFIG block of `linux/toggle-input`, then:
```bash
install -m755 linux/toggle-input linux/toggle-input-remote ~/.local/bin/
```
Test with `toggle-input hdmi`, then `toggle-input native`. Press keys **once**: see the troubleshooting notes on the 8-second delay.

Optional fallback shortcut (Cinnamon): Keyboard → Shortcuts → Custom → `~/.local/bin/toggle-input` on Super+F11.

### 2. SSH both ways
- **Linux → Mac** (normal key; used to wake the Mac's display): enable Remote Login on the Mac, then run `ssh-copy-id <mac-user>@<mac>.local`. On the Mac, also run `sudo pmset -a displaysleep 0`.
- **Mac → Linux** (restricted key): on the Mac, run `ssh-keygen -t ed25519 -N "" -C mac-deskflow-toggle -f ~/.ssh/id_deskflow_toggle`. Add its `.pub` to Linux `~/.ssh/authorized_keys` in the format shown in `linux/authorized_keys.example`. Check it from the Mac with `ssh -i ~/.ssh/id_deskflow_toggle <linux-user>@<linux>.local whoami`, which should print `denied`.

Use mDNS `.local` names rather than IPs, because macOS private Wi-Fi addresses can change the Mac's IP.

### 3. Deskflow
- **Linux:** run `flatpak install flathub org.deskflow.deskflow`. The GitHub `.deb`s target newer Ubuntu than Mint 22.
- **Mac:** download the `macos-arm64.dmg` from [releases](https://github.com/deskflow/deskflow/releases) and check it against `sums.txt`. Then grant **System Settings → Privacy & Security → Accessibility → Deskflow** and **restart Deskflow**. `open "x-apple.systempreferences:com.apple.preference.security?Privacy_Accessibility"` jumps straight to that pane.

**Server (Linux) in the GUI:** choose Server, then Configure Server. Put the Mac on the grid **with an empty square between it and Linux**, so there are no edge links. Name it exactly what the Mac's Deskflow reports, usually the output of `hostname` such as `Mac-mini.localdomain`. Then quit Deskflow and:
```bash
D=~/.var/app/org.deskflow.deskflow/config/Deskflow
cp linux/server-custom.conf $D/            # then fill in the screen names
# in $D/Deskflow.conf, under [server]:
#   externalConfig=true
#   externalConfigFile=/home/<you>/.var/app/org.deskflow.deskflow/config/Deskflow/server-custom.conf
install -m644 linux/deskflow.desktop ~/.config/autostart/
```

**Client (Mac):** choose Client, with the server set to `<linux>.local`. With Deskflow quit, set the keys from `mac/Deskflow.conf.example`. Then:
```bash
mkdir -p ~/bin && install -m755 mac/deskflow-screen ~/bin/      # fill in LINUX / LINUX_USER
cp mac/local.deskflow.autostart.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.deskflow.autostart.plist
```

**First connection:** accept the TLS fingerprint on both sides, after checking it with `openssl x509 -in <tls>/deskflow.pem -noout -fingerprint -sha256`. Each machine's own fingerprint is *supposed* to differ. You can also pre-trust each side by writing `v2:sha256:<lowercase hex, no colons>` into `tls/trusted-clients` (Linux) and `tls/trusted-servers` (Mac).

Keep a spare keyboard/mouse on the Mac. If you ever get stuck there, quitting Deskflow on the Mac returns control to Linux.

## Troubleshooting notes (learned the hard way)

**Monitor / HDMI**
- **HDMI → native takes about 8 s, and repeating the command restarts the delay.** Retry loops leave the panel stuck on HDMI. Send one command and wait. The script locks out repeats.
- **The HDMI input stays black after switching,** even though the Mac says the display is connected. It isn't display sleep. Unplugging and replugging the cable fixes it; turning the Mac's display off and on (`pmset displaysleepnow` then `caffeinate -u`) does the same in software. This doesn't drop the Deskflow session.
- **The panel ignores a set issued right after a read,** hence the 0.5 s pause.

**Deskflow on Cinnamon (X11)**
- **Super+key hotkeys never reach Deskflow:** the window manager grabs the keyboard while Super is held. The Deskflow log still says "registered hotkey", so it fails silently. Use Ctrl+Shift+Fn instead, and avoid Ctrl+Alt+Fn, which switches Linux virtual terminals.
- **The GUI's hotkey editor saves the Windows key as `Meta`,** which X11 couldn't map ("could not map hotkey … mask=0008"). That's why the hotkeys live in `server-custom.conf`, and **the GUI hotkey list is ignored**.
- **Deskflow 1.27 ignores a `screens` section in an external config.** Screens come from the GUI grid; links and options come from the file.
- **Adjacent computers on the grid create edge links,** so the mouse can slide onto the Mac. Without a working hotkey or screen-follow, you're locked out. Keep a gap.
- **A changed Cinnamon shortcut can keep its old key grabbed until re-applied.** This can masquerade as Deskflow working: the screen switches but the keyboard stays put. Check the Deskflow log for `switch from "…" to "…"`.
- **"not starting core process, no core mode set"** means `startCoreWithGui=true` before Server or Client was chosen. Choose a mode.
- **To debug,** set `[log] level=Debug`, `toFile=true`, `file=<path>` on both sides, then read both logs.

**macOS**
- **After a reboot, nothing switches until someone logs in on the Mac.** The LaunchAgent only starts Deskflow after login, so a Mac sitting at the login window never connects. Log in with the spare keyboard (use the screen-only shortcut to see the Mac), or enable **System Settings → Users & Groups → Automatically log in as…**. That isn't available with FileVault on, and is only sensible on a non-personal machine.
- **Keyboard and mouse don't work on the Mac** until Deskflow has Accessibility permission *and* is restarted.
- **The Mac's Deskflow screen name comes from its hostname** (e.g. `Mac-mini.localdomain`), not the friendly computer name. A mismatch shows up as an extra "unrecognized" computer on the server.
- **`/usr/bin/python3` on a Mac without Command Line Tools pops an installer dialog.** Use shell or `sed -i ""` instead.
- **Logitech Options+ may update device firmware.** Single-host Logitech devices (e.g. K650/M650) paired to the Mac by Bluetooth are taken away from the Linux Bolt receiver. Re-pair them with Solaar (`solaar pair`, or the GUI).

**Logitech note:** Solaar can only switch hosts on devices with Easy-Switch (HID++ feature CHANGE HOST 0x1814, e.g. MX Keys S / MX Master 3S). The K650/M650 are single-host, which is why this setup uses Deskflow.
