# obscom

**OBS from the terminal, built for AI agents.**

`obscom` drives [OBS Studio](https://obsproject.com) through its built-in obs-websocket v5 server.
It's one file with zero dependencies. Every command takes `--json`, and `screenshot` saves a PNG
of what's on air, so an agent can check its work before your audience sees it.

```
$ obscom status
OBS 32.1.2 running (websocket 5.7.3)
Scene:     Main
Stream:    off
Recording: no
Mic:       Mic/Aux live (-1.4 dB)
```

## Install

Requires **Node.js 22+**, which provides the global `WebSocket`.

```sh
npm install -g github:andycufari/obscom
```

(npm release as `@cm64/obscom` coming soon.)

In OBS, enable the server under **Tools → WebSocket Server Settings**. obs-websocket is built
into OBS 28 and later.

## Connecting

`obscom` checks these sources in order:

| Setting  | Flag         | Env var        | Fallback                       |
|----------|--------------|----------------|--------------------------------|
| Host     | `--host`     | `OBS_HOST`     | `127.0.0.1`                    |
| Port     | `--port`     | `OBS_PORT`     | OBS's websocket config, else `4455` |
| Password | `--password` | `OBS_PASSWORD` | OBS's websocket config         |

If OBS runs on the same machine, you don't need to configure anything. `obscom` reads the
password from OBS's own `obs-websocket/config.json`:

- macOS: `~/Library/Application Support/obs-studio/plugin_config/obs-websocket/config.json`
- Linux: `~/.config/obs-studio/plugin_config/obs-websocket/config.json` (honors `XDG_CONFIG_HOME`)
- Windows: `%APPDATA%\obs-studio\plugin_config\obs-websocket\config.json`

`obscom` never prints the password and never stores it anywhere.

## Commands

```sh
obscom status                          # is OBS up? scene, stream, recording, mic
obscom scenes                          # list scenes (* = on air)
obscom scene "Main"                    # switch the program scene
obscom sources [scene]                 # sources in a scene, with visibility
obscom show "Camera" [scene]           # make a source visible
obscom hide "Camera" [scene]           # hide it
obscom mute ["Mic/Aux"]                # mute (default: the mic, see below)
obscom unmute ["Mic/Aux"]
obscom volume "Desktop Audio" [-6]     # get or set volume in dB
obscom text "Title" ["Live: building a CLI"]   # get or set a text source
obscom record [start|stop]             # status / start / stop
obscom stream [start|stop] --yes       # status / start / stop: needs --yes
obscom screenshot [source|scene] [--out shot.png] [--width 1280]
obscom raw GetStats                    # any obs-websocket request
obscom raw SetInputSettings '{"inputName":"Title","inputSettings":{"text":"hi"}}'
```

- **Names are forgiving.** Exact match first, then case-insensitive, then a unique substring:
  `obscom scene main` works. An ambiguous or missing name prints the valid options.
- **Sources inside groups** work with `show` and `hide`, and `sources` lists them.
- **Default mic:** `OBS_MIC` if it's set, else the audio input capture in the current scene,
  else OBS's global Mic/Aux.
- **`stream start|stop` refuses to run without `--yes`** (`--si` also works). Going live is
  public, so an agent has to confirm it explicitly.
- **`screenshot`** renders the program output by default, 1280 px wide with the aspect ratio
  kept, and prints the PNG's path. The default folder is `$OBS_SCREENSHOT_DIR`, else
  `<tmp>/obscom-screenshots/`. A capture that isn't on air may have no frames to render, so
  screenshot the program scene in that case.
- **Spanish aliases:** `estado`, `escenas`, `escena`, `fuentes`, `mostrar`, `ocultar`,
  `volumen`, `texto`, `grabar`, `captura`.

## Output and exit codes

`--json` prints a single JSON object per command, on stdout:

```sh
$ obscom scenes --json
{"ok":true,"current":"Main","scenes":["Main","Camera","Starting Soon"]}

$ obscom scene nope --json
{"ok":false,"error":"Nothing named \"nope\" in scenes. Available: Main, Camera, Starting Soon"}
```

| Exit | Meaning |
|------|---------|
| `0`  | ok |
| `1`  | error: bad name, request failed, auth failed |
| `3`  | OBS isn't running or can't be reached (`status` still prints `{"ok":true,"running":false}`) |

## Use with Claude Code and other AI agents

`obscom` is a plain CLI, so any agent that can run shell commands can use it. No MCP server is
needed. A typical loop:

```sh
obscom status --json                 # where are we?
obscom text "Title" "Fixing the login bug"
obscom scene "Screen"
obscom screenshot                    # → /tmp/.../obscom-....png
```

The agent then opens the PNG to check what viewers see: whether the title fits, the right window
is captured and nothing private is on screen. Claude Code reads images natively, so this works out
of the box.

Suggested guardrails for your agent's instructions (`CLAUDE.md` or similar):

- Run `obscom status --json` before changing anything.
- Never run `stream start` or `stream stop` unless the human asks for it in this session.
- After changing a scene, a source's visibility or a text source, take a `screenshot` and look at it.
- Restore any visibility or mute state you changed for testing.

## How it works

`obscom` performs the standard v5 handshake: Hello, then Identify with
`base64(sha256(base64(sha256(password + salt)) + challenge))`, `rpcVersion: 1` and no event
subscriptions. It sends one request per command and then closes the connection. The CLI is a
single file built on Node's global `WebSocket`.

## License

MIT © CM64 Labs
