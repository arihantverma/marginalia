---
title: "Neovim Keymaps, Dotfiles Management with Git & GNU Stow"
date: 2026-01-30
tags: ["neovim", "lua", "dotfiles", "stow", "git", "zsh", "symlinks", "vim"]
---

## Context

Setting up a custom ergonomic Neovim configuration with remapped motion keys, and learning how to manage dotfiles with Git and GNU Stow for backup and portability.

## Learnings

### Neovim Keymap Remapping with Lua

The modern way to remap keys in Neovim is using the `vim.keymap.set()` Lua API.

```lua
vim.keymap.set(mode, lhs, rhs, opts)
```

| Argument | Meaning |
|----------|---------|
| `mode` | Which Vim mode(s) the mapping applies to |
| `lhs` | The key you press (left-hand side) |
| `rhs` | What it executes (right-hand side) |
| `opts` | Options table |

**Example - custom ergonomic motion keys:**

```lua
-- Motion remaps: j=down(default), f=up, k=right, d=left
vim.keymap.set({'n', 'v', 'o'}, 'f', 'k', { desc = 'Move up' })
vim.keymap.set({'n', 'v', 'o'}, 'k', 'l', { desc = 'Move right' })
vim.keymap.set({'n', 'v', 'o'}, 'd', 'h', { desc = 'Move left' })

-- Since 'd' is now left movement, remap delete to 's'
vim.keymap.set({'n', 'v'}, 's', 'd', { desc = 'Delete' })
vim.keymap.set('n', 'ss', 'dd', { desc = 'Delete line' })
vim.keymap.set('n', 'S', 'D', { desc = 'Delete to end of line' })

-- Since 'f' is now up movement, remap find-char to 't'
vim.keymap.set({'n', 'v', 'o'}, 't', 'f', { desc = 'Find character forward' })
```

### Vim Modes Explained

Vim has distinct modes, and mappings are mode-specific:

| Mode | Name | When it's active |
|------|------|------------------|
| `n` | Normal | Default mode, navigating and commanding |
| `v` | Visual | When you've selected text with `v`, `V`, or `Ctrl-v` |
| `o` | Operator-pending | After pressing an operator (`d`, `y`, `c`) waiting for a motion |
| `i` | Insert | When typing text |
| `x` | Visual-only | Like `v` but excludes select mode |
| `s` | Select | Rarely used, like visual but typing replaces selection |

**Why `o` mode matters:** Without it, combinations like `sf` (delete up) wouldn't work. After pressing `s` (delete operator), Vim enters operator-pending mode waiting for a motion. The `o` mode mapping tells Vim that `f` means "up" in that context.

### The `noremap` Option

`{ noremap = true }` means "non-recursive mapping" - it maps to the *original* meaning of the key, not any remapped version.

```lua
-- Without noremap (recursive):
-- If you map a → b, then b → c
-- Pressing 'a' would eventually execute 'c'

-- With noremap (non-recursive):
-- a → original meaning of b, ignoring any remaps on b
```

**Note:** `noremap = true` is actually the default in `vim.keymap.set()`, so it can be omitted. The older `vim.api.nvim_set_keymap()` still works but is more verbose.

### Scalable Neovim Config Structure

For a config that scales as you add more customization, use a namespaced structure:

```
~/.config/nvim/
├── init.lua              # Entry point
└── lua/
    └── user/             # Namespace to avoid plugin collisions
        ├── keymaps.lua   # Key mappings
        └── options.lua   # Editor options (vim.opt)
```

**Why namespace with `user/`?** When you `require('keymaps')`, Neovim searches all `lua/` directories - yours AND plugins. Using `require('user.keymaps')` prevents naming collisions.

**init.lua entry point:**

```lua
require('user.options')
require('user.keymaps')
```

### Dotfiles Management Approaches

Common ways to manage dotfiles with Git:

1. **Symlinks (manual)** - Keep files in git repo, manually symlink to expected locations
2. **GNU Stow** - Automates symlink management based on directory structure
3. **Bare git repo** - Use git with custom work tree pointing to $HOME
4. **Dotfiles managers** - Tools like chezmoi, yadm, dotbot

### GNU Stow - Symlink Farm Manager

Stow is a utility that creates and manages symlinks based on directory structure. Each subdirectory is a "package" that mirrors your home directory structure.

**Mental model:** Think of each folder in your dotfiles as a "transparency sheet." Stow overlays it onto your home directory, creating symlinks wherever there are files.

**Directory structure for stow:**

```
~/Code/po/dotfiles/
├── nvim/
│   └── .config/
│       └── nvim/
│           ├── init.lua
│           └── lua/user/...
└── zsh/
    └── .zshrc
```

The structure inside each package mirrors where files should land relative to `~`.

**Stow command:**

```bash
cd ~/Code/po/dotfiles
stow -t ~ nvim    # Creates ~/.config/nvim → dotfiles/nvim/.config/nvim
stow -t ~ zsh     # Creates ~/.zshrc → dotfiles/zsh/.zshrc
```

| Flag | Meaning |
|------|---------|
| `-t ~` | Target directory is home |
| `-D <package>` | Unstow (remove symlinks) |
| `-R <package>` | Restow (unstow then stow again) |
| `-nv` | Dry-run with verbose output |

**Why Stow is useful:**
- Convention over configuration - directory structure *is* the configuration
- Reversible - `stow -D nvim` removes all symlinks cleanly
- Conflict detection - warns if a real file already exists
- No scripting needed - unlike manual symlinks

### Symlinks Explained

A symlink (symbolic link) is a pointer, not a copy. Both paths access the same underlying file.

```
~/.zshrc → ~/Code/po/dotfiles/zsh/.zshrc
```

- Edit `~/.zshrc` → changes the dotfiles version
- Edit the dotfiles version → changes `~/.zshrc`
- Same file, two paths to access it

### Finding All Stow Symlinks

Use `find` to locate all symlinks pointing to your dotfiles:

```bash
find ~ -maxdepth 3 -type l -lname "*dotfiles*" -exec ls -la {} \;
```

**Breaking down the command:**

| Part | Meaning |
|------|---------|
| `find` | The search command |
| `~` | Start searching from home directory |
| `-maxdepth 3` | Only go 3 levels deep |
| `-type l` | Only find symbolic links (`l` = link) |
| `-lname "*dotfiles*"` | Only links whose target path contains "dotfiles" |
| `-exec ls -la {} \;` | For each result, run `ls -la` on it |

**The `-exec` part:**

| Part | Meaning |
|------|---------|
| `-exec` | Execute a command on each found item |
| `ls -la` | The command to run |
| `{}` | Placeholder for each found file |
| `\;` | End of the `-exec` command (escaped semicolon) |

**Useful alias for .zshrc:**

```bash
alias stowlist='find ~ -maxdepth 3 -type l -lname "*dotfiles*" -exec ls -la {} \;'
```

### Security: What's Safe to Publish in Dotfiles

When making dotfiles public on GitHub, check for:

**NOT safe to publish:**
- API keys or tokens
- Passwords or secrets
- Private SSH/GPG key references
- Database credentials
- Environment variables with secrets

**Safe to publish:**
- Shell configuration (themes, plugins, aliases)
- Editor settings
- Tool configurations
- PATH modifications with standard paths
- Username in paths (e.g., `/Users/yourname/...`) - this is standard and usually matches your GitHub username anyway

## References

- [GNU Stow Manual](https://www.gnu.org/software/stow/manual/stow.html)
- [Neovim Lua Guide](https://neovim.io/doc/user/lua-guide.html)
- [vim.keymap.set documentation](https://neovim.io/doc/user/lua.html#vim.keymap.set())
