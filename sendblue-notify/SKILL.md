---
name: sendblue-notify
description: Text the user's phone when a long-running task, agent turn, or scheduled job finishes — using `@sendblue/cli` for outbound and (optionally) a Claude Code `Stop` hook for automatic fire. TRIGGER when the user says "text me when X is done", "ping my phone", "notify me on completion", "let me know when the build/deploy/migration finishes", "send me an iMessage when…", or asks to wire a hook that texts on agent stop. Composes with [[sendblue-cli]] (the send mechanism) and [[update-config]] (the hook wiring).
---

# Sendblue Notify

Outbound, fire-and-forget notifications from a local Claude Code session, script, or scheduled job to the user's phone via Sendblue. This is the "walk away from the terminal" pattern: kick off something long, get an iMessage when it lands.

This skill owns **when to notify and what to say**. Actual sending goes through [[sendblue-cli]]. Hook wiring (so notifications fire automatically) goes through [[update-config]].

## Prerequisites

The CLI must be installed and authenticated:

```bash
npx @sendblue/cli whoami        # confirms creds
# or, if first run:
npx @sendblue/cli setup
```

The user's phone number must be a verified contact on the account. On the free plan, the contact has to text the Sendblue number once before outbound sends work — confirm with `sendblue contacts` before relying on notify in an unattended workflow.

Cache the destination number once per project rather than re-asking. A `NOTIFY_NUMBER` env var or a one-line `.notify-number` file is fine; defer storage strategy to whatever the surrounding project already does.

## When to notify

Notify is for **long, unattended work** — not chatter. Good triggers:

- Agent turns over ~2 minutes (build, large refactor, migration, dataset crunch).
- `/loop` and `/schedule` jobs that produce a discrete result.
- CI / deploy completion when watched from the terminal.
- Multi-step playbooks where the user has gone heads-down on something else.

Bad triggers (do not silently wire these):

- Every `Stop` event, regardless of duration — produces spam, trains the user to ignore.
- Read-only or sub-second commands.
- Anything inside a tight loop.

If the user asks for "notify me when done" on a short task, do the obvious one-shot inline send (below) and **do not** install a global hook.

## Pattern 1 — One-shot inline send

For a single task, append the send to the command. This is the default — no config changes, no surprise behavior later.

```bash
long_running_thing && \
  npx @sendblue/cli send +15551234567 "✅ done: $(date +%H:%M)" || \
  npx @sendblue/cli send +15551234567 "❌ failed: $(date +%H:%M)"
```

Or, when the result is interesting, include a one-line summary:

```bash
RESULT=$(run-migration 2>&1 | tail -1)
npx @sendblue/cli send +15551234567 "migration done — $RESULT"
```

Notification copy rules:

- **One line, under ~140 chars** — fits in the lock-screen preview.
- **Lead with outcome** — ✅/❌, "done", "failed", "needs review".
- **Include something actionable** — branch name, error tail, PR number, duration.
- **No emojis the user didn't ask for** beyond a single status glyph.
- **No agent self-narration** ("I have completed the task as requested" — just say what happened).

## Pattern 2 — Claude Code `Stop` hook (opt-in, scoped)

When the user explicitly wants automatic notifications for *this* project or session, register a `Stop` hook in `.claude/settings.json` (project-scoped) — never in global settings unless asked.

Defer the actual file edit to the [[update-config]] skill. The hook command itself should:

1. Run cheaply (it fires on *every* `Stop`).
2. Gate on duration — skip sends for turns under a threshold (e.g. 90s) to avoid spam.
3. Never fail the parent — pipe to `|| true` so a notify error doesn't surface as a hook failure.

Minimal command shape:

```bash
[ "$CLAUDE_TURN_DURATION_SECONDS" -ge 90 ] && \
  npx @sendblue/cli send "$NOTIFY_NUMBER" "turn done in ${CLAUDE_TURN_DURATION_SECONDS}s" || true
```

(Adjust the env var names to whatever the hook contract actually provides — verify against the current Claude Code hooks reference before writing the config; the harness owns those names, not this skill.)

When suggesting this to the user, **show the proposed hook config first and get confirmation** before invoking [[update-config]]. Automated outbound messages are a footgun if the threshold is wrong.

## Pattern 3 — End-of-`/loop` or `/schedule` ping

`/loop` and `/schedule` runs are the cleanest fit for notify — they're explicitly unattended and produce discrete results. Append the send to the loop's body so it fires once per iteration's outcome:

```bash
/loop 10m "check deploy; npx @sendblue/cli send +15551234567 \"deploy: \$(deploy-status)\""
```

For `/schedule`, the routine itself can shell out at the end. Same copy rules apply.

## Composing with textme

If the user has [[textme]] installed (njerschow/textme — daemon that lets you *text Claude* from your phone), notify is still useful and not redundant. They run in opposite directions:

- **textme**: phone → Claude (you initiate from your phone).
- **sendblue-notify**: local Claude → phone (Claude initiates from a local session).

You can install both: textme on a server for inbound, notify as a local Stop-hook for outbound. Different problems, same Sendblue account.

## Common Pitfalls

- **Spam from over-eager `Stop` hooks.** Always gate by duration. A user who gets pinged every 4 seconds will rip the hook out within an hour.
- **Hardcoding the destination number in committed files.** Use an env var or gitignored file; the number is per-user, not per-repo.
- **Letting a failed notify fail the parent.** Always trail with `|| true` in hooks; surface the failure in logs, not by aborting the agent turn.
- **Free-plan contact gotcha.** If the destination contact hasn't texted in once on a free-plan account, the send silently fails for the user's purposes (CLI returns the API error but the user just sees "no text arrived"). Verify with `sendblue contacts` before wiring an unattended hook. Or `sendblue upgrade` to the AI Agent plan.
- **PII in notification copy.** Lock-screen previews are visible to anyone holding the phone. Don't embed secrets, customer data, or full error stacks — link to a log or PR instead.
- **Burying the outcome.** "Task complete. Here is a summary of what I did…" wastes the preview line. Lead with ✅/❌ and the verb.

## Links

- Underlying CLI: <https://github.com/sendblue-api/sendblue-cli>
- Sendblue: <https://sendblue.com>
- Related skill: [[sendblue-cli]] (send mechanism), [[update-config]] (hook wiring), [[sendblue-api]] (HTTP alternative for app code)
