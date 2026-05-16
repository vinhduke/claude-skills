# review-past-performance

A Claude Code [skill](https://docs.anthropic.com/en/docs/build-with-claude/agent-skills) that audits your own past Claude Code sessions, finds where the agent wasted effort, and proposes durable fixes you can apply to `CLAUDE.md`, `settings.json`, or as new skills.

The idea: let Claude Code learn from itself. Run this nightly via cron, and over time your agent gets quietly better — because the friction you already paid for becomes permanent context.

## What it does

The skill reads your local Claude Code session transcripts (`~/.claude/projects/*/*.jsonl`) over a configurable time window (default 24h), looks for seven kinds of inefficiency, and produces a Markdown report you can approve.

Signals it catches:

1. **Repeated tool calls** — same `Read` on the same file, near-duplicate `Grep`s
2. **Tool errors** — `is_error: true`, exit code ≠ 0, `not found`, `permission denied`
3. **User corrections** — messages pushing back on the previous assistant turn
4. **Long stalls** — gaps > 5 minutes between events
5. **Plan-mode churn** — plans abandoned and re-written
6. **Permission-prompt loops** — same command requesting approval over and over
7. **Hallucinated identifiers** — wrong file/function/flag names corrected after a `not found`

For each finding, the skill proposes exactly one of: `CLAUDE.md` edit, new skill, `settings.json` change, or helper script — whichever is the least invasive fix that would have prevented it.

## Install

Pick one:

### A. From the `.skill` bundle

Open `review-past-performance.skill` from Claude Code (drag-drop or `/skill install`). The bundle is a regular ZIP — you can also extract it to `~/.claude/skills/` manually.

### B. From source

```bash
git clone <this-repo>
cp -r review-past-performance ~/.claude/skills/
```

On Windows:

```powershell
Copy-Item -Recurse review-past-performance "$env:USERPROFILE\.claude\skills\"
```

Verify it's loaded: open a new Claude Code session and check that `review-past-performance` appears in the available skills.

## Usage

In Claude Code:

```
/review-past-performance
```

Or informally:

```
review my last 24 hours
look at what we did yesterday and find recurring mistakes
audit my recent sessions, window=7d
```

The skill accepts two optional parameters in plain language:

- `window` — `30m`, `2h`, `24h` (default), `7d`, …
- `project` — substring of `cwd` to filter to one repo (default: all projects)

## Example output

```
# Review — 24h, 2026-05-16 07:43 UTC

## Summary
Scanned 4 sessions across 2 repos. 7 cross-repo patterns found.
Biggest issue: 10× exit-127 because Bash tool ran PowerShell cmdlets.

## Per-repo findings
### F:\học dev
- PowerShell cmd in Bash tool — seen 6×
  - Evidence: 6d59f729 @ 2026-05-14T15:16Z — "Get-Item ... | Select-Object Name, Length"
  - Proposed fix: [CLAUDE.md] Add Windows shell rule: prefer POSIX...

## Cross-repo patterns
- Bash + PowerShell mix-up — 10 incidents across 2 projects
- Re-Read same file ≥ 5× — 7 files across 4 sessions
- Python print() crashing on Windows cp1252 — 4 incidents

## Proposed changes (sorted by impact)
1. [CLAUDE.md] Windows Shell rule — ...
2. [CLAUDE.md] Re-reading Files rule — ...
3. [CLAUDE.md] Large File Reads rule — ...

## Next actions
Which proposals do you want to apply?
```

## Real-world result

On the author's machine, one run of this skill produced six concrete rules that were then added to `CLAUDE.md`:

- **Windows Shell** — don't mix PowerShell cmdlets into the Bash tool (caught 10× in 48h)
- **Re-reading Files** — don't `Read` the same file twice in a session (caught one file read **14×**)
- **Large File Reads** — `ls -la` before `Read`ing files > 200KB (caught 4 token-limit failures)
- **Read Before Edit/Write** — Read must precede Edit (caught 13 `File not read` errors across 5 projects)
- **Python Non-ASCII on Windows** — `sys.stdout.reconfigure('utf-8')` for any Python that prints Vietnamese (caught 5 crashes across 4 projects)
- **Path Quoting on Windows** — quote paths with spaces/Vietnamese/parens (caught 11 ENOENT across 6 projects)

Net effect: about 30 lines of new rules vs. dozens of round-trips saved every week.

## Safety

The skill **only proposes**. It never edits `CLAUDE.md`, `settings.json`, source files, or creates skills during a review. The report is the deliverable; application is a separate, user-approved step. This is important if you run it on a schedule — a nightly cron job that silently rewrites your config is exactly the kind of mistake the skill is meant to catch.

Other guarantees:

- Every finding cites at least one `sessionId` + `timestamp` + short snippet.
- No network calls. All transcripts stay local.
- Empty window → empty report. No padding.
- Quotes are clipped (≤ 200 chars) and never include secrets, tokens, or full file contents.

## Schedule

To run nightly, use Claude Code's built-in scheduling:

```
/schedule 0 2 * * * /review-past-performance
```

…or pace it from inside a long-running session with the `loop` skill. The skill itself does **not** register cron jobs — that has to be an explicit, user-triggered action.

## Origin

Inspired by the public idea ["Making Claude Code learn from itself"](../idea.txt) — turn the friction you already paid for into permanent context. The skill was bootstrapped using Anthropic's [`skill-creator`](https://github.com/anthropics/skills/tree/main/skill-creator) workflow.

## Repository layout

```
review-past-performance/
├── SKILL.md      # the skill (frontmatter + instructions)
└── README.md     # this file
```

That's it. No scripts, no dependencies — the skill runs entirely on Claude Code's built-in `Bash`, `Read`, and `Grep` tools.

## License

MIT — do whatever you want with it. Attribution appreciated but not required.
