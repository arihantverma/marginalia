---
title: "Fixing GNU Stow Symlinks After Renaming the Dotfiles Directory"
date: 2026-01-30
tags: ["stow", "symlinks", "dotfiles", "homebrew", "zsh", "shell", "macos"]
---

## Context

I renamed my dotfiles repository folder from `dotfiles` to `.dotfiles`. This broke all the symlinks that GNU Stow had created, because they still pointed to the old `dotfiles` path. The tricky part: my `.zshrc` was itself a symlink managed by stow, so when it broke, my shell lost its configuration—including the PATH setup for homebrew, which meant `stow` wasn't available to fix the problem.

## Learnings

### The Symlink Breakage Chain Reaction

When you rename a directory that contains stowed dotfiles, all symlinks pointing into that directory become invalid. This can create a chicken-and-egg problem:

1. `.zshrc` is a symlink → now broken
2. `.zshrc` normally sets up homebrew in PATH (via `eval "$(/opt/homebrew/bin/brew shellenv)"`)
3. Without `.zshrc` loading, homebrew isn't in PATH
4. `stow` is installed via homebrew, so it's not found
5. You can't run `stow` to fix the symlinks

### The Solution: Bypass PATH Using Absolute Paths

The fix is to source homebrew's environment directly using the absolute path to the `brew` binary:

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

This command has two parts:

#### Part 1: `/opt/homebrew/bin/brew shellenv`

This runs the `brew` binary directly using its **absolute path**, completely bypassing the PATH lookup. The `shellenv` subcommand tells brew to output the shell commands needed to set up its environment. Running it produces something like:

```bash
export HOMEBREW_PREFIX="/opt/homebrew"
export HOMEBREW_CELLAR="/opt/homebrew/Cellar"
export HOMEBREW_REPOSITORY="/opt/homebrew"
export PATH="/opt/homebrew/bin:/opt/homebrew/sbin${PATH+:$PATH}"
export MANPATH="/opt/homebrew/share/man${MANPATH+:$MANPATH}:"
export INFOPATH="/opt/homebrew/share/info:${INFOPATH:-}"
```

#### Part 2: `eval "$(...)"`

- `$(...)` is command substitution—it captures the output of the command inside
- `eval` takes a string and executes it as shell commands
- Combined: the export statements from `brew shellenv` are captured and executed

After running this, `/opt/homebrew/bin` is in your PATH, so `stow` becomes available.

### The Complete Fix

```bash
# 1. Remove the old broken symlinks
rm ~/.config/nvim
rm ~/.zshrc
rm ~/Library/Application\ Support/com.mitchellh.ghostty/config

# 2. Source homebrew to get stow in PATH
eval "$(/opt/homebrew/bin/brew shellenv)"

# 3. Restow from the new directory location
cd ~/path/to/.dotfiles
stow -t ~ nvim
stow -t ~ zsh
stow -t ~ ghostty
```

### Key Insight: Absolute Paths as an Escape Hatch

When your shell environment is broken, you can always fall back to absolute paths. Every binary on your system has a location on disk—PATH is just a convenience for finding them. Common locations:

- Homebrew (Apple Silicon): `/opt/homebrew/bin/`
- Homebrew (Intel Mac): `/usr/local/bin/`
- System binaries: `/usr/bin/`, `/bin/`

You can find where a program lives (when PATH is working) with:

```bash
which stow        # /opt/homebrew/bin/stow
type -a stow      # shows all locations
```

Or search for it:

```bash
find /opt/homebrew -name stow -type f
```

## References

- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Homebrew Shell Environment Documentation](https://docs.brew.sh/Manpage#shellenv)
