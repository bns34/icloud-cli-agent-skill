---
name: icloud-cli
category: productivity
description: Full iCloud CLI tool — Calendar, Reminders, Notes, Find My, and daemon sync. Uses pyicloud web API + IMAP for Notes.
---

# iCloud CLI (icloud-cli-tools)

Access iCloud Calendar, Reminders, Notes, and Find My from Linux via [`icloud-cli-tools`](https://github.com/alan13367/icloud-cli-tools) (pure Python, uses `pyicloud` web API).

## Setup after installation

```bash
# Binary path
~/.venvs/icloud-cli/bin/icloud-cli

# Activation
source ~/.venvs/icloud-cli/bin/activate
```

**Login** (interactive — needs main iCloud password + 2FA):
```bash
~/.venvs/icloud-cli/bin/icloud-cli login
```

Credentials stored in OS keyring; session cookies cached to `~/.config/icloud-cli/session/`.

## Command Reference

### Authentication

```bash
icloud-cli login          # Interactive login with 2FA
icloud-cli logout         # Clear all stored credentials
icloud-cli status         # Check auth status (apple_id, password, session)
```

### Calendar (`📅 Manage iCloud Calendar events`)

```bash
icloud-cli calendar list                    # Events for next 7 days
icloud-cli calendar list --from today --to tomorrow
icloud-cli calendar show <event-id>         # Details of specific event
icloud-cli calendar add -t "Title" -s "2025-06-15 10:00" -e "2025-06-15 11:00"
icloud-cli calendar add -t "Title" -s "..." -e "..." -c "CalendarName" -l "Location" -n "Notes"
icloud-cli calendar delete <event-id>
```

### Reminders (`✅ Manage iCloud Reminders`)

```bash
icloud-cli reminders list                   # Active reminders (all lists)
icloud-cli reminders list --list "Shopping"  # Filter by list name
icloud-cli reminders list --completed        # Include completed reminders
icloud-cli reminders add -t "Buy milk" -d "2025-06-15" -l "Shopping"
icloud-cli reminders add -t "Task" -d "2025-06-15 14:00" --description "Details"
icloud-cli reminders complete <reminder-id>
icloud-cli reminders delete <reminder-id>

# Reminder list management
icloud-cli reminders lists list              # List all reminder lists
icloud-cli reminders lists create "Name" --color blue

# Reminder sections (groups within a list)
icloud-cli reminders sections create "ListName" "SectionName"
```

**⚠️ Performance Note**: First reminders query is **very slow** (2+ minutes) due to pyicloud initiating a CloudKit sync through Apple's API. Subsequent calls are faster once the sync cursor is cached. This is a pyicloud limitation, not a bug in the tool.

### Notes (`📝 Manage iCloud Notes`)

Notes access uses **IMAP** and requires an **app-specific password** generated at [appleid.apple.com](https://appleid.apple.com/account/manage) → *Sign In & Security* → *App-Specific Passwords*.

```bash
icloud-cli notes setup-imap                 # Set up app-specific password (interactive)
icloud-cli notes list                       # List all notes
icloud-cli notes list --folder "FolderName"  # Filter by folder
icloud-cli notes show <note-id>             # View full note content
icloud-cli notes add -t "Title" -b "Body text"
icloud-cli notes add -t "Title" -b "Body" --folder "FolderName"
icloud-cli notes search "keyword"           # Search notes
```

### Find My (`📍 Find My devices`)

```bash
icloud-cli findmy list                      # All devices with name, model, battery, status, location
icloud-cli findmy locate "iPhone"           # GPS coordinates + Google Maps link
icloud-cli findmy play-sound "iPhone"       # Ring your device (write operation!)
icloud-cli findmy lost-mode "iPhone" -p "+1234567890" -m "Please return"  # Lost Mode (write operation!)
```

### Sync & Daemon (`🔄 Background sync daemon`)

```bash
icloud-cli sync                # One-shot sync to local cache
icloud-cli daemon start        # Start background sync (every 15 min)
icloud-cli daemon stop         # Stop daemon
icloud-cli daemon status       # Check daemon status
```

### Output Formats (all commands)

```bash
icloud-cli calendar list -f table   # Rich formatted table (default)
icloud-cli calendar list -f json    # Machine-readable JSON
icloud-cli calendar list -f plain   # Tab-separated for scripting
```

## Configuration

Path: `~/.config/icloud-cli/config.toml`

```toml
[general]
default_format = "table"
verbose = false

[auth]
apple_id = "your@icloud.com"

[sync]
sync_interval_minutes = 15

[calendar]
default_calendar = "Personal"

[reminders]
default_reminder_list = "Reminders"
```

## Bug — `findmy list` fails with AttributeError

**Symptom**: `icloud-cli findmy list` crashes with `AttributeError: 'FindMyiPhoneServiceManager' object has no attribute 'values'`

**Root cause**: pyicloud's `FindMyiPhoneServiceManager.__iter__` yields AppleDevice objects directly, but the installed `icloud_cli/services/findmy.py` calls `devices.values()` which doesn't exist on the manager object. The `FindMyiPhoneServiceManager` has `__iter__` and `__getitem__` (dict-style access by name/index) but not `.values()`.

**Fix applied**: Replaced all three `for device in devices.values():` with `for device in devices:` in `~/.venvs/icloud-cli/lib/python3.11/site-packages/icloud_cli/services/findmy.py`:
- Line 33: `list_devices()` method
- Line 160: `_find_device()` exact match loop
- Line 166: `_find_device()` partial match loop

## Pitfalls

- **Main iCloud password required** — app-specific passwords (used for CalDAV/IMAP) don't work with `pyicloud`'s web API. The first login needs the real iCloud password + 2FA.
- **Login is interactive** — `icloud-cli login` uses `click.prompt` which needs a real TTY. Run it from a terminal, not via an agent background PTY.
- **Session caching** — after first successful login with 2FA, the session cookies are cached so subsequent CLI calls work without re-authenticating (until session expires).
- **Reminders cold-start latency** — first `icloud-cli reminders list` call after a session starts can take 2+ minutes due to pyicloud CloudKit sync fetching all alarm triggers/records. Subsequent calls are faster. Use `timeout 180` if running via script.
- **FindMy `.values()` bug** — the installed 0.1.2 version of icloud-cli-tools has a bug in `findmy.py` iterating over `FindMyiPhoneServiceManager` with `.values()`. Fix applied above.
- **Notes need app-specific password** — run `icloud-cli notes setup-imap` after logging in. Generate the password at appleid.apple.com.
- **Calendar events are not filtered by calendar** — `icloud-cli calendar list` shows events from all iCloud calendars. Filter by calendar name in the output's "Calendar" column.
