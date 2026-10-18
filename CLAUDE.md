# CLAUDE.md — Broker Book

> **Purpose:** Full context for anyone (human or Claude) picking up this project. Read top-to-bottom. Self-sufficient — does not depend on memory system or prior conversation.

## What this project is

**Broker Book** is a single-HTML Progressive Web App for diamond brokers in Surat / Mumbai to track their deals, parties (buyers + sellers), stones, memos, and overdue items. Built as a mobile-first PWA installable from a QR code — no App Store listing needed.

Target user: non-tech-savvy diamond broker who historically tracked deals in a paper register. The app replaces the register.

**Original author:** the mediator (Akshay Shah via `apshah06` GitHub account), who built the entire base app himself with Claude. Ronak forked it, added safety patches, and runs it as a secondary project.

**This is a fork, not Ronak's original work.** All feature development belongs to the mediator; Ronak contributes safety / data-loss-prevention patches.

---

## Repository structure

### Remotes

```
origin   https://github.com/RonakShah92/Broker_App.git    (Ronak's fork)
upstream https://github.com/apshah06/Broker_App.git       (mediator's canonical)
```

Pull mediator updates with:
```bash
git fetch upstream
git merge upstream/main
# resolve any conflicts, re-test, push to origin
```

### Files

- `index.html` — single-file PWA (all HTML/CSS/JS inline, ~1,886 lines)
- `CHANGES.md` — documents the 3 safety patches Ronak added on top of mediator's build
- `TEST_REPORT.md` — browser-automation test report (2026-10-07)
- `.gitignore` — ignores `.DS_Store`, `index-with-safety-patches.html`, `index-backup-before-review.html`, `*.html.bak`
- `index-with-safety-patches.html` — local-only reference copy (gitignored)
- `brokerbook-qr.png` — QR code for installing the PWA (lives in `diamond-seller-client/` currently, could move here)

---

## Current commits (origin/main)

```
2b7015a Re-apply 3 safety patches on top of mediator's regenerated build
7be35b2 Pull in mediator's regenerated build + test report
addf367 Create .nojekyll
```

The pattern: mediator pushes new feature work to upstream → Ronak pulls → Ronak re-applies the 3 safety patches (they are additive and isolated, so re-applying is a small merge) → push to origin.

---

## The 3 safety patches (what Ronak adds on top of mediator's work)

All patches target `index.html` only. They are additive, backwards-compatible, and fall back cleanly if newer browser APIs aren't supported.

### Patch 1 — `navigator.storage.persist()` (eviction protection)

**Where:** right after service worker registration, before the final IIFE close.

**What:** calls `navigator.storage.persist()` on app load. Chrome/Edge grant it silently for installed PWAs → browser won't auto-evict Broker Book data when the device runs low on storage.

**Behavior by browser:**
- Chrome/Edge (installed PWA, both Android + desktop): granted silently
- Firefox: shows one-time permission prompt
- Safari iOS: ignored (iOS doesn't honor this API; ITP still applies)

**Risk:** none. Wrapped in try/catch; no-op on older browsers.

### Patch 2 — Backup button uses native share when available

**Where:** `$("mBackup").addEventListener("click", ...)` handler.

**What:** tries `navigator.share()` with the backup JSON file before falling back to download. On modern Android Chrome, this opens the system share sheet → broker taps WhatsApp → sends backup to any chat (including themselves). **Backup now lives in WhatsApp cloud, retrievable from any device.**

**Fallback:** if `navigator.canShare({files:[...]})` is unsupported, falls back to exact previous download behavior.

**`lastBackup` tracking note:** only updated when the share resolves successfully. If the user cancels, `lastBackup` stays stale → Patch 3's reminder banner keeps nudging. Intentional.

### Patch 3 — Weekly backup-reminder banner on home screen

**Where:** `renderImportBanner()` function.

**What:** adds a second code path to the home-screen banner slot. If broker has data AND hasn't backed up in 7+ days (or never), shows a soft blue banner:
> *"Keep your data safe. Last backup was X days ago. If this phone's Chrome data is cleared, your deals disappear with it."*
> `[Later]` `[Backup now]`

**Priority:** existing import-register banner wins. Backup banner only shows if import banner is NOT showing.

**Session-only "Later":** sets module-level `backupNudgeHidden`, not persisted. App relaunch resets it, so broker is nudged again on next open until they actually back up.

**Combined impact:** these three patches cut ~70% of pilot data-loss risk with 36 lines of additive code.

---

## Service worker version policy

**Current:** `bb-2026-09-30-8` (set by mediator)

**Do NOT bump SW VERSION when patching only `index.html`.** The SW uses a network-first strategy, so it auto-fetches new HTML on next visit. Bumping SW forces a full refresh and invalidates all caches — unnecessary disruption for most HTML-only changes.

**Bump SW VERSION only when:**
- SW itself changes (fetch handlers, caching strategy, cache names)
- You add new static assets that need caching
- You deliberately want to force a full refresh for all users

---

## Local development

```bash
cd /Users/ronakshah/Desktop/side_project/Broker_App
python3 -m http.server 8000
# then open http://localhost:8000 in Chrome
```

### Testing

Before pushing anything, test in Chrome at `localhost:8000`:
1. Fresh install (clear site data) → import a sample register → verify all tabs load
2. Add a deal → check memos → mark overdue → verify banner + colors
3. Click Backup → verify share sheet opens (on Android/supported Chrome) or file downloads (fallback)
4. Wait 7 days OR manually set `S.settings.lastBackup = null` in console → verify Patch 3 banner shows
5. Dismiss banner with "Later" → refresh page → banner should reappear

**Zero console errors required.** The `TEST_REPORT.md` from 2026-10-07 is the baseline — new work should maintain all 7 tests passing.

---

## Rules for future work

### ✅ DO

- Keep safety patches isolated and additive (never couple with feature work)
- Pull mediator's new work regularly (`git fetch upstream && git merge upstream/main`)
- Re-run full test suite in Chrome after any merge
- Update `CHANGES.md` whenever you add a new patch
- Treat the mediator's design decisions as authoritative — our role is safety, not UX

### ❌ DON'T

- Don't touch mediator's feature code (due-date tracking, overdue section, export dialog, etc.) — that's his domain
- Don't bump SW VERSION casually (see policy above)
- Don't add new tabs, dialogs, or UI surfaces without discussing with mediator first (via PR)
- Don't migrate to IndexedDB (not needed at pilot scale — data is <1 MB per broker)
- Don't add cloud sync backend (explicit non-goal; only add if a specific broker asks)
- Don't commit `index-with-safety-patches.html` or any `.bak` files (gitignored for a reason)

---

## How to send patches back to the mediator

The upstream repo (`apshah06/Broker_App`) is public. Patches get sent as pull requests:

```bash
# after committing your safety patches to a feature branch
gh pr create --repo apshah06/Broker_App \
  --title "Safety patches: eviction protection + share-based backup + reminder banner" \
  --body "See CHANGES.md for full details. All 3 patches are additive, isolated, and fall back cleanly on unsupported browsers."
```

The mediator decides whether to merge. If he declines, our fork keeps the patches privately.

---

## Mediator relationship

**Akshay Shah** is the mediator for this project AND the diamond-seller-client project. See `~/Desktop/side_project/diamond-seller-client/roles-and-responsibilities.md` for the formal MOU structure.

For Broker_App specifically: he owns the product; we contribute patches when we see data-loss risks worth addressing. No formal compensation arrangement for this project.

---

## Deployment

The mediator deploys to GitHub Pages from his upstream repo. Ronak's fork is not deployed anywhere publicly — it exists for safety-patch work and as a backup.

The `.nojekyll` file is required so GitHub Pages serves the raw `index.html` without Jekyll processing.
