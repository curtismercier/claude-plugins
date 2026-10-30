# Disk Explorer

Explore what's taking up space on the Mac and give the user actionable recommendations.

Use this skill when the user asks things like: "what's taking up space?", "where is my disk space going?", "show me large files", "what can I delete?", "why is my Mac full?", "explore my disk", "מה תופס מקום?", "מה אפשר למחוק?", "qu'est-ce qui prend de la place?", "¿qué ocupa espacio?", or any question about understanding disk usage without necessarily cleaning yet.

---

## How to run this skill

Run the commands below, analyze the output, and respond in the user's language with clear, prioritized recommendations.

> **Environment note:** This skill works in both Claude Code (native Bash tool) and Cowork (Desktop Commander). Use whichever shell execution method is available in your environment to run the bash commands below.

---

## Step 1 — Overall disk usage

```bash
df -h /System/Volumes/Data | tail -1
```

Report total disk size, how much is used, and how much is free.

> On APFS (macOS Catalina+), `df -h /` reports the small read-only **system** volume, so its "Used" figure (often ~10 GB) is misleading. User data lives on the Data volume above; "Avail" is shared by both. Also check for local Time Machine snapshots, which hold space that no folder shows: `tmutil listlocalsnapshots /`.

---

## Step 2 — Top-level home folder sizes

```bash
du -sh ~/* 2>/dev/null | sort -rh | head -15
```

Identify which top-level folders in the home directory are largest.

`~/*` skips hidden folders, which on developer machines often hold tens of GB (`~/.cache`, toolchains like `~/.pyenv`, `~/.nvm`, `~/.rustup`, model caches). Size them too:

```bash
du -sh ~/.[!.]* 2>/dev/null | sort -rh | head -15
```

---

## Step 3 — Known large locations

Check these locations — they are commonly large and often overlooked:

```bash
du -sh \
  ~/Desktop \
  ~/Downloads \
  ~/Documents \
  ~/Movies \
  ~/Music \
  ~/Library/Developer \
  ~/Library/Application\ Support \
  ~/Library/Caches \
  2>/dev/null | sort -rh
```

---

## Step 4 — Developer-specific large locations (only if ~/Library/Developer exists)

```bash
du -sh \
  ~/Library/Developer/Xcode/DerivedData \
  ~/Library/Developer/CoreSimulator/Devices \
  ~/Library/Developer/Xcode/iOS\ DeviceSupport \
  2>/dev/null | sort -rh
```

Simulator **runtimes** live outside the home folder, in `/Library/Developer/CoreSimulator`, and are often the largest developer item. Inspect them with `simctl` instead of `du`: `du` counts mounted runtime images (`Volumes/`, `Cryptex/Images/bundle/*`) as if they were on disk, so it can report several times the real size.

```bash
xcrun simctl list runtimes            # every runtime Xcode can see
xcrun simctl runtime list -v          # disk-image runtimes, size and last-used date
du -sh /Library/Developer/CoreSimulator/Caches/dyld/*/* 2>/dev/null   # per-runtime shared caches
```

- Keep the runtime the user's current Xcode, and any simulator-driving tool, actually uses. Only recommend removing **older** runtimes.
- A dyld cache folder named after a runtime that is no longer listed is an orphan and can go.
- Older "cryptex bundle" runtimes don't appear in `simctl runtime list` and can't be removed with `simctl runtime delete`. Point the user to Xcode → Settings → Components. If one isn't listed there, removing it needs admin rights (`sudo hdiutil detach` + `sudo rm`), which the user should run themselves.

---

## Step 5 — Find large files (>500 MB) anywhere in home

```bash
find ~ -maxdepth 6 -size +500M -not -path "*/.*" 2>/dev/null | while read f; do du -sh "$f" 2>/dev/null; done | sort -rh | head -20
```

---

## Step 6 — Look for common storage wasters

```bash
# ZIP files next to folders with the same name (often redundant)
find ~/Desktop ~/Downloads ~/Documents -maxdepth 2 -name "*.zip" 2>/dev/null

# Old iOS backups
du -sh ~/Library/Application\ Support/MobileSync/Backup/ 2>/dev/null

# Trash
du -sh ~/.Trash/ 2>/dev/null
```

### Throwaway browser profiles

Browser automation (Puppeteer, Playwright, CDP/DevTools MCP servers, screenshot scripts) creates a new Chromium user-data folder per run, roughly 100–200 MB each, and they pile up. Find them by their `Local State` file, then tell real profiles from empty shells by what they store:

```bash
find ~ -maxdepth 6 -name "Local State" -type f -not -path "*/Library/CloudStorage/*" 2>/dev/null \
  | while read -r f; do d=$(dirname "$f"); p="$d/Default"
      ck=$(sqlite3 "file:$p/Cookies?immutable=1" "select count(*) from cookies" 2>/dev/null || echo 0)
      lg=$(sqlite3 "file:$p/Login Data?immutable=1" "select count(*) from logins" 2>/dev/null || echo 0)
      printf "%s\tcookies=%s\tlogins=%s\t%s\n" "$(du -sh "$d" | cut -f1)" "$ck" "$lg" "$d"; done | sort -rh
ps -axo command | grep -oE -- "--user-data-dir=[^ ]+" | sort -u   # profiles in use right now
```

- **Quick win:** folders with 0 cookies and 0 logins that aren't in use. These are empty shells, typically with names like `*-chrome-<port>`.
- **Review:** anything with saved logins or many cookies. These may be a signed-in automation profile the user depends on.
- **Never:** the user's real browser profiles under `~/Library/Application Support/<Browser>/`.

Puppeteer (`~/.cache/puppeteer`) and Playwright (`~/Library/Caches/ms-playwright`) also keep every browser build they've ever downloaded. Recommend keeping the newest build of each and removing the rest.

### Git worktrees and dev build output

Search for build output in projects: `node_modules`, Rust `target`, `.next`, and especially `.worktrees` folders, where each git worktree carries its own full set of dependencies.

```bash
find ~ -maxdepth 5 -type d \( -name node_modules -o -name target -o -name .next -o -name .worktrees \) -prune 2>/dev/null \
  | xargs -I{} du -sh {} 2>/dev/null | sort -rh | head -20
```

Only call a worktree **safe** to remove after checking all of the following:

- its HEAD commit is on a remote branch: `git branch -r --contains HEAD`
- it has no uncommitted or untracked changes: `git status --porcelain`
- it isn't locked, and no process has files open in it: `lsof +D <path>`
- it has no **ignored** files that exist nowhere else: `git ls-files --others --ignored --exclude-standard --directory`. Local secret files (`.env`, `.dev.vars`) are gitignored, so they never appear in `git status`, and they may only exist in the worktree. Back them up before removing it.

Remove worktrees with `git worktree remove <path>`, without `--force`, so git refuses anything that isn't clean. Folders inside a `.worktrees` directory that `git worktree list` doesn't show are orphans; inspect them before deleting.

---

## Report

Present findings clearly in the user's language, organized by priority:

### 1. Quick wins (safe to clean now)
Items that are clearly redundant or cache-only — e.g. ZIP files next to extracted folders, cache folders, old iOS backups.

### 2. Review recommended (user decides)
Large folders that require user judgment — Downloads, old projects, Movies, iOS device support files.

### 3. Developer bloat (if applicable)
Xcode DerivedData, simulators, device support — large but safe to delete if not actively needed.

### 4. Already optimized
If the disk looks clean, say so clearly. Don't invent problems.

---

## Offer next steps

After the report, offer:
- "I can run a full cleanup now" → triggers the mac-cleanup skill
- "I can help you move large files to cloud storage" → guide the user to iCloud Drive or another service
- "I can set up a weekly automated cleanup" → suggest scheduling

Do not delete anything in this skill. Only analyze and recommend.
