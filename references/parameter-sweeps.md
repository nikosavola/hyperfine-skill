# Parameter sweeps

Use these when the question is "how does runtime scale with X" rather than "which of these fixed commands is faster" -
hyperfine substitutes the value into `{var}` (or a custom placeholder name) and benchmarks each one as a separate
command.

## `-L/--parameter-list`: explicit values

```bash
hyperfine -L threads 1,2,4,8 'make -j {threads}'
```

Runs `make -j 1`, `make -j 2`, `make -j 4`, `make -j 8` and summarizes relative speed across all of them. Values are a
comma-separated list, substituted verbatim - no arithmetic.

Multiple `-L` options combine as a cross product:

```bash
hyperfine -L compiler gcc,clang -L opt O2,O3 '{compiler} -{opt} main.cpp'
```

This runs all 4 combinations (`gcc -O2`, `gcc -O3`, `clang -O2`, `clang -O3`).

## `-P/--parameter-scan`: numeric ranges

```bash
hyperfine -P threads 1 8 'make -j {threads}'
```

Runs one benchmark per integer in `1..8` inclusive. Add `-D/--parameter-step-size` to change the step:

```bash
hyperfine -P delay 0.3 0.7 -D 0.2 'sleep {delay}'
```

Runs `sleep 0.3`, `sleep 0.5`, `sleep 0.7`.

For non-linear progressions (powers of two, etc.), use shell arithmetic inside the benchmarked command itself, since
`-P`/`-D` only step linearly:

```bash
hyperfine -P size 0 3 'sleep $((1<<{size}))'
```

Runs `sleep 1`, `sleep 2`, `sleep 4`, `sleep 8`. `{size}` is still substituted as plain text before the shell evaluates
the arithmetic, so quoting matters - use single quotes around the whole command so `$(( ))` isn't expanded before
hyperfine substitutes the parameter.

Use the bit-shift `1<<{size}` form, not `2**{size}`: hyperfine's default shell is `/bin/sh`, which on Debian/Ubuntu
(including GitHub Actions' `ubuntu-latest`) is dash, and dash's POSIX arithmetic has no `**` operator - it fails the
command outright rather than computing the power. `<<` is POSIX and works everywhere. If you do need `**` for a more
complex expression, pass `--shell=bash` explicitly instead of relying on the default shell.

## Naming swept commands

By default, each parameterized run is labeled with its literal command line, which gets noisy for long commands or wide
sweeps. Combine with `-n/--command-name` (can reference the parameter too) to keep the summary readable, or just read
the `--export-json`/`--export-csv` output instead of the terminal table for a wide sweep - see `export-and-analysis.md`.
