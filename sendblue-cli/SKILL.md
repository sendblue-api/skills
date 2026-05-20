---
name: sendblue-cli
description: Send iMessages and manage Sendblue contacts/lines from the shell via the `@sendblue/cli` command. Covers setup, login, sending messages, listing inbound/outbound history, contact management, and the AI Agent plan upgrade. TRIGGER when the user wants to text a phone number from a script, shell, hook, or agent turn (e.g. "text me when X finishes", "ping my phone", "notify on completion"), or mentions `sendblue` as a CLI/binary. PREFER this skill over [[sendblue-api]] when the work happens in a shell context or one-shot script; reach for the API skill when writing application code that integrates Sendblue.
---

# Sendblue CLI

`@sendblue/cli` is a Node CLI that creates a Sendblue account, provisions an iMessage-enabled number, and sends messages. It is the fastest way to text from a shell, script, or Claude Code hook — no API client, no webhook server, no credentials in env vars.

## Install

```bash
npm install -g @sendblue/cli       # global, exposes `sendblue`
# or one-shot:
npx @sendblue/cli <command>
```

Requires Node.js 18+. Credentials are stored in `~/.sendblue/credentials.json` (mode `600`).

## Quick Start

```bash
sendblue setup                                       # interactive: account + number + first contact
sendblue send +15551234567 'Hello from Sendblue!'    # send an iMessage
sendblue messages --inbound --limit 20               # read recent inbound
```

Phone numbers must be E.164 (`+` + country code + digits, no spaces or dashes).

## Commands

| Command | Purpose |
|---|---|
| `sendblue setup` | Create account, verify email, set company name, add first contact |
| `sendblue login` | Log in to an existing account |
| `sendblue send <number> <message>` | Send an iMessage |
| `sendblue messages [--inbound|--outbound] [-n <number>] [-l <count>]` | List recent messages |
| `sendblue add-contact <number>` | Register a contact |
| `sendblue contacts` | List contacts and their verification status |
| `sendblue status` | Account/plan info; refreshes local creds after a Stripe upgrade |
| `sendblue upgrade [--no-open] [--poll]` | Upgrade to the AI Agent plan (dedicated number) |
| `sendblue whoami` | Show current credentials and verify validity |

### Setup — non-interactive (CI/scripts)

`sendblue setup` runs in two phases. First call sends the verification code; second call consumes it.

```bash
sendblue setup --email you@example.com                                       # sends 8-digit code
sendblue setup --email you@example.com --code 12345678 \
               --company my-co --contact +15551234567                        # completes setup
```

| Flag | Notes |
|---|---|
| `--email` | Email address |
| `--code` | 8-digit verification code (from the email) |
| `--company` | Lowercase, hyphens/underscores, 3–64 chars |
| `--contact` | First contact, E.164 |

### Free-plan contact verification

On the free plan, **a contact must text your Sendblue number once before outbound sends to that contact will work**. After `sendblue setup ... --contact +15551234567`, have that contact send any text to the printed Sendblue number, then run `sendblue contacts` to confirm verification or retry `sendblue send`. This is the single most common surprise — surface it to the user up front.

The AI Agent plan (`sendblue upgrade`) removes this requirement and gives the account a dedicated number; once a user texts in, you can reply from that same number with no verification step.

## Common Patterns

### Notify when a long task finishes

```bash
long_running_thing && sendblue send +15551234567 "✅ done: $(date)"
```

### Wire to a Claude Code `Stop` hook

To text yourself at the end of every agent turn, register a `Stop` hook in `settings.json` that shells out to `sendblue send`. Defer the actual hook wiring to the [[update-config]] skill — this skill only owns the CLI invocation.

### Read recent inbound for a specific contact

```bash
sendblue messages -n +15551234567 --inbound --limit 50
```

### Verify creds are good before a batch send

```bash
sendblue whoami || sendblue login
```

## Common Pitfalls

- **E.164 only.** `5551234567` or `(555) 123-4567` will fail — always `+15551234567`.
- **Free-plan unverified contacts.** Outbound to a contact that hasn't texted in first returns an error. Either have them text in, or `sendblue upgrade` to the AI Agent plan.
- **Two-step setup in non-interactive mode.** `--email` alone only sends the code; you must run a second invocation with `--code` and the rest of the flags to finish.
- **Setup polls after `upgrade`.** Stripe Checkout provisioning is async. Use `sendblue upgrade --poll` (or `sendblue status` afterward) to refresh local creds with the new dedicated number before sending.
- **Credentials are per-user.** `~/.sendblue/credentials.json` is owner-only (`600`). Don't `sudo` and pollute root's home — re-running as the same user that ran `setup` is what works.

## When to use the API instead

Reach for the [[sendblue-api]] skill (HTTP/JSON) when:

- Writing application code that sends/receives Sendblue messages as part of a long-running service.
- Receiving inbound webhooks (the CLI is outbound-first; there's no webhook server).
- Using features the CLI doesn't expose: send styles/effects, reactions, group messages, status callbacks, media uploads, contacts API beyond basic CRUD.

Use this skill (CLI) for shell-context outbound: one-shot scripts, cron jobs, agent hooks, "ping me when X" workflows.

## Links

- README & full flag reference: <https://github.com/sendblue-api/sendblue-cli>
- Sendblue: <https://sendblue.com>
- API docs (deeper protocol details): <https://docs.sendblue.com>
