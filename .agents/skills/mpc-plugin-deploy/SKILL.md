---
name: mpc-plugin-deploy
description: Install, list, update, remove or export VST2 plugins on a real Akai MPC over the network (SSH, developer mode) with tools/mpc_remote.py, and walk the user through setting the MPC up (Enhancements, External Plugin Support, developer mode). Also installs effects on an MPC Sample through its microSD card in SD Card Access mode. Use when the user wants a plugin put on their MPC or MPC Sample, gives an MPC IP address, or asks to back up or manage plugins on the device.
---

# Deploy to an MPC

## MPC Sample: install through its microSD card

An MPC Sample is not installed over the network: do not use `mpc_remote.py` for it. Its plugins go in `VST2 Plugins` on its microSD card, which the computer sees while the MPC Sample is in **SD Card Access** mode (`docs/knob-fx.md`). If the user says "my MPC" and you do not know which model, ask. The same rule applies as for other MPCs: copy, replace or delete a plugin on the card only when the user has asked for that, in this conversation; a clean build is not permission.

1. **Check the plugin.** It must be an **effect**; the MPC Sample does not load instruments, so tell the user and stop for an instrument. It must have passed the build and `host_sim` checks, and `plugins/<Name>/build/` must hold the `"<Manufacturer> - VST - <Name>"` folder with `file64` (the MPC Sample is 64-bit).
2. **One message to the user, then wait for the answer.** Ask them to:
   - put a microSD card in the MPC Sample and connect it to the computer with USB;
   - on the MPC Sample, open **MIDI Config** and, in the **Dev Enhancements (Beta)** section, turn on **Dev Enhancements (Beta)** and **Custom Plug-In Support** (only needed once);
   - enter **SD Card Access** mode from the **Project** screen, so the card appears on the computer;
   - tell you when the card is showing on the computer.

   In the same message, **ask whether you may eject the SD card from the computer when you have finished copying**, and say why you are asking: they might be using that card on the computer for something else. Only eject if they say yes.
3. **Find the card.** macOS: `ls /Volumes`. Linux: `findmnt -rno TARGET,SOURCE,FSTYPE | grep -E ' (/media|/run/media)/'`. Windows (PowerShell): `Get-Volume | Where-Object DriveType -eq 'Removable'`. If exactly one removable volume is there, use it and name it in your next message. If there are none or several, show the list and ask which one. Never write to the system disk or any other drive.
4. **Copy.** Copy the whole folder into `VST2 Plugins` at the top level of the card (create `VST2 Plugins`, spelled exactly like that, if it is missing). If a folder with the same name is already there, this is an update: replace it, and say so. Copy into a staging folder at the card root first, so a failed copy never leaves half a plugin in `VST2 Plugins`:

   ```
   python3 -I - "plugins/<Name>/build/<Manufacturer> - VST - <Name>" "<card mount point>" <<'EOF'
   import os, shutil, sys
   src, card = os.path.normpath(sys.argv[1]), sys.argv[2]
   name = os.path.basename(src)
   plugins = os.path.join(card, "VST2 Plugins"); os.makedirs(plugins, exist_ok=True)
   stage = os.path.join(card, ".mpc-sdk-staging"); shutil.rmtree(stage, ignore_errors=True)
   shutil.copytree(src, os.path.join(stage, name), copy_function=shutil.copyfile,
                   ignore=shutil.ignore_patterns(".DS_Store", "Thumbs.db", "._*"))
   dest = os.path.join(plugins, name)
   if os.path.isdir(dest): shutil.rmtree(dest)
   os.replace(os.path.join(stage, name), dest); os.rmdir(stage)
   def files(root): return {os.path.relpath(os.path.join(r, f), root): os.path.getsize(os.path.join(r, f))
                            for r, _d, fs in os.walk(root) for f in fs if f not in (".DS_Store", "Thumbs.db") and not f.startswith("._")}
   assert files(src) == files(dest), "copy differs from the build folder"
   print("installed", dest)
   EOF
   ```

   It writes nothing outside `VST2 Plugins` and the short-lived `.mpc-sdk-staging` folder. Never delete or change anything else on the card.
5. **Eject (only with permission).** macOS: `diskutil eject "<card mount point>"`. Linux: `udisksctl unmount -b "$(findmnt -no SOURCE "<card mount point>")"`. Windows (PowerShell): `(New-Object -ComObject Shell.Application).Namespace(17).ParseName('E:').InvokeVerb('Eject')` with the card's drive letter, then check the drive has gone. If ejecting fails (the card is busy), say which program may be holding it and let the user eject; never force it. Without permission, tell the user to eject the card on the computer themselves.
6. **Tell the user what to do next:** once the card is ejected, exit **SD Card Access** mode on the MPC Sample. The effect then appears as a Knob FX at the **bottom of the FX Select list**; no restart is needed. Remind them of its 3 knobs and 3 SHIFT knobs (from `plugin.json`). If it does not appear, use `mpc-plugin-troubleshoot`.

## Other MPCs: over the network

`tools/mpc_remote.py` talks to an MPC on the local network over SSH and manages the `VST2 Plugins` folder of one of its drives. It needs only Python 3.9+ and the computer's `ssh` command (use `python3` if `python` is not found).

Offer this after every clean build: "Do you want it on your MPC for a test? What is its IP address?" Use `update` for later builds. Do not connect, `add`, `update`, `import`, `remove` or `restart` without their explicit yes and an address: **The user decides what is installed, updated or removed, and where.** Never run `mpc_remote.py add`/`update`/`import`/`remove`/`restart` unless the user has explicitly asked for that action on that MPC, in this conversation. A request such as "install it on my MPC" or "update it when you're done" covers exactly what it names; a yes to one install does not cover the next build or another MPC, unless they said so ("update it every time you rebuild"). Otherwise finish the build, then ASK whether they want it installed on an MPC and wait for the answer. Silence, a clean build or a passing test is not permission.

## What the user must have done on the MPC

Check these with the user before connecting (the script also diagnoses them, but the user has to do them on the device):

1. MPC OS **3.10.1 or later** (Preferences → Info).
2. **Preferences → Enhancements (Beta)**: turn on **Enable Enhancements**, then **External Plugin Support**. Read the warning with them.
3. **Developer mode** on: Preferences → Info; tap the version value 3 times, the serial number value 2 times, then the firmware version once. This starts the MPC's SSH server. (If it does not seem to work, ask the user to describe what the Info page shows.)
4. The MPC and the computer on the same network. The MPC's IP address is in Preferences, on the Wi-Fi tab (hold Shift to access it) or on the Ethernet tab, depending on how it is connected.

Developer mode is an official, supported switch, but it opens a login on the device: tell the user to switch it off again when finished on a network they do not control. This skill only ever writes inside the `VST2 Plugins` folder of the chosen drive. Never use it, or any SSH access, to change system files, firmware or settings.

## Ask for what you need

- The MPC's **IP address** (ask if not given). A password is only needed if the user has set one; do not guess passwords and do not put one in files. If one is needed pass it with the `MPC_SSH_PASSWORD` environment variable (preferred) or `--password`.
- Which drive, if several are mounted (the script lists them; `--drive` takes the number, name or mount point). Units with an internal NVMe drive list it too (plugins go in `/nvme/VST2 Plugins`). Removable drives may be mounted `noexec`; the script warns, and the internal drive is the fallback.

## Commands

```
python3 tools/mpc_remote.py <ip>                         # interactive menu (needs a real terminal)
python3 tools/mpc_remote.py <ip> info                    # model, serial number, firmware version, CPU, drives
python3 tools/mpc_remote.py <ip> doctor                  # setup diagnostics
python3 tools/mpc_remote.py <ip> restart --yes           # restart the MPC app so it rescans VSTs
python3 tools/mpc_remote.py <ip> list                    [--drive N]
python3 tools/mpc_remote.py <ip> add plugins/<Name>      # an SDK project (its build/ output) ...
python3 tools/mpc_remote.py <ip> add some-plugin.zip     # ... or a zip from the package tool / a download
python3 tools/mpc_remote.py <ip> update plugins/<Name>   # replace an installed copy, no question asked
python3 tools/mpc_remote.py <ip> import <release>.zip [<aarch64>.zip] --out converted --zip   # convert a community mpc-vst-plugins package (needs no hacked MPC OS), save it, then add it
python3 tools/mpc_remote.py <ip> remove "<folder or name>" --yes
python3 tools/mpc_remote.py <ip> export <name> | --all --out <dir>    # always written as zip files
```

Always pass `--drive` and `--yes` when you drive it from an agent shell: there is no terminal to answer prompts, and `--drive` is required when more than one drive is mounted. If the user has several drives and has not said which, run `info` first, show them the list and ask.

Add `--no-color` when output is captured. Exit codes: 0 ok, 1 an operation failed, 2 the MPC is not reachable or not set up (read the diagnostic panel).

## Workflow for "put my plugin on my MPC"

For an MPC Sample, use the MPC Sample section above instead.

1. Make sure the plugin is built and tested (`mpc-plugin-build`, `mpc-plugin-test`): `plugins/<Name>/build/` must hold the `"<Manufacturer> - VST - <Name>"` folder. Never deploy something that has not passed the build and `host_sim` checks.
2. Run `info` (or `doctor`) to confirm the connection and pick the drive.
3. `add plugins/<Name> --drive <d>` (first time) or `update plugins/<Name> --drive <d>` (new build). The tool checks that the ARM build matching the MPC's processor is present, copies via a hidden staging folder, verifies every file size, and keeps the previous version until the new one has arrived intact.
4. After every successful `add`, `update` or `import`, tell the user the MPC only scans for VST2 plugins when its app starts, so the plugin will not appear until it is restarted. **Suggest the restart** and offer to do it with `restart` (see below); remind them to save their project first and only run it once they agree. They can also restart by powering the MPC off and on. Then tell them the plugin is under **External Plugins** in the plugin browser; for a track that already uses an updated plugin, remove and re-add it (a restart also picks up changed skins). If it still does not show after the restart, run `doctor` and check the Enhancements switches.

## Restarting the MPC app

`import` takes zips or folders from the community plugin store (https://sd88me.github.io/mpc-vst-plugins/, made for a modified MPC OS) and rewrites them into the SDK folder format with `tools/legacy_convert.py` (also usable alone, no MPC needed). Give the armv7 and aarch64 zips of one plugin together to get one folder that works on every model. The converted folder (and a zip with `--zip`) is saved to `--out` for redistribution, and installing it asks the same questions as `add`. The plugin binaries are not modified; mention the plugin's licence (recorded in `plugin.json`) before redistributing, and that a plugin written for the modified OS may rely on features the stock host lacks, so test it.

`restart` restarts the MPC application service (`acvs`) so it rescans VST2 plugins; it needs no drive. **Any unsaved project data on the MPC is lost**, so never run it (or pass `--yes`) without the user explicitly agreeing, and remind them to save first. Suggest it after every install or update, since the MPC only rescans VSTs at startup; it is not needed after `list`, `export` or other read-only commands.

## When it will not connect

Read the panel the script prints; it names the cause:

| Panel | Meaning | What to do |
|---|---|---|
| "can't reach" | Timeout: wrong address, different network, client isolation | Check the IP in Preferences (Wi-Fi tab, hold Shift; or Ethernet tab), same Wi-Fi, try Ethernet |
| "SSH server is off" | Connection refused: developer mode is not on | Do steps 1-3 above, then run again |
| "Login refused" | SSH is up but wants a password or another user | Ask the user for it; `--password`/`MPC_SSH_PASSWORD`, `--user` |
| "No drives found" | Nothing mounted under /media | Insert a USB drive / SD card, check the internal storage |
| "read-only" | The drive is write protected or mounted read-only | Another drive, or the card's lock switch |

Remember `doctor` cannot read the Enhancements / External Plugin Support switches remotely; if everything installs fine but the plugin never appears in the browser, those switches (and MPC OS 3.10.1+) are the first thing to check. Then `mpc-plugin-troubleshoot`.

## What the tool does not do

It does not build, rename or edit plugins, and never installs outside `VST2 Plugins`. Apart from the app restart above, it does not run anything else on the device. Pass-through of unknown commands to the device is not offered on purpose.
