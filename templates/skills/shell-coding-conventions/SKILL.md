---
name: shell-coding-conventions
description: Shell scripting conventions for bash and zsh covering strict mode, argument parsing, error handling, quoting, and portability. Load before writing, reviewing, or refactoring shell scripts.
license: MIT
metadata:
  author: Heiko Panjas
  version: "1.0"
---

# Shell Coding Conventions

Read this skill before writing, reviewing, or refactoring shell scripts in this project.
It covers script structure, strict mode, argument parsing, error handling, formatting,
common idioms, and portability between bash and zsh.

---

## Shell Coding Conventions

**Script Structure:**

- Start every script with `#!/usr/bin/env bash`
- Follow the shebang with a header comment describing what the script does: one imperative
  sentence for simple scripts, multiple `#`-prefixed paragraphs (blank `#` line as a
  separator) for anything more involved
- No banner boxes, no author/date/version metadata in the header — that belongs in git history
- Blank line, then strict mode, then the rest of the script
- Example:

  ```bash
  #!/usr/bin/env bash
  # Extract audio from a video as MP3 (320k).

  set -euo pipefail
  ```

**Strict Mode:**

- Every script starts with `set -euo pipefail` immediately after the header comment
- Exception: a batch loop that must survive a single item's failure uses `set -uo pipefail`
  (no `-e`) and instead tallies success/failure counters, exiting nonzero only if anything
  failed:

  ```bash
  set -uo pipefail

  converted=0
  failed=0
  for file in "$dir"/*.mp4; do
      if ffmpeg -i "$file" -c:v libx264 "${file%.mp4}.mkv"; then
          converted=$((converted + 1))
      else
          echo "error: failed to convert $file" >&2
          failed=$((failed + 1))
      fi
  done

  echo "Converted: $converted  Failed: $failed"
  (( failed == 0 )) || exit 1
  ```

- **Under `set -e`, `some_cmd; exit_code=$?` does not capture a real failure** — the script
  aborts on `some_cmd` before the assignment ever runs. Use `exit_code=0; some_cmd ||
  exit_code=$?`, or wrap the command in `if`/`else`, whenever a command's exit code drives
  cleanup or reporting logic:

  ```bash
  # Wrong: never triggers under set -e — the script exits before reaching this line
  some_cmd; exit_code=$?

  # Correct
  exit_code=0
  some_cmd || exit_code=$?
  ```

- **Under `set -o pipefail`, never pipe a long-output producer straight into `grep -q`,
  `head`, or anything that exits after its first match.** The consumer closes the pipe
  early, the producer dies of `SIGPIPE` while it still has output queued, and `pipefail`
  reports that `SIGPIPE` death as the pipeline's failure — even though the consumer already
  matched successfully. Capture the producer's output first, then test the captured string:

  ```bash
  # Wrong: races against SIGPIPE under pipefail
  ffmpeg -filters 2>&1 | grep -q rubberband

  # Correct
  filters=$(ffmpeg -filters 2>&1)
  grep -q rubberband <<< "$filters"
  ```

**Help and Argument Parsing:**

- Handle `-h`/`--help` **before** any dependency check, path resolution, or side-effecting
  work, and exit 0 with zero side effects
- Two tiers, chosen by how many options a script has:
  - **Simple scripts** (few or no options) use an inline guard with a heredoc:

    ```bash
    if [[ $# -lt 1 || "$1" == "-h" || "$1" == "--help" ]]; then
        cat <<EOF
    Usage: $(basename "$0") input.mp4

    Writes <basename>.mp3 (libmp3lame, 320k) next to the input.
    EOF
        [[ "${1:-}" == "-h" || "${1:-}" == "--help" ]] && exit 0
        exit 1
    fi
    ```

  - **Scripts with options** use a `usage()` function, placed immediately after strict mode
    (before any other function), and a `while`/`case` parse loop:

    ```bash
    usage() {
      cat <<EOF
    Usage: $(basename "$0") [OPTIONS] INPUT_DIR OUTPUT

    One-paragraph description of what the script does.

    Arguments:
      INPUT_DIR              Directory containing input files
      OUTPUT                 Output file path

    Options:
      -n, --count COUNT      Number of items to process (default: 3)
      -f, --format FORMAT    Output format (required)
          --dry-run          Preview without making changes
      -h, --help             Show this help and exit

    Example:
      $(basename "$0") --count 2 --format mp4 ./clips out.mp4
    EOF
    }

    count=3
    format=""
    dry_run=0
    positional=()

    while (( $# > 0 )); do
      case "$1" in
        -n|--count)
          if (( $# < 2 )); then
            echo "error: missing value for $1" >&2
            exit 2
          fi
          count="$2"
          shift 2
          ;;
        --count=*)
          count="${1#*=}"
          shift
          ;;
        -f|--format)
          format="$2"
          shift 2
          ;;
        --dry-run)
          dry_run=1
          shift
          ;;
        -h|--help)
          usage
          exit 0
          ;;
        --)
          shift
          positional+=("$@")
          break
          ;;
        -*)
          echo "error: unknown option: $1" >&2
          exit 2
          ;;
        *)
          positional+=("$1")
          shift
          ;;
      esac
    done

    if (( ${#positional[@]} != 2 )); then
      echo "error: expected INPUT_DIR and OUTPUT arguments" >&2
      echo >&2
      usage >&2
      exit 2
    fi
    ```

  - Heredoc section order is fixed: `Usage:` line, blank, one-paragraph description,
    `Arguments:` (if there are positionals), `Options:`, any trailing constraints, `Example:`
  - Spell defaults `(default: X)` and mandatory options `(required)`
  - Indent long-only options (no short form) six spaces so their `--` aligns under the short
    options' `--`
  - Support `--opt=value` as a paired `--opt=*)` case arm using `"${1#*=}"`
  - Collect positional arguments into an array, and support `--` to end option parsing
  - Validate parsed values in a dedicated block after the parse loop — one `if` per rule,
    each with its own message and `exit 2`:

    ```bash
    if [[ -z "$format" ]]; then
      echo "error: the --format option is required" >&2
      exit 2
    fi

    if [[ ! "$count" =~ ^[1-9][0-9]*$ ]]; then
      echo "error: count must be a positive integer: $count" >&2
      exit 2
    fi
    ```

  - Never use `getopts` — it does not support long options

**Exit Codes:**

Use exactly three exit codes, consistently:

- `0` — success, or help requested
- `1` — runtime failure (missing dependency, missing file, output already exists, a command
  failed)
- `2` — usage/argument error (unknown option, missing option value, wrong positional count,
  invalid option value)

**Messages and Output:**

- No color. No ANSI escapes, no `tput`, no `RED=`/`GREEN=` variables — plain text only
- The status-symbol vocabulary is `✓` (success), `✗` (failure), `→` (before/after
  comparison); do not invent additional symbols
- Errors go to stderr; progress and success messages go to stdout:

  ```bash
  if ! command -v ffmpeg >/dev/null 2>&1; then
      echo "error: ffmpeg is not installed" >&2
      exit 1
  fi

  echo "✓ Successfully processed: $input_file"
  original_size=$(du -h "$input_file" | cut -f1)
  new_size=$(du -h "$output" | cut -f1)
  echo "  Original: $original_size → New: $new_size"
  ```

- Error message prefix is always lowercase: `echo "error: ..." >&2` — not `Error:`
- Sub-details are indented two spaces *inside the message string*, not by shell indentation
- Prefer `echo` for plain text (the default); use `printf` only for numeric formatting and
  for safely emitting array contents:

  ```bash
  printf "%.1f cm = %.2f inches\n" "$cm" "$inches"
  printf '%s\n' "${files[@]}"
  ```

- Echo the resolved parameters before doing the actual work, so a run can be understood from
  its own output

**Variables:**

- Use lowercase `snake_case` for ordinary variables; reserve `UPPER_SNAKE_CASE` for
  constants and values meant to be overridable via the environment (`SCRIPT_DIR`,
  `LLM_MODEL`)
- Use `readonly` for true constants that never change after assignment
- Declare `local` variables one per line at the top of a function, mirroring the positional
  parameters:

  ```bash
  extract_frame() {
    local source="$1"
    local timestamp="$2"
    local output="$3"
    ...
  }
  ```

- Quote every expansion in command position: `"$input"`, `"$@"`, `"${array[@]}"`,
  `"$(dirname "$0")"` — including nested command substitution at both levels
- Use `${var}` braces only when the expansion abuts other characters
  (`"${dirname}/${basename}.${extension}"`); leave standalone expansions plain (`"$output"`)
- Use `${1:-}` to probe an optional argument safely under `set -u`, `${var:-default}` for
  real defaults, and `${var:?message}` to assert a required value with a clear error:

  ```bash
  dir="${1:-.}"
  output="${2:?output path is required}"
  ```

- Declare option defaults as a flat block immediately before the parse loop, one assignment
  per line, no blank lines between them; represent booleans as `0`/`1` and test them
  arithmetically:

  ```bash
  count=3
  format=""
  dry_run=0
  ...

  if (( dry_run == 0 )); then
  ```

**Functions:**

- Declare functions as `name() {` — never the `function` keyword
- Place helper functions between strict mode and the option-defaults block; `usage()` comes
  first if present
- Do not wrap the script body in `main()` — scripts run straight-line, top to bottom
- A function that performs an action returns nonzero with a stderr message when the caller
  should decide what to do next, rather than exiting directly:

  ```bash
  normalize_audio() {
    local source="$1"
    local output="$2"
    if ! ffmpeg -i "$source" -ar 44100 -ac 2 "$output"; then
      echo "error: could not normalize audio: $source" >&2
      return 1
    fi
  }
  ```

- A small pure helper returns its result on stdout:

  ```bash
  abs_path() {
    local path="$1"
    echo "$(cd "$(dirname "$path")" && pwd -P)/$(basename "$path")"
  }
  ```

**Formatting:**

- Indent with 2 spaces; spaces only, never tabs
- Put `then`/`do` on the same line as `if`/`while`/`for` — never on their own line
- Prefer `[[ ]]` over `[ ]` for conditionals
- Prefer `(( ))` for arithmetic and numeric comparisons over `[[ -gt ]]`-style tests
- Quote the `case` scrutinee: `case "$1" in`; indent arms one level, arm bodies two levels,
  `;;` on its own line at the body's indentation
- Split long commands across lines with a trailing `\`, one flag group per line, continuation
  indented one level:

  ```bash
  ffmpeg -hide_banner -nostdin \
    -i "$input" \
    -c:v libx264 \
    -preset slow \
    -y "$output"
  ```

- Target roughly 80 columns; long lines are acceptable when they are a single unavoidable
  token, such as an `ffprobe -show_entries` list or a filter graph

**Common Idioms:**

- Check external dependencies up front with `command -v tool >/dev/null 2>&1` — never
  `which`, `hash`, or `type -p` — and fail with a clear message before doing any work:

  ```bash
  for tool in ffmpeg ffprobe; do
    if ! command -v "$tool" >/dev/null 2>&1; then
      echo "error: $tool is not installed" >&2
      exit 1
    fi
  done
  ```

- Pair `mktemp` with an immediate `trap` on the very next line, single-quoted so expansion is
  deferred to trap time, not set time:

  ```bash
  tmpdir="$(mktemp -d)"
  trap 'rm -rf "$tmpdir"' EXIT
  ```

  A double-quoted trap body (`trap "rm -rf $tmpdir" EXIT`) expands immediately when the trap
  is set rather than when it fires — avoid it.
- Decompose a path into its parts with parameter expansion, not `sed`/`awk`:

  ```bash
  dirname="$(dirname "$input_file")"
  filename="$(basename "$input_file")"
  basename="${filename%.*}"
  extension="${filename##*.}"
  output="${dirname}/${basename}-processed.${extension}"
  ```

- Guard against overwriting an existing output before doing any work:

  ```bash
  if [[ -f "$output" ]]; then
      echo "error: output file already exists: $output" >&2
      exit 1
  fi
  ```

- Build a command line with conditional flags as an array, then expand it — never string-
  concatenate a command:

  ```bash
  args=(-hide_banner -i "$input")
  (( dry_run == 0 )) && args+=(-y "$output")
  ffmpeg "${args[@]}"
  ```

- On failure, delete partial output rather than leaving a broken file behind:

  ```bash
  if ! ffmpeg -i "$input" -y "$output"; then
      echo "error: failed to process: $input" >&2
      [[ -f "$output" ]] && rm -f "$output"
      exit 1
  fi
  ```

**Portability — Bash-Only Constructs to Avoid:**

Scripts should behave identically under bash and zsh. The constructs below are common but
bash-only; use the portable replacement instead.

| Avoid | Why it breaks under zsh | Use instead |
| --- | --- | --- |
| `shopt -s nullglob` | zsh has no `shopt` | test existence inside the loop: `[[ -e "$f" ]] \|\| continue` |
| `BASH_REMATCH` after `[[ =~ ]]` | zsh populates `$match`, not `BASH_REMATCH` | a `case` pattern match, or an external `sed`/`awk` |
| `"${arr[0]}"`, `"${arr[1]}"` | bash arrays are 0-indexed, zsh arrays are 1-indexed | avoid numeric indices — use `"$@"`, `shift`, or `set --` |
| `${BASH_SOURCE[0]}` | bash-only variable | `$0` |
| relying on unquoted `$var` to word-split | zsh does not word-split unquoted expansions by default | build an array and expand `"${arr[@]}"` |
| a bare `for f in *.mp4` with no existence check | bash leaves an unmatched glob as a literal string; zsh errors under `nomatch` | combine the glob with the existence test above |

Constructs that are already safe in both shells and need no special handling: `[[ ]]`,
`(( ))`, `local`, `&>` redirection, `<<<` here-strings, `<(...)` process substitution, and
`${#arr[@]}`.
