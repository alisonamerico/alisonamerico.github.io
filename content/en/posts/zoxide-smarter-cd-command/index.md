---
title: "Zoxide - The Smarter cd Command Your Terminal Deserves"
date: 2026-03-26
draft: false
tags: ["zoxide", "terminal", "cli", "productivity", "shell", "fzf"]
categories: ["terminal"]
description: "Discover zoxide, a blazing fast directory jumper written in Rust that learns your most visited directories and lets you navigate with a few keystrokes."
cover: zoxide-command.png
---

> **Level:** beginner  
> **Prerequisites:** basic knowledge of command line and terminal

---

## The Problem with Traditional cd

If you spend any significant time in the terminal, you know the pain of navigating deep directory structures. Even with the use of **AI** today, at some point **YOU** will need to access your directories quickly and intelligently.

```bash
cd ~/projects/web-app/frontend/src/components
cd ../../../../  # Going back up is just as fun
cd /home/user/workspace/my-super-long-project-name
```

It's repetitive, slow, and prone to typos. You probably find yourself using tab completion constantly, or even worse, just memorizing paths that you type too often.

**What if there was a better way?**

---

## What is Zoxide?

**Zoxide** is a smarter `cd` command, inspired by tools like [z](https://github.com/rupa/z) and [autojump](https://github.com/joelmc/autojump). It learns which directories you use most frequently and lets you "jump" to them in just a few keystrokes.

### Key Features

- **Blazing fast**: Written in Rust, zoxide is incredibly fast , even with thousands of entries in its database.
- **Cross-platform**: Works on Linux, macOS, Windows, and BSD.
- **Shell-agnostic**: Supports Bash, Zsh, Fish, PowerShell, Nushell, and more.
- **Fuzzy matching**: Find directories even with partial matches.
- **Interactive mode**: Built-in integration with fzf for visual selection.
- **Frecency-based**: Combines frequency and recency to rank directories intelligently.

---

## Installation

### Linux (Ubuntu/Debian)

```bash
# Official script (recommended)
curl -sSfL https://raw.githubusercontent.com/ajeetdsouza/zoxide/main/install.sh | sh

# Or via apt (Ubuntu 21.04+)
sudo apt install zoxide
```

### macOS

```bash
# Via Homebrew (recommended)
brew install zoxide
```

### Windows

```bash
# Via winget
winget install ajeetdsouza.zoxide

# Or via scoop
scoop install zoxide
```

---

## Shell Configuration

After installing, you need to initialize zoxide in your shell. Add the appropriate line to your shell config file:

### Zsh (most common)

Add to `~/.zshrc`:

```bash
eval "$(zoxide init zsh)"
```

### Bash

Add to `~/.bashrc`:

```bash
eval "$(zoxide init bash)"
```

### Fish

Add to `~/.config/fish/config.fish`:

```bash
zoxide init fish | source
```

> **Important**: Make sure to restart your terminal or source your config file after adding this line.

---

## Basic Usage

Once initialized, zoxide replaces your `cd` with superpowers. Here are the essential commands:

### Jump to a Directory

```bash
z blog            # Jump to highest-ranked directory matching "blog"
z work project    # Jump to directory matching both "work" and "project"
z projects/blog   # Jump to subdirectory matching "projects" and "blog"
```

### Navigate Like Regular cd

```bash
z ~/documents     # Works like regular cd
z projects/       # cd into relative path
z ..              # cd one level up
z -               # cd into previous directory (like cd -)
```

### Interactive Mode (with fzf)

```bash
zi                 # Open fzf to select from all known directories
zi blog            # Open fzf filtered to directories containing "blog"
```

If you don't have fzf installed, zoxide still works , but you should install fzf for the full interactive experience:

```bash
# macOS
brew install fzf

# Linux
sudo apt install fzf   # or: sudo dnf install fzf

# Windows
scoop install fzf
```

---

## Important: How Zoxide Learns Directories

A common misconception is that zoxide automatically discovers all directories on your system. **That's not how it works.** Zoxide learns **only** from directories you visit using zoxide itself.

### Why "no match found" happens

Imagine you create a new project:

```bash
mkdir ~/projects/new-project
cd ~/projects/new-project
z new-project
```

**Result:**
```
zoxide: no match found
```

The directory doesn't exist in zoxide's database because you used `cd` to enter it, not `z`. Creating a folder with `mkdir` doesn't add it to zoxide either.

### How to add directories

**Option 1: Use `z` directly (recommended)**

```bash
z ~/projects/new-project   # Enters AND adds to database
```

**Option 2: Use `zoxide add`**

```bash
zoxide add ~/projects/new-project   # Adds without entering
```

**Option 3: Use interactive mode**

```bash
zi   # Opens fzf - you can type a new path and it will be added
```

### Once added, it sticks

After the first visit with `z`, you can jump there anytime with just:

```bash
z new-project   # Now it works!
```

Zoxide tracks how frequently and recently you visit each directory, so your most-used directories rise to the top.

---

## Query Commands

Want to see what zoxide knows without jumping? Use query commands:

```bash
zoxide query foo          # Show path that 'z foo' would jump to
zoxide query -l           # List all tracked directories with scores
zoxide query --score      # Show directories with their frecency scores
zoxide query -i foo       # Interactive search
```

---

## Importing from Other Tools

If you're migrating from autojump, z.lua, or zsh-z, you can import your existing data:

```bash
# From autojump
zoxide import --from=autojump "/path/to/autojump/db"

# From z (bash/zsh)
zoxide import --from=z "path/to/z/db"
```

---

## Advanced Configuration

### Customizing the Command Prefix

By default, zoxide uses `z` and `zi`. You can change this:

```bash
# Use 'j' instead of 'z'
eval "$(zoxide init zsh --cmd j)"
```

### Environment Variables

```bash
# Show the directory before jumping (like cd)
export _ZO_ECHO=1

# Exclude certain directories
export _ZO_EXCLUDE_DIRS="$HOME:$HOME/private/*"

# Customize fzf options
export _ZO_FZF_OPTS="--height 40% --reverse"
```

---

## Troubleshooting

### "zoxide: command not found"

1. Check if zoxide is installed: `which zoxide`
2. Make sure ~/.local/bin is in your PATH:
   ```bash
   echo $PATH | grep -q ~/.local/bin || export PATH="$HOME/.local/bin:$PATH"
   ```
3. Verify your shell config has the init line

### "no match found"

This is the most common issue and usually happens because:

1. The directory was never visited with zoxide (`cd` doesn't count!)
2. The directory was created with `mkdir` but never entered with `z`
3. The directory name is too generic and doesn't match any entry

**Solutions:**

```bash
# Visit with z (enters AND adds to database)
z /full/path/to/directory

# Or add manually without entering
zoxide add /full/path/to/directory

# Or use interactive mode - you can type new paths there
zi
```

---

## Why Zoxide Over Alternatives?

| Tool | Type | Pros | Cons |
|------|------|------|------|
| **Zoxide** | Rust binary | Fast, modern, well-maintained | Requires installation |
| **[z](https://github.com/rupa/z)** (original) | Shell script | Simple, no dependencies | Slower, less features |
| **[autojump](https://github.com/joelmc/autojump)** | Python | Cross-platform | Slower, less intuitive |
| **[fasd](https://github.com/clvv/fasd)** | Shell script | Quick access to files too | Complex, steeper learning curve |

Zoxide is generally considered the modern standard for directory jumping due to its Rust foundation, speed, and active development.

---

## My Experience

After using zoxide for a while, I can navigate to projects I use daily with just 2-3 keystrokes. Instead of typing `cd ~/projects/blog/alisonamerico.github.io` every time, I just type `z blog`. It learns from my behavior and ranks directories by how frequently and recently I visit them.

The combination of `z` for quick jumps and `zi` for visual selection has completely changed how I navigate the terminal. It's one of those tools that feels like it should have always existed.

---

## Conclusion

Learning to use zoxide is one of the highest ROI (Return on Investment) upgrades you can make to your terminal workflow. It takes minutes to install, requires minimal configuration, and immediately starts making your life easier.

Give it a try , your fingers will thank you.

**Resources:**
- [Zoxide GitHub](https://github.com/ajeetdsouza/zoxide)
- [Official Documentation](https://zoxide.dev)