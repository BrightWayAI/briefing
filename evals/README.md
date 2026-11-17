# Evals

Run from the repository root:

```
claude plugin eval . --scaffold --ablation none --judge-model sonnet
```

`--scaffold` is required. Cases with a `seed.sh` run `_fixtures/seed-config-root.sh`,
which builds a small fictional config folder in the run's workspace: one client, two
contacts, one prospect, plugin settings, an overnight draft. Their prompts carry a short
`append_system_prompt` saying where that folder is, because this plugin has no
session-start hook to resolve it and the run has no shell.

`_fixtures/seed-config-root.sh` is a copy of the one in the cortex repository. Keep the
copies identical; the nucleus ecosystem check fails if they differ. Never put real
people, clients, or figures in it.

## Kinds of case

- **`*-natural-language`**: does plain phrasing reach the right command.
- **`*-quality`**: is the output right for the fixture's facts (grounded, no invented
  facts, the right next steps).
- **Scenario walk-throughs** (everything else): the prompt describes a situation and
  asks how a step handles it. Phrase their graders as a dry run: judge what the response
  says it would do.

Use `--judge-model sonnet`. The default judge graded equivalent answers inconsistently.
