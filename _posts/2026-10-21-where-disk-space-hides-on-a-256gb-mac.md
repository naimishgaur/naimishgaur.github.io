---
layout: post
title: "Where Disk Space Actually Hides on a 256 GB Mac"
date: 2026-10-21 00:00:00 +0530
categories: macos storage cli
---

When a 256 GB Apple Silicon Mac drops below 15 GB of free space, `swap` and filesystem caches run out of room, memory compression spikes, and the machine stalls — even with idle CPU.

## Problem

On a 245.1 GB APFS container (`/dev/disk3`), `df -h /System/Volumes/Data` reported 94% used with 14 GiB free. macOS Storage settings lumps most developer toolchains, package caches, and local model weights into an opaque **System Data** bar that cannot be drilled into.

A naive `du -sh ~/*` also misses the biggest culprits because:

1. APFS shares a single free-space pool across multiple volumes (`Data`, `System`, `Preboot`, `VM`, `Recovery`) plus local Time Machine snapshots.
2. Dotfiles (`~/.ollama`, `~/.bun`, `~/.cache`, `~/.npm`) are skipped by `*` globs.
3. Package managers keep multi-gigabyte installer `.pkg` payloads inside `/opt/homebrew/Caskroom` even after installing the target into `/usr/local` or `/Applications`.
4. Another local user account with `0700` subdirectories (`~/Library`, `~/Downloads`) reports `0B` to an unprivileged `du`, hiding tens of gigabytes in the difference between `diskutil apfs list` and the sum of readable paths.

## Step 1: Check the APFS Container, Not Just `df`

Before scanning files, inspect the APFS container ceiling, volume breakdown, VM swap, and local snapshots:

```bash
# Container-level breakdown across Data, System, Preboot, VM, and Recovery
diskutil apfs list

# Check whether Time Machine local snapshots are pinning deleted blocks
tmutil listlocalsnapshots /
diskutil apfs listSnapshots /System/Volumes/Data

# Check active swap files in /System/Volumes/VM
sysctl vm.swapusage
```

On this machine, the 245.1 GB container broke down as follows:

| APFS Volume / Slice | Consumed (GB) | Notes |
|---|---:|---|
| `Data` (`/System/Volumes/Data`) | 198.6 | User homes, `/Applications`, `/opt`, `/usr/local`, `/Library` |
| `System` (`Macintosh HD`) | 13.7 | Sealed read-only OS snapshot |
| `Preboot` | 10.9 | Boot objects and firmware |
| `VM` (`/System/Volumes/VM`) | 4.3 | Four 1 GB swapfiles (`3.36 GB` active) — never delete live swap |
| `Recovery` | 1.5 | RecoveryOS |
| **Unallocated (Free)** | **15.8** | Shared free pool across all volumes |

If `tmutil listlocalsnapshots /` lists snapshots after a large delete, APFS is still retaining those blocks. Thin them immediately:

```bash
# Request 20 GB of Urgency-4 snapshot thinning
tmutil thinlocalsnapshots / 20000000000 4
```

## Step 2: Rank Hidden Dotfiles and Caches in `$HOME`

Standard directory globs skip dotfiles. Scan explicit hidden directories alongside `~/Library/Caches` and `~/Library/Application Support`:

```bash
du -sh \
  ~/.ollama \
  ~/.cache \
  ~/.bun \
  ~/.npm \
  ~/.local \
  ~/Library/Caches \
  ~/Library/Application\ Support/* \
  2>/dev/null | sort -hr | head -n 20
```

What this surfaced on the machine, ranked by actual space reclaimed or reclaimable:

| Path | Size | Safe Action |
|---|---:|---|
| `~/.ollama/models` | 11.0 GB | `ollama rm <model>` or `rm -rf ~/.ollama` if unused |
| `/opt/homebrew/Caskroom/mactex` | 5.7 GB | Duplicate `.pkg` installer cached after `/usr/local/texlive` (`8.8 GB`) is already unpacked |
| `~/Library/Caches` + `~/.cache` + Trash | 3.0 GB | Safe cache clear (`brew cleanup`, `pip cache purge`, `~/.Trash`) |
| `~/.bun/install/cache` | 3.1 GB | `bun pm cache rm` |
| `/Applications/Ollama.app` | 644 MB | App bundle left behind after `brew uninstall ollama` |
| `~/.npm/_cacache` | 269 MB | `npm cache clean --force` |

## Step 3: Remove Dormant Local LLM Weights

Local model runners (`ollama`, Hugging Face `hub`, `llama.cpp`, LM Studio) store multi-gigabyte weights under dotfiles that never appear in Finder:

```bash
# List and remove unused Ollama models
ollama list
ollama rm gemma4:e2b-mlx gemma4:e2b-it-qat

# Check for Hugging Face, PyTorch, or GGUF weights across $HOME
du -sh ~/.cache/huggingface ~/.cache/torch ~/.lmstudio 2>/dev/null
find ~ -xdev \( -name "*.gguf" -o -name "*.safetensors" -o -name "*.mlx" \) -size +100M -ls 2>/dev/null
```

Removing two dormant Gemma models (`6.5 GB` + `4.3 GB`) immediately moved free space from **14.1 GB to 24.0 GB**. Fully removing Ollama afterward required cleaning both the Homebrew formula and the GUI app bundle:

```bash
brew uninstall ollama
rm -rf /Applications/Ollama.app \
       ~/.ollama \
       ~/Library/Application\ Support/Ollama \
       ~/Library/Caches/com.electron.ollama*
```

## Step 4: Audit Homebrew Caskroom and Package Caches

`brew cleanup --prune=all` clears `~/Library/Caches/Homebrew`, but Homebrew Cask also keeps the original `.pkg` or `.dmg` inside `/opt/homebrew/Caskroom/<cask>/<version>/` for casks installed via `.pkg` installers:

```bash
# Check Caskroom payloads that duplicate already-installed /usr/local or /Applications trees
du -sh /opt/homebrew/Caskroom/* 2>/dev/null | sort -hr | head -n 10

# Clear standard Homebrew, pip, Bun, and npm download caches
brew cleanup --prune=all -s
pip cache purge
bun pm cache rm
npm cache clean --force
```

On this system, `/opt/homebrew/Caskroom/mactex` held a **5.7 GB** `MacTeX.pkg` installer even though TeX Live (`8.8 GB`) was already extracted under `/usr/local/texlive`.

## Step 5: Account for Unreadable Slices (`0700` Directories and Open Unlinked Files)

When `diskutil apfs list` says `Data` consumes **198.6 GB** but summing every readable directory on `/System/Volumes/Data` only reaches **~130 GB**, the missing ~68 GB is not filesystem corruption. There are two places to check:

1. **Other local user accounts on the same Mac:** Each user's `~/Library`, `~/Downloads`, `~/Pictures`, and `~/Movies` are `drwx------` (`0700`). Running `du` without `sudo` silently counts those directories as `0` while printing permission errors to `stderr`:

   ```bash
   # Compare total Data volume usage with the sum of top-level directories using sudo
   sudo du -xhd 1 /System/Volumes/Data
   sudo du -xhd 1 /Users
   ```

2. **Deleted files still held open by a running process:** If you delete a multi-gigabyte log or cache file while a daemon still holds an open file descriptor (`NLINK == 0`), APFS cannot reclaim the blocks until the process closes the descriptor or exits:

   ```bash
   # List open files whose directory link count is 0 (deleted but still occupying blocks)
   lsof +L1
   ```

## Source

- [Homebrew Manpage — `brew cleanup`](https://docs.brew.sh/Manpage)
- [Ollama FAQ — Where are models stored?](https://docs.ollama.com/faq)
- [Apple Support — About Time Machine local snapshots](https://support.apple.com/en-us/102664)
