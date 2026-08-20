# OpenHands PR Review Automation — rbren/rss-parser

An OpenHands cron automation that polls [`rbren/rss-parser`](https://github.com/rbren/rss-parser)
every minute for open pull requests labeled **`ai-review`** and kicks off an AI code review
for each new label event.

## Files

| File | Description |
|---|---|
| `main.py` | The automation script, exported from the running automation's tarball (`GET /api/automation/v1/{id}/tarball`). |
| `automation.json` | The automation's registered metadata, exported via `GET /api/automation/v1/{id}`. |

## How it works

Every minute the automation service runs `python3 main.py`, which is fully deterministic
Python (no LLM) except for the review itself:

1. Lists open PRs on `rbren/rss-parser` via the GitHub REST API.
2. For each PR carrying the `ai-review` label, finds the **latest `labeled` event** for
   that label. Each label event is reviewed at most once and tracked in persistent state,
   so **removing and re-adding the label triggers a fresh review**.
3. For a new label event, it:
   - downloads the PR head commit as a tarball into an isolated workspace,
   - installs the [OpenHands code-review skill](https://github.com/OpenHands/extensions/tree/main/skills/code-review)
     into `.agents/skills/code-review/` in that workspace,
   - starts an OpenHands agent conversation instructed to follow that skill's review
     methodology (data structures, simplicity, security, pragmatism, risk assessment)
     and publish the review to GitHub via `POST /repos/{repo}/pulls/{n}/reviews`,
   - posts an acknowledgement comment on the PR linking to the conversation.
4. On subsequent polls it monitors active review conversations, posts the result as a
   comment if the agent failed to publish a review itself, suppresses stale reviews when
   the PR head SHA has advanced, and cleans up finished checkouts.

State is persisted between runs (KV store when available, local JSON file otherwise), so
already-reviewed label events are never re-processed.

## Configuration

Set as constants near the top of `main.py`:

- `REPOS = ["rbren/rss-parser"]`
- `TRIGGER_LABEL = "ai-review"`
- Trigger: cron `* * * * *` (every minute, UTC)
- Entrypoint: `python3 main.py`, timeout 300s

Requires a `GITHUB_PERSONAL_ACCESS_TOKEN` secret with repo read + issues/PR write access.

---

_This repository's contents were generated and pushed by an AI agent (OpenHands) on behalf of rbren._
