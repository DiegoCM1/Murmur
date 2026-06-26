# Murmur

A fast, local Mac dictation tool. Press a hotkey, speak, and the transcribed text is typed into whatever app has focus. Transcription runs on-device via **MLX Whisper** (Apple Silicon), so audio never leaves the machine.

Fork of [Diwannist/mlx-whisper-dictation](https://github.com/Diwannist/mlx-whisper-dictation), which itself is the MLX port of [foges/whisper-dictation](https://github.com/foges/whisper-dictation).
Rebranded "Murmur."

## Run

```bash
./run.sh                 # sources venv/ and runs the app
# or directly:
python whisper-dictation.py
```

Requires microphone + Accessibility permissions (the app types keystrokes and
listens to global hotkeys). Grant them in System Settings → Privacy & Security.

Common flags (see `parse_args()` for the full list):
- `-m / --model_name` — MLX Whisper repo id. Default `mlx-community/whisper-large-v3-mlx`.
- `-k / --key_combination` — toggle hotkey. Default `cmd_l+alt` on macOS.
- `--k_double_cmd` — toggle with double-press of Right Cmd instead of a combo.
- `-l / --language` — two-letter code(s), comma-separated. Default: auto-detect.
- `-t / --max_time` — auto-stop after N seconds. Default `90`.

## Architecture

Single file: `whisper-dictation.py`. Four classes, wired together in `__main__`.

- **`StatusBarApp(rumps.App)`** — the menu-bar UI and state owner. `started`
  flag, the recording timer, and the elapsed-time title live here. `toggle()`
  is the single entry point both key listeners call.
- **`Recorder`** — captures mic audio with PyAudio on a background thread
  (`_record_impl`), accumulating 16 kHz mono int16 frames while `self.recording`
  is true. On stop, converts to float32 and hands off to the transcriber.
- **`SpeechTranscriber`** — calls `mlx_whisper.transcribe(...)` then "types" the
  result one character at a time via `pynput`'s keyboard controller, with a tiny
  `sleep` between chars so target apps don't drop input.
- **Key listeners** — `GlobalKeyListener` (two-key combo) or
  `DoubleCommandKeyListener` (double Right-Cmd). Exactly one is active per run.

Flow: hotkey → `app.toggle()` → `Recorder.start()` spawns a record thread → hotkey again (or `max_time` timer fires) → `Recorder.stop()` flips the flag → record thread exits its loop, transcribes, and types the text.

## Codebase quirks worth knowing

- **`model_name` is a module-level global**, not passed into `SpeechTranscriber`. It's set in `__main__` (`model_name = args.model_name`) and read inside `transcribe()`. Watch for this if refactoring the transcriber.
- **No explicit model preload.** The eager `load_model(...)` path is commented out; the model loads lazily on the first `mlx_whisper.transcribe()` call, so the first dictation of a session is slower.
- **Recording happens on a raw thread**; `stream.read()` is blocking. PyAudio stream/`PyAudio()` lifecycle is opened and torn down per recording inside `_record_impl`. This is a known fragility area (cleanup, error paths).
- The `transcribe()` type loop swallows exceptions per character (`except: pass`).
- `requirements.txt` and `pyproject.toml` disagree (the latter is inherited from the pre-MLX upstream and lists `openai-whisper`/`triton` pins). `requirements.txt` + `mlx-whisper` is what actually matters for this fork.

## Conventions

- Keep it a single-file tool unless there's a real reason to split. Readability over cleverness — this is a tool to be understood, not an architecture demo.
- Match the existing plain-`print` status logging ("Listening...", "Transcribing...", "Done.").