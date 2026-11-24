# UI, focus, gamepad and the input map

## Layout

- Stretch for any resolution: `display/window/stretch/mode="canvas_items"`, `aspect="expand"`, a base viewport of
  1920x1080. Design at the base size; anchors and containers do the rest.
- Full-screen pages: a root `Control` with `set_anchors_preset(Control.PRESET_FULL_RECT)`, containers
  (`VBoxContainer`, `MarginContainer`) inside. Avoid absolute positions except for HUD corners (anchor presets).
- One style source: a `Theme` resource, or a small static style class (palette constants, cached `FontVariation`s for
  weight and letter spacing). No colours or font paths inline in widget code.
- Readable text over bright 3D: a soft halo / outline (`outline_size`, `font_outline_color` with low alpha) or a dim
  gradient panel behind the column. Check the darkest and brightest scenes the game has.

## Menu pages

A base page class keeps every menu consistent:

- `process_mode = PROCESS_MODE_ALWAYS` (menus work while the tree is paused).
- `open()` fades in (`modulate:a`, 0.3 s tween), waits a frame, then gives focus to the default control; `close()`
  releases focus held inside the page and fades out.
- `mouse_filter = STOP` while open, `IGNORE` while closed (closed pages must not eat clicks).
- A `back_pressed` signal; the UI root routes Esc / gamepad B / the `pause` action to the right page.
- Optional pages: the UI root instances a page scene only if its file exists (same pattern as optional world modules),
  and the parent menu shows its entry only then.

## Focus and gamepad

- Every interactive widget: `focus_mode = FOCUS_ALL`, a visible focus style, `grab_focus()` on the default control when
  a page opens. Neighbours (`focus_neighbor_*`) only where container order is wrong.
- Handle `ui_accept` in `_gui_input` and ignore echoes (`event.is_action_pressed(&"ui_accept") and not event.is_echo()`);
  sliders and option rows take `ui_left` / `ui_right`.
- **(4.7.2)** The built-in `ui_accept` / `ui_cancel` had no joypad button in this project's input map, so A / B did
  nothing in menus while the d-pad (`ui_up/down/left/right`) worked. Add the buttons at startup when missing, and in
  the project's Input Map too:

```gdscript
func _ensure_pad_ui_events() -> void:
	for pair: Array in [[&"ui_accept", JOY_BUTTON_A], [&"ui_cancel", JOY_BUTTON_B]]:
		var has_pad := false
		for ev in InputMap.action_get_events(pair[0]):
			has_pad = has_pad or ev is InputEventJoypadButton
		if not has_pad:
			var jb := InputEventJoypadButton.new()
			jb.device = -1
			jb.button_index = pair[1] as JoyButton
			InputMap.action_add_event(pair[0], jb)
```

- Tabs on shoulder buttons (`JOY_BUTTON_LEFT_SHOULDER` / `RIGHT_SHOULDER`), pause on Start.
- Test every page with keyboard only and with a pad only; a headless flow test can drive pages by emitting the input
  actions (`Input.parse_input_event`) and checking `get_viewport().gui_get_focus_owner()`.

## Input map

- Gameplay actions in `project.godot [input]` with a keyboard and a joypad event each (`move_*` on stick axes with a
  deadzone, `jump` on A, `pause` on Start / Esc).
- Read movement with `Input.get_vector(&"move_left", &"move_right", &"move_forward", &"move_back")`; look on the right
  stick with its own sensitivity setting; mouse look in `_unhandled_input` with `InputEventMouseMotion.relative` while
  the mouse is captured.
- Rebinding: store only the user's overrides (`bindings` section of the settings file, a spec string per event such
  as `"k:<physical_keycode>"`), keep the default events captured at startup, and restore defaults before applying
  overrides on load. Several actions can share one binding group (`jump` + `swim_up`).
- Physical keycodes (`physical_keycode`) for movement, so WASD works on every keyboard layout.

## HUD

- A `CanvasLayer` per layer (HUD, menus, fades) with explicit `layer` numbers; fades on the top layer.
- HUD opacity, crosshair, hints and units are settings; screen effects (damage vignette, cold tint) are small
  canvas_item shaders on full-rect `ColorRect`s driven by the gameplay state.
- Blur behind menus: a full-screen `ColorRect` with a shader that reads `hint_screen_texture` with a mip LOD, tweened
  in with the menu.
