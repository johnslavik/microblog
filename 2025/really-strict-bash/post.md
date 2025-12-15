# Really strict Bash

`set -eu` and its variants (`set -e`, `set -euo pipefail`) are often referred to as Bash’s
“strict mode”.

You usually put them at the top of your script, right under the shebang (`#!`) and any
top‑level comments.

The `-e` and `-u` flags in particular are invaluable when something goes wrong in a Bash
script. They enable behavior that most programming languages provide by default, but
Bash does not.

We will start with a minimal header and progressively make it stricter as new failure
modes appear.

```bash
set -eu
```

## The commonly known flags

### `-e`: exit on error

`-e` tells Bash (and the Bourne shell, i.e. `sh`) to terminate execution immediately when a
standalone command fails.

Conceptually, this is similar to exceptions in other languages. It makes intuitive sense:
most lines in a script depend on earlier ones. If something fails and execution continues,
the script may run in a partially initialized or invalid environment.

Without `-e`, failures are easy to miss and hard to debug.

### `-u`: error on unset variables

`-u` causes parameter expansion to fail when a variable is unset.

This is a life‑saver for cases like:

```bash
rm -rf "$HOME/$CUSTOM_DIR"
```

If `CUSTOM_DIR` is unset, Bash would otherwise expand this to:

```bash
rm -rf "$HOME/"
```

With `-u`, the script fails immediately instead of potentially deleting your entire home
directory.

Both `-e` and `-u` are specified in the POSIX standard (IEEE 1003.1‑2001), documented
[here](https://pubs.opengroup.org/onlinepubs/9799919799/utilities/V3_chap02.html#set).

At this point, our header looks like this:

```bash
set -eu
```

## But this isn’t everything

These two flags are useful, but they are only a small part of what is needed for robust
error handling in Bash.

### `-e` does not catch errors from subshells

Consider this example:

```bash
DIR=$(mktemp -d)
cd $DIR
```

If `mktemp` fails, `-e` will **not** stop the script. The failure happens inside a command
substitution (a subshell), and Bash happily proceeds to the next line.

This snippet has another, more subtle bug.

If `DIR` is empty, `cd $DIR` expands to `cd`, which takes you to `$HOME`.

Quoting matters:

```diff
DIR=$(mktemp)
-cd $DIR
+cd "$DIR"  # if DIR is empty, we stay in the current directory
```

### `-e` does not preserve the correct exit status

Exit statuses carry meaning.

- `130` indicates an interrupt (try: `sleep 5`, then press Ctrl‑C)
- `127` means “command not found” (try: `lkxcvlkm; echo $?`)

In pipelines, `-e` alone is not enough to preserve this information.

### Two more flags to address these problems

#### `-E`: extend `-e` into subshells

`-E` makes `-e` effective in functions, command substitutions, and subshells.
It is not specified by POSIX, but it is supported by Bash and should be used in Bash scripts.

We update our header accordingly:

```diff
-set -eu
+set -Eeu
```

#### `-o pipefail`: fail if any part of a pipeline fails

By default, a pipeline’s exit status is that of its **last** command.

With `pipefail`, the pipeline fails if *any* command in it fails, and the failure propagates.

Updating the header again:

```diff
-set -Eeu
+set -Eeuo pipefail
```

## Error handling is more than exiting

Failing early is good, but *knowing where and why* a failure occurred is even better.

Bash provides additional options that, when combined, make errors predictable and
diagnosable.

### `-E`: make `ERR` traps actually useful

Without `-E`, an `ERR` trap does not run in functions, command substitutions, or subshells.

```bash
set -e
trap 'echo "Error on line $LINENO"' ERR

fail() {
  false
}

fail   # nothing happens
```

With `-E` (which we already enabled):

```bash
set -Ee
trap 'echo "Error on line $LINENO"' ERR

fail() {
  false
}

fail   # Error on line 7
```

Our header remains unchanged here:

```bash
set -Eeuo pipefail
```

### `-T`: inherit traps into functions and subshells

`-T` (also known as `functrace`) complements `-E`. It ensures that `DEBUG` and `ERR` traps
are inherited by functions and subshells.

This matters for logging, cleanup, and stack‑trace‑like diagnostics.

```bash
set -ETe
trap 'echo "ERR at ${BASH_SOURCE}:${LINENO}"' ERR
```

We now tighten the header further:

```diff
-set -Eeuo pipefail
+set -ETeuo pipefail
```

## Pipelines and silent failures

We already enabled `pipefail`, but it is worth seeing *why* it matters.

```bash
set -e

grep "needle" haystack.txt | sort | uniq
echo "done"
```

If `grep` finds no matches, it exits non‑zero — but `uniq` succeeds, so the pipeline as a
whole succeeds. The script prints `done`.

With `pipefail` (already in our header), the failure propagates correctly.

No header change here:

```bash
set -ETeuo pipefail
```

## `-C`: protect files from accidental clobbering

`-C` enables *noclobber*, preventing redirections from overwriting existing files.

This avoids another class of destructive mistakes.

```bash
OUTPUT="$HOME/output.txt"
> "$OUTPUT"
```

If `OUTPUT` accidentally points to `$HOME/.bashrc`, your shell configuration is gone.

With `-C`:

```bash
set -C
> "$OUTPUT"
```

We update the header again:

```diff
-set -ETeuo pipefail
+set -CETeuo pipefail
```

## `-x`: make execution observable

`-x` prints each command as Bash executes it, after expansion.

This is invaluable for debugging CI failures or production issues.

```bash
set -x
cp "$SRC" "$DST"
```

If you want tracing enabled by default, update the header one last time:

```diff
-set -CETeuo pipefail
+set -CETeuxo pipefail
```

(Or make it conditional if you prefer.)

## Syntax checking: fail before doing damage

A useful trick is to force Bash to parse the script again before executing it:

```bash
: Check syntax
bash "$0"
```

If there is a syntax error, Bash exits before any destructive commands run.

However, **this does not work if the script is sourced**:

```bash
source my-script.sh
```

In that case, `$0` refers to the parent shell, not the sourced file. Syntax checking for
sourced scripts must be done explicitly:

```bash
bash -n my-script.sh
```

## Putting it all together

Each flag was added to eliminate a specific failure mode:

- `-e`            exit on errors  
- `-u`            fail on unset variables  
- `-E`            propagate `ERR` traps  
- `-T`            inherit traps into functions and subshells  
- `-o pipefail`   detect failures in pipelines  
- `-C`            prevent accidental overwrites  
- `-x`            trace execution  

Which leaves us with the final, fully evolved header:

```bash
set -CETeuxo pipefail
: Check syntax
bash "$0"
```

It looks intimidating, but every character is there because Bash is *forgiving by default* —
sometimes dangerously so.

Once you internalize these options, your scripts stop failing silently and start failing
early, loudly, and in ways that are easy to diagnose.
