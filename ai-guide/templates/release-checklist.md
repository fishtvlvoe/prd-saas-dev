# Pre-Release Checklist

## Purpose

Ensure every release passes all required verification steps before distribution. Walk through every item; all must pass before building the release.

## When to Use

Before executing the release build command. Every item checked = safe to build.

---

## 1. Version Number (Required -- 3 Locations)

- [ ] Main config file (e.g., `tauri.conf.json`, `electron-builder.json`): `version` updated
- [ ] Backend manifest (e.g., `Cargo.toml`, `package.json` for Electron main): `version` updated
- [ ] Frontend `package.json`: `version` updated

All three locations must have the same version number.

**Version rules**:
- Bug fix -> patch (0.1.0 -> 0.1.1)
- New feature -> minor (0.1.0 -> 0.2.0)
- Breaking change -> major (0.1.0 -> 1.0.0)

---

## 2. Application Icon (Required)

- [ ] Source icon is the latest logo (1024x1024 px)
- [ ] Icon generation command has been re-run (e.g., `pnpm tauri icon src/app-icon.png`)
- [ ] Generated icon files in the icons directory have recent timestamps

**Do not** manually replace individual icon files in the icons directory. Always regenerate from the single source.

---

## 3. Tests Pass

- [ ] Frontend unit tests pass: `pnpm test` / `npm test`
- [ ] Backend tests pass: `cargo test` / backend test command
- [ ] No TypeScript compilation errors: `pnpm tsc --noEmit`

---

## 4. Production Environment Logic

- [ ] Mock backend is disabled in production build (verify `NODE_ENV=production` skips mock)
- [ ] Authentication/license verification works in native environment
- [ ] Core user flow completes end-to-end (e.g., login/activation -> main screen -> core action)

---

## 5. SDD Synchronization

- [ ] All active changes are archived or parked (no incomplete changes left open)
- [ ] `spectra list` returns an empty list (or only intentionally parked changes)

---

## 6. Git Status

- [ ] `git status` shows a clean working tree (no uncommitted changes)
- [ ] All changes are committed and pushed to the correct branch
- [ ] PR has been merged to main (if applicable)

---

## 7. Build Execution

```bash
# Example commands (adjust for your framework)
pnpm tauri build      # Tauri
npm run build         # Electron
```

**Output locations** (verify these exist after building):
- macOS: `.app` and `.dmg` (arm64 and x64 if applicable)
- Windows: `.msi` or `.exe`

---

## 8. Post-Build Verification

- [ ] Open the built application, confirm version number displays correctly
- [ ] Walk through the core user flow (activation -> main screen -> core feature)
- [ ] Verify all primary features work (list, create, edit, export, etc.)
- [ ] Confirm application icon is correct

---

## 9. Release

- [ ] Create GitHub Release with tag format: `vX.Y.Z`
- [ ] Upload all platform artifacts (.dmg arm64, .dmg x64, .msi, etc.)
- [ ] Write release notes describing changes in this version
- [ ] Verify artifact names include architecture identifier (avoid same-name overwrite)

**Artifact naming convention**:
- `AppName-X.Y.Z-aarch64.dmg` (macOS arm64)
- `AppName-X.Y.Z-x86_64.dmg` (macOS x64)
- `AppName-X.Y.Z-x86_64.msi` (Windows)

---

## Common Build Troubleshooting

**Build failure: backend compilation error**
```bash
cd src-tauri && cargo build --release    # Tauri
# Read detailed error messages
```

**Build failure: frontend build error**
```bash
pnpm build    # or: npm run build
# Read Next.js / Vite build error messages
```

**App opens but shows blank screen**
1. Confirm `NODE_ENV=production` (not development)
2. Confirm mock backend is not active in production
3. Check app console log (right-click -> Inspect Element -> Console)

**macOS Gatekeeper blocks the app**
- First time: right-click -> Open -> Allow
- Long-term solution: code signing (Apple Developer account)
