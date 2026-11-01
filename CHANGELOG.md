# Changelog

All notable changes to `obscom` are listed here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/).

## [0.2.0] - 2026-10-31

### Added

- Macros: `obscom run <macro> [args...]` runs a named list of steps from a JSON file as a
  single obs-websocket `RequestBatch` (`haltOnFailure`), so the whole macro is one round trip.
  Steps: `scene`, `show`, `hide`, `mute`, `unmute`, `volume`, `text`, `record`, `stream`,
  `sleep` (a `Sleep` inside the batch), `macro` (with recursion protection) and `raw`.
  Positional arguments `{1}`, `{2}`... and `{*}`. Names, including fuzzy matches and scene
  item ids, are resolved before the batch is sent.
- `run --dry-run` (`-n`) resolves everything and prints the plan without sending it.
- `obscom macros` lists the macros, one line each.
- Macro file lookup: `--macros`, `$OBSCOM_MACROS`, `./obscom.json`,
  `~/.config/obscom/macros.json`, then `~/Library/Application Support/obscom/macros.json`
  (macOS) or `%APPDATA%\obscom\macros.json` (Windows).
- `obscom watch [--json] [--all]` prints OBS events (scene, stream, record, mute, source
  visibility) one per line, for scripts that react without polling.
- `status --short`: one line, like `LIVE 00:12:31 · Main · rec off · mic on`.
- `-q` / `--quiet` prints nothing on success.

### Changed

- About 45 ms faster per command (about 133 ms down to 88 ms on a local OBS). `obscom` now
  speaks WebSocket through a small built-in client on `node:net` instead of loading Node's
  global `WebSocket` (about 40 ms), and loads `child_process` only when a connection fails.
- Requires Node.js 18+ instead of 22+.
- `stream start|stop` inside a macro needs `"confirm": true` on the step and `--yes` on the
  command line. `raw StartStream`, `StopStream` and `ToggleStream` now need `--yes` too.
- Unexpected errors print one line (set `OBSCOM_DEBUG=1` for the stack).
- `mute` and `unmute` make one request fewer.

## [0.1.0] - 2026-10-31

### Added

- First release: a zero-dependency CLI for obs-websocket v5 (Node 22+).
- Commands: `status`, `scenes`, `scene`, `sources`, `show`, `hide`, `mute`, `unmute`,
  `volume`, `text`, `record`, `stream`, `screenshot` and `raw`.
- `--json` on every command, and exit code `3` when OBS can't be reached.
- `stream start` and `stream stop` require `--yes` (`--si` is also accepted).
- `screenshot` saves the program output, or any scene or source, as a PNG and prints its path.
- Reads the password from OBS's own websocket config on macOS, Linux and Windows. It can be
  overridden with `--host`, `--port` and `--password`, or with `OBS_HOST`, `OBS_PORT` and
  `OBS_PASSWORD`.
- Name matching: exact first, then case-insensitive, then a unique substring. Group children
  are supported.
- Spanish command aliases.
