# Prompt Engineering Best Practices for Agent-Swarm

> Compiled from Anthropic docs (Claude 4.6), OpenAI docs (GPT-5), Anthropic's "Building Effective Agents" research, OMO agent prompts, and real-world production patterns (Cursor).
>
> **Date:** 2026-03-20
> **Purpose:** Reference guide for writing agent and sub-agent prompts in agent-swarm.

---

## 1. Foundational Principles

### 1.1 — Be Clear and Direct
State exactly what you want. Treat the prompt like instructions for a brilliant new hire who has zero context on your norms.

**Golden rule:** Show your prompt to a colleague with minimal context. If they'd be confused, the model will be too.

- Be specific about output format and constraints.
- Use numbered steps when order matters.

**Source:** Anthropic Claude Best Practices

### 1.2 — Explain the WHY, Not Just the WHAT
Providing context or motivation behind instructions helps the model generalize beyond the literal instruction.

```
# Bad
NEVER use ellipses

# Good
Your response will be read by a TTS engine, so never use ellipses since TTS can't pronounce them.
```

The model will then also avoid other TTS-unfriendly patterns you didn't explicitly list.

**Source:** Anthropic Claude Best Practices

### 1.3 — Tell the Model What TO DO, Not What NOT to Do
Positive framing produces more reliable results than prohibitions.

```
# Bad
Do not use markdown in your response.

# Good
Your response should be composed of smoothly flowing prose paragraphs.
```

**Source:** Anthropic Claude Best Practices

### 1.4 — Avoid Contradictory Instructions
Contradicting rules within the same prompt causes the model to burn reasoning tokens trying to reconcile them rather than executing the task. Audit prompts for conflicting instructions.

**Source:** OpenAI GPT-5 Prompting Guide (Cursor case study)

### 1.5 — Match Prompt Complexity to Task Complexity
Start simple. Only add structure, sub-agents, and multi-step flows when simpler approaches fall short. A well-tuned single prompt beats an over-engineered pipeline.

**Source:** Anthropic "Building Effective Agents"

---

## 2. Structure & Formatting

### 2.1 — Use XML Tags for Complex Prompts
XML tags help models parse compound prompts unambiguously. Wrap each content type in descriptive tags: `<instructions>`, `<context>`, `<input>`, `<example>`.

Best practices:
- Consistent, descriptive tag names across prompts.
- Nest tags when content has natural hierarchy.
- Use `<example>` / `<examples>` to separate examples from instructions.

**Source:** Anthropic Claude Best Practices, OpenAI (Cursor uses `<instruction_spec>` tags)

### 2.2 — Long Context: Data First, Query Last
Put documents and large inputs at the TOP of the prompt, with instructions and queries at the BOTTOM. This improves response quality by up to 30% for complex multi-document inputs.

**Application to agent-swarm:** When spawning sub-agents with context (PRD, codebase scan, spec), put the context block first, then the task instructions.

**Source:** Anthropic Claude Best Practices

### 2.3 — Ground Responses in Quotes
For tasks involving long documents, ask the model to extract relevant quotes before reasoning. This cuts through noise and anchors conclusions in evidence.

**Application to agent-swarm:** Reviewers and validators should cite specific REQ-NNN sections rather than making abstract claims.

**Source:** Anthropic Claude Best Practices

### 2.4 — Match Prompt Style to Desired Output
The formatting used in your prompt influences the model's output formatting. If you want structured JSON, show JSON. If you want prose, write in prose.

**Source:** Anthropic Claude Best Practices

---

## 3. Role & Identity

### 3.1 — Set a Clear Role
Even a single sentence defining the agent's role focuses behavior and tone. For agent-swarm, each agent already has a clear identity block — keep these concise and front-loaded.

**Source:** Anthropic Claude Best Practices

### 3.2 — Separate Identity from Constraints
Structure prompts so identity (who you are) is distinct from constraints (what you must/must not do). This prevents the model from treating constraints as optional personality traits.

**Pattern from OMO/Prometheus:**
```markdown
## CRITICAL IDENTITY (READ THIS FIRST)
YOU ARE A PLANNER. YOU ARE NOT AN IMPLEMENTER.

## ABSOLUTE CONSTRAINTS
1. INTERVIEW MODE BY DEFAULT
2. OUTPUT LOCATION: GitHub issues
```

**Source:** OMO Prometheus prompt, agent-swarm existing patterns

---

## 4. Examples & Few-Shot

### 4.1 — Include 3-5 Examples for Critical Output Formats
Examples are the most reliable way to steer output format, tone, and structure.

Requirements for good examples:
- **Relevant:** Mirror actual use cases.
- **Diverse:** Cover edge cases; don't just show the happy path.
- **Structured:** Wrap in `<example>` tags.

**Application to agent-swarm:** Sub-agent output contracts (JSON schemas) should include a concrete example alongside the schema definition.

**Source:** Anthropic Claude Best Practices

### 4.2 — Use Bad-vs-Good Example Pairs
Showing both a bad example and a good example is more effective than showing only the good one. The contrast teaches the model what to avoid.

```markdown
- ❌ BAD: "Verify the login endpoint works correctly"
- ✅ GOOD: "POST /api/auth/login with {email: 'test@example.com', password: 'validpass'} → 200"
```

**Source:** Agent-swarm existing testStrategy Rules pattern

---

## 5. Agentic Patterns

### 5.1 — Control Agentic Eagerness
Balance proactivity vs. waiting for user direction. Define when the agent should act autonomously and when it should ask.

For proactive agents:
```xml
<persistence>
Keep going until the task is completely resolved.
Do not ask for confirmation — make reasonable assumptions and document them.
</persistence>
```

For conservative agents:
```xml
<do_not_act_before_instructions>
Default to providing information, not taking action.
Only proceed with edits when explicitly requested.
</do_not_act_before_instructions>
```

**Application to agent-swarm:** Blueprint should be proactive during research (Phase 3) but conservative about external actions (GitHub PRs). BUILD agents should be maximally autonomous within their task scope.

**Source:** OpenAI GPT-5 Prompting Guide

### 5.2 — Define Explicit Stop Conditions
Every agentic prompt should define when the agent's work is done. Without stop conditions, agents either loop or stop prematurely.

```markdown
DONE when:
- All acceptance criteria verified
- PR created and URL returned
- JSON result block output as final message
```

**Source:** OpenAI GPT-5 Prompting Guide, Anthropic "Building Effective Agents"

### 5.3 — Use Self-Clearance Checks
Self-assessment gates let agents decide when to transition between phases without needing external orchestration.

```markdown
CLEARANCE CHECK:
□ Core objective clearly defined?
□ Scope boundaries established?
□ No critical ambiguities remaining?

ALL YES → proceed. ANY NO → continue current phase.
```

**Source:** OMO Prometheus, already adopted in agent-swarm improvement plan

### 5.4 — Tool Preambles for Long Tasks
For tasks with multiple tool calls, have the agent narrate its plan and progress. This helps debugging and gives human observers confidence the agent is on track.

```
Always begin by rephrasing the goal, then outline your plan.
As you execute, narrate each step.
Finish by summarizing completed work.
```

**Source:** OpenAI GPT-5 Prompting Guide

### 5.5 — Parallel Tool Calling
Modern models excel at parallel execution. Explicitly enable it:

```xml
<use_parallel_tool_calls>
If multiple tool calls are independent, make them in parallel.
Never use placeholders or guess missing parameters.
</use_parallel_tool_calls>
```

**Source:** Anthropic Claude Best Practices

---

## 6. Sub-Agent & Stateless Prompt Design

### 6.1 — Complete Context in Every Spawn
Stateless sub-agents get ONE shot. They can't ask follow-up questions. The prompt must contain everything they need:
- Task description
- All input data (spec, codebase context, etc.)
- Output format specification with example
- Constraints and boundaries
- Stop conditions

**Source:** Agent-swarm architecture principle, Anthropic "Building Effective Agents"

### 6.2 — Define Input/Output Contracts
Every sub-agent should have a clear contract:

```markdown
## Input
- spec_content: Full markdown of the spec
- repo: GitHub org/repo

## Output (JSON)
{
  "verdict": "pass | concerns | fail",
  "issues": [{ "severity": "...", "section": "...", "issue": "...", "suggestion": "..." }]
}
```

Include at least one concrete example of the expected output.

**Source:** Agent-swarm existing patterns (DECOMPOSE, reviewers)

### 6.3 — One Concern Per Sub-Agent
Each sub-agent should have a single, focused responsibility. This is more reliable than asking one agent to evaluate multiple unrelated concerns.

**Pattern:** agent-swarm's 4 reviewers (architecture, requirements, scope, testStrategy) outperform a single "review everything" agent.

**Source:** Anthropic "Building Effective Agents" (parallelization/sectioning pattern)

### 6.4 — Push-Based Completion
Sub-agents should auto-announce completion. The orchestrator should NOT poll in a loop.

In OpenClaw: `sessions_spawn(mode="run")` — sub-agent auto-announces when done.

**Source:** OpenClaw architecture, OMO adaptation

---

## 7. Tool Design (Agent-Computer Interface)

### 7.1 — Invest in Tool Documentation
Tool definitions and specs deserve as much prompt engineering attention as the agent prompts themselves. A poorly documented tool causes more failures than a weak prompt.

**Source:** Anthropic "Building Effective Agents" (Appendix 2)

### 7.2 — Poka-Yoke (Error-Proof) Your Tools
Design tool interfaces so it's hard to make mistakes:
- Use absolute paths instead of relative (the model won't get confused after directory changes).
- Use enums instead of free-text for known option sets.
- Validate inputs before execution.

**Source:** Anthropic "Building Effective Agents" (SWE-bench learnings)

### 7.3 — Give the Model Enough Tokens to Think Before Committing
Tool output formats should let the model plan before writing itself into a corner. Avoid formats that require knowing the answer before starting (e.g., diff headers that need line counts upfront).

**Source:** Anthropic "Building Effective Agents"

---

## 8. Thinking & Reasoning

### 8.1 — Prefer General Instructions Over Prescriptive Steps
For reasoning-capable models, "think thoroughly" often produces better results than a hand-written step-by-step plan. The model's reasoning frequently exceeds what a human would prescribe.

**Application to agent-swarm:** Don't micromanage HOW a reviewer reasons. Tell it WHAT to evaluate and let it find the right approach.

**Source:** Anthropic Claude Best Practices

### 8.2 — Use Self-Check Instructions
Append verification checks to catch errors before output:

```
Before finalizing, verify:
- Every P0 requirement has a testStrategy
- No circular dependencies in the dependency chain
- All REQ-NNN references resolve to actual requirements
```

**Source:** Anthropic Claude Best Practices

### 8.3 — Dial Back Aggressive Language on Modern Models
Claude 4.6 and GPT-5 are more responsive to system prompts than previous models. Where you used "CRITICAL: You MUST...", now use "Use this when...". Aggressive language causes overtriggering.

**Application to agent-swarm:** Audit existing prompts for ALL-CAPS commands and soften to natural language where the instruction is already clear.

**Source:** Anthropic Claude Best Practices, OpenAI GPT-5 Prompting Guide (Cursor learnings)

---

## 9. State Management & Long-Horizon Tasks

### 9.1 — Use Structured Formats for State Data
JSON for task status, test results, and dependency maps. Freeform text for progress notes and context.

**Source:** Anthropic Claude Best Practices

### 9.2 — Use Git for State Tracking
Git provides a log of what's been done and checkpoints that can be restored. Agent-swarm already uses GitHub as the state backbone — this aligns perfectly.

**Source:** Anthropic Claude Best Practices

### 9.3 — Context Window Awareness
Tell agents about context compaction behavior so they don't artificially stop early:

```
Your context window will be automatically compacted. Do not stop tasks early
due to token budget concerns. Save progress to a file if needed.
```

**Source:** Anthropic Claude Best Practices

---

## 10. Anti-Patterns to Avoid

### 10.1 — Scope Creep via Requirements
Don't let the model invent requirements the user didn't ask for. Uncertain additions go in Open Questions, not Requirements.

### 10.2 — Premature Architecture
Don't propose microservices when a function works. Match solution complexity to problem complexity.

### 10.3 — Vague Verification
"Verify it works" is not a test strategy. Name a specific action and expected result.

### 10.4 — Dependency Inflation
Only list build dependencies. Runtime calling relationships are NOT build dependencies.

### 10.5 — Gold-Plating Non-Functionals
Don't add "99.99% uptime" to an internal tool for 3 users. Scale NFRs to actual context.

### 10.6 — Over-Specified Task Breakdowns
Task hints should be directional, not prescriptive. BUILD agents need room to make implementation decisions.

**Source:** Agent-swarm existing improvement plan (partially from OMO Prometheus patterns)

---

## Quick Reference: Prompt Template for Stateless Sub-Agents

```markdown
# [AGENT NAME] — [One-Line Role]

## Identity
You are [role]. Your job: [single sentence].

## Input
You receive:
- `field_1`: Description
- `field_2`: Description

## Task
[Clear, specific instructions. What to evaluate/produce.]

## Output Format
Return a JSON block as your final message:
```json
{
  "field": "value"
}
```

<example>
[Concrete example with realistic data]
</example>

## Rules
1. [Constraint]
2. [Constraint]
3. [Stop condition]
```

---

## Applicability Matrix

| Practice | Blueprint | DECOMPOSE | ONBOARD | Reviewers | BUILD | TEST |
|----------|-----------|-----------|---------|-----------|-------|------|
| 1.1 Clear & Direct | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 1.2 Explain WHY | ✅ | ⬜ | ✅ | ⬜ | ⬜ | ⬜ |
| 2.1 XML Tags | ⬜ | ⬜ | ⬜ | ✅ | ✅ | ✅ |
| 2.2 Data First | ✅ | ✅ | ⬜ | ✅ | ✅ | ✅ |
| 4.1 Few-Shot Examples | ✅ | ✅ | ✅ | ✅ | ⬜ | ⬜ |
| 5.1 Eagerness Control | ✅ | ⬜ | ⬜ | ⬜ | ✅ | ⬜ |
| 5.2 Stop Conditions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| 5.3 Self-Clearance | ✅ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |
| 6.1 Complete Context | ⬜ | ✅ | ⬜ | ✅ | ✅ | ✅ |
| 6.2 I/O Contracts | ⬜ | ✅ | ⬜ | ✅ | ✅ | ✅ |
| 8.3 Dial Back Aggression | ✅ | ⬜ | ⬜ | ⬜ | ⬜ | ⬜ |

✅ = Apply now | ⬜ = Apply when agent is built
