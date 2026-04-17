# owl

Owl audits local files for a named issue type, then verifies every finding by reproducing it with a test.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Powered by Claude Code](https://img.shields.io/badge/Powered%20by-Claude%20Code-D97757.svg)](https://github.com/anthropics/claude-code)

Owl is a single fish-shell function that wraps the `claude` CLI. It has two subcommands: `scan` and `check`.

## How it works

`scan` passes each file to `claude` in isolation and writes a report next to it. Findings must cite a specific line. Anything without a line reference gets dropped.

`check` reads that report and asks `claude` to reproduce each finding with a test. Only a passing test marks the finding `CONFIRMED`. Three attempts, then `NOT CONFIRMED` and move on. No severity scores.

Runs are resumable. Owl writes progress to a markdown file as it runs, so an interrupted scan picks up where it left off. If Claude hits a rate limit, Owl parses the reset time and waits.

## Install

Requires the `claude` CLI on `$PATH` and fish.

```fish
curl -o ~/.config/fish/functions/owl.fish https://raw.githubusercontent.com/nerdalytics/owl/trunk/owl.fish
```

## Usage

```fish
owl scan vulnerability
owl scan "memory leak" --include py --include js
owl scan sqli src/ --exclude test.py
owl scan vulnerability src/auth.py
owl scan vulnerability --resume

owl check vulnerability
owl check xss --resume

owl list
owl list vulnerability
```

## Flags

| Flag | What it does |
|---|---|
| `-d, --depth N` | Max directory depth (default: 10) |
| `--include EXT` | Include files by extension (repeatable, `scan` only) |
| `--exclude SUFFIX` | Exclude files by suffix (repeatable, `scan` only) |
| `-i, --ignore BOOL` | Respect `.gitignore` and `.ignore` files (default: true) |
| `--effort VALUE` | Claude effort level (default: max) |
| `--permission-mode` | `acceptEdits`, `plan`, `default`, `auto`, `dontAsk` |
| `--no-memory` / `--memory` | Toggle Claude auto-memory and skills (`scan` defaults off, `check` defaults on) |
| `--state-file PATH` | Progress file (default: `.owl-scn-<slug>.md` or `.owl-chk-<slug>.md`) |
| `--resume` | Resume from progress file |
| `--retry-delay N` | Extra seconds after rate-limit reset (default: 1) |

## Prompt injection

File contents are passed to `claude` as data, not instructions. Owl's system prompt forbids Claude from acting on directives inside the files. Injection attempts get reported as findings rather than executed.

## License

MIT
