# Parslips — Driving the Editor from an Agent or External Tool

The Parslips Eclipse plugin (the editor for the Parsley template language) exposes HTTP
hooks so an external tool — a script, or an AI agent editing files outside Eclipse — can
close the edit→see-it-run loop without a human relaying anything by hand.

This is the reference: endpoints, parameters, response shapes, runtime differences and
setup. The doctrine — what to do when, and the traps — is in `SKILL.md`; it isn't
repeated here.

## Runtimes

The hooks work the same for both runtimes; two things differ:

- **Project type** — `project.base=ng` or `project.base=wo` in `build.properties`
  (selects component types, validation root, template format; absent → probed).
- **App-runtime endpoint URLs** — `log`, `eval` and `problems` live in the app runtime,
  not the plugin. On WebObjects/Wonder (wonder-slim's ERExtensions) the form is
  `…/<App>.woa/<endpoint>`; on ng-objects it's `…/ng/dev/<endpoint>`. Same parameters and JSON on
  both. `/apps` and `/status` report each app's `runtime` (`ng`/`wo`), so build the form
  from that rather than guessing.

## Dev server

`http://localhost:9485` — **loopback only**, no auth (loopback *is* the trust boundary).
Plain `GET`; responses are JSON, except `/console` (plain text), `/watch` (HTML) and a few
fire-and-forget endpoints that answer `ok`. Port is configurable in Eclipse prefs.

| Endpoint | Params | Does |
|---|---|---|
| `/` (or `/help`) | | Self-describing JSON index of every endpoint. Unknown paths answer with it too. |
| `/status` | `app?` | One entry **per launch config** (of the named project, or all): running/mode/uptime, `projectOpen`, `compileErrors`, `registered` port/pid/runtime + `reachable` (a TCP probe). Plus `dialogs`: open modal dialogs. |
| `/refreshProject` | `project?`, `build?`, `clean?` | Refresh project(s) from disk + incremental build. `ok` on a clean build, a JSON `buildErrors` report otherwise. |
| `/problems` | `project?`, `severity?`, `limit?` | Problem markers as JSON, grouped `projects[]` (each with `count`, `shown`, `problems[]`). |
| `/validate` | `component`, `project?` | Validate a component's template; problems as JSON. `found:false` carries a `reason`. |
| `/revalidate` | `project?` | Revalidate EVERY template in a project (or the workspace). Slow: generous timeout. |
| `/purgeMarkers` | `project?` | Delete orphaned untyped problem markers on js/css/html/xml (legacy-validator leftovers). Typed markers untouched. |
| `/elementApi` | `element` (name or comma-list), `project?`, `raw?` | An element's resolved binding API as JSON. `raw=true` returns the `.apiext` XML. |
| `/launch` | `config?`/`app?`, `mode?`, `open?`, `ignoreErrors?`, `allowMultiple?`, `waitForPort?`, `timeout?` | List configs, or start one with preflight and wait-until-ready. Never raises Eclipse's launch dialogs. |
| `/stop` | `app`, `force?` | Stop a running app (terminate, or `force=true` to hard-kill the registered pid). |
| `/restart` | `app`, `refresh?` (+ `/launch` params) | stop → wait for termination → refresh+rebuild named projects → launch. Per-stage results. |
| `/console` | `app`, `tail?` | The launch's console output (default last 100 lines), kept after the process dies. |
| `/dialogs` | `press?`, `close?`, `title?` | List Eclipse's open modal dialogs; `press=BUTTON` presses one; `close=true` closes the topmost; `title=TEXT` targets one. |
| `/breakpoints` | `skipAll?` | List workspace breakpoints; `skipAll=true/false` toggles Skip All Breakpoints. |
| `/openProject` | `project` (or `all`), `related?` | Open a closed project **plus its workspace dependencies** (transitive, pom-resolved), then clean-build. `related=false` for just the one. |
| `/apps` | `name?` | Running apps: port, `runtime`, pid, and the dependencies whose source is open in the workspace. `name` → one app. |
| `/activity` | `since?`, `clear?` | The dev server's request feed: every handled request with query, status, duration and (capped) response. `since=SEQ` is the poll cursor. |
| `/watch` | | A live spectator page (HTML) over `/activity`: narrated requests with a running tally. For the human's screen. |
| `/refresh` | `path` | Refresh one resource path. |
| `/openComponent` | `component`, `app?`, `lineNumber?`, `offset?`, `length?` | Open a component in the editor, revealing a line or a character range (`offset`/`length` as `/validate` reports them). |
| `/openJavaFile` | `className`, `lineNumber?`, `app?` | Open a Java file at a line. |
| `/registerApp` | `name`, `port`, `pid?`, `runtime?` | (App-side, automatic) An app announces itself at startup. You won't call this. |

Older plugin builds expose a subset — the index at `/` lists exactly what a build offers;
a plain `ok` from `/` means only the classic endpoints (`/refreshProject`, `/validate`,
`/launch`, `/stop`, `/apps`) exist.

### `/launch`, `/stop`, `/restart`

```bash
curl -s 'http://localhost:9485/launch'                                        # list configs: {"configs":[{"name":…,"project":…}]}
curl -s 'http://localhost:9485/launch?config=MyApp%20-%20Local'               # by exact config name
curl -s 'http://localhost:9485/launch?app=MyApp&waitForPort=1200&timeout=90'  # by project name; block until ready/dead/timeout
curl -s 'http://localhost:9485/launch?app=MyApp&mode=run'                     # run mode (default is debug)
curl -s 'http://localhost:9485/restart?app=MyApp&refresh=my-model&waitForPort=1200'
curl -s 'http://localhost:9485/stop?app=MyApp'                                # clean terminate
curl -s 'http://localhost:9485/stop?app=MyApp&force=true'                     # kill -9 the registered pid
```

**Resolution.** An exact config name wins; otherwise the query is a project name, and when
the project has several Java-application configs the one whose name contains `local`/`dev`
is preferred. If it still can't choose safely it launches nothing and returns
`{"launched":false,"reason":"ambiguous…","candidates":[…]}`. Only Java-application configs
count (Maven, JUnit etc. are ignored).

**Preflight** — each refusal names the reason and the override:

- project **closed** → `open=true` opens it and its workspace dependencies (transitively,
  pom-resolved — Maven workspace resolution only sees open projects), clean-builds, then
  launches; the response lists `opened`.
- **compile errors** in the project *or any project it depends on* (checked after the
  same incremental pre-launch build Eclipse does) → `errorProjects[]` with the first
  problems of each, and a `hint` naming the clean-rebuild call per project;
  `ignoreErrors=true` overrides.
- config **already running** → use `/restart`, or `allowMultiple=true`.

Launches are made with Eclipse's own prompting disabled, so no launch dialog can appear;
launch failures come back as JSON `error`. Unsaved editor buffers are not saved (the launch
reflects what's on disk).

**`waitForPort=N`** delays the response until the port answers (`"ready":true` with
`startupMillis`), the launched process terminates (`"ready":false`, `reason`, pointer at
`/console`), a modal dialog appears after the launch (`"reason":"blocked by a modal
dialog"` with the dialog), or `timeout` seconds (default 60) pass. Don't know the port yet?
Launch without it, poll `/status?app=NAME` until the running config's `registered.port`
appears, then use `waitForPort` from then on. Mode defaults to **debug** (hot-code-replace
needs it).

`/restart` composes stop → wait for actual termination → refresh+rebuild the projects in
`refresh=` → launch, reporting each stage; all `/launch` parameters pass through.

### `/apps` — port, runtime, and readable dependencies

```bash
curl -s 'http://localhost:9485/apps'                 # {"apps":[…]}
curl -s 'http://localhost:9485/apps?name=MyApp'      # {"found":true,"app":{…}} or {"found":false,…}
```

```json
{ "name":"MyApp","port":1200,"running":true,"runtime":"ng","lastSeen":1780611151951,"pid":"24067",
  "dependencies":[ {"name":"ERExtensions","path":"/Users/you/git/wonder-slim/ERExtensions",
                    "sourceFolders":["/Users/you/git/wonder-slim/ERExtensions/src/main/java"]} ] }
```

- `runtime` (`ng`/`wo`) selects the runtime endpoint form: `ng` → `…:<port>/ng/dev/<endpoint>`,
  `wo` → `…:<port>/cgi-bin/WebObjects/<App>.woa/<endpoint>` (`<App>` is the app's `name`;
  `<endpoint>` is `log`, `eval` or `problems`).
- `running` is a live TCP probe of the port, made when you call; dead entries are dropped
  (apps don't deregister). It says the port answers, not that the app is healthy.
- `dependencies` — the app's dependencies whose **source is open in the workspace**: the
  libraries you can read and fix. Jar-only dependencies are omitted. Computed live.

### `/refreshProject`

```bash
curl -s 'http://localhost:9485/refreshProject?project=MyApp'   # omit project → all open projects
```

`build` defaults true; `clean` defaults false and must stay false while the app runs (a
clean+full build yields no per-class delta for the swapper). Blocks until the build
settles. Returns `ok`, or a JSON report:

```json
{ "refreshed":true, "buildErrors":2,
  "projects":[ {"project":"MyApp","problems":[ {"resource":"src/main/java/…","line":42,"source":"java","message":"…"} ]} ],
  "hint":"the build settled but did NOT compile cleanly — the running app is still on the previous classes" }
```

### `/validate`

```bash
curl -s 'http://localhost:9485/validate?component=ASISearchPage&project=MyApp'
```

```json
{ "component":"ASISearchPage", "found":true,
  "files":["/MyApp/…/ASISearchPage.html"],
  "problems":[ {"severity":"error","line":17,"charStart":420,"charEnd":448,
                "message":"…","file":"/MyApp/…/ASISearchPage.html"} ] }
```

- Refreshes the component's files from disk first, so it sees your edit without a prior
  `/refreshProject` — but it does not make the edit take effect in the app.
- `project` narrows the search to that project; without it (or if the hint fails) every
  open project is searched.
- `"found":false` comes with a `reason`: the hinted project is **closed** (open it with
  `/openProject`; `/status` shows `projectOpen`), the hinted project **doesn't exist**, or
  **no open project has the component**.
- `severity` error/warning/info; `line` 1-based; `charStart`/`charEnd` pair with
  `/openComponent`'s `offset`/`length`. Works for `.wo` bundles and standalone `.html`.

### `/elementApi`

```bash
curl -s 'http://localhost:9485/elementApi?element=WOPopUpButton&project=MyApp'
curl -s 'http://localhost:9485/elementApi?element=str,WOTextField&project=MyApp'   # comma-list; aliases resolve
curl -s 'http://localhost:9485/elementApi?element=WOString&raw=true'               # the .apiext XML instead
```

```json
{ "elements":[
  { "requested":"str", "resolved":"WOString", "kind":"apiext",
    "api":{ "name":"WOString", "content":"forbidden", "unknownAttributes":"forbidden",
            "bindings":[ {"name":"value","required":true,"direction":"pull",
                          "pull":[{"type":"java.lang.Object"}],"push":[], "doc":"…"} ],
            "constraints":[ {"kind":"choose","max":1,"message":"at most one of 'formatter', 'dateformat' or 'numberformat' may be bound","alternatives":[…]} ] } } ] }
```

- Names resolve as a template resolves them: the project's tag aliases
  (`parsley-tag-aliases.properties`, recursive) and the classic shortcuts. `resolved` is
  what the name became; `kind` is `apiext` (rich), `api` (legacy) or `none` (no definition,
  `api` null).
- Per binding: `pull`/`push` arrays of `{type, interpretation?}`; `direction` is
  `pull`/`push`/`both`/`none`; plus `required`, `default`, `defaults`, `deprecated`.
  Constraints carry the generated human `message`.
- In ng-objects projects the definitions are ng-objects' own (`NGString`, `NGCheckbox`…),
  never the WebObjects element of the same name.

### `/console`

```bash
curl -s 'http://localhost:9485/console?app=MyApp&tail=200'
```

First line is a status header (`# config: MyApp - Local  state: terminated  exit: 1`), the
rest the raw console tail, stack traces included. One buffer per launch config, latest
launch wins, ~400k characters. The only source for startup failures and for an app that is
up but broken.

### `/status` and `/problems`

```bash
curl -s 'http://localhost:9485/status?app=MyApp'
curl -s 'http://localhost:9485/problems?project=MyApp&severity=warning&limit=50'
```

```json
{ "apps":[ {"config":"MyApp - Local","project":"MyApp","projectOpen":true,"compileErrors":0,
            "running":true,"mode":"debug","uptimeSeconds":2411,
            "registered":{"port":1200,"pid":"83933","runtime":"wo","reachable":true}},
           {"config":"MyApp - Production","project":"MyApp","projectOpen":true,"compileErrors":0,"running":false} ],
  "dialogs":[] }
```

`/status?app=NAME` matches every launch config of that project (or any config by exact
name), so read the entry with `running:true`. `compileErrors` counts Java errors only.
`reachable` is a TCP probe. `dialogs` lists open modal dialogs (`uiResponsive:false` is
added when the UI thread didn't answer within 3s).

```json
{ "projects":[ {"project":"MyApp","count":36,"shown":3,
                "problems":[ {"resource":"src/main/components/Main.wo/Main.html","line":2,"source":"parsley","message":"…"} ]} ] }
```

`/problems` is always grouped by project. `count` is the true total at the requested
severity; the list is capped by `limit` (its size in `shown`). `source` is `parsley`
(template validation), `java` (JDT), `stock` (untyped legacy leftovers — purgeable) or
`other`. Errors only by default; `severity=warning` includes warnings.

### `/revalidate` and `/purgeMarkers`

```bash
curl -s --max-time 120 'http://localhost:9485/revalidate?project=MyApp'   # {"projects":1,"components":28,"canceled":false}
curl -s --max-time 600 'http://localhost:9485/revalidate'                 # whole workspace — tens of seconds
curl -s 'http://localhost:9485/purgeMarkers'                              # {"deleted":N,"files":M}
```

Template validation is per-file and event-driven, so a Java rebuild never refreshes
template markers. `/revalidate` recomputes them all; `/purgeMarkers` deletes markers of the
exact untyped problem type (never Parsley, JDT or WTP markers) on js/css/html/xml files.
Suspect phantoms → `/problems` (read the `source` mix) → `/revalidate` → `/purgeMarkers`.

### `/dialogs`

```bash
curl -s 'http://localhost:9485/dialogs'                     # {"uiResponsive":true,"dialogs":[{"title":…,"message":…,"buttons":["Proceed","Cancel"]}]}
curl -s 'http://localhost:9485/dialogs?press=Proceed'       # case-insensitive; & mnemonics and "..." ignored
curl -s 'http://localhost:9485/dialogs?close=true'          # the Escape / window-close path
curl -s 'http://localhost:9485/dialogs?press=OK&title=Hot'  # the dialog whose title contains "Hot"
```

### `/breakpoints`

`curl -s 'http://localhost:9485/breakpoints'` lists workspace breakpoints (resource, line,
enabled) and the Skip All Breakpoints state; `?skipAll=true` disarms all, `?skipAll=false`
re-arms.

### `/activity` and `/watch`

```bash
curl -s 'http://localhost:9485/activity'              # {"lastSeq":N,"entries":[{"seq","time","path","query","status","millis","response","responseLength","truncated"}]}
curl -s 'http://localhost:9485/activity?since=120'    # only entries after seq 120
curl -s 'http://localhost:9485/activity?clear=true'   # empty the buffer
```

The last 500 requests the dev server handled, responses capped at 16k characters
(`truncated` says so; `responseLength` is the full size). Requests to `/activity` and
`/watch` are never recorded. Useful when taking over from another session: what was
validated, launched, refreshed, and what came back. `/watch` is the same feed as a live
narrated page for the developer's screen.

## Log endpoint

From the app runtime, **dev mode only**. Port and form from `/apps`:

```
http://localhost:<PORT>/cgi-bin/WebObjects/<App>.woa/log    # WebObjects/Wonder
http://localhost:<PORT>/ng/dev/log                          # ng-objects
```

```bash
curl -s '.../log?tail=50'
curl -s '.../log?contains=MYDEBUG&tail=30'   # contains is case-sensitive
```

Buffer: the last ~2000 lines, captured after logging init (early boot output isn't there);
one entry per event, stack traces included. Dies with the app — `/console` doesn't.

## Eval endpoint

From the app runtime, **dev mode + loopback only**. Evaluates a Java snippet inside the
running JVM, against its live classes, statics and data. Same URL form as `log`, with
`eval` in place of `log`.

```bash
curl -s '.../eval?snippet=1%2B1'
curl -s --data 'MyModel.newContext().performQuery(q).size()' -H 'Content-Type: text/plain' '.../eval'
curl -s '.../eval?reset=true&snippet=…'      # discard the persistent session first
```

```json
{ "status":"ok", "value":"2", "diagnostics":[] }
{ "status":"error", "exception":"java.lang.NumberFormatException: …", "diagnostics":[] }
{ "status":"error", "value":null, "diagnostics":["cannot find symbol …"] }
```

- Snippet from the `snippet` param or the request body **as `text/plain`** — a
  form-encoded body (curl's `--data` default) is split on `=` and `&`, which mangles Java.
- The session persists (`var ctx = …` then use `ctx`); `reset=true` wipes it. Default
  imports: `java.util.*`, `java.util.stream.*`, `java.time.*`.
- `System.out`/`err` go to the app console — read them via the log endpoint.
- A non-terminating snippet hangs its request; a wedged eval needs an app restart.

## Runtime-problems endpoint

From the app runtime, **dev mode only**: the binding-error boxes the app renders into
pages, as data. Same URL form as `log`, with `problems` in place of `log`.

```bash
curl -s '.../problems?tail=20'
curl -s '.../problems?contains=WORepetition'
curl -s '.../problems?clear=true'          # empty the buffer (mark a baseline)
```

```json
{ "problems":[ {"time":1783300000000,"kind":"Unknown key","element":"WOString","message":"…"} ], "count":1 }
```

Bounded to the last ~1000; a problem that renders every request is recorded every request.
The app-side complement to `/validate`: it catches the mistakes that only surface when a
request renders the tag.

## Template conventions

- **ng-objects** — usually **standalone**: one `.html` with inline bindings
  (`<wo:str value="$name" />`).
- **WebObjects/Wonder** — usually **bundle**: a `.wo` folder of `.html` + `.wod`
  (+ optional `.woo`), HTML using `<webobject name="X">` against named `.wod` entries.

## Setup (for the developer)

1. **Install the Parslips plugin** — Eclipse → *Help → Install New Software…*, add
   `https://undur.github.io/parslips/repository/`, pick the **Parslips** feature
   (currently listed as "Parsley Template Editor"), restart. Coexists with WOLips; set
   `project.base=wo` so Parslips wins on WO projects.
2. **Run the app from Eclipse in debug mode** (enables hot-code-replace).
3. **Recommended: enhanced reload** — run on **JBR** with
   `-XX:+AllowEnhancedClassRedefinition` + the **HotswapAgent** java agent; a
   `HOTSWAP AGENT: … reload` line confirms swaps. Without it, only method-body swaps.
4. **The app announces itself** — in dev mode the runtime registers its port and framework
   with the dev server at startup, so the agent discovers both through `/apps`; you don't
   have to tell it anything.
5. **Confirm:** `curl http://localhost:9485/` returns the endpoint index. Refused → plugin
   not loaded or wrong port (Eclipse prefs).
6. **Watch it work:** open `http://localhost:9485/watch` in a browser — a live, narrated
   feed of everything the agent asks the dev server to do.
