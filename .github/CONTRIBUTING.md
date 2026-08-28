# Contributing

## Linting and formatting

Everything runs through [pre-commit](https://pre-commit.com) hooks via [`prek`](https://github.com/j178/prek), a drop-in
pre-commit replacement, so the same versions are used locally and in CI:

```bash
uvx prek run --all-files
```

Install it as a git hook so it runs automatically on every commit:

```bash
uvx prek install
```

## Testing

There's no unit test suite - this is a skill, not a library. `.github/workflows/test.yml`'s `smoke-test` job installs
hyperfine and runs the exact commands `SKILL.md` tells an agent to run (basic benchmark, comparison, parameter sweep,
export formats), so a broken flag or malformed command is caught before merge.

## Evaluating changes to the skill

`evals/evals.json` is a set of realistic prompts, each with a human-readable `expected_output` and a list of verifiable
`expectations`, following the
[Claude Code skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator) format.

If you change `SKILL.md` in a way that could affect how an agent uses hyperfine, run the evals with `/skill-creator` (in
Claude Code, on this repo). It spawns a subagent per eval case with the skill available and a matched baseline subagent
without it, grades each run against the case's `expectations`, and aggregates pass rates and a with/without delta. It
writes everything to `../hyperfine-skill-workspace/` (a sibling of this repo, not inside it, so nothing lands in git by
accident).

When adding new eval cases, keep `expectations` objectively checkable from the transcript (not "the benchmark looks
right", but "the agent passed `-N` for a sub-5ms command"), and add negative controls (prompts that should *not* trigger
the skill, like a profiling request) alongside cases that should.

## Before opening a pull request

- `uvx prek run --all-files` passes.
- Keep commits atomic: one logical change per commit, with an imperative-mood message ("Add x", not "Added x" or "Adds
  x").
- If you change a flag or command in `SKILL.md` or `references/`, verify it against a real `hyperfine --help` / actual
  run rather than from memory - flags and defaults change between versions.

## AI usage policy

Using AI tools to accelerate your workflow, whether for prototyping, writing tests, or improving documentation, is
**encouraged**.

However, as a contributor, you remain **fully responsible** for the code and content you submit. Please ensure the
following:

1. **No "AI slop"**: don't submit unreviewed, low-quality, or redundant AI-generated content.
1. **Verify and test**: all AI-generated content must be reviewed and verified to work as intended.
1. **Maintainability**: the content must be clear, idiomatic, and maintainable by a human.
