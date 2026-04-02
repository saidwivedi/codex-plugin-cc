---
description: Send the current implementation plan to Codex for independent review and critique
argument-hint: '[--wait|--background] [--effort <none|minimal|low|medium|high|xhigh>] [focus areas]'
disable-model-invocation: true
allowed-tools: Read, Write, Bash(node:*), Bash(mktemp:*), Bash(rm:*), AskUserQuestion
---

Send the current implementation plan to Codex for independent critical review.

Raw slash-command arguments:
`$ARGUMENTS`

## Prerequisites

- The Codex plugin must be installed and authenticated. If not, tell the user to run `/codex:setup`.
- There must be an active plan in the current conversation — either from plan mode or a plan you previously generated. If no plan exists, tell the user there is no plan to review.

## Step 1: Extract the plan

Search your conversation context for the most recent implementation plan. Capture:
- The goal/objective
- All steps and sub-steps
- File paths mentioned
- Architectural decisions and trade-offs

If there are multiple plans, use the most recent one.
If no plan exists in the conversation, stop and tell the user.

## Step 2: Write the plan to a temp file

```bash
PLAN_FILE=$(mktemp /tmp/codex-plan-review-XXXXXX.md)
```

Write the extracted plan to that file using the Write tool.

## Step 3: Build the review prompt

Read back the plan file, then construct this prompt for Codex (substituting the actual plan content and any user-provided focus areas):

```
You are reviewing an implementation plan created by Claude Code. Your job is to find flaws, risks, missed edge cases, and suggest improvements. Be specific and actionable — reference steps by number.

<plan>
{PLAN CONTENT}
</plan>

<review_instructions>
Critically review this plan for:
1. Correctness — Will the proposed approach actually work? Are there logical errors?
2. Completeness — Are there missing steps or unhandled edge cases?
3. Risk — What could go wrong? What are the failure modes?
4. Ordering — Are the steps in the right order? Are there dependency issues?
5. Alternatives — Is there a simpler or more robust approach?
6. Architecture — Are the design decisions sound? Any anti-patterns?
</review_instructions>

<user_focus>
{USER FOCUS AREAS or "No additional focus areas specified."}
</user_focus>

<output_contract>
Structure your review as:
- **Verdict**: APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION
- **Critical Issues**: Blocking problems that must be fixed (if any)
- **Warnings**: Non-blocking concerns worth addressing
- **Suggestions**: Optional improvements
- **Summary**: 2-3 sentence overall assessment
</output_contract>
```

## Step 4: Send to Codex

Execution mode:
- If `$ARGUMENTS` includes `--wait`, run in foreground.
- If `$ARGUMENTS` includes `--background`, run in background.
- Otherwise, default to foreground.

Strip `--wait` and `--background` from the prompt text. Preserve `--effort` if provided.

Foreground flow:
```bash
node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task [--effort <effort>] "the constructed review prompt"
```

Return stdout verbatim.

Background flow:
```typescript
Bash({
  command: `node "${CLAUDE_PLUGIN_ROOT}/scripts/codex-companion.mjs" task [--effort <effort>] "the constructed review prompt"`,
  description: "Codex plan review",
  run_in_background: true
})
```
Then tell the user: "Codex plan review started in the background. Check `/codex:status` for progress."

## Step 5: Cleanup

```bash
rm -f "$PLAN_FILE"
```

## Rules

- This is review-only. Do NOT apply changes or modify the plan based on Codex's output.
- Return Codex output verbatim. Do not paraphrase, summarize, or add commentary.
- The plan text comes from YOUR conversation context, not from files on disk.
- After presenting the review, ask the user if they want to incorporate any of the feedback.
- If the plan exceeds ~4000 words, recommend `--background`.
