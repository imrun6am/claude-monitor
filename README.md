# Claude monitor

Reports every Claude Code session start, prompt, model switch, browser action
and session end on a device to your PM tool, so you can see which device used Claude Code on
your account and when. The log lives on the PM tool server (**Claude activity** in its sidebar).
Entries can't be edited there, so they outlast deleted conversations; only the
people allowed to read the log can delete them.

The PM tool side (the `/api/claude-activity` endpoint and the page) ships with
the PM tool. This repo holds only the device installers.

## Install on a device

You need the PM tool's URL and its `CLAUDE_MONITOR_TOKEN` (from the server's
`.env`). The installer asks for the token with the input hidden, so it doesn't
end up in your shell history.

**macOS / Linux**

```bash
curl -fsSL https://raw.githubusercontent.com/imrun6am/claude-monitor/main/install.sh | sudo bash -s -- https://your-pm-tool.example.com
```

**Windows** (PowerShell as Administrator)

```powershell
& ([scriptblock]::Create((irm https://raw.githubusercontent.com/imrun6am/claude-monitor/main/install.ps1))) -Url https://your-pm-tool.example.com
```

Or clone the repo and run `sudo ./install.sh https://your-pm-tool.example.com`
(`.\install.ps1 -Url …` on Windows).

The installer:

1. writes the hook and Claude Code's **managed settings** to
   `/Library/Application Support/ClaudeCode/` (macOS), `/etc/claude-code/` (Linux)
   or `C:\Program Files\ClaudeCode\` (Windows). Only an administrator can change
   these, and a user's own settings can't turn managed hooks off;
2. gives the device a random id that survives reinstalls;
3. hides the Claude desktop app's Chat tab (see below; `--keep-chat` leaves it);
4. sends a test event and tells you if the server didn't accept it.

Restart open Claude Code sessions afterwards. Running it again (say, with a new
URL or token) keeps the same device id.

## The desktop app's Chat tab

No hook runs when someone chats in the Claude desktop app's Chat tab, so those
chats can't be recorded. The installer therefore hides that tab, and the app
opens on its Code tab, which is recorded. It sets the app's policy settings
`desktopHome` = `off` and `chatTabEnabled` = false (the older name, ignored from
December 17, 2026):

- macOS: `/Library/Preferences/com.anthropic.claudefordesktop.plist`
- Windows: `HKLM\SOFTWARE\Policies\Claude`

Quit and reopen the Claude app afterwards. To leave the Chat tab on, install
with `--keep-chat` (`-KeepChat` on Windows); running it that way later shows the
tab again, and so does uninstalling.

This only steers people to the Code tab. Chat on claude.ai in a browser, in the
mobile app and in the Claude in Chrome extension stays open, and on Team and Pro
plans nothing can turn that off. Anthropic documents these two settings for
managed deployments; check on one Mac that the tab goes away with your
account before rolling it out. Some app versions may only honour them from an
MDM profile (Jamf, Kandji, Intune) in the `com.anthropic.claudefordesktop` domain.

## Uninstall

```bash
sudo ./install.sh --uninstall
```

```powershell
.\install.ps1 -Uninstall
```

## What is sent

The event name, session id, project folder, model and first few hundred
characters of each prompt (from Claude Code), plus the computer name, OS user,
OS, entry point (terminal, desktop app, …) and the MAC address of the network
adapter the device's default route uses. On a Mac the computer name is the one
set in System Settings, which doesn't change with the network, and the MAC is
the hardware address, not a private Wi-Fi one. The server adds the IP address
it sees.

When Claude Code drives Chrome through Claude in Chrome, each browser action is
sent too: the tool and action (`navigate`, `computer: left_click`, …) and the
page it goes to. The PM tool keeps the page address without its query string,
and nothing typed or run in the page.

Claude Code names the model when a session starts and when you switch models
(`/model`); the server gives each prompt the latest model its session
reported. The MAC address is reported by the device, so unlike the IP address
it could be faked.

The hook never holds up Claude Code: on macOS and Linux it posts in the
background, and on Windows it hands the event to a hidden copy of itself.

## Limits

- Covers Claude Code only: the terminal, the desktop app's Code tab and Remote
  Control sessions on a device that has it installed. claude.ai chat in a
  browser or the mobile app isn't covered.
- Claude in Chrome shows up only when Claude Code drives Chrome through it.
  Chatting with the extension directly in Chrome doesn't run Claude Code's
  hooks, so it isn't recorded; your Claude account's admin tools are the place
  to look for that.
- A device without it reports nothing. Usage going up with no matching events
  points to such a device.
- An administrator on the device can uninstall it; what was already sent stays.
- Events sent while the device is offline are lost.
- The token is the same on every device and sits in the hook, which every user
  on the device can read (the hook runs as them). Anyone holding it can post
  events with any device name, so the IP address is the one field they can't
  fake. If a device leaks it, set a new `CLAUDE_MONITOR_TOKEN` on the server
  and run the installer again on every device.
- If `managed-settings.json` already exists for another reason, the installer
  stops; merge `managed-settings.example.json` into it by hand.
