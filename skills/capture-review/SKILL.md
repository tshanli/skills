---
name: capture-review
description: Records Standards / Adversarial findings from a code-review report into this repo's review ledger, and promotes a pattern to a rule in .agents/rules/code-review-rules.md once it has appeared in 2 different places. Only run by review-and-capture; not for reviewing code.
disable-model-invocation: true
---

# Capture Review

Input: the verbatim Standards and/or Adversarial sections of one code-review report (either may be absent).
All reads and writes go through the script. **Never** read or edit `.agents/review-memory.jsonl` or `.agents/rules/code-review-rules.md` directly.

## Steps

1. From the repo root, run:
   `python3 ~/.agents/skills/capture-review/scripts/capture.py list`
   Each output line is: `id  axis  status  count  pattern`.

2. Go through the findings one by one:
   - Keep only findings under the **Standards** and **Adversarial** headings. Discard any Spec content.
   - Discard one-off issues: typos, a single bad name, logic specific to this one requirement. Keep only issues that could recur in a different file.
   - Compare the **core anti-pattern** against the listing, ignoring file names, variable names and line numbers:
     - Same pattern: set `match` to the existing id. **Match `rejected` entries too**; never create a new id because an entry was rejected. The script keeps counting it but never promotes it again.
     - New pattern: set `match` to `null` and give a new kebab-case `id`.
     - **When unsure, treat it as new.** A false merge promotes a wrong rule, which costs more than a missed merge.
   - `file`: repo-relative path of the file the finding is in; `null` if the report gives none.
   - New patterns also need a `pattern` (one sentence naming the core problem) and a `rule` (one line starting with `- `), written in English:
     - Standards: `- **[Context]**: Never X; do Y instead (documented standard, repeatedly violated; see <standards file>).`
     - Adversarial: `- **[Context]**: Never X; do Y instead. Why: <one-sentence failure mode>.`

3. Submit all decisions in one call:
   ```bash
   python3 ~/.agents/skills/capture-review/scripts/capture.py apply <<'EOF'
   {"findings":[
     {"axis":"adversarial","match":null,"id":"unchecked-nullable","pattern":"Iterates a nullable collection without a null check","rule":"- **[Collection iteration]**: Never iterate a nullable collection directly; check for null first. Why: crashes when null is passed.","file":"src/a.ts"},
     {"axis":"standards","match":"magic-number","file":"src/b.cs"}
   ]}
   EOF
   ```
   If nothing is worth recording, run `python3 ~/.agents/skills/capture-review/scripts/capture.py skip` instead.

4. If the output starts with `[Review Memory] ERROR:`, fix the JSON as the message says and **retry once**. If it still fails, return the error verbatim.

5. Return **only** the last line of the script output, with no explanation. Keep any `NOTE:` it contains; it tells the user how to fix their setup.
