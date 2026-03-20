---
name: configure
description: Set up the QQ Bot channel — save AppID:AppSecret credentials and review access policy. Use when the user pastes QQ Bot credentials, asks to configure QQ Bot, asks "how do I set this up" or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /qqbot:configure — QQ Bot Channel Setup

Writes the credentials to `~/.claude/channels/qqbot/.env` and orients the
user on access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Credentials** — check `~/.claude/channels/qqbot/.env` for
   `QQBOT_CREDENTIALS`. Show set/not-set; if set, show AppID and mask the
   secret (`123456:A3F...`).

2. **Access** — read `~/.claude/channels/qqbot/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list user OpenIDs
   - Pending pairings: count, with codes and sender IDs if any

3. **What next** — end with a concrete next step based on state:
   - No credentials → *"Run `/qqbot:configure <AppID>:<AppSecret>` with
     your QQ Bot credentials from the QQ Open Platform."*
   - Credentials set, policy is pairing, nobody allowed → *"Send a private
     message to your QQ Bot. It replies with a code; approve with
     `/qqbot:access pair <code>`."*
   - Credentials set, someone allowed → *"Ready. Send a private message to
     your bot to reach the assistant."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture user OpenIDs you don't know.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/qqbot:access policy allowlist`. Do this proactively.
4. **If no, people are missing** → *"Have them DM the bot; you'll approve
   each with `/qqbot:access pair <code>`. Run this skill again once
   everyone's in and we'll lock it."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"Send a private message to your bot to capture your own ID first."*
6. **If policy is already `allowlist`** → confirm this is the locked state.

Never frame `pairing` as the correct long-term choice.

### `<AppID>:<AppSecret>` — save credentials

1. Treat `$ARGUMENTS` as the credentials (trim whitespace). Format:
   `AppID:AppSecret` (numeric ID, colon, secret string).
2. `mkdir -p ~/.claude/channels/qqbot`
3. Read existing `.env` if present; update/add the `QQBOT_CREDENTIALS=` line,
   preserve other keys. Write back, no quotes around the value.
4. Confirm, then show the no-args status so the user sees where they stand.

### `clear` — remove credentials

Delete the `QQBOT_CREDENTIALS=` line (or the file if that's the only line).

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/qqbot:access` take effect immediately, no restart.
