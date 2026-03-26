# Auto Tailscle for Windows

**Usage:**

1. Go to [admin console → Keys](https://login.tailscale.com/admin/settings/keys) and generate an auth key (reusable + tagged + pre-approved is ideal for servers - tagged devices have key expiry disabled by default, so the `NeedsLogin` state the VBS warns about won't happen).
2. Paste it into `$AUTH_KEY` at the top.
3. Optionally set `$MONITOR_PEER_IP` to a Tailscale IP you want to ping-check, and `$EXTRA_UP_FLAGS` for things like `--ssh` or `--advertise-routes`.
4. Run from an elevated PowerShell:
   ```powershell
   Set-ExecutionPolicy Bypass -Scope Process -Force
   .\Tailscale_Setup.ps1
   ```

## Tailscale Setup Script

The script is idempotent - safe to re-run. It will overwrite the old VBS, stop the old task, and re-register everything cleanly.

```powershell
# ================================================================
#  Tailscale_Setup.ps1
#
#  Installs Tailscale, authenticates with an auth key, configures
#  service auto-recovery, and deploys a VBS health-monitor as a
#  Scheduled Task running under SYSTEM.
#
#  Run once from an elevated PowerShell prompt:
#    Set-ExecutionPolicy Bypass -Scope Process -Force
#    .\Tailscale_Setup.ps1
# ================================================================

# ======================== CONFIGURATION =========================
# Generate a key at  https://login.tailscale.com/admin/settings/keys
# Recommendation: Reusable + Tagged + Pre-approved for servers.
$AUTH_KEY        = "tskey-auth-XXXXXXXXXXXXX"

# Optional: a Tailscale IP to ping-check each cycle (leave "" to skip)
$MONITOR_PEER_IP = ""

# Optional: extra flags for the initial `tailscale up` (space-separated)
# e.g.  "--advertise-routes=10.0.0.0/24 --ssh --accept-routes"
$EXTRA_UP_FLAGS  = ""
# ================================================================

# --- Paths (should not need editing) ---
$TAILSCALE_EXE  = "C:\Program Files\Tailscale\tailscale.exe"
$TAILSCALED_EXE = "C:\Program Files\Tailscale\tailscaled.exe"
$TAILSCALE_IPN  = "C:\Program Files\Tailscale\tailscale-ipn.exe"
$DATA_DIR       = "C:\ProgramData\Tailscale"
$VBS_PATH       = Join-Path $DATA_DIR "Tailscale_Monitor.vbs"
$LOG_FILE       = Join-Path $DATA_DIR "Monitor.log"
$STOP_FILE      = Join-Path $DATA_DIR "Monitor_Stop.flag"
$TASK_NAME      = "Tailscale_Monitor"

# ================================================================
# HELPERS
# ================================================================
function Write-Step([int]$n, [string]$msg) {
    Write-Host "`n[Step $n] $msg" -ForegroundColor Cyan
}
function Write-OK([string]$msg)   { Write-Host "  [OK]  $msg" -ForegroundColor Green }
function Write-Warn([string]$msg) { Write-Host "  [!!]  $msg" -ForegroundColor Yellow }
function Write-Err([string]$msg)  { Write-Host "  [ERR] $msg" -ForegroundColor Red }

# Fix #4: parameter renamed from $Args → $Command
function Invoke-Tailscale {
    param([string[]]$Command)
    & $TAILSCALE_EXE @Command 2>&1
}

# ================================================================
# STEP 1 - Elevation check                              (Fix #7)
# ================================================================
Write-Host ""
Write-Host "================================================================" -ForegroundColor Magenta
Write-Host "  Tailscale Server Setup (auth-key edition)"                      -ForegroundColor Magenta
Write-Host "================================================================" -ForegroundColor Magenta

Write-Step 1 "Checking administrator privileges..."
$isAdmin = ([Security.Principal.WindowsPrincipal] `
    [Security.Principal.WindowsIdentity]::GetCurrent()
).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)

if (-not $isAdmin) {
    Write-Err "This script must be run as Administrator."
    Write-Err "Right-click PowerShell -> 'Run as administrator', then retry."
    exit 1
}
Write-OK "Running elevated."

# ================================================================
# STEP 2 - Validate prerequisites                       (Fix #10)
# ================================================================
Write-Step 2 "Validating prerequisites..."

$allBinaries = (Test-Path $TAILSCALE_EXE) -and
               (Test-Path $TAILSCALED_EXE) -and
               (Test-Path $TAILSCALE_IPN)

if (-not $allBinaries) {
    if (-not (Get-Command winget -ErrorAction SilentlyContinue)) {
        Write-Err "winget is not available and Tailscale is not installed."
        Write-Err "Either install Tailscale manually from https://tailscale.com/download"
        Write-Err "or install winget first (App Installer from the Microsoft Store)."
        exit 1
    }
    Write-OK "winget found."
} else {
    Write-OK "Tailscale binaries already present."
}

if ($AUTH_KEY -match "^tskey-auth-X+$" -or [string]::IsNullOrWhiteSpace($AUTH_KEY)) {
    Write-Err "AUTH_KEY has not been set."
    Write-Err "Generate one at: https://login.tailscale.com/admin/settings/keys"
    Write-Err "Paste it into the `$AUTH_KEY variable at the top of this script."
    exit 1
}
Write-OK "Auth key provided."

# ================================================================
# STEP 3 - Install Tailscale                             (Fix #10)
# ================================================================
Write-Step 3 "Installing Tailscale..."

if ($allBinaries) {
    Write-OK "Already installed - skipping."
} else {
    Write-Warn "Installing via winget..."
    $proc = Start-Process -FilePath "winget" `
        -ArgumentList "install -e --id Tailscale.Tailscale --accept-package-agreements --accept-source-agreements" `
        -Wait -NoNewWindow -PassThru
    if ($proc.ExitCode -ne 0) {
        Write-Err "winget exited with code $($proc.ExitCode)."
        exit 1
    }
    Write-OK "winget install finished."
}

# ================================================================
# STEP 4 - Wait for service readiness                    (Fix #12)
# ================================================================
Write-Step 4 "Waiting for the Tailscale service (up to 3 min)..."

# Attempt to start the service if it exists but is stopped
$svc = Get-Service -Name "Tailscale" -ErrorAction SilentlyContinue
if ($svc -and $svc.Status -ne "Running") {
    Write-Warn "Service exists but is not running - starting..."
    Start-Service -Name "Tailscale" -ErrorAction SilentlyContinue
}

$maxWait = 180; $elapsed = 0; $ready = $false
while ($elapsed -lt $maxWait) {
    Start-Sleep -Seconds 5; $elapsed += 5
    $exeOK = Test-Path $TAILSCALE_EXE
    $svc   = Get-Service -Name "Tailscale" -ErrorAction SilentlyContinue
    Write-Host "  ${elapsed}s - exe=$exeOK  service=$($svc.Status)" -ForegroundColor DarkGray
    if ($exeOK -and $svc -and $svc.Status -eq "Running") {
        $ready = $true; break
    }
}
if (-not $ready) {
    Write-Err "Tailscale service did not start within 3 minutes. Reboot and re-run."
    exit 1
}
Write-OK "Service is running."

# ================================================================
# STEP 5 - Authenticate with auth key           (Fixes #1, #5, #12)
# ================================================================
Write-Step 5 "Connecting with auth key (--unattended)..."

$upArgs = [System.Collections.Generic.List[string]]::new()
$upArgs.Add("up")
$upArgs.Add("--auth-key=$AUTH_KEY")
$upArgs.Add("--unattended")
if ($EXTRA_UP_FLAGS -ne "") {
    $EXTRA_UP_FLAGS -split "\s+" | ForEach-Object { $upArgs.Add($_) }
}

$result = (Invoke-Tailscale -Command $upArgs.ToArray()) | Out-String
if ($LASTEXITCODE -ne 0) {
    Write-Err "tailscale up failed:"
    Write-Err $result.Trim()
    exit 1
}
Write-OK "tailscale up succeeded."

# Poll for Running state instead of a fixed sleep  (Fix #12)
$timeout = 60; $t = 0
while ($t -lt $timeout) {
    try {
        $statusObj = (Invoke-Tailscale -Command @("status","--json")) |
                     Out-String | ConvertFrom-Json
        if ($statusObj.BackendState -eq "Running") { break }
    } catch { }
    Start-Sleep -Seconds 3; $t += 3
}
if ($t -ge $timeout) {
    Write-Warn "Backend did not reach 'Running' within ${timeout}s - continuing."
} else {
    Write-OK "BackendState = Running."
}

# ================================================================
# STEP 6 - Configure service auto-recovery              (Fix #3)
# ================================================================
Write-Step 6 "Configuring service failure recovery..."
sc.exe failure Tailscale reset= 0 actions= restart/5000/restart/10000/restart/30000 | Out-Null
Write-OK "Auto-restart on failure: 5 s / 10 s / 30 s."

# ================================================================
# STEP 7 - Write VBS health monitor          (Fixes #6,8,9,11,13,15)
# ================================================================
Write-Step 7 "Creating VBS monitor at $VBS_PATH..."

if (-not (Test-Path $DATA_DIR)) {
    New-Item -ItemType Directory -Path $DATA_DIR -Force | Out-Null
}

$vbsContent = @"
' ================================================================
'  Tailscale_Monitor.vbs     (auto-generated $(Get-Date -Format 'yyyy-MM-dd HH:mm'))
'  Runs as SYSTEM via Scheduled Task.
'  Checks Tailscale health every 60 s, logs to:
'    $LOG_FILE
'  To stop gracefully, create the file:
'    $STOP_FILE
' ================================================================
Option Explicit

Const TAILSCALE_EXE = "C:\Program Files\Tailscale\tailscale.exe"
Const LOG_FILE      = "$LOG_FILE"
Const STOP_FILE     = "$STOP_FILE"
Const MONITOR_PEER  = "$MONITOR_PEER_IP"
Const INTERVAL_MS   = 60000
Const MAX_LOG_BYTES = 5242880  ' 5 MB

Dim objShell, fso
Set objShell = CreateObject("WScript.Shell")
Set fso      = CreateObject("Scripting.FileSystemObject")

' ----------------------------------------------------------------
' WriteLog - append to log file with 5 MB rotation      (Fix #8)
' ----------------------------------------------------------------
Sub WriteLog(msg)
    On Error Resume Next
    Dim logDir, f
    logDir = fso.GetParentFolderName(LOG_FILE)
    If Not fso.FolderExists(logDir) Then fso.CreateFolder logDir

    If fso.FileExists(LOG_FILE) Then
        If fso.GetFile(LOG_FILE).Size > MAX_LOG_BYTES Then
            If fso.FileExists(LOG_FILE & ".old") Then
                fso.DeleteFile LOG_FILE & ".old", True
            End If
            fso.MoveFile LOG_FILE, LOG_FILE & ".old"
        End If
    End If

    Set f = fso.OpenTextFile(LOG_FILE, 8, True)
    f.WriteLine Now & " | " & msg
    f.Close
    On Error GoTo 0
End Sub

' ----------------------------------------------------------------
' RunAndCapture - run a command, return stdout           (Fix #13)
' ----------------------------------------------------------------
Function RunAndCapture(cmd)
    Dim tmpFile, ts, content
    tmpFile = fso.BuildPath(fso.GetSpecialFolder(2), fso.GetTempName())
    objShell.Run "cmd /c " & cmd & " > """ & tmpFile & """ 2>&1", 0, True
    content = ""
    On Error Resume Next
    If fso.FileExists(tmpFile) Then
        Set ts = fso.OpenTextFile(tmpFile, 1)
        If Not (ts Is Nothing) Then
            content = ts.ReadAll()
            ts.Close
        End If
        fso.DeleteFile tmpFile, True
    End If
    On Error GoTo 0
    RunAndCapture = content
End Function

' ----------------------------------------------------------------
' GetBackendState - parse JSON for BackendState          (Fix #9, #11)
' ----------------------------------------------------------------
Function GetBackendState()
    Dim output, regex, matches
    output = RunAndCapture("""" & TAILSCALE_EXE & """ status --json")
    Set regex = CreateObject("VBScript.RegExp")
    regex.Pattern = """BackendState""\s*:\s*""([^""]+)"""
    regex.IgnoreCase = False
    Set matches = regex.Execute(output)
    If matches.Count > 0 Then
        GetBackendState = matches(0).SubMatches(0)
    Else
        GetBackendState = "Unknown"
    End If
End Function

' ----------------------------------------------------------------
' IsServiceRunning - check the Windows service
' ----------------------------------------------------------------
Function IsServiceRunning()
    Dim output
    output = RunAndCapture("sc query Tailscale")
    IsServiceRunning = (InStr(output, "RUNNING") > 0)
End Function

' ----------------------------------------------------------------
' EnsureConnected - idempotent bring-up                  (Fix #1)
' ----------------------------------------------------------------
Sub EnsureConnected()
    WriteLog "Running: tailscale up --unattended"
    objShell.Run "cmd /c """ & TAILSCALE_EXE & """ up --unattended", 0, True
    WScript.Sleep 5000
End Sub

' ----------------------------------------------------------------
' CanPingPeer - optional peer reachability check
' ----------------------------------------------------------------
Function CanPingPeer()
    If MONITOR_PEER = "" Then
        CanPingPeer = True
        Exit Function
    End If
    Dim output
    output = RunAndCapture("""" & TAILSCALE_EXE & """ ping --c 3 " & MONITOR_PEER)
    CanPingPeer = (InStr(output, "pong") > 0)
End Function

' ================================================================
' MAIN LOOP                                              (Fix #6, #15)
' ================================================================
WriteLog "=== Monitor started ==="
Call EnsureConnected()

Dim state, checkCount
checkCount = 0

Do While True
    ' --- Graceful shutdown ---                         (Fix #15)
    If fso.FileExists(STOP_FILE) Then
        WriteLog "Stop file detected. Exiting gracefully."
        On Error Resume Next
        fso.DeleteFile STOP_FILE, True
        On Error GoTo 0
        WScript.Quit 0
    End If

    checkCount = checkCount + 1

    ' --- Ensure the Windows service is alive ---
    If Not IsServiceRunning() Then
        WriteLog "Tailscale service is not running. Attempting net start..."
        objShell.Run "cmd /c net start Tailscale", 0, True
        WScript.Sleep 15000
    End If

    ' --- Check backend state via JSON ---              (Fix #9)
    state = GetBackendState()

    If state = "Running" Then
        ' Healthy - optionally verify peer reachability
        If MONITOR_PEER <> "" Then
            If Not CanPingPeer() Then
                WriteLog "Ping to " & MONITOR_PEER & " failed. Re-running tailscale up..."
                Call EnsureConnected()
            End If
        End If
        ' Hourly heartbeat (every 60 checks at 60 s each)
        If checkCount Mod 60 = 0 Then
            WriteLog "Heartbeat: state=Running, checks=" & checkCount
        End If

    ElseIf state = "NeedsLogin" Then
        WriteLog "WARNING: BackendState=NeedsLogin. Node key may have expired."
        WriteLog "Manual fix required: tailscale up --auth-key=<key> --unattended"

    Else
        ' Stopped, Starting, NoState, Unknown, etc.
        WriteLog "BackendState='" & state & "'. Running tailscale up --unattended..."
        Call EnsureConnected()
        WScript.Sleep 5000
        state = GetBackendState()
        If state <> "Running" Then
            WriteLog "Still not Running (state='" & state & "'). Will retry next cycle."
        Else
            WriteLog "Recovered - BackendState=Running."
        End If
    End If

    WScript.Sleep INTERVAL_MS
Loop
"@

Set-Content -Path $VBS_PATH -Value $vbsContent -Encoding ASCII -Force

if (Test-Path $VBS_PATH) {
    Write-OK "VBS monitor written."
} else {
    Write-Err "Failed to write $VBS_PATH"
    exit 1
}

# ================================================================
# STEP 8 - Register Scheduled Task as SYSTEM             (Fix #2)
# ================================================================
Write-Step 8 "Registering Scheduled Task '$TASK_NAME'..."

# Clean up legacy Startup-folder VBS from the old script
$legacyVBS = "$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\Tailscale_Startup.vbs"
if (Test-Path $legacyVBS) {
    Remove-Item $legacyVBS -Force -ErrorAction SilentlyContinue
    Write-Warn "Removed old Startup-folder VBS: $legacyVBS"
}

# Stop existing task to avoid duplicate processes
$existing = Get-ScheduledTask -TaskName $TASK_NAME -ErrorAction SilentlyContinue
if ($existing -and $existing.State -eq "Running") {
    Stop-ScheduledTask -TaskName $TASK_NAME -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 3
    Write-Warn "Stopped previous instance of the monitor task."
}

$action    = New-ScheduledTaskAction -Execute "wscript.exe" -Argument "`"$VBS_PATH`""
$trigger   = New-ScheduledTaskTrigger -AtStartup
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest
$settings  = New-ScheduledTaskSettingsSet `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries `
    -ExecutionTimeLimit ([TimeSpan]::Zero) `
    -RestartCount 3 `
    -RestartInterval (New-TimeSpan -Minutes 1)

Register-ScheduledTask -TaskName $TASK_NAME `
    -Action $action -Trigger $trigger `
    -Principal $principal -Settings $settings `
    -Description "Monitors Tailscale backend state and reconnects if needed." `
    -Force | Out-Null

Write-OK "Task registered (SYSTEM, AtStartup, no time limit)."

Start-ScheduledTask -TaskName $TASK_NAME
Write-OK "Monitor started."

# ================================================================
# STEP 9 - Verify connectivity                          (Fix #11)
# ================================================================
Write-Step 9 "Verifying Tailscale connectivity..."

try {
    $statusObj = (Invoke-Tailscale -Command @("status","--json")) |
                 Out-String | ConvertFrom-Json
    Write-OK "BackendState = $($statusObj.BackendState)"
} catch {
    Write-Warn "Could not parse tailscale status JSON."
}

$myIP = ((Invoke-Tailscale -Command @("ip","-4")) | Out-String).Trim()
if ($myIP) {
    Write-OK "This device's Tailscale IPv4: $myIP"
} else {
    Write-Warn "Could not retrieve Tailscale IPv4."
}

if ($MONITOR_PEER_IP -ne "") {
    Write-Host "  Pinging $MONITOR_PEER_IP..." -ForegroundColor DarkGray
    $ping = (Invoke-Tailscale -Command @("ping","--c","3",$MONITOR_PEER_IP)) | Out-String
    if ($ping -match "pong") {
        Write-OK "Ping to $MONITOR_PEER_IP: OK"
    } else {
        Write-Warn "Ping to $MONITOR_PEER_IP failed (peer may be offline)."
        Write-Warn "The monitor will keep checking every 60 s."
    }
}

# ================================================================
# DONE
# ================================================================
Write-Host ""
Write-Host "================================================================" -ForegroundColor Magenta
Write-Host "  Setup complete.  Summary:" -ForegroundColor Green
Write-Host ""
Write-Host "  Tailscale:  connected (--unattended)" -ForegroundColor Green
Write-Host "  Service:    auto-restarts on failure" -ForegroundColor Green
Write-Host "  Monitor:    Scheduled Task '$TASK_NAME' (SYSTEM)" -ForegroundColor Green
Write-Host "  Log file:   $LOG_FILE" -ForegroundColor Green
Write-Host ""
Write-Host "  Management commands:" -ForegroundColor Yellow
Write-Host "    Stop monitor:    New-Item '$STOP_FILE'" -ForegroundColor White
Write-Host "    Restart monitor: Start-ScheduledTask '$TASK_NAME'" -ForegroundColor White
Write-Host "    Uninstall task:  Unregister-ScheduledTask '$TASK_NAME' -Confirm:`$false" -ForegroundColor White
Write-Host "    View log:        Get-Content '$LOG_FILE' -Tail 50" -ForegroundColor White
Write-Host "================================================================" -ForegroundColor Magenta
```

## Changelogs

| # | Problem | Fix |
|---|---------|-----|
| 1 | Missing `--unattended` | Added to **every** `tailscale up` call (PS1 + VBS) |
| 2 | Startup folder = headless dead | Replaced with **Scheduled Task** running as `SYSTEM` |
| 3 | VBS reimplements the service | Kept as optional hardened monitor + configured **service auto-recovery** via `sc.exe failure` |
| 4 | `$Args` shadows automatic variable | Renamed to `$Command` |
| 5 | Browser login on headless server | Replaced with **`--auth-key`** |
| 6 | `DoReconnect` calls `down` first | Removed `down` - just calls `tailscale up --unattended` (idempotent) |
| 7 | No elevation check | Fail-fast admin check at the top |
| 8 | VBS has zero logging | `WriteLog` to file with **5 MB rotation** |
| 9 | Single-peer-only monitoring | Primary check is now **`BackendState` from JSON**; peer ping is optional |
| 10 | winget may not exist | Checked before install; exits with guidance if missing |
| 11 | Fragile human-readable parsing | Both PS1 and VBS use `--json` / `ConvertFrom-Json` / regex on JSON |
| 12 | Hardcoded 30 s sleep for browser | Eliminated - auth key is instant; remaining waits use polling loops |
| 13 | Weak temp-file naming | `fso.GetTempName()` with proper cleanup in `Finally`-style blocks |
| 14 | Confusing step numbering | Flat sequential Steps 1-9 called linearly from main |
| 15 | No graceful VBS shutdown | Monitor exits cleanly when a **stop file** is detected; documented `Stop` / `Start` / `Uninstall` commands |

