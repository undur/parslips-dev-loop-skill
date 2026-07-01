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

When you edit a **Java** file on disk, Eclipse doesn't notice — it only tracks edits
made through its own editor. So your change sits there: no recompile, no
revalidation, and a running app keeps using stale `.class` files. From the outside
your edit appears to do nothing. The hooks below are how you make Eclipse act on a
disk edit. **Don't conclude a Java edit "had no effect" until you've refreshed.**

**Templates are different: in dev mode they are NOT cached.** A `.html`/`.wod` edit
is picked up on the next request with no refresh and no restart — just save and
reload the page. So a template-only change never needs a restart; you only
`/validate` it to catch mistakes before rendering. (Restarting after a pure template
edit — as is easy to do out of habit — is wasted work.)

## When to do what

| You just… | Do this |
|---|---|
| Edited a template (`.html`/`.wod`) | `GET /validate?component=NAME` — confirm it's error-free. Not cached in dev → **no refresh/restart**, just reload the page |
| Edited a Java class | `GET /refreshProject?project=NAME` — refresh + incremental build (reloads live; see below) |
| Need the app's port, or which deps you can read | `GET /apps` — running apps + their source-available dependencies |
| Need to start / stop an app | `GET /launch?app=NAME` / `GET /stop?app=NAME` |
| Need to see what the app logged | `GET …/<App>.woa/log` (WO) or `…/ng/dev/log` (ng), `?contains=…&tail=…` (port from `/apps`) |
| Aren't sure the dev server is up | `GET /refreshProject` (probe) before anything else |

## Always probe first

Before relying on the loop, confirm the dev server answers:

```bash
curl -s -o /dev/null -w '%{http_code}' http://localhost:9485/refreshProject
```

`200` → you're good. Connection refused → **stop and tell the human** Eclipse isn't
running or the plugin isn't loaded (don't keep retrying or work blind). The dev
server is loopback-only with no auth, so all calls are plain local `curl`.

## Know your source reach

`GET /apps?name=APP` returns, besides the port, a `dependencies` array — the app's
dependencies whose **source is open in the workspace**, each with its on-disk `path`
and `sourceFolders`. These are the libraries you can actually read and edit; jar-only
dependencies are omitted. So when a question crosses into a framework or dependency
(why does `ERExtensions` do X, is the bug in `helium5` or here), check `/apps` first:
if it's listed, read its real source at the given path instead of guessing from
behavior — and you can fix bugs across that boundary, not just in the app. If it's not
listed, you don't have its source in this workspace; say so rather than inventing it.

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
didn't resolve (check spelling/project). It refreshes from disk itself, so you
needn't call `/refreshProject` first. **`/refreshProject` does not validate
templates** — validation is editor-driven — so always `/validate` explicitly after
a template edit.

## Refresh + build after a Java edit

```bash
curl -s 'http://localhost:9485/refreshProject?project=MyApp'   # omit project → all open projects
```

Keep the default **incremental** build — it produces the per-class delta hot-swap
needs (same as an in-editor save). **Never pass `clean=true` in the normal loop**:
it forces a full rebuild that does *not* hot-swap and will make the app *stop*
picking up your changes. The call blocks until the build settles, so on return the
new classes exist.

### What reloads vs. what needs a restart

**This developer runs the JetBrains Runtime (JBR) with DCEVM**, which reloads far
more than a stock JVM: not just method bodies but **structural changes too** — new
and removed methods/fields, changed signatures, and new classes all reload live via
`/refreshProject`, no restart. Treat the default assumption as **"a Java edit hot-reloads;
don't restart."**

Only genuinely heavy changes need an app restart:
- **classpath changes** (new/updated dependency, changed `pom.xml`/build path)
- **project-structure changes** (new source folder, module layout)
- occasional very complex reloads DCEVM can't apply cleanly

A stock JVM (plain debug mode, no DCEVM) is much more limited — it swaps only method
bodies and needs a restart for any shape/constructor change. If you're unsure which
runtime is in play, `GET /apps` and the human-side setup notes in
`references/endpoints.md` clarify it; but for THIS project, assume JBR+DCEVM and don't
restart for ordinary Java edits. If a refresh genuinely doesn't take (rare), a restart
is the fallback — not the default.

## Start and stop apps

Restarts are rarely needed (see above — templates aren't cached, and JBR+DCEVM
reloads structural Java changes). But when one genuinely is — a classpath or
project-structure change, or the app isn't running yet — you can do it yourself
rather than asking the human:

```bash
curl -s 'http://localhost:9485/stop?app=MyApp'      # stop it (clean terminate)
curl -s 'http://localhost:9485/launch?app=MyApp'    # start it (debug mode — needed for hot reload)
```

`/launch` with no arg lists the configs. **Be careful which config you start:** a
project often has several (e.g. `MyApp - Local`, `MyApp - Production`). The endpoint
prefers a `local`/`dev` config and refuses to guess when it's ambiguous — if you get
`{"launched":false,"candidates":[…]}`, pick an exact name; **don't** blindly launch
something that might be Production. If a hot reload wedged the JVM and a clean stop
won't take, `/stop?app=MyApp&force=true` hard-kills it.

## Read the app's console

The running app serves its recent log over HTTP in dev mode. The URL form depends on
the runtime — same parameters on both:

```bash
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/log?contains=MYDEBUG&tail=40'   # WebObjects/Wonder
curl -s 'http://localhost:1200/ng/dev/log?contains=MYDEBUG&tail=40'                         # ng-objects
```

Discover the app's port and name yourself from the dev server's registry — apps
announce themselves at startup, so `curl -s 'http://localhost:9485/apps'` returns
each app's `name` and `port`; build the log URL from that. To diagnose: add a
uniquely greppable marker
(`log.info("MYDEBUG …")`), refresh/restart, exercise the app, read back with
`contains=MYDEBUG`, then remove the marker. This replaces asking the human to paste
the console. Buffer is the last ~2000 lines, captured after logging init (so early
boot output isn't there).

## A full iteration

```bash
# template edit → validate before rendering
curl -s 'http://localhost:9485/validate?component=SomeComponent&project=MyApp'
# java edit → rebuild (incremental)
curl -s 'http://localhost:9485/refreshProject?project=MyApp'
# exercise the app, then read what it logged
curl -s 'http://localhost:1200/cgi-bin/WebObjects/MyApp.woa/log?contains=MYDEBUG&tail=40'
```

## More detail

`references/endpoints.md` has the full endpoint list (`/openComponent`,
`/openJavaFile`, `/refresh`), parameter tables, runtime differences (ng vs WO),
template-format conventions, and the human-side setup (plugin install, JBR +
HotswapAgent). Read it when you need an endpoint not covered above or are helping
the developer set things up.
