---
title: Herdr Pi agent detection on Windows, and recovering sessions without a restart
period: 2026-09
status: resolved and verified
applies_to: Herdr 0.8.2 on Windows, Pi coding agent installed through npm
---

# Herdr Pi agent detection on Windows, and recovering sessions without a restart

**Symptom.** Two Pi coding agent sessions were running inside Herdr tabs but never appeared in the sidebar's Agents tab. `herdr agent list` returned only the OpenCode panes. `herdr pane get <pane_id>` on a Pi pane returned:

```json
{
  "agent_status": "unknown",
  "pane_id": "w3:p7",
  "terminal_title": "π - <workspace>"
}
```

The `"agent"` key was absent entirely.

**Status.** Root cause found. Both live sessions were restored without restarting them, and future `pi` launches are detected. Verified in `herdr agent list` and the sidebar.

## How Herdr detects agents

Herdr is a Rust application compiled as `herdr.exe`. Its pane module (`src/pane.rs`, `src/platform/windows.rs`) polls the Windows process tree:

1. `CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)` followed by `Process32FirstW` and `Process32NextW`.
2. For each pane, it starts at the pane's root shell PID (`shell_pid`, for example `powershell.exe`) and walks descendants by `th32ParentProcessID`.
3. It reads `PROCESSENTRY32W.szExeFile`, strips `.exe`, lowercases it, and matches it against a hard-coded agent table:

```rust
// Reverse-engineered from herdr.exe 0.8.2 at file offset 0x231af5
match exe_name_lower {
    "pi" => Some(Agent::Pi),
    "opencode" => Some(Agent::OpenCode),
    "claude" => Some(Agent::Claude),
    "cursor" => Some(Agent::Cursor),
    "devin" => Some(Agent::Devin),
    "gemini" => Some(Agent::Gemini),
    _ => None,
}
```

4. On a match it logs `agent changed pane=... previous_agent=None agent=Some(Pi) process=pi` and sets `pane.agent`.
5. The sidebar only renders panes where `pane.agent.is_some()`.

Herdr also accepts lifecycle reports over its named pipe from the Pi extension (`pane.report_agent`, `pane.report_agent_session`). It ignores those reports for panes whose process scanner has not verified an agent executable. IPC alone does not establish agent presence.

## Why Pi misses

Claude Code and OpenCode ship native executables (`claude.exe`, `opencode.exe`), so the process in the pane matches.

Pi (`@earendil-works/pi-coding-agent`) is pure JavaScript. npm on Windows installs three shims:

- `pi` for MSYS/Cygwin
- `pi.cmd` for cmd
- `pi.ps1` for PowerShell

None is an `.exe`. Launching `pi` runs `node.exe` with `cli.js`. Herdr sees:

```text
powershell.exe -> node.exe
```

`node.exe` is generic, so the matcher returns `None`.

## The extension path bug

The Pi-side extension `herdr-agent-state.ts` also had a Windows path check:

```ts
currentAgentSessionPath =
  typeof file === "string" && file.startsWith("/") ? file : undefined;
```

Windows session paths start with a drive letter (`C:\...`) or a UNC prefix (`\\...`), never `/`. The value stayed `undefined`, so session correlation never worked even after detection was restored.

Fixed:

```ts
currentAgentSessionPath =
  typeof file === "string" &&
  (file.startsWith("/") || /^[a-zA-Z]:[\\/]/.test(file) || file.startsWith("\\\\"))
    ? file
    : undefined;
```

## Fix 1: a native launcher named `pi.exe`

A small C# console executable wraps `node.exe` and `cli.js`. It must:

1. Find `node.exe` on `PATH` or in the standard install location.
2. Find `@earendil-works/pi-coding-agent/dist/bundle/cli.js`.
3. Use `UseShellExecute = false` without redirecting I/O, so the child shares the host console buffer and raw TTY mode works.
4. Trap `Console.CancelKeyPress` with `e.Cancel = true`, so Ctrl+C does not kill the wrapper and Node handles turn cancellation.
5. Support `--watch-pid <PID>` as a companion watcher for sessions that are already running.

```csharp
using System;
using System.Diagnostics;
using System.IO;

class Program {
    static int Main(string[] args) {
        // Mode 1: companion process watching an existing running pi PID
        if (args.Length >= 2 && args[0] == "--watch-pid") {
            int targetPid;
            if (int.TryParse(args[1], out targetPid)) {
                try {
                    Process target = Process.GetProcessById(targetPid);
                    target.WaitForExit();
                } catch {
                    // Target process already exited
                }
            }
            return 0;
        }

        // Mode 2: interactive wrapper launching node.exe with pi cli.js
        string nodePath = FindNode();
        if (string.IsNullOrEmpty(nodePath)) {
            Console.Error.WriteLine("Error: node.exe not found in PATH or standard install locations.");
            return 1;
        }

        string cliPath = FindCli();
        if (string.IsNullOrEmpty(cliPath)) {
            Console.Error.WriteLine("Error: @earendil-works/pi-coding-agent cli.js not found.");
            return 1;
        }

        string cmdArgs = "\"" + cliPath + "\"";
        foreach (string arg in args) {
            cmdArgs += " " + Quote(arg);
        }

        ProcessStartInfo psi = new ProcessStartInfo();
        psi.FileName = nodePath;
        psi.Arguments = cmdArgs;
        psi.UseShellExecute = false;

        // Prevent wrapper from terminating on Ctrl+C so child Node process can handle turn interrupts
        Console.CancelKeyPress += (sender, e) => {
            e.Cancel = true;
        };

        try {
            using (Process p = Process.Start(psi)) {
                p.WaitForExit();
                return p.ExitCode;
            }
        } catch (Exception ex) {
            Console.Error.WriteLine("Failed to launch pi: " + ex.Message);
            return 1;
        }
    }

    static string Quote(string arg) {
        if (string.IsNullOrEmpty(arg)) return "\"\"";
        if (arg.IndexOf(' ') >= 0 || arg.IndexOf('"') >= 0 || arg.IndexOf('\t') >= 0) {
            return "\"" + arg.Replace("\"", "\\\"") + "\"";
        }
        return arg;
    }

    static string FindNode() {
        string standard = @"C:\Program Files\nodejs\node.exe";
        if (File.Exists(standard)) return standard;

        string pathEnv = Environment.GetEnvironmentVariable("PATH") ?? "";
        string[] dirs = pathEnv.Split(';');
        foreach (string dir in dirs) {
            string trimmed = dir.Trim();
            if (trimmed.Length == 0) continue;
            try {
                string candidate = Path.Combine(trimmed, "node.exe");
                if (File.Exists(candidate)) return candidate;
            } catch { }
        }
        return null;
    }

    static string FindCli() {
        string appData = Environment.GetFolderPath(Environment.SpecialFolder.ApplicationData);
        string defaultPath = Path.Combine(appData, @"npm\node_modules\@earendil-works\pi-coding-agent\dist\bundle\cli.js");
        if (File.Exists(defaultPath)) return defaultPath;

        string userProfile = Environment.GetFolderPath(Environment.SpecialFolder.UserProfile);
        string altPath = Path.Combine(userProfile, @"AppData\Roaming\npm\node_modules\@earendil-works\pi-coding-agent\dist\bundle\cli.js");
        if (File.Exists(altPath)) return altPath;

        return null;
    }
}
```

Compile with the .NET Framework compiler that ships with Windows. No Visual Studio needed:

```powershell
$csc = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe"
& $csc -out:"$env:USERPROFILE\bin\pi.exe" "$env:TEMP\pi_launcher.cs"

Copy-Item "$env:USERPROFILE\bin\pi.exe" -Destination "$env:APPDATA\npm\pi.exe" -Force
```

Then make the npm shims delegate to it, so even a direct `pi.cmd` or `pi.ps1` invocation lands on the wrapper:

```cmd
@ECHO off
IF EXIST "%~dp0pi.exe" (
  "%~dp0pi.exe" %*
  EXIT /b %ERRORLEVEL%
)
GOTO start
```

```powershell
#!/usr/bin/env pwsh
$basedir=Split-Path $MyInvocation.MyCommand.Definition -Parent

if (Test-Path "$basedir/pi.exe") {
  if ($MyInvocation.ExpectingInput) {
    $input | & "$basedir/pi.exe" $args
  } else {
    & "$basedir/pi.exe" $args
  }
  exit $LASTEXITCODE
}
```

## Fix 2: attach a watcher to a live session without restarting it

Win32 `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` lets a process be created with a chosen parent rather than the caller. Spawn `pi.exe --watch-pid <node_pid>` directly under the pane's shell PID. Herdr then sees a `pi.exe` descendant and upgrades the pane, while the running Node process and its work are untouched.

```csharp
using System;
using System.Diagnostics;
using System.Runtime.InteropServices;

class Program {
    const uint EXTENDED_STARTUPINFO_PRESENT = 0x00080000;
    const uint CREATE_NO_WINDOW = 0x08000000;
    const int PROC_THREAD_ATTRIBUTE_PARENT_PROCESS = 0x00020000;
    const uint PROCESS_ALL_ACCESS = 0x1F0FFF;

    [StructLayout(LayoutKind.Sequential)]
    struct PROCESS_INFORMATION {
        public IntPtr hProcess;
        public IntPtr hThread;
        public uint dwProcessId;
        public uint dwThreadId;
    }

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct STARTUPINFOEX {
        public STARTUPINFO StartupInfo;
        public IntPtr lpAttributeList;
    }

    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct STARTUPINFO {
        public uint cb;
        public string lpReserved;
        public string lpDesktop;
        public string lpTitle;
        public uint dwX;
        public uint dwY;
        public uint dwXSize;
        public uint dwYSize;
        public uint dwXCountChars;
        public uint dwYCountChars;
        public uint dwFillAttribute;
        public uint dwFlags;
        public short wShowWindow;
        public short cbReserved2;
        public IntPtr lpReserved2;
        public IntPtr hStdInput;
        public IntPtr hStdOutput;
        public IntPtr hStdError;
    }

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern IntPtr OpenProcess(uint processAccess, bool bInheritHandle, int processId);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool InitializeProcThreadAttributeList(IntPtr lpAttributeList, int dwAttributeCount, int dwFlags, ref IntPtr lpSize);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool UpdateProcThreadAttribute(IntPtr lpAttributeList, uint dwFlags, IntPtr Attribute, IntPtr lpValue, IntPtr cbSize, IntPtr lpPreviousValue, IntPtr lpReturnSize);

    [DllImport("kernel32.dll", SetLastError = true, CharSet = CharSet.Unicode)]
    static extern bool CreateProcess(
        string lpApplicationName,
        string lpCommandLine,
        IntPtr lpProcessAttributes,
        IntPtr lpThreadAttributes,
        bool bInheritHandles,
        uint dwCreationFlags,
        IntPtr lpEnvironment,
        string lpCurrentDirectory,
        ref STARTUPINFOEX lpStartupInfo,
        out PROCESS_INFORMATION lpProcessInformation);

    [DllImport("kernel32.dll", SetLastError = true)]
    static extern bool CloseHandle(IntPtr hObject);

    static int Main(string[] args) {
        if (args.Length < 2) {
            Console.WriteLine("Usage: herdr-reparent <parent_pid> <exe_path> [cmdline]");
            return 1;
        }

        int parentPid = int.Parse(args[0]);
        string exePath = args[1];
        string cmdLine = args.Length > 2 ? args[2] : ("\"" + exePath + "\"");

        IntPtr hParent = OpenProcess(PROCESS_ALL_ACCESS, false, parentPid);
        if (hParent == IntPtr.Zero) {
            Console.WriteLine("OpenProcess failed: " + Marshal.GetLastWin32Error());
            return 1;
        }

        IntPtr lpSize = IntPtr.Zero;
        InitializeProcThreadAttributeList(IntPtr.Zero, 1, 0, ref lpSize);
        IntPtr lpAttributeList = Marshal.AllocHGlobal(lpSize);
        InitializeProcThreadAttributeList(lpAttributeList, 1, 0, ref lpSize);

        IntPtr lpValue = Marshal.AllocHGlobal(IntPtr.Size);
        Marshal.WriteIntPtr(lpValue, hParent);

        UpdateProcThreadAttribute(
            lpAttributeList,
            0,
            (IntPtr)PROC_THREAD_ATTRIBUTE_PARENT_PROCESS,
            lpValue,
            (IntPtr)IntPtr.Size,
            IntPtr.Zero,
            IntPtr.Zero);

        STARTUPINFOEX siex = new STARTUPINFOEX();
        siex.StartupInfo.cb = (uint)Marshal.SizeOf(typeof(STARTUPINFOEX));
        siex.lpAttributeList = lpAttributeList;

        PROCESS_INFORMATION pi = new PROCESS_INFORMATION();
        bool success = CreateProcess(
            exePath,
            cmdLine,
            IntPtr.Zero,
            IntPtr.Zero,
            false,
            EXTENDED_STARTUPINFO_PRESENT | CREATE_NO_WINDOW,
            IntPtr.Zero,
            null,
            ref siex,
            out pi);

        if (!success) {
            Console.WriteLine("CreateProcess failed: " + Marshal.GetLastWin32Error());
            return 1;
        }

        Console.WriteLine("Spawned PID: " + pi.dwProcessId + " with Parent PID: " + parentPid);
        CloseHandle(pi.hProcess);
        CloseHandle(pi.hThread);
        CloseHandle(hParent);
        return 0;
    }
}
```

Compile and attach:

```powershell
$csc = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe"
& $csc -out:"$env:USERPROFILE\bin\herdr-reparent.exe" "$env:TEMP\herdr_reparent.cs"

# <shell_pid> is the pane's root shell, <node_pid> the already-running pi Node process
& "$env:USERPROFILE\bin\herdr-reparent.exe" <shell_pid> "$env:USERPROFILE\bin\pi.exe" `
  "`"$env:USERPROFILE\bin\pi.exe`" --watch-pid <node_pid>"
```

## Verification

```powershell
& "$env:LOCALAPPDATA\Programs\Herdr\bin\herdr.exe" agent list
```

Expected: the Pi panes appear with `"agent": "pi"` and `"agent_status": "idle"`.

`%APPDATA%\herdr\herdr-server.log` shows the transition:

```text
INFO herdr::pane: agent changed pane=16 previous_agent=None agent=Some(Pi) process=pi pgid=Some(<pid>)
INFO herdr::pane: agent changed pane=17 previous_agent=None agent=Some(Pi) process=pi pgid=Some(<pid>)
```

New launches should resolve to the wrapper:

```powershell
(Get-Command pi).Source
# %USERPROFILE%\bin\pi.exe
```

## Evidence

- `herdr pane process-info --pane w3:p7` showed `powershell.exe -> node.exe` and no agent match.
- Both live panes were restored by spawning `--watch-pid` companions under the pane shell PIDs with the reparenter. The server log recorded `agent changed ... agent=Some(Pi) process=pi` for both, and `herdr agent list` showed both with `agent: "pi"` and `agent_status: "idle"`.
- The launcher and reparenter were compiled with `csc.exe` and placed on `PATH`; the npm `pi.cmd` and `pi.ps1` shims were updated to delegate to `pi.exe`.
- The extension's `updateSessionRef` was patched for drive-letter and UNC paths.

## What generalizes

- Herdr needs a matching executable in the pane's process tree. IPC lifecycle messages alone do not register an agent.
- npm-installed agents that lack a native binary need a small `.exe` shim for process-scanned launchers. Put it ahead of the npm shims on `PATH`.
- A live process can be given a new parent without being restarted. A lightweight watcher under the pane shell is enough to restore detection while the real session keeps working.
- Never test Windows paths with `.startsWith("/")`. Cover drive letters and UNC paths.
