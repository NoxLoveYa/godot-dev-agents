# Sourcing, licences, credits, Git LFS

## Sources

| Source | What | Licence |
|---|---|---|
| Poly Haven (`polyhaven.com`) | models (glTF), PBR textures, HDRIs | CC0 |
| ambientCG (`ambientcg.com`) | PBR materials, some models and HDRIs | CC0 |
| OpenGameArt (`opengameart.org`) | sprites, models, sounds, music | mixed: filter to CC0, check each item |
| Internet Archive (`archive.org`) | field recordings, public-domain film and audio | per item: Public Domain Mark / CC0 only |
| Freesound (`freesound.org`) | sound effects, ambiences | per item: CC0 filter |
| NASA (e.g. SVS CGI Moon Kit) | moon albedo and other imagery | generally public domain; check the item page |

## Poly Haven API

```python
import json, urllib.request
UA = {"User-Agent": "<project>-asset-fetch"}            # a descriptive client name; no personal data
def files(asset_id):
    req = urllib.request.Request(f"https://api.polyhaven.com/files/{asset_id}", headers=UA)
    with urllib.request.urlopen(req) as r:
        return json.load(r)
f = files("rock_moss_set_01")
gltf = f["gltf"]["2k"]["gltf"]                         # {"url": ..., "include": {relative path: {"url": ...}}}
tex = f["Diffuse"]["2k"]["jpg"]["url"]                 # textures: Diffuse, nor_gl, Rough, AO, Displacement, arm
hdr = f["hdri"]["4k"]["hdr"]["url"]                    # HDRIs
```

`https://api.polyhaven.com/assets?t=models` (or `textures`, `hdris`) lists assets with metadata. Download each file
to `<dest>.part`, then rename; skip files that exist; a small thread pool is fine for I/O. Use OpenGL normal maps
(`nor_gl`) for Godot.

## Untrusted downloads

Every download batch goes to its own new folder outside the project (`tools/<area>/raw/`, gitignored). Only ffmpeg,
Pillow or isolated `python -I` scripts kept elsewhere read them; nothing in them is executed.

## Credits

- During a parallel wave each package writes `docs/credits_pending/<package>.md`; the integration package merges them
  into `CREDITS.md` (and `assets/audio/CREDITS.md` for sound). Nobody else edits the shipped credits mid-wave.
- One row per file or set: asset, URL, author, licence (with link for CC BY: the full attribution line, e.g.
  "CC BY 3.0 https://creativecommons.org/licenses/by/3.0/"), edits made. Generated assets: "procedural, <tool>".
- Next to each downloaded audio file, a small JSON (title, author, licence, URL) written by the fetch script.
- A credits screen can read `CREDITS.md` at runtime (it then needs `*.md` in the export include filter; the smoke test
  counts its lines).

## Git LFS

`.gitattributes` at the repo root:

```
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.exr filter=lfs diff=lfs merge=lfs -text
*.hdr filter=lfs diff=lfs merge=lfs -text
*.glb filter=lfs diff=lfs merge=lfs -text
*.bin filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.ttf filter=lfs diff=lfs merge=lfs -text
*.res filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text
*.pck filter=lfs diff=lfs merge=lfs -text
```

`.import` files and `.tscn` / `.tres` text stay in plain git (they diff). `.godot/` is never committed. Check a commit
with `git lfs ls-files` before pushing.
