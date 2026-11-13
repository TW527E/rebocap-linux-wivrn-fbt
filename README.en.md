[简体中文](README.md) · [繁體中文](README.zh-TW.md) · **English** · [日本語](README.ja.md)

# Rebocap × Linux × WiVRn full-body tracking (VRChat)

Connect **Rebocap** inertial full-body tracking to **VRChat** on Linux, without SteamVR, through **WiVRn + xrizer**,
with a Meta Quest headset.

Rebocap now has an official Linux version, but it only ships a **SteamVR driver**. If you use SteamVR, the official
version is all you need and you don't need this repository. This repository is for people who **use WiVRn instead
of SteamVR**: a CLI turns Rebocap's trackers into SolarXR trackers and feeds them to WiVRn.

It covers the setup I actually use, installation from scratch, everyday use, and every problem I hit along the way
with its fix.

## Two setups

| | **A. Official Linux version (recommended)** | B. Windows version in Wine (Bottles) |
|---|---|---|
| Rebocap | Official native Linux app | Windows version installed in Bottles |
| Receiver | The official app uses USB directly | Wine COM port + setupapi patch |
| Link to the CLI | Rebocap itself listens on TCP `127.0.0.1:20480` | `bridge.exe` turns the named pipe into TCP `127.0.0.1:36850` inside Wine |
| What you need | Rebocap, the CLI, this repository | Rebocap, Bottles, the rebocap-wine patch, bridge.exe, the CLI, this repository |
| Tracker firmware | Can be updated with the official app (I went from v15 to v17 and it feels more stable) | Don't update under Wine |

Setup A drops three layers (Wine, the patch and bridge.exe) and is much simpler to start and stop. Use A unless the
official version doesn't run on your machine. Setup B stays as a fallback and as a record of how it was first made to work.

```
Setup A:
Quest headset → WiVRn → CLI ⇄ Rebocap (native Linux, :20480)
Rebocap trackers → Rebocap → CLI → SolarXR → WiVRn → xrizer → VRChat

Setup B:
Quest headset → WiVRn → CLI ⇄ bridge.exe (Wine, :36850) ⇄ Rebocap (Wine)
Rebocap trackers → Rebocap → bridge.exe → CLI → SolarXR → WiVRn → xrizer → VRChat
```

The launcher `rebocap-fbt` supports both and detects which one is running: if port 20480 is listening it uses
setup A, otherwise it looks for bridge.exe on 36850.

## Tested setup

| Item | Version / setting |
|---|---|
| OS | Arch Linux (Omarchy, Hyprland), NVIDIA RTX 3070 |
| Headset | Meta Quest, WiVRn (Wi-Fi or USB) |
| WiVRn | wivrn-server / wivrn-dashboard 26.9 |
| xrizer | xrizer-git 0.6.r6 (the rebocap-wivrn-bridge author tested 0.5; 0.6 works too) |
| Rebocap (setup A) | Official Linux version `rebocap-linux-preview-0.1-1`, tracker firmware v17 |
| Rebocap (setup B) | Windows version in Flatpak **Bottles**, runner soda-11.0-10 (with ntsync), tracker firmware v15 |
| Trackers | Usually 10: waist, chest, both upper legs, both lower legs, both feet, both upper arms (the forearm trackers moved to the lower legs). For testing, 6: waist, both upper legs, both lower legs, chest |

## Projects used

| Project | Purpose | Setup |
|---|---|---|
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | The CLI (`monad2steamvr`): relays head and tracker poses between WiVRn and Rebocap and publishes the trackers through SolarXR. A fork of [akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge) with a fix for a moved WiVRn play space (libmonado) and without the two 10 ms polling waits on tracker poses (about 10 ms less latency on average) | A, B |
| This repository | The `rebocap-fbt` launcher, the VRChat binding generator and this guide | A, B |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | setupapi patch so Rebocap in Wine recognizes the receiver | B |
| [TW527E/rebocap-linux-compat](https://github.com/TW527E/rebocap-linux-compat) | `bridge.exe`: turns Rebocap's named pipe into TCP inside Wine. A fork of [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat) with Bottles/ntsync fixes | B |

## Installation (from scratch)

Do the common steps first, then setup A or B.

### Common 1. WiVRn and xrizer

Install wivrn-server, wivrn-dashboard and xrizer following WiVRn's instructions, and make sure VRChat works without full-body tracking.

### Common 2. The CLI

```sh
sudo pacman -S rust openxr pkgconf
git clone https://github.com/TW527E/rebocap-wivrn-bridge ~/Projects/rebocap-wivrn-bridge
cd ~/Projects/rebocap-wivrn-bridge && cargo build --release
```

### Common 3. The launcher

```sh
git clone https://github.com/TW527E/rebocap-linux-wivrn-fbt ~/Projects/rebocap-linux-wivrn-fbt
ln -s ~/Projects/rebocap-linux-wivrn-fbt/scripts/rebocap-fbt ~/.local/bin/rebocap-fbt
```

The launcher finds the CLI (`monad2steamvr`) in this order: `REBOCAP_CLI`, `PATH`, a `rebocap-wivrn-bridge` next to
this repository, then `rebocap-wivrn-bridge` under `~/Projects`, `~/Downloads`, `~/src` and `~/`. If it lives
elsewhere, or other paths differ, override them with `REBOCAP_CLI`, `REBOCAP_BOTTLE`, `REBOCAP_RECEIVER` and
`REBOCAP_XR_RUNTIME_JSON`. The ports can be changed with `REBOCAP_NATIVE_PORT` (default 20480) and
`REBOCAP_WINE_PORT` (default 36850).

### Setup A: official Linux version (recommended)

1. Download the Linux installer following the [official instructions](https://doc.rebocap.com/zh-Hans/docs/tutorial/software_install) and run it:
   ```sh
   chmod +x rebocap-linux-preview-*.run
   ./rebocap-linux-preview-*.run
   ```
   I installed it to `~/.local/opt/rebocap`. Start Rebocap from the app menu or run `~/.local/opt/rebocap/rebocap-ui`.
2. Give yourself access to the receiver. The official script writes `/etc/udev/rules.d/70-rebocap.rules` and adds you
   to one of the `dialout` / `uucp` / `plugdev` groups (`uucp` on Arch):
   ```sh
   sudo bash ~/.local/opt/rebocap/linux/install_linux_permissions.sh --install --user "$USER"
   ```
   Then log in again and replug the receiver.
3. Open Rebocap and update the tracker firmware if it asks. **Don't unplug the receiver or close the app during the update.**
4. **Don't** install the official SteamVR driver (`data/rebocap_driver`). We don't use SteamVR, and port 20480 takes
   only one client at a time, so a connected SteamVR driver would lock out the CLI.

Bottles, rebocap-wine and bridge.exe are not needed.

> While it runs, the official app detaches the receiver from `cdc_acm` and talks USB directly, so
> `/dev/ttyACM*` and `/dev/serial/by-id/...` disappear. That's expected; the launcher looks for USB device 248a:8002 instead.

### Setup B: Windows version in Wine (Bottles)

#### B1. Receiver permissions

Your user must be in the group that owns `/dev/ttyACM*` (`uucp` on Arch) so access survives a reboot:

```sh
sudo usermod -aG uucp "$USER"   # log in again afterwards
```

I also added a udev rule `/etc/udev/rules.d/70-rebocap.rules`; it is optional:

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="248a", ATTRS{idProduct}=="8002", MODE="0660", TAG+="uaccess"
```

#### B2. Rebocap in Bottles

1. Give Bottles device access so Wine can reach the serial port: `flatpak override --user --device=all com.usebottles.bottles`.
2. Create a bottle (mine is called `Rebocap`, win64) and install Rebocap.
3. Apply the [rebocap-wine](https://github.com/TW527E/rebocap-wine) setupapi patch so Rebocap can enumerate the receiver.
4. Map the receiver to a COM port in the bottle's `dosdevices`, for example `com33 -> /dev/ttyACM0`.

#### B3. bridge.exe

```sh
git clone https://github.com/TW527E/rebocap-linux-compat
cd rebocap-linux-compat
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32   # needs mingw-w64-gcc
cp bridge.exe ~/.var/app/com.usebottles.bottles/data/bottles/bottles/Rebocap/drive_c/Program\ Files\ \(x86\)/Rebocap/
```

If you'd rather not compile it, download `bridge.exe` from [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest).

You **don't** need its `install.py` or the SteamVR driver.

Adding `bridge.exe` to the bottle as a program in the Bottles GUI makes it easy to start every day.

### (Optional) Right stick click = jump

If your controller's A button is broken, you can make a right stick click jump too. Generate xrizer bindings:

```sh
./xrizer-bindings/make-vrchat-bindings.py      # writes ~/.config/xrizer-bindings/oculustouch.json
```

Then put this at the very start of VRChat's Steam launch options:

```
XRIZER_CUSTOM_BINDINGS_DIR=/home/<you>/.config/xrizer-bindings
```

The right stick click normally does "Udon Stick Click", which only a few worlds use. A still jumps. Rerun the
generator after VRChat updates. To undo it, remove the launch option.

## Everyday use

**Without full-body tracking**: as usual. Start WiVRn → connect the headset → start VRChat. You don't need Rebocap or the launcher.

**With full-body tracking**, **the order matters**:

1. Start **WiVRn** (wivrn-dashboard) and make sure the server is running. **Don't connect the headset yet.**
2. Start **Rebocap**:
   - Setup A: open the official Linux app.
   - Setup B: start Rebocap in Bottles, then bridge.exe.
3. In a terminal:
   ```sh
   rebocap-fbt          # 10 points (default): waist, chest, upper legs, lower legs, feet, upper arms. Detects the setup
   rebocap-fbt 12       # 12 points: the 10 plus both forearms
   rebocap-fbt 6        # 6 points
   rebocap-fbt 10 --usb # connect the headset over USB once ready
   rebocap-fbt --wine   # force setup B (--native forces setup A)
   ```
   The launcher's messages are in Traditional Chinese. The first lines show which setup it uses, for example
   `模式：官方 Linux 版 Rebocap（127.0.0.1:20480）` (mode: official Linux Rebocap).
4. When "**SolarXR 準備好了**" (SolarXR is ready) appears (also as a desktop notification), **connect the headset**.
5. Switch Rebocap to **VR mode** and **calibrate** with the headset on.
6. Start **VRChat** from Steam and press **Calibrate FBT** in the quick menu.

The launcher checks the body parts:

```
有資料的部位：0 1 2 3 4 5 6 7 9 10
  ⚠ 已發布但沒有資料：…
  ⚠ 有資料但沒有發布：…
```

The first line lists the parts that send data. A ⚠ line means parts are "published but sending no data" or
"sending data but not published": the body-part argument doesn't match the trackers you wear, so fix the argument.

**Stopping**:

1. Press **Ctrl+C** in the launcher's terminal. It stops the CLI. In setup B it also asks the bridge to exit through
   `bridge.quit` (`--keep-bridge` leaves it running).
2. Close the **Rebocap** window.
3. Handle WiVRn as usual.

**Important**: after restarting the CLI or changing body parts, **reconnect the headset** and **restart VRChat**.
WiVRn reads the tracker list only when the headset connects.

### Body part numbers

| No. | Part | No. | Part |
|---|---|---|---|
| 0 | Waist | 7 | Chest |
| 1 / 2 | Left / right upper leg | 9 / 10 | Left / right upper arm |
| 3 / 4 | Left / right lower leg | 11 / 12 | Left / right forearm |
| 5 / 6 | Left / right foot | | |

VRChat assigns trackers to hips, feet, knees, chest and elbows by **position** during calibration, so the 6-point
kit's lower leg trackers can serve as feet.

The CLI's default `0,5,6` only suits kits with foot trackers. The 6-point kit has none: 5 and 6 are virtual nodes
Rebocap builds, which send only brief bursts of data, so with the default VRChat won't find your feet.

### USB connection

1. The headset needs developer mode and must allow USB debugging from this computer. The prompt appears inside
   the headset, so put it on to see it.
2. Run `adb devices`; the state should be `device`. `unauthorized` means you haven't allowed it in the headset yet.
3. Use `rebocap-fbt --usb`, or press "Connect via USB" in wivrn-dashboard.

By hand:

```sh
adb reverse tcp:9757 tcp:9757
adb shell am start -a android.intent.action.VIEW -d "wivrn+tcp://127.0.0.1:9757" org.meumeu.wivrn
```

## Updating

```sh
cd ~/Projects/rebocap-wivrn-bridge && git pull && cargo build --release   # CLI
cd ~/Projects/rebocap-linux-wivrn-fbt && git pull                         # launcher (the symlink picks it up)
```

Update the official Linux Rebocap with `maintenancetool` in its install folder, or reinstall from a new installer.

## Latency

Up to the CLI the two setups differ very little: setup A skips Wine's named pipe, bridge.exe's polling and Wine's
serial emulation, saving roughly 1–3 ms. The bigger difference is in the CLI: in the original CLI, tracker poses
waited in two 10 ms polling loops, adding about 10 ms on average (up to ~20 ms) with uneven update spacing.
[TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) sends them to WiVRn as soon as they arrive.

## Problems and fixes

### Both setups

| Symptom | Cause | Fix |
|---|---|---|
| CLI: `Failed to connect to monado service process` | The WiVRn server isn't running (an open dashboard doesn't mean the server is running) | Press "Start server" in the dashboard |
| The CLI can't find WiVRn | The system has no `active_runtime.json` | Set `XR_RUNTIME_JSON=/usr/share/openxr/1/openxr_wivrn.json` (the launcher does this) |
| The CLI prints `invalid SolarXR frame` plus usage | Full-width punctuation in the body-part argument (for example `12。`) | Use only ASCII digits and commas |
| VRChat doesn't find the feet | The body-part argument doesn't match the trackers you wear | Read the launcher's body-part check and fix the argument |
| Trackers drift off the body after moving the play space | The CLI can't read WiVRn's STAGE offset (log shows `libmonado unavailable`) | Use the TW527E CLI fork and check that WiVRn's runtime JSON has `MND_libmonado_path` |

### Setup A (official Linux version)

| Symptom | Cause | Fix |
|---|---|---|
| The launcher says the receiver is missing, but Rebocap sees it | The official app detaches the receiver from `cdc_acm` and uses USB directly, so `/dev/ttyACM*` disappears | The launcher now looks for USB device 248a:8002 |
| The CLI connects but never gets data | Rebocap isn't in VR mode yet, or is updating firmware | Wait for the update to finish, switch to VR mode and calibrate |
| The launcher warns that something is already connected to 20480 | The official SteamVR driver (or another CLI) holds the port, which accepts one client | Close SteamVR / the other CLI |

### Setup B (Wine / Bottles)

The full debugging log is in [docs/troubleshooting.md](docs/troubleshooting.md) (Traditional Chinese). In short:

| Symptom | Cause | Fix |
|---|---|---|
| bridge.exe says it is listening, but connections never get an answer | Blocking socket calls are broken in this Wine (soda-11.0-10 + ntsync) | The bridge uses non-blocking sockets with event waits (fixed in the fork) |
| No waist tracker, foot data comes and goes, the bridge keeps printing `0 bytes` | Pipe reads report 0 bytes and the message is lost | Read only after `PeekNamedPipe` reports data (fixed in the fork) |
| The bridge uses a lot of CPU and Rebocap stops responding | A pipe read stays pending and its I/O status never updates | Same fix |
| Rebocap crashes after `kill`ing the bridge from a terminal | wineserver segfaults while handling the killed process | **Never kill Wine processes from the host**; use `bridge.quit` |
| Rebocap shows "network verification failed, exiting in 10 seconds" and hangs | wineserver is gone, so all networking inside Wine fails | End the whole bottle and start it again. It isn't a network problem |
| Bottles' "Stop" hangs | The bridge is a console program and Rebocap ignores Windows shutdown messages | Use `bridge.quit` first, then close the Rebocap window by hand |

## License

The scripts and documents in this repository are MIT licensed. The `bridge.exe` code comes from
dogesoulseller/rebocap-linux-compat (MIT). The CLI is developed by akazdayo/rebocap-wivrn-bridge (my changes are in
TW527E/rebocap-wivrn-bridge); this repository only links to it and contains none of its code. Rebocap is JXQ's
proprietary software; this repository contains none of its files.
