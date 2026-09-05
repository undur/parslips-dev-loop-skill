---
name: parslips-skill
description: >-
  Use this skill in WebObjects, ng-objects, Wonder or Parsley projects whenever the
  running app has to see a change, or you need to see the running app. That means:
  after editing any .java, .html or .wod file on disk (Eclipse doesn't notice disk
  edits, so nothing takes effect until you refresh through this skill); when the
  browser still shows old behavior after your edit; to validate a template for errors
  without rendering it; to start, stop or restart the app; to find out what is running
  and on which port; to read what the app logged, or what it printed while starting;
  to check whether a Java change hot-swapped or needs a restart; to look up an
  element's real bindings ("what can I bind on WOPopUpButton?"); to run a Java snippet
  inside the live app; or to learn which dependencies have source open in the
  workspace. It drives the Parslips Eclipse plugin's dev server (HTTP on
  localhost:9485) and the app's own dev endpoints (log, eval, problems). Triggers on
  projects with .wo component bundles, <wo:...> or <webobject> template tags,
  build.properties with project.base, WOComponent/NGComponent classes, or
  wonder-slim/ERExtensions.
---

# Parslips

You're editing a WebObjects / ng-objects project whose developer runs Eclipse with the
**Parslips** plugin (the editor for the Parsley template language). Parslips exposes HTTP
hooks that let you — an agent editing files on disk — make changes take effect, validate
templates, launch and observe the app, and read its output, without the human relaying
anything by hand.

## The two rules

Eclipse only tracks edits made through its own editor. A file you change on disk sits
unnoticed: no recompile, no resource copy, and the running app keeps its old state. From
the outside your edit appears to do nothing.

**After editing ANY project file, run `/refreshProject`. Templates, `.wod` files and
static resources too, not just Java.** Templates and resources aren't compiled, but the
app reads them from the output folder (`target/classes`), which Eclipse only refreshes
when it notices a change — and a disk edit is never noticed until you refresh. Assume
nothing has changed until you have.

**Hand the workspace back settled.** The moment you stop, the human will open a
component, run the app, or launch it from Eclipse — and they assume the workspace
matches the disk. So before you finish a task, pause for their input, or report back,
run `/refreshProject?project=NAME` for **every project you touched** — a dependency you
edited counts as much as the app; refreshing only the app leaves the dependency unbuilt —
and confirm each answers `ok`. Then `/problems?project=NAME` on those projects. Never
hand back a mid-state: unrefreshed edits, half-copied resources, an unbuilt change, or
errors you meant to fix later. If you stopped an app that was running when you started,
start it again. If you must leave something broken, say so in your report, so the human
doesn't have to do the close/clean/rebuild dance to find out.

## When to do what

| You just… | Do this |
|---|---|
| Edited **anything** in the project | `GET /refreshProject?project=NAME` — always, first. Check the response (below). |
| Finished, pausing, or reporting back | `GET /refreshProject?project=NAME` for **every** project you touched, each answering `ok`, then `GET /problems?project=NAME` on them — leave the workspace settled for the human (rule two) |
| Edited a template (`.html`/`.wod`) | …then `GET /validate?component=NAME` — refresh makes it take effect, validate catches mistakes; they're separate |
| Want to know what's running | `GET /status?app=NAME` — one entry **per launch config** of the project; read the one with `running:true` |
| Need the app's port, framework, or readable dependencies | `GET /apps?name=NAME` — port, `runtime` (`ng`/`wo`, picks the endpoint URL form), and the dependencies whose source is open in the workspace |
| Need to start the app | `GET /launch?app=NAME&waitForPort=PORT` — blocks until it answers or provably failed |
| Workspace cold (projects closed) | `GET /launch?config=NAME&open=true&waitForPort=PORT&timeout=300` — opens the project + its workspace dependencies, clean-builds, launches, waits |
| Need a restart (classpath change, broken swap, wedged reload) | `GET /restart?app=NAME&refresh=PROJ1,PROJ2&waitForPort=PORT` — stop, rebuild, launch, wait, in one call |
| Launch refused for compile errors in a *dependency* | Run the refusal's `hint` (`/refreshProject?project=DEP&clean=true`), then retry — stale build state is the usual cause |
| Launch/startup failed, app died, or app answers 500 to everything | `GET /console?app=NAME&tail=200` — the Eclipse console, kept after the process dies |
| A call hangs, or a wait ends `blocked by a modal dialog` | `GET /dialogs` — Eclipse's modal dialogs (title, message, buttons); `?press=BUTTON` answers one. **Check this before "fixing" anything** |
| Suspect compile errors | `GET /problems?project=NAME` — the Problems view as JSON (`count` is the true total; entries carry a `source`) |
| App slow/frozen only under Eclipse | `GET /breakpoints` — a forgotten breakpoint; `?skipAll=true` disarms them all |
| Need what the app logged | `GET …/<App>.woa/log` (WO) or `…/ng/dev/log` (ng), `?contains=…&tail=…` — port and runtime from `/apps` |
| Need an element's real bindings | `GET /elementApi?element=NAME&project=NAME` — the editor's resolved API as JSON; don't reverse-engineer the Java |
| Want to run code in the live JVM | `GET …/<App>.woa/eval` (WO) or `…/ng/dev/eval` (ng), `?snippet=…` — a REPL inside the running app |
| Need the binding errors the app rendered | `GET …/<App>.woa/problems` (WO) or `…/ng/dev/problems` (ng) — the inline error boxes as JSON |
| Markers look stale (survive rebuilds) | `GET /revalidate?project=NAME` re-validates every template; `GET /purgeMarkers` removes orphaned legacy markers |
| Picking up after another session | `GET /activity` — every request the dev server handled, with responses; the human can watch live at `/watch` |
| Unsure what this build offers | `GET /` — the self-describing index; trust it over this document when they disagree |

## Probe first

```bash
curl -s http://localhost:9485/
```

A JSON index → you're good. Connection refused → **stop and tell the human**: Eclipse
isn't running or the plugin isn't loaded; don't retry blindly. (A plain `ok` means an
old plugin build with only the classic endpoints — `/refreshProject`, `/validate`,
`/launch`, `/stop`, `/apps`.) The dev server is loopback-only with no auth.

## Traps — the lessons that cost time

**Check the refresh response.** Plain `ok` means the build settled clean. A JSON body
with `buildErrors` means your edit did **not** compile: the app still runs the previous
classes, and exercising it now tests nothing. Fix the listed problems first.

**The build settling is not the swap landing.** `ok` means the `.class` files exist; the
JVM redefines the classes a beat later. Wait ~2–3s after a refresh before exercising a
Java change, and re-exercise once before concluding a swap failed. (Templates don't race
— they're read per render.)

**Recognise a broken hot swap — then restart, don't debug.** This stack (JBR + DCEVM +
HotswapAgent) reloads structural changes live, so restarts are rare. But a swap can
corrupt a class: lambdas get renumbered, an observer or callback registered against the
old numbering breaks, and the app starts failing on *every* request. The signature is a
`NoSuchMethodError`, `NoSuchFieldError` or `AbstractMethodError` in the console —
especially one naming a `lambda$N` method — after a refresh, on code that is fine. The
app's own log endpoint is dead at that point too, so `/console` is your only source.
`/restart` cures it. Don't spend a minute "fixing" the code.

**Never `clean=true` while the app is running.** A clean+full rebuild produces no per-class
delta for the swapper, so the running app stops picking up changes until restarted. Clean
belongs to build recovery and to freshly opened projects (where `/openProject` and
`/launch?open=true` do it for you, safely, because nothing is running yet).

**Never verify or "fix" the loop with Maven.** Eclipse resolves inter-project dependencies
inside the workspace; the local Maven repository plays no part. A `mvn compile` failing
against a stale installed jar is a false alarm — `/problems` is the truth, and `mvn
install` only wastes time and hides that nothing was broken.

**A hang or a timeout is usually a dialog.** Eclipse's modal prompts are invisible to you.
Before diagnosing anything after a request hangs or a wait times out, `GET /dialogs` — and
answer it with `?press=BUTTON`. Dev-server launches never raise Eclipse's own launch
prompts (compile errors, save, switch-to-debug are all decided and reported as data), but
hot-code-replace failures and other tooling still can.

**`reachable` is a port probe, not health.** `/status` and `/apps` report `running` and
`reachable` from a TCP connect. An app whose every request returns 500 (see the broken
swap above) is "reachable". To confirm health, fetch a page or the app's log endpoint.

**`/status?app=NAME` returns every launch config of the project** — one-off main classes
and a Production config included, all sharing the same registered block. Read the entry
whose `running` is true; don't take the first one.

**`found:false` from `/validate` tells you why.** Its `reason` distinguishes a closed
project (open it: `/openProject?project=NAME`, or check `projectOpen` in `/status`), an
unknown project name, and a component nobody has. Each needs a different fix.

**Know your source reach; never invent framework internals.** When a question crosses
into a framework or dependency (why does `ERExtensions` do X, is the bug in `helium5` or
here), check `/apps?name=APP` first: its `dependencies` are the ones whose source is open
in the workspace, with their on-disk `path`. If it's listed, read the real source there
and fix bugs across that boundary. If it isn't, you don't have its source — say so
plainly rather than guessing at its behavior.

**Don't launch Production.** `/launch` prefers a `local`/`dev` config and refuses to guess
when ambiguous — `{"launched":false,"candidates":[…]}`. Pick an exact name from the list.

**A JVM the dev server can't reach.** A swap occasionally leaves the JVM holding its port
but answering nothing, and not reacting to a clean `/stop`. `/stop?force=true` needs a
registered pid; when there is none (`/status` says `running:false` while the port is
held), `lsof -nP -iTCP:PORT -sTCP:LISTEN`, `kill -9` the pid, then `/launch`. (A fresh
ng-objects launch in development mode first tries to evict whatever holds its port, so
`/launch` alone sometimes clears a half-dead instance — but a truly wedged JVM ignores
that too, and `kill -9` is the fallback.)

## Validate templates — your highest-value habit

Templates are long strings with weak definition discovery; mistakes hide until render.
After a template edit do **both**: `/refreshProject` (so the change takes effect) and
`/validate?component=NAME` (to catch mistakes before rendering). `problems` empty →
clean; each problem has `severity`, `line`, `charStart`/`charEnd`, `message`, `file`.
`/refreshProject` does not validate, and `/validate` does not make an edit take effect.

## Ask what an element can do

Don't read an element's Java to work out its bindings. `/elementApi?element=NAME` returns
them as data: each binding's `pull`/`push` types and `direction`, `required`, `default`,
`deprecated`, plus cross-binding `constraints` with their plain-English `message`, and the
`content`/`unknownAttributes` policies. Names resolve as a template resolves them
(`str` → `WOString` → `ERXWOString`, classic shortcuts included); `resolved` says what the
name became, `kind:none` means no definition exists. `raw=true` returns the `.apiext` XML.

## Two logs, two jobs

- **`/console?app=NAME`** — the raw Eclipse console, captured by the plugin and kept after
  the process dies. Startup failures, `waitForPort` reporting termination, and an app
  broken by a bad swap live only here.
- **The app's log endpoint** (`…/<App>.woa/log` or `…/ng/dev/log`, port and form from
  `/apps`) — filterable with `contains=`, dies with the app. For debugging markers: add
  `log.info("MYDEBUG …")`, refresh, exercise, read back `?contains=MYDEBUG`, remove.

## Inside the app: `eval` and `problems`

`…/eval?snippet=…` (same URL form as `log`) evaluates a Java snippet inside the running
JVM against its live classes and data — a REPL in the process, with a persistent session.
POST long snippets as `text/plain` (form encoding shreds `=` and `&`). `…/problems`
returns the binding-error boxes the app rendered, as JSON: `clear=true`, exercise, read
back — only the errors that exercise produced. Details and shapes in the reference.

## A full iteration

```bash
curl -s 'http://localhost:9485/refreshProject?project=MyApp'                      # any edit → refresh first
curl -s 'http://localhost:9485/validate?component=SomeComponent&project=MyApp'    # template edit → also validate
sleep 2                                                                           # Java edit → let the swap land
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/log?contains=MYDEBUG&tail=40'   # then read what it logged
# …and before handing back: every touched project refreshed and clean
curl -s 'http://localhost:9485/refreshProject?project=my-model'                   # the dependency you edited too, not just the app
curl -s 'http://localhost:9485/problems?project=MyApp'                            # {"projects":[]} → settled
```

## More detail

`references/endpoints.md` is the reference: every endpoint with parameters and response
shapes, the runtime endpoints (`log`, `eval`, `problems`) in full, ng-vs-WO differences,
template conventions, and the developer-side setup (plugin, JBR + HotswapAgent, `/watch`).
