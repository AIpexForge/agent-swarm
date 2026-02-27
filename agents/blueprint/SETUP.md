# Blueprint Agent — OpenClaw Setup Guide

## Prerequisites

- OpenClaw gateway running
- Telegram bot token for Blueprint (create via [@BotFather](https://t.me/BotFather))
- `gh` CLI authenticated with access to AIpexForge org

---

## Step 1: Create the Telegram Bot

1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. `/newbot` → name it "Blueprint" (or "Blueprint Agent")
3. Username: e.g., `aipexforge_blueprint_bot`
4. Copy the bot token

---

## Step 2: Add the Agent to OpenClaw

```bash
openclaw agents add blueprint
```

Or manually add to `~/.openclaw/openclaw.json`:

```json5
{
  agents: {
    list: [
      // ... existing agents ...
      {
        id: "blueprint",
        name: "Blueprint",
        workspace: "~/.openclaw/workspace-blueprint",
        agentDir: "~/.openclaw/agents/blueprint/agent",
        model: "anthropic/claude-opus-4-6"
      }
    ]
  }
}
```

---

## Step 3: Configure Telegram Account + Binding

Add to `~/.openclaw/openclaw.json`:

```json5
{
  // Under channels.telegram.accounts:
  channels: {
    telegram: {
      accounts: {
        blueprint: {
          botToken: "<BLUEPRINT_BOT_TOKEN>",
          dmPolicy: "allowlist",
          allowFrom: ["tg:<YOUR_TELEGRAM_ID>"],
          streaming: "off"
        }
      }
    }
  },

  // Under bindings:
  bindings: [
    {
      agentId: "blueprint",
      match: {
        channel: "telegram",
        accountId: "blueprint"
      }
    }
  ]
}
```

---

## Step 4: Set Up the Workspace

```bash
mkdir -p ~/.openclaw/workspace-blueprint

# Copy prompt as agent instructions
cp agents/blueprint/PROMPT.md ~/.openclaw/workspace-blueprint/AGENTS.md
```

Create `~/.openclaw/workspace-blueprint/IDENTITY.md`:
```markdown
# IDENTITY.md

- **Name:** Blueprint
- **Creature:** Planning agent — structured, thorough, opinionated. Turns conversations into specs.
- **Vibe:** Direct, methodical, no-fluff. Asks the right questions, writes the right docs.
- **Emoji:** 📐
```

Create `~/.openclaw/workspace-blueprint/SOUL.md`:
```markdown
# SOUL.md

You are Blueprint, the PLAN agent in the agent-swarm system.

Your single job: turn a conversation into a spec that other agents can build from.

## Rules
- Be conversational during interviews, not robotic
- Be opinionated — if something is vague, propose a concrete answer
- Be thorough — ambiguity in the spec becomes bugs in the code
- Be efficient — skip questions the codebase already answers
- The PRD is the contract. BUILD and TEST work from YOUR output.
```

---

## Step 5: Restart Gateway

```bash
openclaw gateway restart
```

---

## Step 6: Verify

```bash
openclaw agents list --bindings
openclaw channels status --probe
```

DM the Blueprint bot on Telegram to verify it responds.

---

## Architecture

Blueprint is a **standalone agent** with its own Telegram bot — not a sub-agent of Servo.

**What Blueprint handles directly (phases 1-4):**
- Pre-interview codebase scan
- Discovery interview (3 rounds)
- Optional research (spawns research sub-agents)
- PRD generation

**What Blueprint delegates to sub-agents (phases 5-7):**
- Validation (13-check quality scoring)
- Review (4 parallel reviewers: architecture, requirements, scope, testStrategy)
- GitHub output (PR + issue creation)

**Phase 8 (handoff):** Blueprint sends the summary message, then the session ends.

---

## Cron Mode (Future)

Cron configuration for PR comment handling and task decomposition will be added separately.
The cron job will use `find-work.sh` to check for:
- `plan:draft` PRs with unaddressed review comments
- Merged spec PRs needing task decomposition
