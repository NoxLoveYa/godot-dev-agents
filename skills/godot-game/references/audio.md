# Audio in Godot 4

## Buses

- The layout lives in `default_bus_layout.tres` (`audio/buses/default_bus_layout`). Typical: `Master`, `Ambience`,
  `SFX`, `Music`, `UI`, `Underwater`. One package owns the file per wave.
- Effects by index (`AudioServer.get_bus_effect(bus, i)`): document which index is which and who toggles it (e.g.
  Ambience effect 0 = low-pass, toggled by the player when underwater; effect 1 = high-shelf for snow muffling, owned
  by the ambience mixer).
- A bus can be added at runtime (`AudioServer.add_bus`, `set_bus_send`) when a module needs one the layout lacks;
  check `get_bus_index(name) >= 0` first.
- Volumes from settings: linear slider -> `linear_to_db()`; ducking (pause menu) as a tween on an offset in dB, not on
  the user's value.
- A filter toggled only while one state runs (underwater low-pass) must be reset when that state is left another way
  (to the main menu while underwater).

## Players

- `AudioStreamPlayer` (non-positional: ambience beds, UI), `AudioStreamPlayer3D` (positional: set `unit_size`,
  `max_distance`, `attenuation_model`, `bus`), pooled for one-shots (a small ring of players, round robin), never one
  new node per sound.
- Loops: the import setting of `.ogg` / `.wav` (`loop` in the `.import` file, or `AudioStreamOggVorbis.loop`). Seamless
  loops need a crossfaded seam baked into the file.
- `AudioStreamRandomizer` for variations (pitch / volume random) of one-shots.

## Ambience mixers

- Layers (wind in leaves, birds, insects, rain, water lapping) each with a weight function of the world state (time of
  day, weather, season, distance to water, under canopy). Weights in a table; the mixer computes them at a few Hz and
  smooths volumes in dB.
- Event voices (thunder after `distance / 343` s, owl calls at night, creaks in wind gusts) scheduled from the state
  owner's signals, so picture and sound agree (one source of truth for the strike).
- Measure each file's RMS once (offline) and store levels in a data file, so the mixer sets
  `volume_db = target_db - file_rms_db`. Names say nothing about loudness.
- CPU: the mixer's `_process` should stay well under 0.1 ms; a headless perf probe can time it.

## Listing audio files at runtime

**(4.7.2)** In an exported `.pck`, `DirAccess.get_files_at("res://assets/audio")` lists `*.ogg.import` / remap names,
not the `.ogg` sources. Use `ResourceLoader.list_directory("res://assets/audio")`, which lists the loadable resources
in both editor and export, and filter by prefix (`bird_*`).

## Testing audio

- Headless: the mixer's layer weights for named scene states, delays (thunder), filter states, the settings keys. Run
  with `--fixed-fps 60` (the dummy audio driver still runs the mixer logic).
- Offline render: Movie Maker writes a `.wav` next to the frames; analyse its RMS per window to confirm what plays.
  That is a windowed run.
- Spectrogram PNGs of raw and built files let an agent judge content (birdsong bands, rain hiss, clipping) without
  listening.
