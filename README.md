# Akitsu — Arbiter Akitsu Web App Launcher

A simple Windows launcher that opens Arbiter Akitsu’s web app in a clean Microsoft Edge window.

---

## Usage

1. Download `Akitsu.exe` from this repository or the Releases page.  
2. Double-click to open Arbiter Akitsu in Edge.  
3. (Optional) Right-click → Pin to Start or Pin to Taskbar.

---

## Build It Yourself

1. Install ps2exe:
   ```powershell
   Install-Module ps2exe -Scope CurrentUser
   ```
2. Compile:
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
