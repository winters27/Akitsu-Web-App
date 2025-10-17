# Arbiter Akitsu Web App Launcher

A simple Windows launcher that opens Arbiter Akitsu’s web app in a clean Microsoft Edge window.

---

## Usage

1. Download `Akitsu.exe` from this repository or the Releases page.  
2. Double-click to open Arbiter Akitsu in Edge.  
3. (Optional) Right-click → Pin to Start or Pin to Taskbar.

---

## Build It Yourself

1. Install ps2exe via PowerShell:
   ```powershell
   Install-Module ps2exe -Scope CurrentUser
   ```

2. [Download](https://github.com/winters27/Akitsu-Web-App/blob/main/Akitsu.ps1)
 or copy .ps1 into notepad and save as Akitsu.ps1:
   ```powershell
   # Akitsu.ps1 — Launches Arbiter Akitsu Web App in Microsoft Edge app-mode (PS 5.1 safe)
   $ErrorActionPreference = 'Stop'
   
   # ===== CONFIG =====
   $Url           = 'https://miceapp.arbiterstudio.com/#/project/items'
   $DesiredWidth  = 1200
   $DesiredHeight = 800
   $MaxWaitSec    = 12
   $PollMs        = 200
   $EdgeProfile   = $null   # e.g. "Default" or $null
   # ==================
   
   # Edge path
   $edge64  = 'C:\Program Files\Microsoft\Edge\Application\msedge.exe'
   $edge32  = 'C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe'
   $EdgeExe = if (Test-Path $edge64) { $edge64 } elseif (Test-Path $edge32) { $edge32 } else { 'msedge.exe' }
   
   # Build args
   $Args = "--app=`"$Url`""
   if ($EdgeProfile) { $Args += " --profile-directory=`"$EdgeProfile`"" }
   
   # Launch Edge (don’t wait)
   $launched = Start-Process -FilePath $EdgeExe -ArgumentList $Args -PassThru
   
   # ---- Win32 interop (safe Add-Type for PS 5.1) ----
   $cs = @"
   using System;
   using System.Runtime.InteropServices;
   
   public static class User32 {
       public delegate bool EnumWindowsProc(IntPtr hWnd, IntPtr lParam);
   
       [DllImport("user32.dll")] public static extern bool EnumWindows(EnumWindowsProc lpEnumFunc, IntPtr lParam);
       [DllImport("user32.dll")] public static extern bool IsWindowVisible(IntPtr hWnd);
       [DllImport("user32.dll")] public static extern uint GetWindowThreadProcessId(IntPtr hWnd, out uint lpdwProcessId);
       [DllImport("user32.dll")] public static extern bool SetWindowPos(IntPtr hWnd, IntPtr hWndInsertAfter, int X, int Y, int cx, int cy, uint uFlags);
       [DllImport("user32.dll")] public static extern bool ShowWindow(IntPtr hWnd, int nCmdShow);
   
       public const int SW_RESTORE = 9;
       public const UInt32 SWP_NOZORDER = 0x0004;
       public const UInt32 SWP_NOACTIVATE = 0x0010;
       public const UInt32 SWP_SHOWWINDOW = 0x0040;
   }
   "@
   try { Add-Type -TypeDefinition $cs -Namespace WinAPI -Name User32 -ErrorAction Stop } catch {}
   
   # Windows Forms for Screen.WorkingArea (center calc)
   try { Add-Type -AssemblyName System.Windows.Forms -ErrorAction Stop } catch {}
   
   function Get-AppPid {
       param([string] $url)
   
       Start-Sleep -Milliseconds 600  # let the app host spawn
   
       # Prefer CIM (gives CommandLine & CreationDate). Fall back to Get-Process.
       try {
           $procs = Get-CimInstance Win32_Process -Filter "Name='msedge.exe'" -ErrorAction Stop
       } catch {
           $procs = Get-Process msedge -ErrorAction SilentlyContinue | ForEach-Object {
               # Best-effort CommandLine from process (may be empty here)
               [pscustomobject]@{
                   ProcessId   = $_.Id
                   CommandLine = $null
                   CreationDate = $_.StartTime  # native DateTime
                   _NativeDate  = $true
               }
           }
       }
   
       if (-not $procs) { return $null }
   
       $needle = '--app="' + $url + '"'
   
       # Normalize CreationDate to real DateTime for sorting
       $normalized = $procs | ForEach-Object {
           $cmd = $_.CommandLine
           $dt  = $null
           if ($_.PSObject.Properties.Name -contains 'CreationDate' -and $_.CreationDate) {
               if ($_.CreationDate -is [datetime]) {
                   $dt = $_.CreationDate
               } else {
                   # DMTF → DateTime (CIM returns strings)
                   try { $dt = [Management.ManagementDateTimeConverter]::ToDateTime([string]$_.CreationDate) } catch { $dt = Get-Date }
               }
           } elseif ($_.PSObject.Properties.Name -contains 'StartTime' -and $_.StartTime) {
               $dt = $_.StartTime
           } else {
               $dt = Get-Date
           }
           [pscustomobject]@{ ProcessId = $_.ProcessId; CommandLine = $cmd; When = $dt }
       }
   
       $match = $normalized |
           Where-Object { $_.CommandLine -and ($_.CommandLine -like "*$needle*") } |
           Sort-Object @{ Expression = { $_.When }; Descending = $true } |
           Select-Object -First 1
   
       if ($match) { return [int]$match.ProcessId }
       return $null
   }
   
   function Get-TopLevelWindowsByPid {
       param([int]$Pid)
       $handles = New-Object System.Collections.Generic.List[System.IntPtr]
       $cb = [WinAPI.User32+EnumWindowsProc]{
           param([IntPtr] $h, [IntPtr] $l)
           if ([WinAPI.User32]::IsWindowVisible($h)) {
               [uint32]$wpid = 0
               [void][WinAPI.User32]::GetWindowThreadProcessId($h, [ref]$wpid)
               if ($wpid -eq $Pid) { $handles.Add($h) | Out-Null }
           }
           return $true
       }
       [WinAPI.User32]::EnumWindows($cb, [IntPtr]::Zero) | Out-Null
       return $handles
   }
   
   try {
       # Wait for the app window to exist (by PID)
       $deadline  = (Get-Date).AddSeconds($MaxWaitSec)
       $targetPid = $null
       while (-not $targetPid -and (Get-Date) -lt $deadline) {
           $targetPid = Get-AppPid -url $Url
           if (-not $targetPid) { Start-Sleep -Milliseconds $PollMs }
       }
   
       if (-not $targetPid) {
           try { $targetPid = (Get-Process msedge | Sort-Object StartTime -Descending | Select-Object -First 1).Id } catch {}
       }
   
       if ($targetPid) {
           $wins = Get-TopLevelWindowsByPid -Pid $targetPid
           if ($wins.Count -gt 0 -and [Type]::GetType('System.Windows.Forms.Screen', $false)) {
               $wa = [System.Windows.Forms.Screen]::PrimaryScreen.WorkingArea
               $X  = [int](($wa.Width  - $DesiredWidth)  / 2) + $wa.Left
               $Y  = [int](($wa.Height - $DesiredHeight) / 2) + $wa.Top
   
               foreach ($h in $wins) {
                   [WinAPI.User32]::ShowWindow($h, [WinAPI.User32]::SW_RESTORE) | Out-Null
                   [WinAPI.User32]::SetWindowPos(
                       $h, [IntPtr]::Zero, $X, $Y, $DesiredWidth, $DesiredHeight,
                       [WinAPI.User32]::SWP_NOZORDER -bor [WinAPI.User32]::SWP_NOACTIVATE -bor [WinAPI.User32]::SWP_SHOWWINDOW
                   ) | Out-Null
               }
           }
       }
   } catch {
       # Swallow unexpected issues so the launcher still opens the app
   }

   ```
4. Edit the paths to your .ps1 and .ico, then paste into PowerShell:
   ```powershell
   $in  = "C:\Path\To\Akitsu.ps1"
   $out = "C:\Path\To\Akitsu.exe"
   $ico = "C:\Path\To\Akitsu.ico"

   Invoke-ps2exe `
     -inputFile  $in `
     -outputFile $out `
     -iconFile   $ico `
     -noConsole `
     -title      "Akitsu" `
     -description "Launches Arbiter Akitsu's Web App in Microsoft Edge app-mode" `
     -product    "Akitsu Portable App" `
     -company    "Arbiter Studio" `
   ```

---
