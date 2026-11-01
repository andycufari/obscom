# obscom

**OBS from the terminal, built for AI agents.**

`obscom` drives [OBS Studio](https://obsproject.com) through its built-in obs-websocket v5 server.
It's one file with zero dependencies. Every command takes `--json`, macros run several steps as
one atomic round trip, and `screenshot` saves a PNG of what's on air, so an agent can check its
work before your audience sees it.

```
$ obscom status --short
offline · Main · rec off · mic on

$ obscom run work "Building a CLI"
ok work 74ms
```

## Install

Requires **Node.js 18+**. There are no dependencies.

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
obscom status [--short]                # is OBS up? scene, stream, recording, mic
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
obscom run <macro> [args...] [--dry-run]   # run a macro (see Macros)
obscom macros                          # list macros, one per line
obscom watch [--json] [--all]          # print OBS events as they happen
obscom raw GetStats                    # any obs-websocket request
obscom raw SetInputSettings '{"inputName":"Title","inputSettings":{"text":"hi"}}'
```

- **Names are forgiving.** Exact match first, then case-insensitive, then a unique substring:
  `obscom scene main` works. An ambiguous or missing name prints the valid options.
- **Sources inside groups** work with `show` and `hide`, and `sources` lists them.
- **Default mic:** `OBS_MIC` if it's set, else the audio input capture in the current scene,
  else OBS's global Mic/Aux.
- **Going live needs `--yes`.** `stream start|stop` refuses to run without `--yes` (`--si`
  also works), and so does `raw StartStream`, `StopStream` or `ToggleStream`. Going live is
  public, so an agent has to confirm it explicitly.
- **`screenshot`** renders the program output by default, 1280 px wide with the aspect ratio
  kept, and prints the PNG's path. The default folder is `$OBS_SCREENSHOT_DIR`, else
  `<tmp>/obscom-screenshots/`. A capture that isn't on air may have no frames to render, so
  screenshot the program scene in that case.
- **Spanish aliases:** `estado`, `escenas`, `escena`, `fuentes`, `mostrar`, `ocultar`,
  `volumen`, `texto`, `grabar`, `captura`, `correr`.

## Macros

A macro is a named list of steps. `obscom run <name>` sends the whole list to OBS as **one
obs-websocket `RequestBatch`**: one round trip, executed in order, and it stops at the first
step that fails. Names are resolved before anything is sent, so a typo fails the whole macro
up front instead of leaving OBS half-changed.

Example `obscom.json`:

```json
{
  "_comment": "obscom macros. Keys starting with _ are ignored.",
  "curtain": [{ "scene": "Curtain" }, { "mute": "Mic" }],
  "work": [
    { "scene": "Main" },
    { "unmute": "Mic" },
    { "text": ["Title", "Today: {1}"] },
    { "show": "Camera" }
  ],
  "camera": [{ "scene": "Camera" }, { "volume": ["Mic", 0] }],
  "brb": [{ "macro": "curtain" }, { "text": ["Title", "Back in a minute"] }],
  "intro": [{ "scene": "Curtain" }, { "sleep": 3000 }, { "macro": "work" }],
  "go-live": [{ "scene": "Curtain" }, { "stream": "start", "confirm": true }]
}
```

```sh
$ obscom macros
curtain: scene Curtain · mute Mic
work: scene Main · unmute Mic · text Title="Today: {1}" · show Camera
...

$ obscom run work "Building a CLI" --dry-run
dry work: scene Main · unmute Mic · text Title="Today: Building a CLI" · show Camera

$ obscom run work "Building a CLI"
ok work 74ms
```

**Steps** mirror the CLI verbs. Each step is an object with one verb:

| Step | Does |
|------|------|
| `{"scene": "Main"}` | switch the program scene |
| `{"show": "Camera"}`, `{"show": ["Camera", "Main"]}` | show a source; `hide` is the same |
| `{"mute": "Mic"}`, `{"mute": true}` | mute an input, or the default mic; `unmute` is the same |
| `{"volume": ["Mic", -6]}` | set volume in dB |
| `{"text": ["Title", "Live: {1}"]}` | set a text source |
| `{"record": "start"}` | `start` or `stop` recording |
| `{"stream": "start", "confirm": true}` | `start` or `stop` streaming (see Safety) |
| `{"sleep": 500}` | wait, in ms (0 to 50000), inside the batch |
| `{"macro": "curtain"}`, `{"macro": ["work", "{1}"]}` | run another macro |
| `{"raw": ["SetInputSettings", {...}]}`, `{"raw": "GetStats"}` | any obs-websocket request |

- **Arguments:** `{1}`, `{2}`... are the arguments after the macro name, and `{*}` is all of
  them joined by spaces. A missing argument is an error. A nested `{"macro": "name"}` without
  arguments receives the caller's arguments.
- **`show`/`hide` without a scene** use the scene the macro last switched to, else the current
  program scene. In `work` above, `Camera` is looked up in `Main`.
- **Recursion** (a macro that ends up calling itself) is an error.
- **Transitions:** a `scene` step returns as soon as the transition starts. If a later step
  should wait until it ends, add a `sleep` at least as long as the transition.
- **`--dry-run`** (`-n`) connects to OBS, resolves every name and prints the plan without
  sending it. With `--json` it prints the exact requests.

**Safety:** `stream start` and `stream stop` (and `raw` `StartStream`, `StopStream`,
`ToggleStream`) are refused inside a macro unless the step has `"confirm": true` **and** the
command line has `--yes`. The check runs before anything is sent, so a refused macro changes
nothing.

**Where macros live.** `obscom` uses the first of these that exists:

1. `--macros <file>`
2. `$OBSCOM_MACROS`
3. `./obscom.json` in the current directory
4. `~/.config/obscom/macros.json` (honors `XDG_CONFIG_HOME`)
5. macOS: `~/Library/Application Support/obscom/macros.json`;
   Windows: `%APPDATA%\obscom\macros.json`

Files are not merged. If `--macros` or `$OBSCOM_MACROS` points to a missing file, that's an
error.

## Watching events

`obscom watch` keeps the connection open and prints one line per OBS event: scene changes,
stream and record state, mute changes, and source visibility. A director script can react to
these without polling.

```sh
$ obscom watch --json
{"event":"CurrentProgramSceneChanged","ts":1793496613193,"sceneName":"Main"}
{"event":"InputMuteStateChanged","ts":1793496613367,"inputMuted":true,"inputName":"Mic"}
{"event":"StreamStateChanged","ts":1793496620011,"outputActive":true,"outputState":"OBS_WEBSOCKET_OUTPUT_STARTED"}
```

Without `--json` the lines are short (`scene Main`, `mute Mic`, `stream STARTED`).
`--all` prints every event OBS sends except the high-volume ones such as audio meters. `watch`
exits with `0` when OBS shuts down and `3` if the connection drops.

## Output and exit codes

Output is short by default because agents pay for every token. Actions print one line, macros
print `ok <name> <ms>` (time measured inside the process), and `-q` (`--quiet`) prints nothing
on success. Errors are a single line on stderr.

`--json` prints a single JSON object per command, on stdout, including errors:

```sh
$ obscom scenes --json
{"ok":true,"current":"Main","scenes":["Main","Camera","Curtain"]}

$ obscom scene nope --json
{"ok":false,"error":"Nothing named \"nope\" in scenes. Available: Main, Camera, Curtain"}
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
obscom status --short                # where are we?
obscom run work "Fixing the login bug"   # one call, one round trip
obscom screenshot                    # → /tmp/.../obscom-....png
```

The agent then opens the PNG to check what viewers see: whether the title fits, the right window
is captured and nothing private is on screen. Claude Code reads images natively, so this works out
of the box.

Suggested guardrails for your agent's instructions (`CLAUDE.md` or similar):

- Run `obscom status --short` before changing anything.
- Prefer macros for anything with more than one step, and try new ones with `--dry-run`.
- Never run `stream start` or `stream stop` unless the human asks for it in this session.
- After changing a scene, a source's visibility or a text source, take a `screenshot` and look at it.
- Restore any visibility or mute state you changed for testing.

## How it works

`obscom` performs the standard v5 handshake: Hello, then Identify with
`base64(sha256(base64(sha256(password + salt)) + challenge))` and `rpcVersion: 1`. Commands
send their requests and then close the connection. Macros go out as a single `RequestBatch`
with `haltOnFailure` and serial real-time execution, which is what lets `sleep` run inside the
batch. `watch` subscribes to events only for as long as it runs.

It speaks WebSocket through a small built-in client on `node:net` instead of Node's global
`WebSocket`, which takes about 40 ms to load. On a local OBS, a command takes about 90 ms
end to end, of which roughly 70 ms is Node starting up. The handshake and a request take
about 5 ms. `obscom` only looks for the OBS process when the connection fails, to tell you
whether OBS is closed or its websocket server is off.

## License

MIT © CM64 Labs
