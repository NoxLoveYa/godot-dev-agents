# Windows and PowerShell traps

## Godot is a GUI-subsystem program

`godot.exe` (editor binary) is built for the Windows GUI subsystem: started from PowerShell with `&`, the shell does
not wait for it and its output may not appear. Pick one:

```powershell
# 1. pipe: the pipeline waits for the process and receives stdout / stderr
& $godot --headless --path $proj -s $script 2>&1 | ForEach-Object { $_ -replace '\x1b\[[0-9;]*m', '' } | Out-File -Encoding utf8 $log

# 2. Start-Process with redirection and a PID to wait on (use this when a plain pipeline hangs on a headless run)
$p = Start-Process -FilePath $godot -ArgumentList $argv -NoNewWindow -PassThru `
     -RedirectStandardOutput $log -RedirectStandardError "$log.err"
if (-not $p.WaitForExit(600000)) { Stop-Process -Id $p.Id; "TIMEOUT" }
$code = $p.ExitCode
```

Some Godot downloads ship a `*_console.exe` wrapper next to the editor binary; it is a console program, so the shell
waits and prints. Use it when your install has one.

`Start-Process -ArgumentList` with an array joins it with spaces: quote arguments that contain spaces yourself
(`'"' + $a + '"'`).

## Calling scripts

- From PowerShell, call runner scripts as `& .\shots.ps1 name "--cam=spawn --hour=12"`. A child
  `powershell -File script.ps1 "--cam=..."` **drops quoted arguments that start with `--`**.
- `-Only a,b` passed through `-File` arrives as one string: split it inside the script
  (`$Only = @($Only | ForEach-Object { $_ -split "," } | Where-Object { $_ })`).
- Long runs in the background: `Start-Process powershell -ArgumentList '-File', 'x.ps1' -RedirectStandardOutput log`
  and wait on the returned PID (the agent's Waiting rule).

## Paths

- Keep the Godot path in one variable per script, read from an environment variable or a gitignored local config,
  so shared scripts carry no machine paths: `$godot = $env:GODOT_EXE`.
- GDScript accepts forward slashes in absolute paths (`load("<repo>/tools/x.gd")` with the drive letter and `/`);
  prefer them in `-s` helpers.

## Focus

Windowed Godot runs open a focusable window. Neither external window-guard scripts (resetting the foreground window)
nor the project's no-focus overrides stopped it from stealing focus from a fullscreen game in practice. Prefer
`--headless`; when windowed runs are needed, warn the owner, batch them, and run them when the owner is not playing.

## Python next to Godot

- Run tool scripts with `python -I` (isolated: no current-folder imports), never from a folder with stray `.py` files.
- Downloads go to their own new folders and are read only by ffmpeg or isolated scripts kept elsewhere.
- CPU-bound batches: parallel processes (cores - 2), not threads.
