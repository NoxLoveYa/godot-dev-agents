# Regenerate and re-import an asset

```powershell
# tools/<asset>/regen.ps1 [variants...]   (paths from the environment or a gitignored local config)
param([string[]]$Variants = @())
$py = $env:PYTHON_EXE; $godot = $env:GODOT_EXE; $proj = "<project>"
& $py -I "$PSScriptRoot\generate_x.py" @Variants
Get-ChildItem "$proj\assets\x" -Filter "x_*.png*" | Remove-Item -Force          # stale extracted textures
& $godot --headless --path $proj --import 2>&1 | Out-Null
& $py -I "$PSScriptRoot\patch_import.py"
& $godot --headless --path $proj --import 2>&1 | Out-Null                         # applies the patched options
& $godot --headless --path $proj --import 2>&1 | Select-String -Pattern 'ERROR|WARNING'   # should print nothing
& $godot --headless --path $proj -s "$PSScriptRoot\check_contract.gd" 2>&1 | Out-File -Encoding utf8 "$PSScriptRoot\out\contract.txt"
```

Then:

1. Read `out/contract.txt`: all PASS.
2. Renders (windowed, when the brief allows): the preview scene through the shot harness, the views the brief lists.
3. If a game system reads the asset (bone names, animation names, node names), run that system's headless test.
4. Commit the generator, the patch script, the `.glb`, the `.import` files and the extracted textures together
   (binaries through LFS).
