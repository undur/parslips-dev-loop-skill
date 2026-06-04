# parslips-dev-loop (Claude skill)

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that teaches an
agent to close the edit→build→run loop in **WebObjects / ng-objects / Wonder**
projects by driving the [Parslips](https://undur.github.io/parslips/repository/)
Eclipse plugin's dev server and the running app's log endpoint. (Parslips is the
Eclipse plugin; Parsley is the template language it edits.)

With it, an agent editing files on disk can:

- **Validate a component template** for errors without rendering the page
  (`/validate`)
- **Refresh + rebuild** so Eclipse picks up disk edits and the running app hot-swaps
  them (`/refreshProject`)
- **Read the running app's console** over HTTP instead of asking you to paste it
  (`…/App.woa/log`)
- Know **what hot-swaps vs. what needs an app restart**

It exists because Eclipse doesn't notice edits made outside its own editor — so
without these hooks, an agent's disk edits silently do nothing.

## Install (personal — all your projects)

Clone, then symlink the skill folder into your personal skills directory:

```bash
git clone https://github.com/undur/parslips-dev-loop-skill.git
ln -s "$PWD/parslips-dev-loop-skill/parslips-dev-loop" ~/.claude/skills/parslips-dev-loop
```

(A symlink means `git pull` updates the skill in place. Or just `cp -r` the
`parslips-dev-loop/` folder into `~/.claude/skills/` if you prefer a copy.)

## Install (project — shared via a repo)

Drop the `parslips-dev-loop/` folder into a project's `.claude/skills/` and commit
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
- The `/log` endpoint needs the app runtime to provide it (wonder-slim ERExtensions
  on Wonder)

Full details — every endpoint, parameters, runtime differences, and setup — are in
[`parslips-dev-loop/references/endpoints.md`](parslips-dev-loop/references/endpoints.md).

## License

MIT — see [LICENSE](LICENSE).
