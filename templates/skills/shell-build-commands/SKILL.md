---
name: shell-build-commands
description: shellcheck, shfmt, and bats commands for linting, formatting, and testing shell scripts. Load when linting, formatting, or testing shell scripts.
license: MIT
metadata:
  author: Heiko Panjas
  version: "1.0"
---

# Shell Build Commands

Read this skill when you need to lint, format, or test shell scripts in this project.

---

## Build Commands

### Setup

```bash
# Install shellcheck, shfmt, and bats (macOS via Homebrew)
brew install shellcheck shfmt bats-core

# Debian/Ubuntu
sudo apt-get install shellcheck bats
# shfmt has no apt package on most distros; install via go or a release binary
```

### Linting

```bash
# Lint a single script, targeting bash explicitly
shellcheck -s bash script.sh

# Lint every tracked script
shellcheck -s bash *.sh

# Include optional checks (e.g. quote-all-variables) on top of the defaults
shellcheck -s bash -o all script.sh
```

### Formatting

This project ships no `.editorconfig` for shell, so pass settings explicitly as flags
rather than relying on shfmt's config discovery:

```bash
# Check formatting without writing (2-space indent, indent switch cases,
# add a space after redirect operators, keep binary operators at line end)
shfmt -i 2 -ci -sr -d script.sh

# Rewrite the file in place
shfmt -i 2 -ci -sr -w script.sh

# Format every tracked script in place
shfmt -i 2 -ci -sr -w *.sh
```

shfmt can also read these settings from an `.editorconfig`, including non-standard
`[[bash]]`/`[[zsh]]` sections that match by shebang rather than file extension — useful if
this project later adds one instead of passing flags.

### Testing

```bash
# Run a bats test file
bats script.bats

# Run every test file in a directory
bats test/
```

### Running

```bash
# Execute directly (uses the shebang)
./script.sh

# Execute under bash explicitly, ignoring the shebang
bash script.sh

# Execute under zsh explicitly, to verify portability
zsh script.sh

# Trace execution for debugging
bash -x script.sh
```

**Important**: Run `shellcheck -s bash` before every commit that touches a shell script, and
prefer fixing the flagged construct over suppressing it with a `# shellcheck disable=`
comment — most shellcheck findings (unquoted expansions, unchecked command substitution) are
real portability or correctness issues, not false positives.
