# Parslips — Driving the Editor from an Agent or External Tool

The Parslips Eclipse plugin (which edits the Parsley template language) exposes a
few HTTP hooks so an external tool — a script, or an AI agent editing files outside
Eclipse — can close the edit→see-it-run loop without a human relaying anything by
hand.

Written for whoever's driving them. If you're an agent in a WebObjects/ng-objects
project whose developer runs Eclipse with this plugin, this is your guide; humans,
see **Setup** at the end.

## Why

Edit a file **on disk** and Eclipse doesn't notice — it only tracks edits made
through its own editor. So the change sits there: no recompile, no revalidation, and
a running app keeps using stale `.class` files. Two HTTP surfaces fix that:

1. **Dev server** (this plugin, inside Eclipse) — refresh/rebuild, validate a
   template, open a component.
2. **Log endpoint** (the app's runtime) — read the running app's console.

Together: edit → refresh → app reloads → read the log; and validate a template on
disk before you ever render it.

## Runtimes

The hooks work the same for both runtimes; two things differ:

- **Project type** — `project.base=ng` or `project.base=wo` in `build.properties`
  (selects component types, validation root, template format; absent → probed).
- **App-runtime endpoint URLs** — the `log`, `eval` and `problems` endpoints live in the
  app runtime, not this plugin, and the URL form differs by runtime: on WebObjects/Wonder
  (wonder-slim's ERExtensions) it's `…/<App>.woa/<name>`; on ng-objects it's
  `…/ng/dev/<name>`. Same parameters and JSON shapes on both — only the mount path differs.
  **You don't have to guess which:** `/apps` (and `/status`) report each app's `runtime`
  (`ng`/`wo`), so read it from there and build the right form.

## Dev server

`http://localhost:9485` — **loopback only**, no auth (loopback *is* the trust
boundary). Plain `GET`; responses are JSON (`/console` is plain text; `/refreshProject`
and a few fire-and-forget endpoints answer `ok` on success). Port is configurable in
Eclipse prefs.

| Endpoint | Params | Does |
|---|---|---|
| `/` (or `/help`) | | Self-describing JSON index of every endpoint. Unknown paths answer with it too. |
| `/status` | `app?` | Ground truth per launch config: running/mode/uptime, project open state, compile errors, registered port/pid/runtime + reachability. |
| `/refreshProject` | `project?`, `build?`, `clean?` | Refresh project(s) from disk + incremental build. Returns `ok` on a clean build, a JSON `buildErrors` report otherwise. Use after editing. |
| `/problems` | `project?`, `severity?`, `limit?` | Problem markers as JSON; errors only by default. `count` is the true total (list capped by `limit`, size in `shown`); each entry carries a `source`: `parsley` (template validation), `java` (JDT), `stock` (purgeable legacy leftovers), `other`. |
| `/validate` | `component`, `project?` | Validate a component's template, return problems as JSON. |
| `/revalidate` | `project?` | Revalidate EVERY template in a project (or whole workspace) — the bulk cure for stale/phantom template markers (a Java clean/rebuild never refreshes them). Slow: use a generous timeout. |
| `/purgeMarkers` | `project?` | Delete orphaned untyped (exact-stock-type) PROBLEM markers on js/css/html/xml — legacy-validator leftovers nothing will ever revalidate. Typed markers (template validation, JDT, WTP) untouched. |
| `/elementApi` | `element` (name or comma-list), `project?`, `raw?` | The resolved binding API of one or more elements as JSON — bindings with pull/push types and direction, required/default/deprecation, constraints with generated messages, content/unknownAttributes policies. Alias-aware. `raw=true` returns the `.apiext` XML. |
| `/refresh` | `path` | Refresh one resource path. |
| `/apps` | `name?` | Discover running apps and their ports (so you don't have to be told the port). |
| `/launch` | `config?`/`app?`, `mode?`, `open?`, `ignoreErrors?`, `allowMultiple?`, `waitForPort?`, `timeout?` | List launch configs, or start one — with preflight (closed project, compile errors, already running each refuse with a named override) and optional wait-until-ready. |
| `/stop` | `app`, `force?` | Stop a running app (terminate, or `force=true` to hard-kill). |
| `/restart` | `app`, `refresh?` (+ `/launch` params) | stop → wait for termination → refresh+rebuild named projects → launch. Per-stage results. |
| `/console` | `app`, `tail?` | The launch's console output (default last 100 lines) — kept after the process dies; the startup-failure post-mortem. |
| `/breakpoints` | `skipAll?` | List workspace breakpoints; `skipAll=true/false` toggles the Skip All Breakpoints master switch. |
| `/openProject` | `project` (or `all`), `related?` | Open a closed project **plus its workspace dependencies** (transitive, pom-resolved). `related=false` for just the one. |
| `/openComponent` | `app?`, `component`, `lineNumber?`, `offset?`, `length?` | Open a component, reveal a position. |
| `/openJavaFile` | `className`, `lineNumber`, `app?` | Open a Java file at a line. |
| `/registerApp` | `name`, `port`, `pid?`, `runtime?` | (App-side, automatic) An app announces its port (and framework, `ng`/`wo`) at startup. You won't call this. |

(Older plugin builds lack `/`, `/status`, `/problems`, `/console`, `/restart`,
`/breakpoints` and the `/launch` preflight/wait parameters — probe `/` first; a plain
`ok` response means you're on an old build and should use the classic subset.)

### `/launch` and `/stop` — start and stop apps

Start an app without the developer doing it by hand:

```bash
curl -s 'http://localhost:9485/launch'                       # list configs: {"configs":[{"name":…,"project":…}]}
curl -s 'http://localhost:9485/launch?config=MyApp%20-%20Local'   # launch by exact config name
curl -s 'http://localhost:9485/launch?app=MyApp'             # launch by project name
curl -s 'http://localhost:9485/launch?app=MyApp&mode=run'    # run mode (default is debug)
curl -s 'http://localhost:9485/launch?app=MyApp&waitForPort=1200&timeout=90'   # block until ready/dead/timeout
```

**Preflight — `/launch` refuses instead of lying.** Each refusal names the reason and
the parameter that overrides it:

- Project **closed** in the workspace → `{"launched":false,"reason":"project … is closed"}`;
  pass `open=true` — it opens the project **and its workspace dependencies**
  (transitively, pom-resolved; the response lists what it opened). Opening just the
  target would only trade the refusal for build-path errors, since Maven workspace
  resolution only sees open projects.
- Project has **compile errors** → the errors are listed; fix them or pass `ignoreErrors=true`.
- Config **already running** → use `/restart` (or `allowMultiple=true` if you really
  mean a second instance).

**`waitForPort=N`** delays the response until something listens on port N
(`"ready":true` with `startupMillis`), the launched process terminates
(`"ready":false` — go read `/console`), or `timeout` seconds (default 60) pass.
Prefer it over hand-rolled polling: it also distinguishes "slow startup" from
"died at startup", which a port poll alone cannot.

**`/restart`** composes the full cycle — stop, wait for actual termination, refresh
and rebuild the projects named in `refresh=proj1,proj2`, then launch (all `/launch`
parameters pass through):

```bash
curl -s 'http://localhost:9485/restart?app=MyApp&refresh=my-model&waitForPort=1200'
```

Mode defaults to **debug** — the dev loop (hot-code-replace via JBR + DCEVM +
HotswapAgent) needs a debug JVM, so launch in debug unless you have a reason not to.

Only **Java application** launch configurations are considered (and listed) — Maven
builds, JUnit runs etc. in the workspace's config pool are ignored, even on an exact
name match, since "launching the app" means running its main class.

**Choosing the config matters.** A project often has several configs for different
environments (e.g. `MyApp - Local`, `MyApp - Production`). Resolution: an exact config
name wins; otherwise the query is treated as a project name, and when a project has
several configs it prefers one whose name contains `local`/`dev`. If it still can't
choose safely it launches **nothing** and returns the candidates — so you never
accidentally fire Production. If you get `{"launched":false,"candidates":[…]}`, pick an
exact name from that list.

Stop a running app:

```bash
curl -s 'http://localhost:9485/stop?app=MyApp'             # clean terminate (Eclipse, or graceful kill)
curl -s 'http://localhost:9485/stop?app=MyApp&force=true'  # kill -9 — for a wedged JVM (e.g. DCEVM choked on a big reload)
```

Default is a clean terminate via Eclipse (or a graceful `kill` of the registered pid if
Eclipse doesn't own the launch). `force=true` hard-kills the registered pid — reach for
it when a hot reload wedged the JVM and a clean stop won't take.

### `/apps` — discover an app's port and which dependencies you can read

Apps register their port with the dev server at startup, so you can look up where one
is running by name instead of being told the port or guessing:

```bash
curl -s 'http://localhost:9485/apps'                 # list of running apps
curl -s 'http://localhost:9485/apps?name=MyApp'      # {"found":true,"app":{…}} or {"found":false,…}
```

Each app entry looks like:

```json
{
  "name":"MyApp","port":1200,"running":true,"runtime":"ng","lastSeen":1780611151951,"pid":"24067",
  "dependencies":[
    {"name":"ERExtensions","path":"/Users/you/git/wonder-slim/ERExtensions",
     "sourceFolders":["/Users/you/git/wonder-slim/ERExtensions/src/main/java", …]},
    …
  ]
}
```

Three things you get from this:

- **`runtime`** — `"ng"` or `"wo"`, the app's framework, so you build the right runtime endpoint
  URL form directly: `"ng"` → `…:<port>/ng/dev/<name>`, `"wo"` → `…:<port>/cgi-bin/WebObjects/<name>.woa/<name>`.
  No probing or inferring it from the dependency list. (Absent for apps running an older build
  that doesn't announce it — fall back to inference then.)
- **Port** — combined with `runtime`, that's the whole log/eval/problems URL. `running` is a
  live reachability check done when you call `/apps` (a TCP probe of the port), so the list
  only shows apps that are actually up — dead entries are dropped, since apps don't deregister
  on shutdown. `lastSeen` (epoch-millis of the last startup announcement) is a recency hint.
- **`dependencies`** — the app's dependencies whose **source is open in the workspace**:
  their project name, on-disk path, and source folders. These are the libraries you can
  actually read and edit (e.g. to understand a framework's behavior, or fix a bug across
  the boundary). Jar-only dependencies are deliberately omitted — if it's not here, you
  don't have its source in this workspace. Computed live, so opening/closing a project in
  Eclipse changes the list without restarting the app.

### `/refreshProject` — make Eclipse pick up disk edits

```bash
curl -s 'http://localhost:9485/refreshProject?project=MyApp'   # omit project → all open projects
```

- `build` defaults true (`build=false` to skip).
- `clean` defaults false — **leave it.** Incremental builds yield the per-class delta
  hot-swap needs (same as an in-editor save); `clean=true` forces CLEAN+FULL, which
  *breaks* hot reload. Use it only to recover a corrupted build.

The call blocks until the build settles, so on return the new `.class` files exist.

**Refresh is mandatory for every edit:** any change to a project file — Java OR
template — needs `/refreshProject` to take effect. Assume nothing changed until you
refresh. (`/validate` checks a template but does not make it take effect.)

**Hot-swap reach:** this project runs the **JetBrains Runtime (JBR) with DCEVM +
HotswapAgent** (one combined stack), which reloads **structural changes too** —
new/removed methods and fields, signature changes, new classes, and constructor
changes all reload live on `/refreshProject`, no restart. So after refreshing, an
ordinary edit is live and you **don't** restart. Only heavy changes need a restart on
top of the refresh: **classpath changes** (new/updated dependency, `pom.xml`/build-path
edits), **project-structure changes** (new source folder/module), or the rare reload the
agent can't apply. A stock debug JVM without DCEVM+HotswapAgent is far more limited —
method bodies only, restart for any shape/constructor change — but that is not this
setup. If a refresh genuinely doesn't take (rare), a restart is the fallback, not the
default.

### `/validate` — catch template errors without rendering

Templates are long strings with weak type/definition discovery; mistakes often hide
until render time. This runs Parsley's validator on a named component and hands back
the problems.

```bash
curl -s 'http://localhost:9485/validate?component=ASISearchPage&project=MyApp'
```

```json
{ "component":"ASISearchPage", "found":true,
  "files":["/MyApp/.../ASISearchPage.html"],
  "problems":[ {"severity":"error","line":17,"charStart":420,"charEnd":448,
                "message":"…","file":"/MyApp/.../ASISearchPage.html"} ] }
```

- Refreshes from disk first — reflects your edits without a prior `/refreshProject`.
- Empty `problems` → clean. `"found":false` → name didn't resolve.
- `severity` error/warning/info; `line` 1-based; `charStart`/`charEnd` pair with
  `/openComponent`'s `offset`/`length`. Re-validating clears prior problems. Works
  for `.wo` bundles and standalone `.html`.
- **`/refreshProject` does not validate templates** (validation is editor-driven) —
  call `/validate` explicitly after a template edit.
- The `$`-in-a-plain-HTML-attribute trap is one of the things it catches: `style="$x"`
  on a raw `<div>` renders the literal `$x` (plain tags don't evaluate bindings) — a
  warning nudging you toward a `wo:` element/container. So `/validate` covers that
  mistake too, not just `wo:`-tag binding errors.

### `/elementApi` — an element's real bindings, as data

The authoritative answer to "what can I bind on this tag, and how" — the editor's own
resolved element API over HTTP, so you never reverse-engineer bindings from an element's
Java source.

```bash
curl -s 'http://localhost:9485/elementApi?element=WOPopUpButton&project=MyApp'
curl -s 'http://localhost:9485/elementApi?element=str,WOTextField&project=MyApp'   # comma-list; aliases resolve
curl -s 'http://localhost:9485/elementApi?element=WOString&raw=true'               # canonical .apiext XML instead
```

```json
{ "elements":[
  { "requested":"str", "resolved":"WOString", "kind":"apiext",
    "api":{ "name":"WOString", "content":"forbidden", "unknownAttributes":"forbidden",
            "bindings":[ {"name":"value","required":true,"direction":"pull",
                          "pull":[{"type":"java.lang.Object"}],"push":[], "doc":"…"} ],
            "constraints":[ {"kind":"choose","max":1,"message":"at most one of 'formatter', 'dateformat' or 'numberformat' may be bound","alternatives":[…]} ] } } ] }
```

- `element` is required — one name or a comma-separated list. `project`/`app` is an
  optional hint; without it, the first open project where the element resolves wins.
- Names resolve the way a template resolves them — through the project's tag aliases
  (`str` → `WOString` → `ERXWOString`) AND the classic tag shortcuts (`link` →
  `WOHyperlink`, `textfield` → `WOTextField`), so the tag you see in a template resolves.
  `resolved` reports what the name became; `kind:none` means it's genuinely undefined.
- Per binding: `pull`/`push` are arrays of `{type, interpretation?}` (interpretation is
  e.g. `"truthy"`); `direction` is the derived `pull`/`push`/`both`/`none`; plus
  `required`, `default`, `defaults`, `deprecated`.
- Constraints carry a generated human `message` (the same sentence the hover shows), so
  you don't decode the typed rule yourself.
- `kind`: `apiext` (rich), `api` (legacy, thinner), or `none` (no definition — `api` is
  null). `raw=true` returns the `.apiext` XML for `apiext` elements (null otherwise).

### `/console` — the Eclipse console, including post-mortem

The app's own log endpoint (below) only exists once the app is up. `/console` serves
the launch's console output captured by the plugin — and keeps it after the process
dies, which is precisely when you need it:

```bash
curl -s 'http://localhost:9485/console?app=MyApp&tail=200'
```

First line is a status header (`# config: MyApp - Local  state: terminated  exit: 1`),
the rest is the raw tail. Use it whenever a launch "succeeded" but nothing listens,
whenever `waitForPort` reports the process terminated, and for anything the app printed
before its logging/HTTP came up. One buffer per launch config, latest launch wins,
capped at ~400k characters.

### `/status` and `/problems` — workspace ground truth

```bash
curl -s 'http://localhost:9485/status?app=MyApp'      # running? mode? uptime? project open? compile errors? registered port reachable?
curl -s 'http://localhost:9485/problems?project=MyApp'  # the Problems view as JSON (errors by default; severity=warning for more)
```

`/status` without `app` covers every launch config — the one-call answer to "what is
this workspace running right now". `/problems` is the check to run when a refresh
reported build errors, or before concluding that an edit "did nothing".

### `/breakpoints` — the forgotten-breakpoint check

An app that is inexplicably slow or frozen **only when Eclipse-launched** may just be
sitting on a forgotten breakpoint. `curl -s 'http://localhost:9485/breakpoints'` lists
them (resource, line, enabled) plus the Skip All Breakpoints state;
`?skipAll=true` disarms them all non-destructively, `?skipAll=false` re-arms.

## Log endpoint

From the app runtime. **Dev mode only.** The URL form depends on the runtime:

```
http://localhost:<PORT>/cgi-bin/WebObjects/<App>.woa/log    # WebObjects/Wonder (wonder-slim ERExtensions)
http://localhost:<PORT>/ng/dev/log                          # ng-objects
```

Get `<PORT>` and `<App>` from `/apps` (above) rather than guessing — e.g.
`curl -s 'http://localhost:9485/apps'` returns each app's name and port.

```bash
curl -s '.../log?tail=50'
curl -s '.../log?contains=MYDEBUG&tail=30'   # contains is case-sensitive
```

Replaces "paste the console." Diagnose: add a unique marker
(`log.info("MYDEBUG …")`), refresh/restart, exercise the app, read back
`?contains=MYDEBUG`, then remove the marker.

Buffer: last ~2000 lines, captured after logging init (so early *boot* output isn't
there); one entry per event, stack traces included.

## Eval endpoint

From the app runtime. **Dev mode + loopback only.** Evaluates a Java snippet inside the
running JVM, against the app's live classes/statics/data — not a separate `jshell`. URL
form mirrors the log endpoint:

```
http://localhost:<PORT>/cgi-bin/WebObjects/<App>.woa/eval    # WebObjects/Wonder
http://localhost:<PORT>/ng/dev/eval                          # ng-objects
```

```bash
curl -s '.../eval?snippet=1%2B1'                                   # snippet as a query param
curl -s --data 'MyModel.newContext().performQuery(q).size()' -H 'Content-Type: text/plain' '.../eval'   # POST the body as text/plain
curl -s '.../eval?reset=true&snippet=…'                           # discard the persistent session first
```

```json
{ "status":"ok", "value":"2", "diagnostics":[] }
{ "status":"error", "exception":"java.lang.NumberFormatException: …", "diagnostics":[] }
{ "status":"error", "value":null, "diagnostics":["cannot find symbol …"] }
```

- The snippet comes from the `snippet` param, or the request body **sent as
  `text/plain`**. A form-encoded body (curl's `--data` default) gets split on `=` and
  `&` — which mangles most real Java — so pass `-H 'Content-Type: text/plain'` when
  POSTing the body. (On ng, an accidental form-encoded body returns a clear error
  telling you this; on WO the raw body survives, but text/plain is the portable idiom.)
- **Persistent session**: `var ctx = …` in one call, use `ctx` in the next. `reset=true`
  wipes it. Default imports: `java.util.*`, `java.util.stream.*`, `java.time.*`.
- `System.out`/`err` from a snippet go to the app console — read them via the log endpoint.
- The point: verify logic against the app's *own* objects and a real data context (a live
  Cayenne `ObjectContext`, the running app singleton) instead of reconstructing them.
- Loopback-only (arbitrary code execution). A non-terminating snippet hangs its request;
  a wedged eval needs an app restart.

## Runtime-problems endpoint

From the app runtime. **Dev mode only.** The binding-error boxes the app renders into
pages (🐶 ng / 🌿 Parsley), as data instead of scraped HTML:

```
http://localhost:<PORT>/cgi-bin/WebObjects/<App>.woa/problems    # WebObjects/Wonder
http://localhost:<PORT>/ng/dev/problems                          # ng-objects
```

```bash
curl -s '.../problems?tail=20'
curl -s '.../problems?contains=WORepetition'
curl -s '.../problems?clear=true'          # empty the buffer (mark a baseline)
```

```json
{ "problems":[ {"time":1783300000000,"kind":"Unknown key","element":"WOString","message":"…"} ], "count":1 }
```

- `contains=`/`tail=` filter like the log endpoint. `clear=true` empties the buffer.
- Snapshot-then-`clear`, exercise the app, read back → only the errors that exercise
  produced. The app-side complement to `/validate`: `/validate` catches template mistakes
  statically in the editor; this catches the ones that only surface when a request renders.
- Bounded to the last ~1000 problems; a problem that renders every request is recorded
  every request (that repetition is signal — it's still live).

## A loop

```bash
curl -s '.../9485/validate?component=SomeComponent&project=MyApp'   # check the template
curl -s '.../9485/refreshProject?project=MyApp'                     # rebuild Java (incremental)
curl -s '.../MyApp.woa/log?contains=MYDEBUG&tail=40'               # read the result
```

No change after a refresh? First give the class redefinition a beat (~2-3s) and
re-exercise — the build settling and the JVM swap landing are two moments. Still
nothing → re-run the refresh; restart only for classpath/project-structure changes.

## Template conventions

- **ng-objects** — usually **standalone**: one `.html` with inline bindings
  (`<wo:WOString value="$name" />`).
- **WebObjects/Wonder** — usually **bundle**: a `.wo` folder of `.html` + `.wod`
  (+ optional `.woo`), HTML using `<webobject name="X">` against named `.wod` entries.

## Setup (for the developer)

1. **Install the Parslips plugin** — Eclipse → *Help → Install New Software…*, add
   `https://undur.github.io/parslips/repository/`, pick the **Parslips** feature
   (currently listed as "Parsley Template Editor" in the install dialog), restart.
   (Coexists with WOLips; set `project.base=wo` so Parslips wins on WO projects.)
2. **Run the app from Eclipse in debug mode** (enables hot-code-replace).
3. **Recommended: enhanced reload** — run on **JBR** with
   `-XX:+AllowEnhancedClassRedefinition` + the **HotswapAgent** java agent; a
   `HOTSWAP AGENT: … reload` line confirms swaps. Without it, only method-body swaps.
4. **Log endpoint** — automatic in dev mode from the app runtime. Share the app's
   port and `.woa` name with the agent.
5. **Confirm:** `curl http://localhost:9485/` should return the endpoint index. Refused →
   plugin not loaded or wrong port (check Eclipse prefs).
