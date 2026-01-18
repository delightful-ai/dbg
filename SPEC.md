Got it – I’ve updated the design accordingly to ensure it feels instantly familiar, like LLDB/GDB, while keeping it optimized for bash-only usage.


# DBG v0.1 Technical Specification

A bash-first debugger built on DAP, designed for noninteractive use with output that feels like a great “paper debugger” you can drive with one-shot commands.

This spec is opinionated on one thing above all: **the output format is the product**. If you cannot *trust it*, *scan it*, and *act on it* without an interactive UI, the debugger is just a protocol adapter with ambition.

DBG’s CLI should feel instantly familiar to anyone who’s used **gdb/lldb**: `b`, `c`, `n`, `s`, `fin`, `bt`, `p`, `frame`, `thread`, `info`, `list`, `disas`, `x/...`.
The difference is packaging: **no REPL required**, no TUI required. You “tap” the session with commands; every tap prints a curated artifact.

---

## 1) Product definition

DBG is a noninteractive CLI debugger that:

* Talks to debug adapters via **DAP**.
* Can be used entirely from a plain bash terminal.
* Produces **decision-ready, grounded artifacts** by default.
* Preserves **gdb/lldb muscle memory** via familiar commands and aliases.

DBG’s mental model:

* There is a **session**.
* The target is either **running** or **stopped**.
* When stopped, there is a **focused thread** and a **focused frame**.
* Commands either:

  * control execution (continue/step), or
  * render a view (stop/bt/locals/etc), or
  * mutate debugging state (breakpoints/watches).

---

## 2) Goals and non-goals

### Goals

1. **Comprehension-first defaults**
   Every “stop” output answers, on the first screen:

   * why did we stop?
   * where are we?
   * what’s the relevant context (stack + code + locals)?
   * what’s likely actionable next?

2. **Trustworthy grounding**
   Every code excerpt must be tied to:

   * source path (or DAP sourceReference),
   * line range,
   * and a stable handle.

3. **Scanability and low repetition**
   Grounding appears **per fragment**, not per line.

4. **Predictable, scriptable behavior**
   No prompts. No hidden state in a REPL. Everything is addressable via handles.

5. **DAP-native but adapter-tolerant**
   Capabilities vary; DBG adapts, degrades gracefully, and explains why.

### Non-goals

* Replacing IDE UIs.
* Guaranteeing advanced features (reverse debugging, data breakpoints) across all adapters.
* A “dump everything” default mode.
* PTY-driven interactive program I/O. DBG works without PTY.

---

## 3) System architecture

### 3.1 Processes

DBG uses two cooperating components:

* **`dbg`** (CLI): short-lived; you run it for each command.
* **`dbg-agent`**: long-lived per session; owns state, runs/attaches adapters, caches stop context, records events.

This is how we get “familiar debugger state” without a REPL.

### 3.2 Session transport

* `dbg` communicates with `dbg-agent` over a **local IPC channel**:

  * Unix domain socket on Unix-like systems.
  * Named pipe / TCP loopback fallback where needed.

### 3.3 Session working directory + discovery

* When a session is created, DBG writes a session pointer file at:

  * `./.dbg/session` (project-local, repo-friendly)
* Commands run within that directory automatically target that session.
* Override with `--session <handle|path>`.

### 3.4 DAP adapter lifecycle

`dbg-agent` either:

* **spawns** an adapter process and speaks DAP over stdio, or
* **connects** to an adapter server (TCP) as configured.

`dbg-agent` implements:

* DAP message framing (Content-Length headers),
* request/response correlation,
* event ingestion and persistence,
* capability negotiation,
* and a state machine.

---

## 4) Core concepts

### 4.1 Session, stop, focus

* **Session**: an ongoing debug context (adapter + debuggee).
* **Stop**: a point where the debuggee is paused (breakpoint, step, exception, signal, etc).
* **Focus**:

  * `focus.thread`: selected thread for stepping and stack views.
  * `focus.frame`: selected frame for locals/eval/source.

DBG persists focus in the session so “`frame 2` then `locals`” works across one-shot commands.

### 4.2 Handles

Handles are stable identifiers printed in output and accepted as input.

**Grammar**

```
Handle := "@" Type ":" Body
Type   := [A-Z][A-Z0-9]*
Body   := ASCII without spaces (recommend: [A-Za-z0-9._-]+)
```

**Core handle types**

* `@S:<id>` session
* `@P:<id>` debuggee process (DBG-level)
* `@A:<id>` adapter instance
* `@ST:<n>` stop (monotonic integer per session)
* `@T:<id>` thread (DAP threadId)
* `@F:<stop>.<index>` frame within a stop’s stack (example `@F:17.0`)
* `@B:<id>` breakpoint (DBG-level id)
* `@IB:<id>` instruction breakpoint (DBG-level id)
* `@DB:<id>` data breakpoint (DBG-level id)
* `@W:<id>` watch expression (DBG-level id)
* `@E:<n>` event in the session timeline
* `@SN:<id>` snapshot bundle
* `@SR:<n>` source reference (DAP sourceReference)

**Handle lifetime rules**

* `@S`, `@B`, `@W`, `@E`, `@SN` are stable across execution.
* `@ST` refers to recorded stop summaries (stable).
* `@F:<stop>.<index>` is stable for that stop record.
* Any “live expansion” of variables is only valid while stopped; snapshots can preserve rendered values for later review.

### 4.3 Location spec

Locations are accepted in gdb/lldb-ish forms:

* `file:line`
* `file:line:col` (col optional; adapter support varies)
* `fn:<name>` (function breakpoint)
* `addr:<hex>` (instruction breakpoint / disas anchor)
* `@SR:<n>:line` (for non-file sources)

**Examples**

* `dbg b src/main.c:42`
* `dbg b fn:main`
* `dbg ibreak addr:0x401234`
* `dbg list @SR:12:88`

---

## 5) Output format contract

Inspired by the “RAQL artifacts” approach: structured, skimmable, grounded, and budgeted.

### 5.1 Output modes

Global `--only` controls map vs territory:

* `--only artifact` (default)
  Curated, human-first output: semantic headers, code snippets, grouping.
* `--only nav`
  Location-first, grep-like navigation lines. Minimal excerpts.
* `--only counts`
  Metrics only (counts, summaries). No code excerpts.

There is also `--jsonl` as an opt-in escape hatch, but it is not the design center of this tool.

### 5.2 Output atoms

#### A) Artifact preamble (one line)

Must be log-friendly and instantly scannable.

Example:

```
dbg stop  ->  STOPPED breakpoint @B:3  thread @T:1  frame @F:17.0  [@ST:17 @S:9c2f]
```

#### B) Summary block (key-value list)

Compact, diff-friendly, parseable if needed.

Example:

```
# Summary
session: @S:9c2f
process: pid=4123  state=stopped
reason: breakpoint  bp=@B:3  hits=5
thread: @T:1  name="main"
frame: @F:17.0  fn=foo::bar  at src/foo.rs:87
time: 2026-01-18T00:12:31Z
```

#### C) Section headings

* `#` for sections
* `##` for groups

#### D) Fragment header and grounding

Two styles depending on mode.

**Artifact mode (semantic-first)**

```
SRC  fn foo::bar(x: i32) -> Result<()> 
at src/foo.rs:82-96  [@F:17.0]
```

**Nav mode (location-first)**

```
src/foo.rs:87  SRC  fn foo::bar(x: i32) -> Result<()>  [@F:17.0]
```

#### E) Code blocks (guttered)

* Normal lines: `| `
* Anchor/highlight lines: `> | `

Example:

```
| let y = compute(x)?;
> | if y == 0 {
|     return Err(Error::Zero);
| }
```

#### F) Elision messages (honest + actionable)

Always say:

* what was omitted
* how many
* which knob shows more

Example:

```
… omitted 12 more frames (showing 5/17). Add: --show frames=all
```

#### G) Alternatives for ambiguity (noninteractive picker)

When input is ambiguous, print a numbered list with handles.

Example:

```
# Alternatives (matched "main")

1. fn main at src/main.c:12  [fn:main]
2. fn main at src/tools/main.c:9  [fn:main@src/tools/main.c]

Tip: rerun as `dbg b src/main.c:12`
```

---

## 6) Global flags and knobs

### 6.1 Global flags

Available on all commands unless noted.

* `--session <@S:…|path>`: select session
* `--only {artifact|nav|counts}`
* `--show <knobs>`: expansion controls (see below)
* `--max-lines <N>`: max lines per code fragment (default varies)
* `--max-width <N>`: truncate long lines (default 140)
* `--max-value <N>`: truncate value renderings (default 240 chars)
* `--depth <N>`: variable expansion depth (default 1)
* `--timeout <dur>`: for wait/resume commands
* `--project-root <path>`: influences “user frame” heuristics
* `--color {never|auto|always}` (default `never`)

### 6.2 `--show` knobs

A single composable mechanism, RAQL-style.

Syntax:

* `--show key=value[,key=value...]`
* Values: integer `K` or `all`

Keys (v0.1):

* `frames` (default 5)
* `threads` (default 8)
* `locals` (default 12)
* `args` (default 8)
* `watches` (default 6)
* `scopes` (default 8)
* `breaks` (default 50)
* `output` (default 20 lines)
* `events` (default 30)
* `disasm` (default 24 instructions)

Examples:

* `--show frames=all`
* `--show locals=30,args=all`
* `--show output=200`

---

## 7) CLI surface

DBG is organized into command families that “earn their place” by having a distinct output contract and intent.

### 7.1 Top-level commands (canonical)

```
dbg start        Create a session and prepare a launch (like “target create”)
dbg attach       Attach to an existing process and stop it
dbg connect      Connect to an already-running DAP adapter server

dbg run          Begin execution (like gdb/lldb “run”)
dbg c|continue   Resume execution from a stop
dbg n|next       Step over
dbg s|step       Step into
dbg fin|finish   Step out (until returning from current frame)
dbg until        Run until location
dbg pause        Interrupt execution
dbg restart      Restart the debuggee (if supported)
dbg kill         Terminate debuggee
dbg detach       Disconnect without terminating (if supported)
dbg wait         Wait for next stop/exit/output event

dbg stop         Print the current stop postcard
dbg report       Print a fuller “case file” for the current stop

dbg bt           Backtrace
dbg frame        Select or show current frame
dbg up/down      Move frame focus
dbg threads      List threads
dbg thread       Select thread focus

dbg list         List source around a location
dbg source       Show source by @SR:… or file, with grounding
dbg disas        Disassemble around location
dbg x            Examine memory (gdb-like)

dbg p|print      Evaluate expression in current frame
dbg locals       Show locals for current frame
dbg args         Show arguments for current frame
dbg scopes       List scopes for current frame
dbg vars         Expand a variable reference

dbg set          Set variable / expression (if supported)
dbg watch        Manage watch expressions (evaluate on each stop)

dbg b|break      Manage source/function breakpoints
dbg tbreak       Temporary breakpoint (one-shot)
dbg ibreak       Instruction breakpoints
dbg watchpoint   Data breakpoints (watchpoints)
dbg exception    Exception breakpoints / filters

dbg logs         Show captured output (stdout/stderr)
dbg events       Show session timeline (stops/output/bp changes)

dbg snapshot     Save or show persistent bundles of stop context

dbg info         gdb-like alias group (info b, info threads, ...)
dbg dap          Raw DAP utilities (trace, send request)
dbg batch        Run a sequence of dbg commands from a file/stdin
```

### 7.2 Aliases for familiarity

* `dbg b` == `dbg break`
* `dbg c` == `dbg continue`
* `dbg n` == `dbg next`
* `dbg s` == `dbg step`
* `dbg fin` == `dbg finish`
* `dbg bt` == `dbg bt`
* `dbg p` == `dbg print`
* `dbg l` == `dbg list` (optional alias; default off to avoid conflict with shell `l`)

---

## 8) Command specifications

Below, each command defines:

* purpose
* key flags
* behavior
* output contract

### 8.1 `dbg start`

Create a session, initialize adapter, prepare launch configuration. By default, DBG behaves like “lldb before `run`”: it does not start program execution until you `dbg run`.

**Synopsis**

```
dbg start --adapter <name|path> [--connect <mode>] \
  [--launch <json-file>|--launch-inline <json>] \
  [--cwd <dir>] [--env KEY=VAL ...] \
  -- <program> [args...]
```

**Notes**

* If `--launch` is provided, it is passed as adapter-specific launch arguments.
* If `-- <program> [args]` is provided, DBG injects these into launch args using a configurable mapping (see Config).

**Behavior**

* Spawns or connects to adapter.
* Performs DAP initialize handshake.
* Sends `launch` request (adapter-specific arguments).
* Waits for adapter `initialized` event.
* Applies stored breakpoints/watch settings to the new session.
* Does **not** send `configurationDone` until `dbg run` (unless `--run` is used).

**Flags**

* `--run`: automatically do `dbg run` after setup.
* `--stop-at-entry {on|off}` (default `on`): requests entry stop if supported by adapter args/mapping.
* `--console {capture|inherit}` (default `capture`): whether to prefer capturing output via DAP `output` events vs inheriting process output.

**Output**
Artifact mode:

1. Preamble: session created
2. Summary: adapter, process plan, state `ready`
3. Sections:

   * `# Adapter`
   * `# Launch config (resolved)` (redacted sensitive env)
   * `# Next`

     * “Tip: run with `dbg run`”

Example skeleton:

```
dbg start  --  ./app  ->  SESSION READY  [@S:9c2f]

# Summary
session: @S:9c2f
adapter: @A:python  transport=stdio
state: ready (not running)
stop-at-entry: on
breakpoints: 3 configured
watches: 2 configured

# Next
Tip: begin execution with `dbg run`
```

### 8.2 `dbg run`

Begins execution of a prepared launch by sending `configurationDone` (or adapter equivalent) and then waits for the next stop/exit by default.

**Synopsis**

```
dbg run [--no-wait] [--timeout <dur>]
```

**Default behavior**

* Starts running.
* Blocks until:

  * next stop, or
  * termination/exit, or
  * timeout.

**Output**

* On stop: prints the **Stop Postcard** (same as `dbg stop`).
* On exit: prints an **Exit Summary**.

Elision example:

```
… program exited (code=1). Add: --show output=all
```

### 8.3 `dbg continue` / `dbg next` / `dbg step` / `dbg finish` / `dbg until`

Execution control commands all share a key contract:

**Contract**: by default they behave like “do + wait + show”.

**Common synopsis**

```
dbg continue [--no-wait] [--timeout <dur>]
dbg next     [--count N] [--no-wait] [--timeout <dur>]
dbg step     [--count N] [--no-wait] [--timeout <dur>]
dbg finish   [--no-wait] [--timeout <dur>]
dbg until    <location> [--no-wait] [--timeout <dur>]
```

**Stop output**
All print the Stop Postcard, plus a small “delta” section:

* changed locals (best-effort)
* changed watches (best-effort)

Example delta section:

```
# Changed
locals: x: 3 -> 4, err: nil -> "EOF"
watches: req.id: 91 -> 92
```

If DBG cannot compute deltas (adapter doesn’t expose stable values), it prints:

```
# Changed
note: delta unavailable (adapter does not provide stable variable identity)
```

### 8.4 `dbg pause`

Interrupt execution.

```
dbg pause [--timeout <dur>]
```

Output:

* if pause leads to stop: Stop Postcard.
* if timeout: prints status + hint.

### 8.5 `dbg restart`

Restart the debuggee if supported.

```
dbg restart [--timeout <dur>]
```

If unsupported:

* prints a clear error and suggests `dbg kill && dbg run` (or `dbg start --run`) depending on adapter.

### 8.6 `dbg kill` / `dbg detach`

* `dbg kill`: terminate the debuggee (DAP terminate or disconnect with terminate).
* `dbg detach`: disconnect but keep debuggee alive, if supported.

Both print a summary artifact.

---

## 9) The Stop Postcard

This is the flagship artifact. It must be *good enough that stepping feels like debugging, not archaeology*.

### 9.1 `dbg stop`

Print the current stop postcard (or last stop).

```
dbg stop [--stop <@ST:n>] [--thread <@T:id>] [--frame <@F:x.y|index>]
```

### 9.2 Stop Postcard sections (fixed order)

1. **Summary**
2. **Where you are**
3. **Source**
4. **Stack**
5. **Locals**
6. **Watches**
7. **Recent output**
8. **Notes** (reason-specific)

### 9.3 Stop heuristics (critical)

DBG must assemble context with heuristics, not raw dumps.

#### Heuristic A: “Nearest project frame”

If the top frame is outside project root (stdlib, runtime, vendor), DBG shows:

* current frame (truth)
* nearest frame whose source is inside `project-root` (actionable)

Example:

```
# Where you are
NOW  fn std::panicking::begin_panic
at /rustc/...  [@F:17.0]

NEAREST PROJECT FRAME
fn myapp::handler::serve
at src/handler.rs:201  [@F:17.6]
Tip: focus it with `dbg frame @F:17.6`
```

#### Heuristic B: “Line-referenced locals first”

In `Locals`, variables referenced on the current source line are shown first (best-effort token match), then the rest.

#### Heuristic C: “Error-shaped names get priority”

Within the budget, prefer names matching:
`err`, `error`, `errno`, `exception`, `status`, `code`, `result`, `ret`.

#### Heuristic D: “Reason-aware extras”

* If stopped on exception and adapter supports exception info:

  * show exception type/message in Notes
* If signal/crash:

  * include Registers (brief) and Disassembly snippet (brief)
  * include fault address if available
* If breakpoint:

  * show breakpoint condition/log/hit count

### 9.4 Default budgets for stop postcard

* source context: 9 lines (4 before, 1 anchor, 4 after)
* frames: 5
* locals: 12
* args: 8 (shown within Locals as a group if present)
* watches: 6
* output: 20 lines

All expansions via `--show`.

### 9.5 Example stop postcard (skeleton)

```
dbg stop  ->  STOPPED breakpoint @B:3  thread @T:1  frame @F:17.0  [@ST:17 @S:9c2f]

# Summary
session: @S:9c2f
process: pid=4123  state=stopped
reason: breakpoint  bp=@B:3  hits=5
thread: @T:1  name="main"
frame: @F:17.0  fn=myapp::handler::serve  at src/handler.rs:201

# Where you are
SRC  fn myapp::handler::serve(req: Request) -> Result<Response>
at src/handler.rs:188-244  [@F:17.0]

# Source
at src/handler.rs:201-209

| let user = repo.load_user(req.user_id)?;
> | if user.is_none() {
|     return Err(Error::NotFound);
| }
| let plan = planner.plan(&user.unwrap())?;

# Stack
1. fn myapp::handler::serve(...)  at src/handler.rs:201  [@F:17.0]
2. fn myapp::server::dispatch(...) at src/server.rs:88   [@F:17.1]
3. fn tokio::runtime::...          at …                  [@F:17.2]
… omitted 9 more frames (showing 3/12). Add: --show frames=all

# Locals
args:
- req: Request{ user_id=91, ... }     [preview truncated, add: --max-value=800]

locals:
- user_id: 91
- user: None
- err: nil
… omitted 14 more locals (showing 4/18). Add: --show locals=all

# Watches
- @W:2  req.user_id = 91
- @W:5  err         = nil

# Recent output
(stdout)
| INFO request id=8f3 ... route=/users/91
… omitted 120 lines (showing 1/121). Add: --show output=all

# Notes
bp: @B:3  condition: user_id == 91
```

---

## 10) Stack, threads, and frames

### 10.1 `dbg bt`

Backtrace of the focused thread by default.

```
dbg bt [--thread <@T:id>] [--show frames=K|all] [--with-source] [--with-locals]
```

Default output:

* list frames with function + location.
* If `--with-source`, include a tiny 3-line snippet per top frames (budgeted).
* If `--with-locals`, include locals for top 1 frame (or each, budgeted).

### 10.2 `dbg threads`

List threads, highlight focused and stopped.

```
dbg threads [--show threads=K|all]
```

Artifact mode groups:

* `# Stopped threads`
* `# Running threads`

### 10.3 `dbg thread`

Select focus thread.

```
dbg thread <@T:id|id>
```

Output: summary + “Tip: now `dbg bt` refers to this thread”.

### 10.4 `dbg frame`, `dbg up`, `dbg down`

* `dbg frame` shows or sets focused frame.
* `up/down` moves relative.

```
dbg frame [<@F:stop.idx|idx>]
dbg up [N]
dbg down [N]
```

On success, prints a mini postcard:

* frame header + 5-line source snippet + locals preview.

---

## 11) Source views

### 11.1 `dbg list`

gdb/lldb-like source listing around a location, defaulting to current frame.

```
dbg list [<location>] [--context N]
```

Output is a single `# Source` fragment with grounding and guttered lines.

### 11.2 `dbg source`

Show a full source file or DAP sourceReference slice.

```
dbg source show <file|@SR:n> [--range a-b]
dbg source info <file|@SR:n>
```

* `info` prints metadata (path, origin, checksums if available).

---

## 12) Expressions, locals, variables

### 12.1 `dbg print` / `dbg p`

Evaluate an expression in the focused frame.

```
dbg print <expr> [--context {repl|watch|hover}] [--format {auto|raw}]
```

Output contract:

* preamble includes expression (truncated with full in Notes if needed)
* `# Result` shows:

  * type (if adapter provides)
  * value preview
* optional `# Related` section:

  * if the value is complex and adapter provides a variablesReference, print a `vars` handle for expansion.

Example:

```
dbg p  user  ->  VALUE  [@ST:17 @S:9c2f]

# Summary
expr: user
frame: @F:17.0

# Result
type: Option<User>
value: None
```

### 12.2 `dbg locals` / `dbg args`

Render locals/args for the focused frame.

```
dbg locals [--depth N] [--show locals=K|all]
dbg args   [--depth N] [--show args=K|all]
```

Output groups by scope and uses elision messages.

### 12.3 `dbg scopes`

List scopes of the focused frame.

```
dbg scopes [--show scopes=K|all]
```

### 12.4 `dbg vars`

Expand a variable reference.

```
dbg vars <selector> [--depth N]
```

Selectors can be:

* a variable name in current scope (`user`)
* a scope-qualified selector (`locals.user`)
* a handle emitted by DBG when a complex value is shown (implementation detail but stable within stop)

If expansion is not available (running state), DBG explains and suggests `dbg stop` or `dbg pause`.

### 12.5 `dbg set`

Set a variable or expression value if supported.

```
dbg set var <name> <value-expr>
dbg set expr <lvalue-expr> <value-expr>
```

If unsupported, return a clear error:

* “adapter does not support setVariable/setExpression”
* suggestion: use different adapter or restart with debug build.

---

## 13) Breakpoints

### 13.1 `dbg break` / `dbg b`

```
dbg break add <location> [--if <expr>] [--hit <n|expr>] [--log <fmt>] [--disabled]
dbg break del <@B:id|id>...
dbg break enable <@B:id|id>...
dbg break disable <@B:id|id>...
dbg break list [--show breaks=K|all]
dbg break clear [--all|--file <file>]
dbg break edit <@B:id|id> [--if ...] [--hit ...] [--log ...] [--enabled|--disabled]
```

**DAP-driven implementation constraint (important)**

* Source breakpoints are managed per-file; DBG maintains a canonical set and re-sends full sets when mutated.

**Output contract**

* `break list` groups by file:

  * each breakpoint shows id, enabled, location, condition/log/hit, verified/unverified if adapter reports.

Example:

```
# Breakpoints
## src/handler.rs
- @B:3  enabled  at src/handler.rs:201
  if: user_id == 91
  hits: 5  verified: yes

## src/repo.rs
- @B:8  disabled at src/repo.rs:55
  log: "load_user id={id}"
```

### 13.2 `dbg tbreak`

Temporary breakpoint (one-shot). Alias to `break add --once`.

```
dbg tbreak <location> [--if <expr>]
```

### 13.3 `dbg ibreak`

Instruction breakpoints.

```
dbg ibreak add <addr:0x...> [--if <expr>] [--hit ...] [--log ...]
dbg ibreak list
dbg ibreak del <@IB:id|id>...
```

Only available if adapter supports instruction breakpoints; otherwise explain.

### 13.4 `dbg watchpoint`

Data breakpoints (watchpoints).

```
dbg watchpoint add <expr|addr:...> [--access {read|write|readwrite}]
dbg watchpoint list
dbg watchpoint del <@DB:id|id>...
```

Capability-gated.

### 13.5 `dbg exception`

Exception breakpoints are adapter/language-specific; DBG exposes what the adapter advertises.

```
dbg exception list
dbg exception enable <filter-id>...
dbg exception disable <filter-id>...
dbg exception set [--break-on {thrown|uncaught|user-unhandled}]  (when supported)
```

Output includes filter ids and descriptions, plus what’s enabled.

---

## 14) Memory, registers, disassembly

### 14.1 `dbg regs`

Registers are typically exposed as a scope or adapter feature; DBG uses whatever is available.

```
dbg regs [--show regs=K|all]
```

If only available via scopes, DBG prints:

* “registers scope present”
* and renders it.

### 14.2 `dbg x` (gdb-like memory examine)

```
dbg x/<count><fmt> <addr-expr>
```

Examples:

* `dbg x/16x $rsp`
* `dbg x/64b addr:0x7ffeefbff000`

If adapter supports readMemory, DBG reads and formats.

Formats:

* `x` hex
* `d` signed decimal
* `u` unsigned decimal
* `b` bytes
* `c` chars (printable)
* `s` string (null-terminated best-effort, bounded)

### 14.3 `dbg disas`

```
dbg disas [<location|addr-expr>] [--show disasm=K|all]
```

Output:

* disassembly with an anchor marker at current instruction (if known)
* grounding includes address range
* elision with `--show disasm=all`

---

## 15) Logs and events

### 15.1 `dbg logs`

Captured stdout/stderr (typically via DAP `output` events).

```
dbg logs [--stdout|--stderr|--all] [--tail N] [--since <@E:n|time>]
```

Output is line-oriented with minimal decoration, plus grounding metadata in Summary.

### 15.2 `dbg events`

Session timeline: stops, continues, output markers, breakpoint changes, thread changes, adapter notices.

```
dbg events [--show events=K|all] [--kind <k>] [--since <@E:n|time>] [--follow]
dbg events show <@E:n>
```

`--follow` is “tail -f style”: prints new events until interrupted.

---

## 16) Snapshots and reports

### 16.1 `dbg report`

A richer case file than `dbg stop`, meant for “paste into an incident” or for agents to reason without follow-ups.

```
dbg report [--stop <@ST:n>]
```

Fixed sections:

1. Summary
2. Stop postcard (embedded)
3. Breakpoints (relevant subset: nearby + triggering bp)
4. Watches
5. Recent output (longer tail than stop by default)
6. Environment (cwd, args, adapter, launch mapping summary)
7. Notes + suggested next commands (minimal, concrete)

Budgets are still enforced with actionable elisions.

### 16.2 `dbg snapshot`

Persist a bundle of context to disk (for later review or sharing).

```
dbg snapshot save [--stop <@ST:n>] [--out <path>]
dbg snapshot list
dbg snapshot show <@SN:id>
```

Snapshot content is a rendered artifact (human-readable) plus optional machine-readable sidecar if configured.

---

## 17) `dbg info` compatibility layer

For gdb familiarity:

```
dbg info breakpoints   (alias of dbg break list)
dbg info threads       (alias of dbg threads)
dbg info frame         (alias of dbg frame)
dbg info locals        (alias of dbg locals)
dbg info args          (alias of dbg args)
```

---

## 18) `dbg batch`

Run multiple DBG commands from stdin or a file, like a gdb command file but using DBG subcommands.

```
dbg batch --file commands.dbg
dbg batch < commands.dbg
```

Rules:

* Each non-empty non-comment line is a `dbg` command **without** the leading `dbg`.
* Outputs are concatenated; each command’s artifact begins with its own preamble.
* Batch stops on first failure unless `--keep-going`.

---

## 19) `dbg dap` (power tools, not the main UX)

```
dbg dap caps
dbg dap trace on|off
dbg dap send <RequestName> --args <json>
```

This exists to debug adapters and DBG itself. It should not be required for normal use.

---

## 20) Heuristics and determinism

### 20.1 Deterministic ordering

To keep artifacts stable and diffable:

* Breakpoints: sorted by file, then line, then column.
* Threads: by threadId (but focused/stopped groups first).
* Frames: preserve adapter-provided order (stack truth).
* Locals: heuristic ordering within budgets (line-referenced first), then alphabetical for determinism.

### 20.2 Project root detection

Used for “nearest project frame” heuristics.

Order:

1. `--project-root`
2. `.dbg/config` value
3. git root (if detectable)
4. current working directory

### 20.3 Value rendering rules

* Truncate long values to `--max-value`.
* Render collections with:

  * count summary (length)
  * first K items (K derived from budgets/depth)
* Always print how to expand.

---

## 21) DAP integration rules (the “engine room”)

### 21.1 Handshake and configuration choreography

DBG agent must implement the DAP lifecycle:

* initialize request first
* then launch/attach
* wait for initialized event
* set breakpoints/exceptions/instruction breakpoints as configured
* send configurationDone (triggered by `dbg run` unless `dbg start --run`)

If an adapter doesn’t follow the choreography, DBG:

* continues best-effort,
* records a warning event,
* and avoids lying in output.

### 21.2 Capability gating

DBG never pretends a feature exists. If unsupported, commands:

* fail with a clear reason
* suggest alternatives where possible

### 21.3 Variable lifetime constraints

DBG treats live variable handles as stop-scoped:

* If running, variable expansion fails with “must be stopped”
* Snapshots preserve rendered values so you can inspect older stops without re-querying the adapter.

---

## 22) Configuration

### 22.1 Config files

Searched in order:

1. `./.dbg/config.toml`
2. `~/.config/dbg/config.toml`

### 22.2 Adapter profiles

Profiles provide:

* adapter command + transport
* mapping for `-- <program> [args]` into launch args
* defaults for budgets / show knobs per language if desired

Example shape (illustrative):

```toml
[project]
root = "."

[defaults]
max_value = 240
max_lines = 120

[adapters.myadapter]
command = ["path/to/adapter"]
transport = "stdio"

[profiles.default]
adapter = "myadapter"
launch_defaults = { stopAtEntry = true }
program_key = "program"
args_key = "args"
cwd_key = "cwd"
env_key = "env"
```

---

## 23) Exit codes

* `0` success
* `1` internal error
* `2` CLI usage error
* `3` session not found / cannot connect to agent
* `4` target state error (e.g., requires stop but running)
* `5` capability unsupported
* `6` ambiguous input (Alternatives printed)
* `7` timeout
* `8` adapter failure / disconnected unexpectedly
* `9` debuggee terminated unexpectedly (DBG still reports details in output)

---

## 24) Acceptance tests (golden artifacts)

These are snapshot-worthy, like RAQL “golden tests”:

1. **Stop postcard quality**

   * Stopped at breakpoint in project code
   * Shows source snippet with anchor
   * Shows stack and locals with elision knobs

2. **Nearest project frame**

   * Stop inside runtime/stdlib
   * Outputs both NOW frame and NEAREST PROJECT FRAME

3. **Stepping delta**

   * `dbg next` shows Changed locals/watches

4. **Breakpoint mutation correctness**

   * Multiple breakpoints across same file
   * Adding/removing one results in correct adapter breakpoint set replacement

5. **Exception stop**

   * Exception includes type/message in Notes

6. **Crash stop**

   * Signal includes brief regs/disasm snippets (when available)

7. **No prompts ever**

   * Ambiguity yields Alternatives list and exit code 6

---

## 25) What “instantly familiar” looks like in practice

A gdb/lldb user should be able to do this without thinking:

```bash
dbg start --adapter lldb -- ./app arg1
dbg b src/main.c:42
dbg run
dbg bt
dbg n
dbg p x
dbg c
```

…and each command prints enough context that there’s no “now I need a UI” moment.

---

If you want, I can also write a **help-text spec** (exact `dbg --help` and `dbg <cmd> --help` outputs) and a **golden-output suite** for `stop`, `bt`, `break list`, `threads`, `print`, `report`, including exact elision strings and default budgets. That tends to flush out the last 10% of ambiguity where tools get mushy.

