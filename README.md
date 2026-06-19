# Owl

Owl audits local files for a named issue type, then tries to confirm each finding by reproducing it with a test.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Powered by Claude Code](https://img.shields.io/badge/Powered%20by-Claude%20Code-D97757.svg)](https://github.com/anthropics/claude-code)

Owl is a single-file fish script that drives an agent CLI — `claude` by default, or any binary you pass with `agent=`. It is agent-agnostic: owl owns prompt building, file iteration, progress/resume, rate-limit retry and the per-file timeout, while you supply the agent's own flags (forwarded verbatim) and wire the prompt with `p=`/`s=`. It has three subcommands: `scan`, `check`, and `list`.

## How it works

`scan` passes each file to `claude` in isolation and writes a report next to it. Findings must cite a specific line. Anything without a line reference gets dropped.

`check` reads that report and asks `claude` to reproduce each finding with a test. Only a passing test marks the finding `CONFIRMED`. Three attempts, then `NOT CONFIRMED` and move on. No severity scores.

Runs are resumable. Owl writes progress to a markdown file as it runs, so an interrupted scan picks up where it left off. If Claude hits a rate limit, Owl parses the reset time and waits. A per-file `--timeout` (default 20 min) kills a stalled agent so one hung file can't block the run — that file is left unmarked and retried on `--resume`.

## Install

Requires a Claude-compatible agent CLI on `$PATH` (`claude` by default) and fish.

```fish
curl -o ~/.config/fish/functions/owl.fish https://raw.githubusercontent.com/nerdalytics/owl/trunk/owl.fish
```

## Usage

Owl is agent-agnostic. It owns prompt building, file iteration, state, rate-limit
retry, the per-file timeout and resume; you supply the agent's own flags. owl's own
options use a `key=value` convention. Any token starting with `-` or `--` is forwarded
verbatim to the agent. The bare keywords `resume` and `help` are reserved.

You wire the prompt and system prompt to your agent with `p=` and `s=`:

- `p=<flag>` — owl appends `<flag> <prompt>` to the agent invocation (e.g. `p=-p`, `p=exec`).
  Omit `p=` and owl does not inject the prompt at all.
- `s=<flag>` — owl appends `<flag> <system-prompt>` (e.g. `s=--append-system-prompt`).
  Omit `s=` and the system prompt is not sent.

```fish
owl scan vulnerability p=-p s=--append-system-prompt --permission-mode=acceptEdits
owl scan "memory leak" p=-p include=.py,.js
owl scan sqli p=-p src/ exclude=.test.py
owl scan vulnerability p=-p src/auth.py
owl scan vulnerability agent=codex p=exec --full-auto src/
owl scan vulnerability p=-p resume

owl check vulnerability p=-p s=--append-system-prompt
owl check xss p=-p resume

owl list
owl list vulnerability
owl scan help
```

## Params

owl's own params use `key=value`. Anything starting with `-`/`--` is forwarded to the agent.

| Param | What it does |
|---|---|
| `agent=NAME\|PATH` | Agent binary name (resolved on `$PATH`) or path (default: `claude`) |
| `depth=N` | Max directory depth (default: 10) |
| `include=EXT,EXT` | Include files by extension, comma-separated (`scan` only) |
| `exclude=SFX,SFX` | Exclude files by suffix, comma-separated (`scan` only) |
| `ignore=BOOL` | Respect `.gitignore` and `.ignore` files (default: true, `scan` only) |
| `memory=BOOL` | Allow agent auto-memory and skills (`scan` defaults `false`, `check` defaults `true`) |
| `p=FLAG` | Prompt-delivery flag — owl appends `<FLAG> <prompt>`. Omit to not inject the prompt |
| `s=FLAG` | System-prompt-delivery flag — owl appends `<FLAG> <system-prompt>`. Omit to not send it |
| `state-file=PATH` | Progress file (default: `.owl-scn-<agent>.<slug>.md` or `.owl-chk-<agent>.<slug>.md`) |
| `retry-delay=N` | Extra seconds after rate-limit reset (default: 1) |
| `timeout=N` | Max seconds per file before a stalled agent is killed (`0` = off, default: 1200) |

Bare keywords: `resume` (resume from the progress file), `help` (show owl's help — note
`--help` is forwarded to the agent instead).

Any `-`/`--` flag (e.g. `--full-auto`, `--permission-mode=acceptEdits`) is forwarded to the
agent verbatim. Valued forwarded flags use `=`; lone booleans are passed through alone.
On `resume`, owl replays the stored params and forwarded args; CLI params override stored ones.

## Known limitations

1. **Rate-limit and auth handling assume Claude Code's output, and are hardcoded (not
   configurable).** Between files owl scans the agent's output for Claude-specific strings:
   - it pauses on a rate-limit notice matching `resets <time> (<tz>)`, parses that time and
     sleeps until the reset (plus `retry-delay` seconds), falling back to a 30-minute backoff
     if it can't parse the time;
   - it aborts the whole run on `Not logged in`.

   Other agents' rate-limit and authentication messages won't be recognized, so owl will not
   pause or abort for them.
2. **Isolation is Claude-shaped.** With `memory=false` (the `scan` default) owl runs the agent
   under `env -i`, setting `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` and preserving only `$HOME`,
   `$PATH`, `$TMPDIR`, `$USER`, and `$SECURITYSESSIONID`. Agents that authenticate via other
   environment variables must use `memory=true`.

## Prompt injection

File contents are passed to `claude` as data, not instructions. Owl's system prompt forbids Claude from acting on directives inside the files. Injection attempts get reported as findings rather than executed.

## License

MIT
