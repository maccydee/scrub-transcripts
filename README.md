# scrub-transcripts

[![Claude skill](https://img.shields.io/badge/Claude-skill-8A2BE2.svg)](https://docs.claude.com/en/docs/claude-code/skills)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Python 3.9+](https://img.shields.io/badge/python-3.9%2B-blue.svg)](scripts/scrub.py)
[![Tests](https://github.com/maccydee/scrub-transcripts/actions/workflows/tests.yml/badge.svg)](https://github.com/maccydee/scrub-transcripts/actions/workflows/tests.yml)

A Claude Code skill that finds passwords, API keys and tokens sitting in Claude's local
session logs and replaces them with `[REDACTED:<kind>]`.

Claude Code writes every session to disk as plain JSONL. If you paste a token into a
prompt, or Claude runs `cat .env`, that value stays in `~/.claude/projects` and in your
prompt history until you delete it.

```
DRY-RUN: 8 findings in 3 of 3 files (0.0s)
  kind   password 3, github_token 2, telegram_bot_token 1, anthropic_key 1, url_password 1
  role   user 4, tool_result 3, assistant 1
  store  transcripts 6, prompt-history 2

kind                      role         len  file:line                         context (already redacted)
telegram_bot_token        tool_result   46  -Users-sam-webapp/sess-2.jsonl:2  TELEGRAM_BOT_TOKEN=[REDACTED:telegram_bot_token]⏎ANTHROPIC_API_KEY=[REDACTED:anthropic_ke…
anthropic_key (sk-ant-…)  tool_result   73  -Users-sam-webapp/sess-2.jsonl:2  …N=[REDACTED:telegram_bot_token]⏎ANTHROPIC_API_KEY=[REDACTED:anthropic_key]⏎FTP_PASS=[REDACTED:passw
github_token (ghp_…)      user          44  -Users-sam-webapp/sess-1.jsonl:1  here's my github token [REDACTED:github_token] and the db password is [REDACTED:password], de…
url_password              assistant     14  -Users-sam-webapp/sess-1.jsonl:2  psql postgres://app:[REDACTED:url_password]@db.internal:5432/app -c 'select 1'
github_token (ghp_…)      user          44  .claude/history.jsonl:1           here's my github token [REDACTED:github_token] and the db password is [REDACTED:password], de…
```

That is real output from the demo profile in [`examples/make_demo.py`](examples/make_demo.py).
Every secret in it is fake. The report never prints a secret. It shows the kind, the
length, the public vendor prefix and a context line that has already been redacted.

## Why a skill and not just a grep

Asking Claude to "clean the secrets out of my logs" without this skill goes wrong in a
specific way. The model greps for the secrets to find them, and the grep output
lands in the new session's transcript. In our eval, three plain runs left 7, 8 and 10
copies of the planted secrets in their own logs while cleaning up. One of them also asked
the user to type the password into chat. With the skill, two of the three runs left none.
The third leaked one value through a report bug, which is fixed and covered by a test.
It's a small sample, but it shows the failure is real.

The skill has two parts.

- **A scanner** (`scripts/scrub.py`, standard library only). It knows 23 token
  formats, including Anthropic, OpenAI, GitHub, AWS, Google, Slack, Stripe,
  Telegram and private keys. It also catches labelled values such as `DB_PASSWORD=…`,
  `Password: …`, `my password is …`, `--password=…` and `postgres://user:pass@host`.
- **Rules for the model.** Dry run first, never echo a secret, never ask for one in
  chat, and tell the user to rotate anything real.

## Install

```bash
git clone https://github.com/maccydee/scrub-transcripts ~/.claude/skills/scrub-transcripts
```

Then ask Claude something like *"I pasted my API key into a session earlier, get it out
of the logs"*, or run `/scrub-transcripts`. The skill runs on Sonnet (`model: sonnet`),
because the script does the detection and the model only runs it and reports.

You can also run the script directly:

```bash
python3 ~/.claude/skills/scrub-transcripts/scripts/scrub.py            # dry run, changes nothing
python3 ~/.claude/skills/scrub-transcripts/scripts/scrub.py --since 2  # last 2 days only
python3 ~/.claude/skills/scrub-transcripts/scripts/scrub.py --apply    # redact in place
```

## What it scans

| store | where |
|---|---|
| Claude Code sessions, including subagents | `~/.claude/projects/**/*.jsonl` |
| Prompt history | `~/.claude/history.jsonl` |
| Claude desktop app sessions | `local-agent-mode-sessions` and `claude-code-sessions` in the app's data dir (macOS, Windows, Linux) |
| With `--all` | `~/.claude/file-history` (snapshots of files Claude edited, often `.env`), `shell-snapshots`, `tasks`, `plans` |

`--root <dir>` scans a copied profile or a backup instead of your live home directory.

## How `--apply` writes

- It rewrites only the lines that contain a finding. Every other line stays byte for byte.
- It re-parses each rewritten line as JSON and checks the line count before replacing the file.
- It writes a temp file in the same directory, then renames it into place. The original
  modification time is kept, so the order of your session list doesn't change.
- It skips files modified in the last 10 minutes, because a live session may still be
  writing to them. It also leaves a file alone if it changes during the scrub.
- It keeps no backup, because a backup of a secret is just another copy of it.

## Limits

- Redacting your local copy doesn't un-send anything. The value already went to the
  API, and it may be in Time Machine or a synced dotfiles repo. **Rotate the credential.**
- A password typed on its own with no label can't be told apart from ordinary text. For
  those, put the value in a file yourself and pass `--values-file`. Don't paste it into
  chat, because that creates another copy.
- The labelled-value rules will occasionally redact something harmless, like a
  job-board slug stored under `"token"`. In old logs that costs nothing.
- If a thinking block contained a secret, redacting it breaks that block's signature.
  The transcript still reads fine, but that exact session may refuse to resume.

## Tests

```bash
python3 -m unittest discover -s tests -v
```

## Licence

MIT
