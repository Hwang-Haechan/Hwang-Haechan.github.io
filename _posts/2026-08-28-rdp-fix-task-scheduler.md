---
title: "Scheduling an RDP Fix Script with Windows Task Scheduler: Setup, Verification, and a `pause` Gotcha"
date: 2026-08-28 00:00:00 +0900
categories: [Windows, Sysadmin]
tags: [rdp, task-scheduler, schtasks, batch-script, windows-server]
---

This post walks through automating a Remote Desktop (RDP) troubleshooting batch script with Windows Task Scheduler — how to register it, how to check on it (both GUI and CLI), how to read its result codes, and a classic beginner trap involving the `pause` command that makes a scheduled task look like it's stuck forever.

## 1. The script being scheduled

The starting point was a `.bat` file meant to fix RDP error `0x904` / extended error `0x7` on a Windows Server target machine. It does four things, in order:

1. Re-enables the "Remote Desktop" firewall rule group
2. Restarts the `TermService` (Remote Desktop Services) — this also forces Windows to discard any corrupted/expired self-signed RDP certificate and generate a fresh one
3. Ensures RDP is enabled via the `fDenyTSConnections` registry value
4. Restarts `UmRdpService` (the RD listener helper service)

At the very end, the script has:

```bat
echo.
pause
```

This detail turns out to matter a lot later.

## 2. Registering the task: GUI vs. `schtasks`

There are two equivalent ways to schedule the script to run daily at 3:00 AM.

### GUI method (`taskschd.msc`)

1. `Win + R` → `taskschd.msc`
2. **Action → Create Task...** (not "Create Basic Task" — it has fewer options)
3. **General tab**: check *"Run whether user is logged on or not"* and *"Run with highest privileges"*
4. **Triggers tab**: New → *On a schedule* → *Daily* → Start time `3:00:00 AM`
5. **Actions tab**: New → *Start a program* → point it at the `.bat` file's full path
6. Save, entering admin credentials when prompted

### CLI method (`schtasks`)

```cmd
schtasks /Create /TN "RDP_Fix_Daily" /TR "C:\Scripts\fix_rdp.bat" /SC DAILY /ST 03:00 /RU SYSTEM /RL HIGHEST /F
```

| Option | Meaning |
|---|---|
| `/TN` | Task Name |
| `/TR` | Task Run — the program/script path |
| `/SC DAILY` | Schedule type = daily |
| `/ST 03:00` | Start Time = 3:00 AM |
| `/RU SYSTEM` | Run as the SYSTEM account — always elevated, no stored password needed |
| `/RL HIGHEST` | Run Level = highest privileges |
| `/F` | Force — overwrite if a task with the same name already exists |

Because `net stop termservice` briefly drops the RDP service, it's worth confirming no one is actively connected around 3 AM before relying on this in production.

## 3. Checking the task

### Via CLI

```cmd
schtasks /Query /TN "RDP_Fix_Daily" /V /FO LIST
```

- `/V` = verbose (full details)
- `/FO LIST` = output format, as a list

⚠️ A common typo here is mistyping `/FO` as something unrelated (e.g. accidentally pasting in an unrelated tool name). `schtasks` doesn't recognize unknown flags and will throw a syntax error like:

```
ERROR: Invalid Syntax. Value expected for '/FO'.
```

The flag must be exactly `/FO` followed by `LIST`, `TABLE`, or `CSV`.

### Via GUI

1. `taskschd.msc` → click **"Task Scheduler Library"** itself (not a subfolder) in the left tree
2. The task list appears in the center pane
3. Select the task to see its **General / Triggers / Actions / Conditions / Settings / History** tabs
4. The right-hand action pane lets you Run / End / Disable / Delete / view Properties directly

Note: built-in folders like `Microsoft`, `MySQL`, `GoogleSystem`, etc. are created by other software and are unrelated — a custom task registered without a folder path shows up directly under the Library root.

### Running it on demand

```cmd
schtasks /Run /TN "RDP_Fix_Daily"
```

This triggers an immediate run, independent of the schedule — useful for testing. Because it runs in the background/non-interactively, anything in the script waiting on user input has no one to answer it (see Section 5).

## 4. Reading the "Last Result" code

`schtasks /Query` reports a **Last Result** field. These aren't always plain Win32 error codes — Task Scheduler has its own status codes, several of which just mean "still working," not "failed."

| Decimal | Hex | Meaning |
|---|---|---|
| `0` | `0x0` | Completed successfully |
| `267008` | `0x41300` | Ready (hasn't run yet / idle) |
| `267009` | `0x41301` | **Currently running** |
| `267010` | `0x41302` | Disabled |
| `267011` | `0x41303` | Has never been run |

Seeing `267009` right after triggering a run is completely normal — it just means the task is still in progress.

## 5. The trap: a task that never finishes

In this case, `267009` was still showing **two minutes** after the run started — well past the ~10 seconds this kind of script should take.

### Ruling out the obvious suspect

```cmd
sc query termservice
```
returned `STATE: 4 RUNNING`, `WIN32_EXIT_CODE: 0` — meaning the service had already restarted cleanly. So the actual RDP-related work in the script had already finished.

```cmd
query session
```
showed the current session as an active RDP connection (`rdp-tcp#1`, Administrator), plus several stale disconnected sessions — useful context, but not the cause of the hang.

### The real cause: `pause`

The script's final line, `pause`, waits for a keypress before continuing. That's harmless when someone double-clicks the `.bat` file interactively — a console window opens, they see *"Press any key to continue..."*, and pressing a key ends it.

But Task Scheduler runs scripts **non-interactively**, in a background session with no visible console and no one to press a key. `pause` blocks forever, so the batch process — and therefore the scheduled task — never actually exits, even though every real command in the script already completed successfully.

### Fix

**Immediate**: force-stop the hung run —
```cmd
schtasks /End /TN "RDP_Fix_Daily"
```

**Permanent**: open the `.bat` file and delete the trailing `pause` line. The informational `echo` lines above it can stay; only `pause` itself needs to go. No task re-registration is needed — Task Scheduler already points at that file path, so the fix takes effect on the very next run.

## 6. Locating a file's full path

Since Task Scheduler needs an exact path, a few quick ways to find one:

- **File Explorer**: search for the filename → right-click → *Properties* → the **Location** field is the folder path (the file name isn't included)
  - `Location: D:\` means the file sits at the drive root → full path is `D:\filename.bat`
  - `Location: D:\Scripts` → full path is `D:\Scripts\filename.bat`
  - Rule of thumb: **Location + `\` + filename = full path**
- **CLI search**: `dir C:\filename.bat /s` (`/s` recurses into subfolders; swap the drive letter if needed)
- **PowerShell**: `Get-ChildItem -Path C:\ -Filter "filename.bat" -Recurse -ErrorAction SilentlyContinue`

If the task is already registered, the verbose `schtasks /Query` output's **"Task To Run"** field also shows the exact path currently in use.

## TL;DR

- Register daily scheduled tasks either via `taskschd.msc` (GUI) or `schtasks /Create ... /SC DAILY /ST 03:00 /RU SYSTEM /RL HIGHEST` (CLI)
- Check status with `schtasks /Query /TN "<name>" /V /FO LIST` — mistyping the `/FO` flag causes a syntax error, not a real query
- Task Scheduler's "Last Result" codes aren't always errors: `267009` (`0x41301`) simply means *"still running"*
- If a task stays "running" far longer than expected, check the actual work first (service state, sessions) before assuming failure
- **`pause` in a batch script will hang forever under Task Scheduler**, because there's no interactive console to press a key in — always remove `pause` (and similar blocking prompts) from scripts meant to run unattended
- A file's full path = its Explorer "Location" property + `\` + the filename
