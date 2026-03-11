# Eval Metadata Files

Detailed assertion specifications for the highest-signal test cases. Each JSON file contains:

1. **Assertions** — Specific checks with `critical` flags and `reason` explanations
2. **Expected pipelines/tool calls** — The correct answer for structural verification
3. **Common mistakes** — Specific errors to watch for and their fixes
4. **Skill references** — Which SKILL.md section or reference file teaches this

## File Naming

Files are named `{eval_id}-{eval_name}.json` matching entries in `../evals.json`.

## Not all evals have metadata files

Only evals with complex patterns, non-obvious correctness criteria, or high common-mistake rates get metadata files. Simpler evals (e.g., change stream syntax) rely on their `evals.json` assertions alone.

## Critical vs non-critical assertions

`"critical": true` marks assertions where failure indicates a fundamental problem — the response would either crash the processor, mislead the user into data loss, or demonstrate that the skill's core guidance was not applied.

## Common mistakes format

```json
{
  "mistake": "What the model does wrong",
  "error": "What happens as a result",
  "fix": "The correct approach"
}
```

These mirror real failure modes observed in eval runs across iterations 14-16.
