# Debug loop

Bounded, one error at a time. Applies to the agent's own errors too.

1. **Read the log.** Run the failing command headless with output to a file; grep `SCRIPT ERROR|Parse Error|ERROR:|WARNING:`
   (ignore the exit-time "still in use at exit" / "leaked at exit" lines a headless run prints). The first error is
   usually the cause; later ones are fallout.
2. **Read around the cited line** and what it touches: the caller, the node path or group it looks up, the signal that
   fires it, the data it reads.
3. **Reproduce in a headless `-s` script** that sets the state directly (seeded, `--fixed-fps 60`), so each attempt
   takes seconds. Windowed reproduction only for bugs that need the GPU (a shader, a SubViewport chain), and only as the
   brief allows.
4. **One hypothesis**, written down. Typical causes:
   - a node path or group renamed by another package;
   - `_ready` order (a child read before the parent set it up; use `call_deferred` or an explicit init);
   - process mode during pause; process priority (a reader ran before the writer this frame);
   - a shared resource (mesh, material) changed by someone else;
   - export-only: a file the export filter dropped, `DirAccess` in a `.pck`, binary conversion of text resources;
   - a test-only artefact: unseeded RNG, wall-clock waits instead of frames, a real `user://` file.
5. **Fix the cause**, run the reproduction, then the project's existing headless tests for that area.
6. **At most 5 attempts per error**, with a running log (attempt, hypothesis, fix, result). Then stop, undo what did not
   help, report what you learned.

Never "fix" by deleting the feature, disabling the script, catching and ignoring the error, loosening a test, or
adding a sleep.
