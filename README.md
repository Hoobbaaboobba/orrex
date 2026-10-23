<div align="center">

# orrex

**See how your Go code actually runs.**
One command in a project folder — and you get a live map of it in the browser:
flowcharts with goroutine lanes, request paths from HTTP route to SQL, where errors are born and lost,
how `context` flows, and what goroutines really did at runtime.

[Install](#install) · [What it shows](#what-it-shows) · [CI](#in-ci) · [Русский](README.ru.md)

<img src="docs/function.png" alt="Flowchart of a function: the goroutine gets its own lane, channels and ctx are rails on the right" width="100%">

<sub>A function with a goroutine, a channel, <code>select</code> on <code>ctx.Done()</code> and a deferred <code>cancel()</code>. Every block is clickable and opens the line in your editor.</sub>

</div>

---

```sh
cd ~/my-go-service
orrex            # → http://127.0.0.1:7777
```

No config, no annotations, no code changes. orrex reads the source with the Go type checker,
watches the disk and redraws on every save. Everything stays on your machine.

> The interface is in Russian for now. Code, diagrams and file names are, of course, as in your project.

## What it shows

### Overview — what to look at first
Numbers, a heatmap of packages (size, complexity, goroutines, locks, SQL, lost errors, hints) and a ranked
list of findings: build errors → data races → broken `context` chains → leaks → transactions and SQL → the rest.
Repeated findings of the same rule fold into one line, so a code-style habit doesn't drown real bugs.

<img src="docs/overview.png" alt="Overview: stats, ranked findings and a package heatmap" width="100%">

### Request path — from HTTP route to the database
A sequence diagram built from the code: pick a route (`net/http`, chi, gin, echo — group prefixes included)
and follow calls through interfaces down to SQL. Switch the implementation behind an interface
(`Cached` → `Postgres`) right on the arrow. Error wrapping and HTTP statuses are marked along the way.

<img src="docs/path.png" alt="Request path GET /orders/{id}: handler, cache, Postgres, SELECT, error wrapping, statuses" width="100%">

### Error flow
Where an error is created, wrapped with `%w`, wrapped **without** `%w` (the cause is lost for `errors.Is`),
swallowed, logged or turned into an HTTP response. Everything else is dimmed.

<img src="docs/errors.png" alt="Error flow mode in a handler: 400, 404, 500" width="100%">

### Concurrency
Goroutines, channels and mutexes of the whole project as one graph: who sends, who receives, who closes,
who locks what. After a recorded trace, mutex nodes show real contention.

<img src="docs/concurrency.png" alt="Concurrency graph: goroutines, channels, mutexes" width="100%">

### Context
How `context.Context` travels from `main` to the database: where it is created, where it is replaced with
`context.Background()` (and cancellation stops working), who actually listens to `Done()`.

<img src="docs/context.png" alt="Context propagation graph" width="100%">

### Handlers — see what the client gets
Next to an HTTP handler's flowchart there is a response preview for every path: status, headers and body.
JSON is built from the literal or from the type, honouring json tags and `omitempty`; your own helpers like
`writeJSON(w, status, v)` are inlined at the call site. Warns when a header is set after the body is written,
`WriteHeader` is called twice, or JSON goes out without `Content-Type`. Works with `net/http`, gin and echo.

### Calls — without leaving the flowchart
Click a call to a project function to open a side panel: what it reads from and writes to its arguments, its exits,
defers and goroutines, what it does with errors — and which branches run with the values known at the call site.
Hovering highlights the lines that prepare the arguments and read the result. Expand the callee right inside the call
node or open its flowchart (with breadcrumbs back). For stdlib calls — the signature and a pkg.go.dev link.

### Deadlocks and goroutine leaks — before you run anything
- a goroutine sends to an unbuffered channel while the function may return on a timeout or `ctx.Done()` — the goroutine hangs forever;
- a channel no other goroutine can see, a `nil` channel, `close` in a loop, double `close`, send after `close`;
- `wg.Wait()` with no `wg.Done()`;
- a function holds a mutex and calls another one that locks the same mutex; two mutexes locked in opposite orders in different places.

Record a trace, and a run that hung or panicked goes to the top of the overview — with the line where `main` is stuck.

### And also
- **Architecture** — packages and imports; large projects fold into folders, indirect imports are hidden.
- **Package** — types, methods, interfaces and who implements them.
- **Data** — tables from migrations (`*.sql`, PostgreSQL and MySQL syntax) linked to the functions that read and write them; N+1, `SELECT *`, SQL built with `fmt.Sprintf`, unclosed `rows`, transactions without `Rollback`.
- **Runtime** — record a goroutine trace (`go test -trace`) and replay it over the flowchart; run the race detector and see races on the graph.
- **⌘K** — jump to any function, type, table, route or finding.
- Export diagrams to PNG (graphs also to SVG).

## Install

orrex needs **Go 1.22 or newer** on your machine (tested on 1.22–1.26). On the first run it downloads the
engine's dependencies and builds it with the same Go your project uses — about 15–40 seconds, once; progress is
shown both in the terminal and in the browser. After that it starts in a second. If your project's `go.mod`
asks for a newer Go than installed, Go downloads that version itself — orrex shows it as a separate step.

**macOS — Apple Silicon**
```sh
curl -fsSL https://github.com/Hoobbaaboobba/orrex/releases/latest/download/orrex-aarch64-apple-darwin.tar.gz | tar -xz orrex
sudo mv orrex /usr/local/bin/
```

**macOS — Intel**
```sh
curl -fsSL https://github.com/Hoobbaaboobba/orrex/releases/latest/download/orrex-x86_64-apple-darwin.tar.gz | tar -xz orrex
sudo mv orrex /usr/local/bin/
```

**Linux — x86_64 / arm64** (static binary, any distro)
```sh
curl -fsSL https://github.com/Hoobbaaboobba/orrex/releases/latest/download/orrex-$(uname -m | sed 's/arm64/aarch64/')-unknown-linux-musl.tar.gz | tar -xz orrex
sudo mv orrex /usr/local/bin/
```

**Windows 10/11** (PowerShell)
```powershell
$dir = "$env:LOCALAPPDATA\Programs\orrex"
Invoke-WebRequest https://github.com/Hoobbaaboobba/orrex/releases/latest/download/orrex-x86_64-pc-windows-gnu.zip -OutFile "$env:TEMP\orrex.zip"
Expand-Archive "$env:TEMP\orrex.zip" -DestinationPath $dir -Force
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path", "User") + ";$dir", "User")
```
Open a new terminal afterwards. The binary is not code-signed yet: if SmartScreen complains, choose
*More info → Run anyway*. The race detector on Windows additionally needs `gcc` (MinGW-w64).

Or download an archive from [Releases](https://github.com/Hoobbaaboobba/orrex/releases) and put `orrex` anywhere in `PATH`.
If macOS refuses to open a binary downloaded in the browser, run `xattr -d com.apple.quarantine orrex`.

## Usage

```sh
orrex                          # open the browser UI (http://127.0.0.1:7777)
orrex serve --port 8000 --no-open --editor zed
orrex check                    # no browser: build errors and hints in the terminal
orrex doctor                   # what orrex sees: project root, Go, engine, files
```

- Works in any folder with `go.mod` — or even just a `main.go` without one.
- "Open in editor" uses `--editor`, `$ORREX_EDITOR`, `$VISUAL`, `$EDITOR` or the first found of
  `cursor`, `code`, `zed`, `goland`, `subl`.
- Traces and `-race` run on a temporary copy of the project — your files are never touched.
- The server listens on `127.0.0.1` only. Nothing is sent anywhere.

## In CI

```sh
orrex check --format sarif > orrex.sarif     # GitHub code scanning
orrex check --format json                    # summary + findings for scripts
orrex check --fail-on hint                   # exit 1 on any finding (default: only on build errors)
orrex check --disable error-lost-cause,struct-padding
```

```yaml
# .github/workflows/orrex.yml
jobs:
  orrex:
    runs-on: ubuntu-latest
    permissions: { security-events: write }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: stable }
      - run: |
          curl -fsSL https://github.com/Hoobbaaboobba/orrex/releases/latest/download/orrex-x86_64-unknown-linux-musl.tar.gz | tar -xz orrex
          ./orrex check --format sarif > orrex.sarif
        continue-on-error: true
      - uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: orrex.sarif }
```

Exit codes: `0` — ok, `1` — findings (see `--fail-on`), `2` — orrex itself failed.

## Status

Early, but already useful on real projects — tested on [caddy](https://github.com/caddyserver/caddy):
3.5k functions analysed in under 2 seconds.

- Go only for now. Builds for macOS (Apple Silicon, Intel), Linux (x86_64, arm64) and Windows x64.
- Hints are heuristics — 34 rules. They can be noisy on code written in a different style;
  turn rules off with `--disable`.
- The source code is not published yet.

Bug reports and ideas are welcome in [Issues](https://github.com/Hoobbaaboobba/orrex/issues).

## License

MIT — see [LICENSE](LICENSE). Bundled third-party components are listed in [THIRD_PARTY.md](THIRD_PARTY.md).
