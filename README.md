# hyperfine-skill

[![Test](https://github.com/nikosavola/hyperfine-skill/actions/workflows/test.yml/badge.svg)](https://github.com/nikosavola/hyperfine-skill/actions/workflows/test.yml)

An [Agent Skill](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) that teaches an AI agent
how to benchmark a command with [hyperfine](https://github.com/sharkdp/hyperfine) correctly, instead of eyeballing a
single `time` invocation or guessing at flags it hasn't used before.

## Install

```bash
npx skills add nikosavola/hyperfine-skill
```

<details>
<summary>Install manually instead</summary>

Clone into your agent's skills directory. The folder name must match the skill's `name` field (`hyperfine`):

```bash
git clone https://github.com/nikosavola/hyperfine-skill.git ~/.claude/skills/hyperfine
```

</details>

## Requirements

- [`hyperfine`](https://github.com/sharkdp/hyperfine) itself. `SKILL.md` covers installing it (`brew`, `cargo`, `conda`,
  `apt`, or a prebuilt binary from the releases page) if it isn't already on `PATH`.

## How it works

Ask an agent to benchmark a command, compare two implementations, or check whether a change regressed runtime. Rather
than researching hyperfine's flags from scratch every time, it follows `SKILL.md`: pick the right warmup/shell/run-count
flags for the situation, export machine-readable results instead of eyeballing the terminal table, and read hyperfine's
own outlier/cache warnings instead of ignoring them. Longer-tail topics (parameter sweeps, the full export schema) live
in `references/` and are only pulled in when actually needed, so the common case stays a short read.

See [SKILL.md](SKILL.md) for the full instructions given to the agent.

## What hyperfine provides

hyperfine runs a command repeatedly (auto-detecting a sensible run count, or use `-r` to fix one), discards a warmup
period on request, and reports mean/median/stddev with outlier detection - all things a single `time` call can't give
you. It also accounts for shell startup overhead, which is why `-N`/`--shell=none` matters for very fast commands.

## Limitations

- hyperfine measures whole-process wall-clock time from outside. It has no idea where time goes *inside* the program -
  that's a profiler's job (`perf`, `py-spy`, etc.), not hyperfine's.
- Below roughly a millisecond, hyperfine's own overhead (process spawn, and the shell's if not disabled with `-N`)
  starts to dominate the signal. It's the wrong tool for microbenchmarking a fast in-process function; a language-native
  benchmark harness measures that correctly.
- Results are only as clean as the machine they were measured on: background load, thermal throttling, and power-state
  switching all show up as the outliers hyperfine warns about.
