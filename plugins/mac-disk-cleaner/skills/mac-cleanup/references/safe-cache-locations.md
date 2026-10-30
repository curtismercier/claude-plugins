# Safe Mac Cache Locations

Reference guide for what is safe to delete and what must never be touched.

## Safe to delete (auto-regenerated)

| Location | What it is | Typical size |
|----------|-----------|--------------|
| `~/Library/Caches/Google/` | Chrome network/disk cache | 1–5 GB |
| `~/Library/Caches/pip/` | Python package download cache | 50–500 MB |
| `~/Library/Caches/node-gyp/` | Node native addon headers | 50–200 MB |
| `~/Library/Caches/org.swift.swiftpm/` | Swift Package Manager cache | 200 MB–1 GB |
| `~/.gradle/caches/` | Gradle build cache | 200 MB–2 GB |
| `~/Library/Caches/Homebrew/` | Homebrew download cache | 50–500 MB |
| `~/Library/Caches/yarn/` | Yarn package cache | 100 MB–1 GB |
| `~/Library/Caches/com.todesktop.*` | Desktop app update cache | 100 MB–1 GB |
| `~/Library/Caches/com.*.ShipIt/` | Sparkle auto-update cache | 50–500 MB |
| pnpm store | Run `pnpm store prune`, which removes only packages no project references | 1–10 GB |
| `~/.cache/uv/` | uv Python package cache. Run `uv cache clean` | 0.5–5 GB |
| `~/.cache/puppeteer/`, `~/Library/Caches/ms-playwright/` | Downloaded headless browser builds. Keep the newest build of each tool; older ones are unused | 0.5–3 GB per build |

## Safe with caution

| Location | Notes |
|----------|-------|
| `~/Library/Caches/com.apple.Safari/` | Safari cache — safe, but Safari must be closed first |
| `~/Library/Caches/com.spotify.client/` | Spotify cache — will re-download on next play |
| Xcode derived data: `~/Library/Developer/Xcode/DerivedData/` | Safe if not actively building — regenerated on next build |
| iOS Simulators: `~/Library/Developer/CoreSimulator/Devices/` | Can be large (10–50 GB). Remove unused simulators via Xcode > Preferences > Platforms |
| Simulator runtimes: `/Library/Developer/CoreSimulator/` | Keep the runtime the current Xcode uses; remove only older ones. Use `xcrun simctl list runtimes`, not `du`, because `du` counts mounted runtime images as if they were on disk |
| Automation browser profiles (Chromium folders containing a `Local State` file, outside `~/Library/Application Support`) | Delete only if they have no cookies and no saved logins and aren't in use (`--user-data-dir` in `ps`). Profiles with logins are a decision for the user |

## NEVER delete

| Location | Why |
|----------|-----|
| `~/Library/Application Support/Google/Chrome/` | Passwords, bookmarks, history, extensions, login sessions |
| `~/Library/Application Support/` (general) | App data, preferences, local databases |
| `~/Library/Keychains/` | Passwords and certificates |
| `~/Library/Mail/` | Email data |
| `~/Library/Mobile Documents/` | iCloud sync folder |
| `/System/`, `/Library/` (root) | macOS system files |
| `~/.ssh/` | SSH keys |
| `~/.gnupg/` | GPG keys |

## Large items worth flagging (not deleting)

- **Desktop folder > 3 GB** — likely contains files that should be moved to cloud storage or an external drive
- **Downloads folder > 5 GB** — old downloads often forgotten; user should review manually
- **Documents folder > 20 GB** — flag if unusually large; suggest cloud backup
- **ZIP files next to their extracted folder** — the ZIP is usually redundant after extraction
