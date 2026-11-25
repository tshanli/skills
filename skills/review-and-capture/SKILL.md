---
name: review-and-capture
description: Runs code-review, then records reusable lessons in this repo's review ledger. Use when the user runs /review-and-capture or asks to review and capture lessons.
---

# Review and Capture

## 1. Review

- If a skill named `supermatt:code-review` is available, invoke it. Otherwise invoke `code-review`.
- Pass the user's arguments (fixed point, etc.) unchanged. Run it fully and show the report to the user.

## 2. Collect findings

Take the verbatim `Standards` and `Adversarial` sections of the report, keeping file paths (either section may be absent). Ignore the `Spec` section.
If neither section has findings, end your reply with `[Review Memory] nothing to record` and stop.

## 3. Capture

The capture steps live in `~/.agents/skills/capture-review/SKILL.md`.

- **If you can spawn a subagent with a model override** (Claude Code's Agent tool has a `model` parameter): spawn a general-purpose subagent with `model: haiku`. Its prompt is exactly:
  `Run cat ~/.agents/skills/capture-review/SKILL.md and follow its steps exactly on these review findings:`
  followed by the collected sections verbatim. Add nothing else.
- **Otherwise**: run `cat ~/.agents/skills/capture-review/SKILL.md` yourself and follow its steps on the collected sections.

## 4. Report

Append the single `[Review Memory] ...` line to the end of your reply verbatim, even if it is an error. Never edit the ledger or the rules file yourself.

