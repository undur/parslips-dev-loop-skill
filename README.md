# parslips-skill

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that teaches an
agent to close the edit→build→run loop in **WebObjects / ng-objects / Wonder**
projects by driving the [Parslips](https://undur.github.io/parslips/repository/)
Eclipse plugin's dev server and the running app's own dev endpoints (log, eval,
runtime problems). (Parslips is the Eclipse plugin; Parsley is the template language
it edits.)

With it, an agent editing files on disk can:

- **Refresh + rebuild** so Eclipse picks up disk edits and the running app hot-swaps
  them (`/refreshProject`) — and get a build-error report instead of a false `ok`
  when the edit didn't compile
- **Validate a component template** for errors without rendering the page
  (`/validate`)
- **See what's running** — per launch config: running/mode/uptime, project open
  state, compile errors, the app's registered port (`/status`)
- **Start, stop and restart apps itself** (`/launch`, `/stop`, `/restart`) — launch
  refuses with a named reason (closed project, compile errors, already running)
  instead of pretending to succeed, and `waitForPort` blocks until the app answers
  or provably died
- **Go from a fully closed workspace to a running app in one call** —
  `launch?open=true` opens the project *and its workspace dependencies*
  (pom-resolved, transitive), clean-builds them, launches and waits (or `/openProject`
  to open a project + its dependencies without launching)
- **Read startup failures post-mortem** (`/console` — the Eclipse console over
  HTTP, kept after the process dies) and **read the running app's log**
  (`…/App.woa/log`) instead of asking you to paste either
- **Check the Problems view** (`/problems`) and **hunt forgotten breakpoints**
  (`/breakpoints`, with a Skip All toggle)
- **Look up an element's real API** — bindings with their types and pull/push
  direction, required flags, and cross-binding constraints with generated messages
  (`/elementApi`) — instead of reading the element's Java source to work it out
- **Run code inside the running app** — a Java REPL in the live JVM, against the
  app's own objects and data (a real Cayenne context, the running singleton), not a
  separate jshell (`…/App.woa/eval`)
- **Read the runtime binding errors** the app rendered into its pages — the inline
  error boxes as JSON, instead of scraping them out of rendered HTML
  (`…/App.woa/problems`)
- **See and answer Eclipse's modal dialogs** (`/dialogs`) — the stop-the-world
  prompts an agent otherwise can't see, and launches that never raise them
- **Read (or watch) everything the agent asked the dev server to do** — `/activity`
  as JSON for the next session, `/watch` as a live narrated page for you
- Know **what hot-swaps vs. what needs an app restart** — the timing traps
  (build-settled is not swap-landed) and how to recognise a swap that corrupted a
  class (restart, don't debug)

It exists because Eclipse doesn't notice edits made outside its own editor — so
without these hooks, an agent's disk edits silently do nothing. The dev server is
self-describing: `GET http://localhost:9485/` returns a JSON index of every
endpoint, and the skill teaches the agent to trust that index over its own docs
(older plugin builds expose a smaller endpoint set; the skill degrades gracefully).

## Install (personal — all your projects)

Clone, then symlink the skill folder into your personal skills directory:

```bash
git clone https://github.com/undur/parslips-skill.git
ln -s "$PWD/parslips-skill/parslips-skill" ~/.claude/skills/parslips-skill
```

(A symlink means `git pull` updates the skill in place. Or just `cp -r` the
`parslips-skill/` folder into `~/.claude/skills/` if you prefer a copy.)

## Install (project — shared via a repo)

Drop the `parslips-skill/` folder into a project's `.claude/skills/` and commit
it; anyone (or any agent) working in that repo gets it automatically.

## Permissions (so the calls don't prompt)

The skill drives the dev server with `curl` to `localhost`. To let an agent run the
loop without approving each call, add an allowlist to your settings
(`~/.claude/settings.json` for personal, or a project's `.claude/settings.json`).
See [`settings.snippet.json`](settings.snippet.json) for a ready-to-merge block —
it allowlists `curl` to `localhost:9485` (the dev server) and the local app port.
Adjust the app port to match yours.

## What it triggers on

Working in a project with `.wo` component bundles, `<wo:…>`/`<webobject>` template
tags, `build.properties` with `project.base`, `WOComponent`/`NGComponent` classes,
or wonder-slim/ERExtensions.

## Requirements (on the developer's side)

- Eclipse running the **Parslips** plugin
  (`https://undur.github.io/parslips/repository/`)
- The app launched from Eclipse in **debug mode** (JBR + HotswapAgent recommended
  for broad hot reload)
- The app-runtime endpoints (`log`, `eval`, `problems`) are served by the framework
  in dev mode — recent wonder-slim/ERExtensions or ng-objects. `eval` runs code in
  the live JVM and is restricted to loopback clients. An older framework build serves
  a smaller set (or just `log`); the editor-side endpoints (`/validate`,
  `/elementApi`, …) come from the Parslips plugin and are independent of it.

Full details — every endpoint, parameters, runtime differences, and setup — are in
[`parslips-skill/references/endpoints.md`](parslips-skill/references/endpoints.md).

## License

MIT — see [LICENSE](LICENSE).
