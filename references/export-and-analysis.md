# Export formats and programmatic analysis

Timing units differ by format: JSON and CSV are always in seconds, regardless of `--time-unit`. The Markdown, AsciiDoc,
and org-mode table exports follow whatever `--time-unit` is set to (or an auto-picked unit if it isn't set) - see the
display-only options at the bottom of this file.

## `--export-json FILE`

The most complete format: summary statistics plus every individual run's time, so you can compute anything beyond
mean/stddev/median yourself (percentiles, a histogram, a t-test between two commands).

```json
{
  "results": [
    {
      "command": "echo hi",
      "mean": 0.00033025575124470355,
      "stddev": 0.0003870453532064868,
      "median": 0.00027484069999999985,
      "user": 0.00017302347457627114,
      "system": 0.0003869804449152538,
      "min": 0.0,
      "max": 0.0099595837,
      "times": [0.0, 0.0001552576999999998, "... one entry per run"],
      "exit_codes": [0, 0, "... one entry per run"],
      "memory_usage_byte": [0, 0, "... peak RSS per run, one entry per run"],
      "parameters": { "threads": "4" }
    }
  ]
}
```

`parameters` is only present when the command came from `-L`/`-P`. Read it with `jq`:

```bash
jq '.results[] | {command, mean, stddev, median}' results.json
jq '.results | sort_by(.mean) | .[0].command' results.json   # fastest command
```

## `--export-csv FILE`

One row per command (not per run): `command,mean,stddev,median,user,system,min,max`. When the command came from a
`-L`/`-P` parameter sweep, one `parameter_<name>` column is appended per swept variable (e.g. `parameter_threads`), so a
script reading this file with a fixed column list should append those rather than assuming a static header. Use CSV when
you just need the summary numbers in a spreadsheet; use JSON if you need the individual run times.

## `--export-markdown FILE`

A ready-to-paste GitHub-flavored table with a `Relative` column showing the speed multiplier against the fastest
command. Best default when the result is going into a PR description, issue comment, or chat reply - don't hand-format a
table from the JSON/CSV when this already produces one.

If you used `--reference` to pin a specific baseline (SKILL.md's Step 2), note that this `Relative` column ignores it -
it's always normalized to whichever command was fastest, not your chosen reference. The terminal summary's "N.NN times
slower/faster than `<reference>`" line is reference-aware; the exported table is not. If you need reference-relative
numbers in the exported table itself, compute them from each command's `mean` in the JSON export instead.

## `--export-asciidoc FILE` / `--export-orgmode FILE`

Same summary table as the Markdown export, in AsciiDoc or Emacs org-mode syntax, for docs written in those formats.

## Multiple export formats at once

All `--export-*` flags can be combined in a single invocation; hyperfine writes each file from the same run:

```bash
hyperfine --export-json results.json --export-markdown results.md 'the command'
```

## Per-iteration output logging

`--output` controls what happens to the *benchmarked command's* stdout/stderr, not hyperfine's own export files. To
capture what each individual run printed (not just its timing), use the `$HYPERFINE_ITERATION` environment variable,
which hyperfine sets to the current 0-based iteration number for each command:

```bash
hyperfine 'my-command > output-${HYPERFINE_ITERATION}.log'
```

## Display-only options (don't affect exported numbers)

- `--time-unit {microsecond,millisecond,second}` - forces the unit shown in the terminal and in Markdown/AsciiDoc/org
  exports (JSON and CSV stay in seconds regardless).
- `--sort {auto,command,mean-time}` - controls the order of the terminal comparison summary and the markup exports;
  `auto` orders by time in the summary but by input order in markup tables.
- `--style {auto,basic,full,nocolor,color,none}` - controls terminal color/interactivity, irrelevant when output is
  piped or exported.
