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

**What Blueprint handles directly (phases 0-4, 6-7):**
- Working plan file / PRD draft (Phase 0)
- Targeted codebase scan (Phase 1)
- Discovery interview — 3 rounds (Phase 2)
- Research — prior art + technical (Phase 3)
- PRD generation (Phase 4)
- GitHub output — PR + issue creation (Phase 6)
- Handoff + feedback loop (Phase 7)

**What Blueprint delegates to sub-agents (phases 5, 8):**
- Review — 6 parallel agents: 5 reviewers + validator (Phase 5)
- Task decomposition — post-merge only (Phase 8)

**Human gate:** The spec PR (Phase 6) is the review gate. User provides feedback → Blueprint re-runs Phase 5 → updates PR. Loop until merged. Decompose (Phase 8) only triggers after merge.
