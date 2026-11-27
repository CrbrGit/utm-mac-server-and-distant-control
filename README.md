# UTM on a headless Mac: SSH control and VM autostart

> **Status:** personal notes, shared as-is in case they help someone hitting the
> same problem. Not a maintained project: issues and pull requests may go
> unanswered. Fork it and adapt it freely (0BSD license).

## Goal

Use a Mac (here a Mac mini) as an always-on VM server for QEMU and Apple
Virtualization VMs, without sitting in front of it:

- control the VMs from other hosts over plain SSH (list, start, stop, exec…);
- have selected VMs come back up on their own after a reboot or power failure;
- run nothing extra: no web UI, no reverse proxy, no open port besides SSH.

Two small scripts do this:

- **`utmctl`** (wrapper): run UTM's `utmctl` from SSH sessions.
- **`utm-autostart`**: start chosen VMs automatically at boot.

## Why the wrapper is needed

UTM's `utmctl` drives the UTM app through Apple Events. macOS checks Apple Events
permission against the process that started the chain. From SSH that is `sshd`,
which can never show the consent prompt, so every call fails:

```
Error from event: The operation couldn't be completed. (OSStatus error -1743.)
NOTE: utmctl does not work from SSH sessions or before logging in.
```

Things that do **not** fix it:

- `sudo launchctl asuser <uid> sudo -u <user> utmctl …`: still attributed to `sshd`.
- System Settings → Privacy & Security → Automation: `sshd` never gets an entry, and there is no "+" button.
- A PPPC configuration profile is only honoured when pushed by MDM; editing `TCC.db` requires disabling SIP.

What works: a **LaunchAgent in the GUI session** (`gui/<uid>`). launchd starts
`utmctl` itself, inside the logged-in session, and UTM accepts it with no prompt.
UTM's `utmctl` is inside `UTM.app` and signed the same way (probable reason, not verified).

The wrapper does that on every call: it builds a one-shot LaunchAgent with
`plutil` (arguments are passed verbatim), bootstraps it, polls `launchctl print`
for the exit code, prints stdout/stderr, cleans up and exits with utmctl's code.

## Files

| File | Installed as | Purpose |
|---|---|---|
| `utmctl` | `/usr/local/bin/utmctl` | SSH-safe wrapper around `/Applications/UTM.app/Contents/MacOS/utmctl` |
| `utm-autostart` | `/usr/local/bin/utm-autostart` | Starts VMs listed in `/etc/utm-autostart.conf` |
| (created) | `/etc/utm-autostart.conf` | One VM UUID or exact name per line, `#` comments |
| (created) | `~/Library/LaunchAgents/local.utm-autostart.plist` | Runs `utm-autostart` at GUI login |

## Mac setup (GUI, once, at the console or via Screen Sharing)

Both scripts need a logged-in GUI session of the user that owns the VMs, so the
Mac must log in by itself and never go to sleep. Menu names are from recent
macOS (14/15) and may differ slightly between versions.

1. **Install UTM** at `/Applications/UTM.app` and create the VMs as that user.
2. **Turn FileVault off** (required for automatic login):
   System Settings → Privacy & Security → FileVault.
3. **Automatic login**: System Settings → Users & Groups → "Automatically log in as" → the VM user.
4. **Open UTM at login**: System Settings → General → Login Items → "Open at Login" → `+` → UTM.
   Recommended; `utm-autostart` also waits up to ~2 min for UTM to answer.
5. **Never sleep, restart after power loss**: System Settings → Energy →
   enable "Prevent automatic sleeping when the display is off" and
   "Start up automatically after a power failure".
   CLI equivalent: `sudo pmset -a sleep 0 autorestart 1`.
6. **Remote Login (SSH)**: System Settings → General → Sharing → Remote Login → on.
7. **Screen Sharing** (optional): same Sharing pane. Lets you reach the GUI to
   click a permission prompt or look at UTM without a screen attached.

Without a GUI session neither script can work.

All commands below tolerate leading spaces when pasted (no heredocs).

## Install

### 1. Copy the scripts (from the machine holding this directory)

```bash
scp utmctl utm-autostart <user>@<mac>:/tmp/
```

### 2. Install the wrapper (on the Mac)

> **Warning:** if `/usr/local/bin/utmctl` is a symlink to UTM's binary (UTM's docs
> suggest creating one), **remove it first**. Copying onto the symlink would
> overwrite UTM's real `utmctl`, and the wrapper would then call itself forever.

```bash
ls -l /usr/local/bin/utmctl; sudo rm -f /usr/local/bin/utmctl
sudo install -m 755 /tmp/utmctl /usr/local/bin/utmctl; sudo install -m 755 /tmp/utm-autostart /usr/local/bin/utm-autostart; rm /tmp/utmctl /tmp/utm-autostart
file /Applications/UTM.app/Contents/MacOS/utmctl
```

The last line must say `Mach-O … executable`. If it says `shell script`, reinstall UTM.

### 3. Create the autostart config

List VMs with `utmctl list` (on the Mac, or through the wrapper over SSH), then:

```bash
printf '%s\n' '# UTM VMs to start at login (UUID or exact name)' <VM-UUID> | sudo tee /etc/utm-autostart.conf >/dev/null; sudo chmod 644 /etc/utm-autostart.conf
```

Keep it world-readable (644): the LaunchAgent runs as the user, not root.
Prefer UUIDs: names break on rename or when a clone has the same name.

### 4. Create the LaunchAgent

```bash
P=~/Library/LaunchAgents/local.utm-autostart.plist; mkdir -p ~/Library/LaunchAgents; plutil -create xml1 $P
plutil -insert Label -string local.utm-autostart $P
plutil -insert ProgramArguments -array $P
plutil -insert ProgramArguments.0 -string /bin/bash $P
plutil -insert ProgramArguments.1 -string /usr/local/bin/utm-autostart $P
plutil -insert RunAtLoad -bool true $P
```

No log files are configured on purpose; launchd discards the output.

### 5. Load and verify

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.utm-autostart.plist
/usr/local/bin/utm-autostart
```

The manual run prints `started` / `already started` per VM. Then reboot once and
check the VMs came up (`ssh <user>@<mac> /usr/local/bin/utmctl list`).

## Usage

Remote control (non-interactive SSH usually lacks `/usr/local/bin` in `PATH`, so use the full path):

```bash
ssh <user>@<mac> /usr/local/bin/utmctl list
ssh <user>@<mac> /usr/local/bin/utmctl start <VM>
ssh <user>@<mac> /usr/local/bin/utmctl stop <VM>
ssh <user>@<mac> /usr/local/bin/utmctl --stdin file push <VM> /tmp/x < localfile
```

Autostart list: edit `/etc/utm-autostart.conf` with `sudo`. Applies at next login.
VMs already running are skipped, so re-running is harmless.

## Wrapper limitations

- Needs the user logged in on the GUI (checked; fails with a clear message).
- Output is shown when the command finishes, not streamed.
- Stdin is only forwarded with `--stdin` (by default it would hang interactive calls).
- Timeout 600 s by default; override with `UTM_TIMEOUT=<seconds>`.
- The exit code is parsed from `launchctl print`, whose format Apple does not
  guarantee; a macOS update could break it.
- Locally on the Mac, calls also go through launchd (slightly slower, still works).

## Troubleshooting

- A VM did not autostart: run `/usr/local/bin/utm-autostart` in a terminal on the Mac and read its output.
- If an "allow to control UTM" prompt ever appears on the Mac's screen, click Allow
  (use Screen Sharing if headless). None appeared on the original setup.

## Uninstall

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/local.utm-autostart.plist; rm ~/Library/LaunchAgents/local.utm-autostart.plist
sudo rm /usr/local/bin/utm-autostart /usr/local/bin/utmctl /etc/utm-autostart.conf
```

Optionally restore UTM's own CLI symlink:
`sudo ln -s /Applications/UTM.app/Contents/MacOS/utmctl /usr/local/bin/utmctl`

## Background

UTM has no built-in "start VM at launch"; the maintainer suggests starting VMs at
macOS login instead ([UTM discussion #5913](https://github.com/utmapp/UTM/discussions/5913)).
The `utm://start` URL scheme is unreliable ([UTM issue #6346](https://github.com/utmapp/UTM/issues/6346)).

If you want a browser UI with VM consoles (serial and VNC), see
[utm-webapp](https://github.com/naveenrajm7/utm-webapp).
