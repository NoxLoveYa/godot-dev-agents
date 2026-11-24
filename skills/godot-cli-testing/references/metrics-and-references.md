# Contact sheets, metrics and verdicts against reference images

Looks are accepted by three things together: numeric metrics, a contact sheet next to the reference, and a written
verdict per "must show" item. Numbers alone pass ugly pictures; eyes alone drift.

## Look targets (in the design doc)

| Target | Reference | Must show | Numeric check |
|---|---|---|---|
| Golden hour | `tools/ref/golden_hour.jpg` | low warm backlit sun near the treeline, haze glowing toward the sun, glitter path on the water, lifted shadows, pale teal upper sky | upper-sky hue 160-200° with saturation < 0.35; sun blob within 40 px of its projected position; water column under the sun ≥ 2x mean water luma; midtone R/B ≥ 1.15; 5th-percentile luma ≥ 0.04 |
| Shallows | `tools/ref/underwater_plants.jpg` | streaming ribbon weed, lime where sunlit, golden gravel, warm green-gold water | underwater half mean hue 60-110°, never 150-230°; ≥ 8% "lime" pixels (hue 65-95°, S ≥ 0.5, V ≥ 0.6); bottom R/B ≥ 1.3; a deep control shot keeps hue 180-210° |
| Rain | `tools/ref/rain.jpg` | flat light, bright grey sky, dark ringed water, wet stone | shadow opacity ≤ 0.3; water luma ≤ 0.6 x sky luma; wet / dry luma ≤ 0.8 at the same view |
| Night | (none) | readable silhouettes, moon path | mean luma 0.04-0.13; sky ≥ 1.8 x treeline luma |

When a reference exists only as a text description, use its checklist and numbers until the image arrives, and re-run
the comparison then.

## Metric conventions (`metrics.py <png> <check>`)

- Load as float RGB in 0..1 (sRGB values as stored).
- Luma: Rec.709 `0.2126 R + 0.7152 G + 0.0722 B`.
- Hue: **circular mean** of HSV hue weighted by `S x V` (dark and grey pixels count little).
- Regions: fixed fractions of the frame (upper sky = rows above the horizon of a level camera; "underwater half" =
  bottom half for a camera at the waterline) or a projected point (the sun from the same astronomy as the game, the
  camera's yaw / pitch / fov).
- Ratios (R/B, wet / dry luma) over the same region of two shots from the same camera.
- Print `PASS` / `FAIL name (value, bounds)` and exit 1 on any FAIL.

## Contact sheets (`contact.py`)

- `contact.py <ref> <shot> <out.png> [--diff]`: reference | shot (| 4x absolute difference), both scaled to 540 px
  high, labelled with file names.
- `contact.py --grid <out.png> <img>... --cols N --labels a,b,c`: time-lapses, weather sets, before / after pairs.
- Look at the sheet yourself before reporting it.

## Written verdict

One line per must-show item: `item: yes / partly / no, what you see, what would fix it`. Then the overall call
(accept, or the next change). Keep before / after pairs (`out/polish/before_*.png`, `after_*.png`) for every polish
round so the owner can judge the change.

## Regression of a look

A "no-op" shot after a contract change (stubs in, nothing meant to change) compared with the pre-change render (diff
sheet, mean absolute difference per region) proves the contracts did not move the picture.
