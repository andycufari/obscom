# Changelog

All notable changes to `obscom` are listed here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and versions follow
[Semantic Versioning](https://semver.org/).

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
