---
name: parslips-dev-loop
description: >-
  Use this skill after you edit any WebObjects, ng-objects, Wonder, or Parsley
  file on disk to make the change take effect in the running app. Invoke
  immediately after modifying .java, .html, or .wod files in WO/ng components —
  Eclipse doesn't see disk edits made outside its editor, so the change won't take
  effect until you trigger a refresh through this skill. Also use when: the
  browser still shows old behavior after your edit; you want to validate a
  template for errors without rendering it; you added debug logging and want to
  read what printed; you need to check whether a Java change hot-swapped or
  requires a restart; you need to know an app's port or which of its dependencies
  have source open in the workspace; or changes "aren't reflecting" in the running
  app. The skill
  drives the Parslips Eclipse plugin's dev server (HTTP on localhost:9485) and the
  running app's log endpoint to refresh, rebuild, validate templates, and fetch
  console output — closing the edit→build→run loop without manual steps or asking
  the human to paste the console. Triggers on projects with .wo component bundles,
  <wo:...> or <webobject> template tags, build.properties with project.base,
  WOComponent/NGComponent classes, or wonder-slim/ERExtensions.
---

# Parslips Dev Loop

You're editing a WebObjects / ng-objects project whose developer runs Eclipse with
the **Parslips** plugin (which edits the Parsley template language). Parslips
exposes HTTP hooks that let you — an agent editing files on disk — make changes
take effect, validate templates, and read the app's console, without the human
relaying anything by hand.

## The core problem

When you edit **any file in the project** on disk — Java source, templates (`.html`/`.wod`), anything — Eclipse doesn't notice, because it only tracks edits made through
its own editor. So your change sits there and the running app keeps using the old
state. From the outside your edit appears to do nothing.

**So the rule is simple and absolute: after editing ANY project file, run
`/refreshProject`. Always — templates and Java alike. Assume nothing changes until
you've refreshed.** This is the one habit that makes the whole loop work; the most
common mistake is skipping the refresh for a "mere template edit" and then wondering
why the page looks unchanged.

## When to do what

| You just… | Do this |
|---|---|
| Edited a template (`.html`/`.wod`) | `GET /refreshProject?project=NAME` (**always**), then `GET /validate?component=NAME` to catch errors |
| Edited a Java class | `GET /refreshProject?project=NAME` — refresh + incremental build (reloads live; see below) |
| Edited **any** project file | `GET /refreshProject` — no exceptions; assume nothing changed until you do |
| Want to know what's running | `GET /status` (or `?app=NAME`) — running/mode/uptime, project open state, compile errors, registered port |
| Need the app's port, framework, or which deps you can read | `GET /apps` — running apps + their `runtime` (`ng`/`wo`, so you build the right endpoint URL), port, and source-available dependencies |
| Need to start an app | `GET /launch?app=NAME&waitForPort=PORT` — blocks until it answers or provably failed |
| Workspace cold (projects closed) | `GET /launch?config=NAME&open=true&waitForPort=PORT&timeout=300` — opens the project + its workspace dependencies, clean-builds, launches, waits |
| Need to restart (classpath change, wedged reload) | `GET /restart?app=NAME&refresh=PROJ1,PROJ2&waitForPort=PORT` — the whole stop/refresh/launch/wait cycle in one call |
| Launch/startup failed, or app died | `GET /console?app=NAME&tail=200` — the Eclipse console, readable even after the process died |
| Suspect compile errors | `GET /problems?project=NAME` — the Problems view as JSON |
| App inexplicably slow/frozen only under Eclipse | `GET /breakpoints` — a forgotten breakpoint on a hot class; `?skipAll=true` disarms them all |
| Need to see what the app logged | `GET …/<App>.woa/log` (WO) or `…/ng/dev/log` (ng), `?contains=…&tail=…` (port from `/apps`) |
| Need an element's real bindings/types/constraints | `GET /elementApi?element=NAME&project=NAME` — the editor's own resolved API as JSON; beats reading the element's Java source |
| Want to run code inside the live app (inspect real objects/data) | `GET …/<App>.woa/eval` (WO) or `…/ng/dev/eval` (ng), `?snippet=…` — a REPL in the running JVM (loopback only) |
| Need the runtime binding errors the app rendered | `GET …/<App>.woa/problems` (WO) or `…/ng/dev/problems` (ng) — the inline error boxes as JSON, no HTML-scraping |
| Aren't sure the dev server is up / what it offers | `GET /` — self-describing JSON index of all endpoints |

(If `/status` or `/` return 404-ish "unknown" responses, the developer is running an
older plugin build without these endpoints — fall back to `/refreshProject` as the
probe, `/launch`+port-polling for starts, and the app's own log endpoint for output.)

## Always probe first

Before relying on the loop, confirm the dev server answers:

```bash
curl -s http://localhost:9485/
```

A JSON endpoint index → you're good (and the index tells you exactly what this plugin
build offers — trust it over this document when they disagree). Plain `ok` → an older
plugin build without the index; the core endpoints (`/refreshProject`, `/validate`,
`/launch`, `/stop`, `/apps`) still work. Connection refused → **stop and tell the
human** Eclipse isn't running or the plugin isn't loaded (don't keep retrying or work
blind). The dev server is loopback-only with no auth, so all calls are plain local
`curl`.

## Know your source reach

`GET /apps?name=APP` returns, besides the port, a `dependencies` array — the app's
dependencies whose **source is open in the workspace**, each with its on-disk `path`
and `sourceFolders`. These are the libraries you can actually read and edit; jar-only
dependencies are omitted. So when a question crosses into a framework or dependency
(why does `ERExtensions` do X, is the bug in `helium5` or here), check `/apps` first:
if it's listed, read its real source at the given path instead of guessing from
behavior — and you can fix bugs across that boundary, not just in the app. If it's not
listed, you don't have its source in this workspace; say so rather than inventing it.

## Look up an element's API instead of reading its source

When you need to know an element's real bindings — their types, which way they flow
(pull/push), what's required, the cross-binding constraints — don't read the element's
Java to reverse-engineer it. Ask the editor, which already knows:

```bash
curl -s 'http://localhost:9485/elementApi?element=WOPopUpButton&project=MyApp'
# multiple at once:
curl -s 'http://localhost:9485/elementApi?element=WOString,WOTextField&project=MyApp'
```

Each element comes back as JSON: `bindings` (with `pull`/`push` types, a `direction` of
`pull`/`push`/`both`/`none`, `required`, `default`, `deprecated`), `constraints` (with
their generated human `message`, e.g. *"exactly one of 'checked' or 'value' must be
bound"*), and the `content`/`unknownAttributes` policies. Names resolve the way a template
resolves them — through the project's tag aliases (`str` → `WOString` → `ERXWOString`)
**and** the classic tag shortcuts (`link`, `textfield`, `string`) — so you can type the tag
you see in a template. The `resolved` field tells you what the name actually became. Add
`raw=true` to get the canonical `.apiext` XML instead of the interpreted view.

This is the editor's hover, as data — the authoritative answer to "what can I bind on
this tag", without opening a file. (`kind` tells you where it came from: `apiext`, legacy
`api`, or `none` when the element has no definition.)

## Validate templates — your highest-value habit

Templates are long strings with weak type/definition discovery, so mistakes are
easy and often stay hidden until the page renders. After editing a template,
validate it:

```bash
curl -s 'http://localhost:9485/validate?component=ASISearchPage'
# add &project=MyApp when a name might be ambiguous across projects
```

Returns JSON: `problems` empty → clean; each problem has `severity`, `line`
(1-based), `charStart`/`charEnd`, `message`, `file`. `"found":false` → the name
didn't resolve (check spelling/project). **`/refreshProject` does not validate
templates** (validation is editor-driven) and `/validate` does not make your template
edit take effect in the running app — they're separate. So after a template edit do
**both**: `/refreshProject` (so the change takes effect) **and** `/validate` (to catch
mistakes before rendering).

## Refresh + build after a Java edit

```bash
curl -s 'http://localhost:9485/refreshProject?project=MyApp'   # omit project → all open projects
```

Keep the default **incremental** build — it produces the per-class delta hot-swap
needs (same as an in-editor save). **Never pass `clean=true` while the app is
running**: a clean+full rebuild doesn't produce that delta, so the running app
stops picking up changes (restart to recover). Clean has exactly two homes: build
recovery, and freshly opened projects — where `/openProject` and `/launch?open=true`
already do it for you, safely, because nothing is running yet. The call blocks until
the build settles, so on return the new classes exist.

**Check the response.** Plain `ok` = the build settled clean. A JSON body with
`buildErrors` = your edit did NOT compile — the app is still running the previous
classes, and exercising it now tests nothing. Fix the listed problems (or
`GET /problems?project=NAME` for the full list) before drawing any conclusions.

**The build settling is not the swap landing.** `ok` means the .class files exist;
the JVM's actual class redefinition follows a beat later. If you exercise the app
in the same breath as the refresh, a Java change can appear "not to have taken".
Wait ~2-3s after the refresh before exercising Java changes (template changes
don't race — they're read per render); if a change still doesn't show, re-exercise
once before concluding the swap failed.

**Never verify or "fix" the loop with Maven.** Eclipse resolves inter-project
dependencies **within the workspace** (open projects reference each other's source
directly), so the local Maven repository plays no part in this dev cycle. A `mvn
compile` that fails against a **stale installed jar** (e.g. a sibling project's
artifact missing a newly added class) is a false alarm — Eclipse builds fine, and
the answer is `GET /problems?project=NAME`, not `mvn install`. Installing artifacts
to "fix" such an error wastes time and hides that the workspace was never broken.
Maven is for release/CI builds, not the edit→run loop.

### What reloads vs. what needs a restart

First: **you must `/refreshProject` regardless** — reloading is what a refresh *does*,
so nothing (Java or template) takes effect without it. The question here is only
whether, after the refresh, the change reloads into the running app or additionally
needs a restart.

**This dev environment is the JetBrains Runtime (JBR) with DCEVM + HotswapAgent**
(they run together as one stack). That combination reloads far more than a stock JVM:
not just method bodies but **structural changes too** — new/removed methods and fields,
changed signatures, new classes, constructor changes — all reload live on
`/refreshProject`, no restart. Treat the default assumption as **"refresh, and it
reloads; don't restart."**

Only genuinely heavy changes need an app restart on top of the refresh:
- **classpath changes** (new/updated dependency, changed `pom.xml`/build path)
- **project-structure changes** (new source folder, module layout)
- the occasional very complex reload the agent can't apply cleanly

A stock JVM (plain debug mode, without DCEVM+HotswapAgent) is much more limited — method
bodies only, restart for any shape/constructor change — but that is not this setup. Here,
assume JBR+DCEVM+HotswapAgent and don't restart for ordinary edits. If a refresh genuinely
doesn't take (rare), a restart is the fallback — not the default.

## Start and stop apps

Restarts are rarely needed — after the mandatory `/refreshProject`, JBR+DCEVM+HotswapAgent
reloads both template and structural Java changes live. But when one genuinely is
needed — a classpath or project-structure change, or the app isn't running yet — you
can do it yourself rather than asking the human:

```bash
# one call for the whole cycle: stop, wait for termination, rebuild projects, launch, wait for ready
curl -s 'http://localhost:9485/restart?app=MyApp&refresh=my-model,MyApp&waitForPort=1200'

# or the pieces individually:
curl -s 'http://localhost:9485/stop?app=MyApp'                          # clean terminate
curl -s 'http://localhost:9485/launch?app=MyApp&waitForPort=1200'       # start (debug mode) and block until ready
```

With `waitForPort` the response only comes back once the app answers on that port
(`"ready":true` + `startupMillis`), the process died (`"ready":false` with a pointer
at `/console` — go read the startup failure there), or the timeout (default 60s)
passed. **No more hand-rolled polling loops.**

Don't know the port? It lives in the launch config's arguments, so you can't read it
up front. Launch **without** `waitForPort`, then poll `/status?app=NAME` — the app
registers its port with the dev server at startup, and `registered: {port, reachable}`
appearing (reachable true) is your readiness signal. `/console` also shows it (WO apps
print `WOPort=N` early in startup). Once known, use `waitForPort` from then on.

`/launch` refuses — with a named reason and the parameter that overrides it — instead
of pretending to succeed, when:
- the **project is closed** in the workspace → pass `open=true`, which opens the
  project **and its workspace dependencies** (transitively, pom-resolved — Maven
  workspace resolution only sees open projects, so opening just one isn't enough)
  and clean-builds them (freshly opened projects need a clean: their persisted
  build state is stale and an incremental build no-ops); `/openProject?project=NAME`
  (or `all`) does the same standalone. So a completely cold workspace to a running
  app is ONE call: `/launch?config=NAME&open=true&waitForPort=PORT&timeout=300`.
- the project has **compile errors** → fix them (see the listed problems) or `ignoreErrors=true`
- a launch of the config is **already running** → use `/restart`, or `allowMultiple=true`

`/launch` with no arg lists the configs. **Be careful which config you start:** a
project often has several (e.g. `MyApp - Local`, `MyApp - Production`). The endpoint
prefers a `local`/`dev` config and refuses to guess when it's ambiguous — if you get
`{"launched":false,"candidates":[…]}`, pick an exact name; **don't** blindly launch
something that might be Production. If a hot reload wedged the JVM and a clean stop
won't take, `/stop?app=MyApp&force=true` hard-kills it.

## Startup failures: read the Eclipse console

The app's own log endpoint only exists once the app is up — a launch that dies during
startup is invisible there. The plugin captures every launch's console, and keeps it
after the process dies:

```bash
curl -s 'http://localhost:9485/console?app=MyApp&tail=200'
```

First line is a status header (`# config: … state: running|terminated exit: N`), the
rest is the raw console tail — stack traces and all. This is the post-mortem for
"launched":true-but-nothing-listening, and the place to look whenever `waitForPort`
reports the process terminated.

## Read the app's log

(Not the same as `/console` above: the **log endpoint** lives in the running app,
is filterable with `contains=`, and dies with the app; `/console` is the raw Eclipse
console, unfiltered, and survives the app's death. Debugging markers → log endpoint;
startup failures → `/console`.)

The running app serves its recent log over HTTP in dev mode. The URL form depends on
the runtime — same parameters on both:

```bash
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/log?contains=MYDEBUG&tail=40'   # WebObjects/Wonder
curl -s 'http://localhost:1200/ng/dev/log?contains=MYDEBUG&tail=40'                         # ng-objects
```

Discover the app's port, name and framework yourself from the dev server's registry —
apps announce themselves at startup, so `curl -s 'http://localhost:9485/apps'` returns
each app's `name`, `port`, and `runtime` (`ng`/`wo`); build the log URL from that (the
`runtime` tells you which form — `…/ng/dev/log` vs `…woa/log` — so you don't guess). To diagnose: add a
uniquely greppable marker
(`log.info("MYDEBUG …")`), refresh/restart, exercise the app, read back with
`contains=MYDEBUG`, then remove the marker. This replaces asking the human to paste
the console. Buffer is the last ~2000 lines, captured after logging init (so early
boot output isn't there).

## Run code inside the running app (`eval`)

The app serves an `eval` endpoint in dev mode that evaluates a Java snippet **inside its
own JVM**, against its real live classes, statics and data — a REPL in the running
process, not a separate `jshell`. Same URL shapes as `log`:

```bash
# WebObjects/Wonder:
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/eval?snippet=1%2B1'
# ng-objects:
curl -s 'http://localhost:1200/ng/dev/eval?snippet=1%2B1'
# a real question — POST the snippet as the body when it's long. Send it as text/plain:
# most real Java contains = and &, which a form-encoded body (curl's --data default) gets
# shredded on. text/plain is the reliable idiom.
curl -s --data 'MyModel.newContext().performQuery(q).size()' -H 'Content-Type: text/plain' 'http://localhost:1200/.../eval'
```

Response is JSON: `{"status":"ok","value":"…","diagnostics":[]}`, or `status:"error"` with
an `exception` or compiler `diagnostics`. The session is **persistent** — `var ctx = …`
in one call, use `ctx` in the next; `reset=true` starts fresh. Printed output
(`System.out`) goes to the app's console, so read it back via the `log` endpoint. This is
how you verify logic against the app's *own* objects and a live data context instead of
reconstructing them — e.g. check what a keypath actually returns, or what a query yields.

Restricted to loopback clients (it's arbitrary code execution) and dev mode only. A
snippet that loops forever hangs its request — if you wedge it, the app needs a restart.

## Read the runtime binding errors (`problems`)

When a template binding fails at runtime, the app renders an inline error box into the
page (the 🐶/🌿 boxes). Instead of scraping rendered HTML for them, read them as data:

```bash
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/problems?tail=20'   # WebObjects/Wonder
curl -s 'http://localhost:1200/ng/dev/problems?contains=WORepetition'           # ng-objects
```

JSON: `{"problems":[{"time","kind","element","message"}],"count"}`. `contains=`/`tail=`
filter like `log`; `clear=true` empties the buffer — snapshot-then-clear to mark a clean
baseline, exercise the app, then read back only the errors that exercise produced. This is
the app-side complement to `/validate` (which catches template mistakes statically, in the
editor): `problems` catches the ones that only surface when a real request renders the tag.

## A full iteration

```bash
# ANY project edit (template or Java) → refresh first, always — nothing takes effect otherwise
curl -s 'http://localhost:9485/refreshProject?project=MyApp'
# template edit → also validate to catch mistakes before rendering
curl -s 'http://localhost:9485/validate?component=SomeComponent&project=MyApp'
# Java edit → give the class redefinition a beat to land before exercising
sleep 2
# exercise the app, then read what it logged
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/log?contains=MYDEBUG&tail=40'
```

## More detail

`references/endpoints.md` has the full endpoint list (`/openComponent`,
`/openJavaFile`, `/refresh`), parameter tables, runtime differences (ng vs WO),
template-format conventions, and the human-side setup (plugin install, JBR +
HotswapAgent). Read it when you need an endpoint not covered above or are helping
the developer set things up.
