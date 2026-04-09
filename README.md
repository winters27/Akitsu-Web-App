# Arbiter Akitsu Web App Launcher

A simple Windows launcher that opens Arbiter Akitsu’s web app in a clean Microsoft Edge window.

---

## Usage

1. Download `Akitsu.exe` from this repository or the Releases page.  
2. Double-click to open Arbiter Akitsu in Edge.  
3. (Optional) Right-click → Pin to Start or Pin to Taskbar.

---## Build It Yourself (Native C#)

This repository provides a cleaner, faster, zero-dependency alternative to `ps2exe` by compiling directly via the Windows built-in C# compiler. 

1. Download or clone this repository.
2. Edit `Akitsu.cs` if you wish to tweak window dimensions or Chromium flags.
3. Open PowerShell and cleanly compile the `.exe` (which permanently embeds the `.ico`) by running:

```powershell
& "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe" /target:winexe /out:Akitsu.exe /win32icon:Akitsu.ico Akitsu.cs
```

4. Your new `Akitsu.exe` will be generated with zero console-flashing overhead!
