**中文** · [README.md](./README.md)

<p align="center">
  <a href="https://github.com/huanweide/family-showcase"><img src="https://img.shields.io/badge/Engineering%20Health%20Family-18%20axes-22c55e" alt="Engineering Health Family"></a>
  <a href="https://github.com/huanweide/chaineye/stargazers"><img src="https://img.shields.io/github/stars/huanweide/chaineye" alt="Stars"></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/huanweide/chaineye" alt="License"></a>
  <a href="https://nodejs.org"><img src="https://img.shields.io/badge/node-%3E%3D18-brightgreen" alt="Node"></a>
</p>

# chaineye (dependency-chain eye) — see who breaks before you change the code

> One-liner: you change one file; it instantly tells you "which other files will break because of it."

You have been there: you changed a low-level function and another, totally unrelated file crashed, and you had no idea you missed something. chaineye does exactly this — it scans your repo, draws the "who depends on whom" graph, and when you ask "I changed X, who is affected?", it answers in a flash.

**Real-life analogy**: like installing a "pipe detector" in your house. You plan to change a pipe on the second floor; it first tells you "this pipe connects to the toilet and water heater on the first floor — think before you touch it." chaineye is the pipe detector for code — you move one thing, it tells you where the shockwave reaches.

> [!WARNING]
> **This axis has been merged into the [doctors](https://github.com/huanweide/doctors) monorepo and is no longer maintained.**
> The code now lives at [`doctors/packages/chaineye/`](https://github.com/huanweide/doctors/tree/main/packages/chaineye) and runs as the `doctors chain` subcommand (or `node packages/chaineye/index.js`). This repository no longer accepts changes — please use <https://github.com/huanweide/doctors>.

**Part of the [Engineering Health Family](https://github.com/huanweide/family-showcase)** (18 zero-dependency checkup axes). Standalone tool — see [doctors](https://github.com/huanweide/doctors) (monorepo) and [enghealth](https://github.com/huanweide/enghealth) (unified runner).

## What it does (in plain words)

- **See the blast radius before you change X**: `chaineye impact src/core/parser.js` → lists every file that will be affected. Know before you touch, don't take the blame.
- **Who is secretly using this file**: `chaineye why src/core/parser.js` → who directly imports it.
- **Catch circular dependencies (the ones that blow up)**: `chaineye cycles` → finds "A depends on B, B depends on A" dead loops.
- **Find orphan files**: `chaineye orphans` → files nobody uses and that use nothing; safe to delete.
- **Find hub files (the most dangerous)**: `chaineye hubs` → the most-depended-on files; changing them has the widest impact.
- **As a CI gate**: `chaineye . --max-cycles 0 --fail-on-cycle` → blocks the merge the moment a circular dependency appears.

## How to use (three steps, no install, no config)

```bash
# 1. Scan the current repo + run the CI gate
npx chaineye .

# 2. I'm about to change parser.js — who suffers?
npx chaineye impact src/core/parser.js

# 3. Block circular deps in CI (no merge if a cycle exists)
npx chaineye . --max-cycles 0 --fail-on-cycle
```

Clone it and `node index.js .` runs immediately — zero dependencies, zero configuration.

## Why it, instead of madge / dependency-cruiser

Those two can draw graphs too, but when you just want "who is affected if I change X", you first have to `npm i` a bunch of stuff, configure a pile of rules, and study the manual. chaineye hammers on exactly one thing — blast radius: zero-dependency, single-file, works out of the box. The big tools consider this feature too small; the OSS ones consider it too light. chaineye fills exactly that gap.

## Who should use this

- You refactor or review in unfamiliar repos and need to know the impact surface before changing a low-level file.
- You want a CI gate that blocks circular dependencies and flags hub files.

## Who should NOT use this

- You need deep architectural visualization dashboards with rich config — chaineye is intentionally a single-purpose, zero-config CLI.
- You rely on package-manager-level dependency graphs of third-party modules — chaineye only analyzes in-repo relative imports.

## How it works

1. **Scan**: recursively walk the repo, ignoring `node_modules` / `.git` / `dist`, collecting JS/TS/Python/Go source files.
2. **Parse**: regex-extract relative imports (`import` / `require` / dynamic `import()` / Python `from .x` / Go `import "./x"`), resolving them to real in-repo paths (auto-appending extensions, directory indexes).
3. **Build graph**: a dependency DAG, `A → B` meaning "A depends on B", with a reverse edge maintained in parallel.
4. **Analyze**: blast radius = transitive closure of reverse edges; circular deps = three-color DFS; orphans = in-degree and out-degree both 0; hubs = highest in-degree.
   Results are deterministic: the same repo scanned twice yields identical output, so it can serve as a regression gate in CI.

## Supported languages

| Language | Import forms recognized |
|----------|-------------------------|
| JavaScript / TypeScript | `import ... from` / `import` / `export ... from` / `require()` / dynamic `import()` |
| Python | `import x` / `from x import y` / `from .x import y` / `from ..x.y import z` (relative imports across extensions) |
| Go | relative paths inside `import "x"` / `import ( "x" "y" )` |

Only in-repo relative imports are analyzed; third-party bare modules (e.g. `react`) are not counted as dependency edges.

## CI gate example (GitHub Actions)

```yaml
- name: dependency impact gate
  run: npx chaineye . --max-cycles 0 --fail-on-cycle
```

Exit codes: pass `0`, fail `1` — can directly block the merge.

## Why it stays maintainable

Pure static parsing, no external dependencies, no network; small, stable logic surface. Adding a language is just one more branch in `parseImports`.

## License

[MIT](./LICENSE) © huanweide

---

Maintained by **huanweide** · [GitHub](https://github.com/huanweide)

## More open-source tools by huanweide

- 📚 Long-form creation — AI novel workshop, local rule engine, zero-dependency: [novel-smith](https://github.com/huanweide/novel-smith)
- 🎓 Course assistant — Zhihuishu auto-play/continue/AI answers: [zhihuishu-helper](https://github.com/huanweide/zhihuishu-helper)
- 🎬 Subtitle / video — one-click Bilibili subtitle extraction (official subtitles, zero key): [bili-subtitle-universal](https://github.com/huanweide/bili-subtitle-universal)
- 📺 Live subtitle — live/meeting real-time subtitles: [live-subtitle](https://github.com/huanweide/live-subtitle)
- ✂️ Video summary — one-click long-video summary: [video-brief](https://github.com/huanweide/video-brief)
- 🖥️ Overview — all of huanweide's open-source projects: [family-showcase](https://github.com/huanweide/family-showcase)
- 🩺 Code health — zero-dependency multi-axis static checkup: [doctors](https://github.com/huanweide/doctors)
