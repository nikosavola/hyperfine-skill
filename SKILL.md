---
name: hyperfine
description: Benchmarks the wall-clock runtime of shell commands using hyperfine, runs a command many times, accounts for shell startup overhead, and reports mean/median/stddev with outlier detection. Use whenever the user wants to know how fast a command is, wants to compare the speed of commands or implementations, is checking whether a change regressed runtime, or wants to sweep a parameter (threads, input size, flags) to see how runtime scales - even if they never say "benchmark" or "hyperfine" (e.g. "which of these is faster", "did this get slower"). Do not use for in-process microbenchmarks under about 1ms, where process-spawn overhead dominates - point the user to a language-native benchmark harness instead. Do not use for profiling where time goes inside a program - hyperfine only measures whole-process wall-clock time from outside.
when_to_use: |
  - "How fast is X" / "benchmark this command" / "time how long X takes".
  - "Is X faster than Y" / "compare the performance of A vs B".
  - "Did this change make the build/script slower" (before/after comparison).
  - Sweeping a parameter (thread count, input size, flag value) to see how runtime scales.
  - Do NOT use for microbenchmarking a function/expression that runs in well under a millisecond - hyperfine's own process-spawn overhead (several ms, more with a shell) will swamp the signal. Point the user to a language-native benchmarking library instead.
  - Do NOT use for profiling (where inside the program time goes) - hyperfine only measures wall-clock time of the whole process, it doesn't sample call stacks or line-level timings.
argument-hint: <command to benchmark> (e.g. "make build", "curl https://example.com", or two+ commands to compare)
---

# Benchmarking with hyperfine

Don't estimate how fast a command is, and don't time it with one `time` invocation - a single run is noise. Use
`hyperfine`, which runs the command enough times to get a statistically meaningful mean, stddev, and outlier check, and
already accounts for shell startup overhead so you're not measuring bash instead of the command.

## Step 0: Confirm hyperfine is available

```bash
command -v hyperfine
```

If missing, install it (prefer the OS/language package manager already in use in the project):

```bash
brew install hyperfine          # macOS / Linuxbrew
cargo install hyperfine          # any platform with a Rust toolchain
conda install -c conda-forge hyperfine
sudo apt install hyperfine       # Debian/Ubuntu (may lag behind the latest release)
```

Or download a prebuilt binary from <https://github.com/sharkdp/hyperfine/releases>.

## Step 1: Run the benchmark

Basic form:

```bash
hyperfine 'the command to benchmark'
```

Reach for these flags based on what the command actually needs - don't just run the bare default for anything that isn't
a simple, already-fast command:

| Situation                                                                   | Flag                                                 | Why                                                                                                                                                                                                                                      |
| --------------------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Command runs in a few ms or less                                            | `-N` (`--shell=none`)                                | Skips the intermediate shell entirely so its spawn time can't swamp the measurement. Only works if the command needs no shell features (pipes, `&&`, globbing); if it does, quote it as one shell command and accept the shell overhead. |
| Command is I/O- or cache-sensitive                                          | `-w, --warmup <N>`                                   | Runs N throwaway iterations first so filesystem/page caches are warm before the timed runs start.                                                                                                                                        |
| Need to clear caches every run                                              | `-p, --prepare <CMD>`                                | Runs `CMD` before *every* timed run, e.g. a command that drops filesystem caches.                                                                                                                                                        |
| One-time setup (build, server start)                                        | `-s, --setup <CMD>`                                  | Runs `CMD` once before each command's whole set of runs, not per-run.                                                                                                                                                                    |
| Need to tear something down per run (a server you started with `--prepare`) | `-C, --conclude <CMD>`                               | Runs `CMD` after each timed run. Requires hyperfine >= 1.19.                                                                                                                                                                             |
| Command is slow (seconds+)                                                  | `-m/--min-runs`, `-M/--max-runs`, or `-r/--runs <N>` | Hyperfine defaults to at least 10 runs with no default cap: a 45s command becomes a 7.5-minute benchmark unless you bound it. Use `-r 3` for something you know is slow and consistent, or `-M 5` to cap an otherwise auto-tuned run.    |
| Command is expected to fail / return non-zero                               | `-i, --ignore-failure[=<MODE>]`                      | Without this, hyperfine aborts the whole benchmark on the first non-zero exit.                                                                                                                                                           |
| Program special-cases output going to `/dev/null` (e.g. `grep` skips work)  | `--output=pipe`                                      | Feeds output through a real pipe instead of hyperfine's default `null`, so the program can't take a shortcut that a real caller wouldn't get.                                                                                            |

## Step 2: Comparing commands

Pass multiple commands to get a relative-speed summary against the fastest one:

```bash
hyperfine -n 'old' 'old-impl-command' -n 'new' 'new-impl-command'
```

`-n/--command-name` labels each command in the output; without it hyperfine just prints the command line itself, which
gets unreadable when commands are long. Add `--reference '<cmd>'` (hyperfine >= 1.19, optionally paired with
`--reference-name`, hyperfine >= 1.20) to compare everything against a specific baseline instead of whichever command
happened to be fastest.

For sweeping a parameter (thread counts, input sizes, compiler flags) across many values instead of listing commands by
hand, see `references/parameter-sweeps.md`.

## Step 3: Read the result - don't eyeball the pretty table for anything you'll act on

The terminal table is fine for a quick "which is faster" glance. For anything you're going to quote a number from,
report in a PR, or gate a decision on, export machine-readable output and read the actual numbers instead of guessing
from the printed `±`:

```bash
hyperfine --export-json results.json 'the command'
jq '.results[] | {command, mean, median, stddev}' results.json
```

`--export-markdown FILE` is the right choice when the result is going straight into a PR description or a message to the
user - it produces a ready-to-paste comparison table. See `references/export-and-analysis.md` for the full JSON schema,
CSV/AsciiDoc/org-mode formats, and per-iteration logging via `$HYPERFINE_ITERATION`.

## Step 4: Understand the warnings

Hyperfine prints exactly these warnings, and they're informative, not decorative - act on them rather than ignoring:

| Warning (as printed)                                                                    | Meaning                                                                                                                          | Fix                                                                                                       |
| --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| "Command took less than 5 ms to complete..."                                            | Below this, shell startup noise dominates the true signal.                                                                       | Add `-N`/`--shell=none` if the command doesn't need a shell.                                              |
| "The first benchmarking run for this command was significantly slower than the rest..." | Cold cache on the first run.                                                                                                     | Add `--warmup`, or `--prepare` to clear caches deliberately every run.                                    |
| "Statistical outliers were detected..."                                                 | Some runs were far from the median - likely system noise (another process, thermal throttling, a laptop switching power states). | Re-run on a quieter machine, or add `--warmup`/`--prepare` if not already set.                            |
| "Ignoring non-zero exit code."                                                          | Printed once you've already passed `-i`; not an error, just confirming the flag took effect.                                     | Nothing to fix - only worth double-checking that a non-zero exit isn't masking a real bug in the command. |

## Reporting results

State the mean and stddev (or point at the exported table), name which command was faster and by how much (hyperfine
prints this as "N.NN ± N.NN times faster"), and mention any warning that fired rather than silently dropping it - a
result with an unresolved outlier warning is weaker evidence than one without.
