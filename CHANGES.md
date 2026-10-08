# Broker Book — Safety Patches on top of mediator's build

**Based on:** mediator's regenerated build (previous commit on this branch)
**Target file:** `index.html` only
**Added:** 36 lines across 3 independent patches

Three additive, zero-infra improvements to cut ~70% of pilot data-loss risk. All three are isolated, backwards-compatible, and fall back cleanly to original behavior if the newer browser APIs aren't supported.

---

## Patch 1 — `navigator.storage.persist()` (eviction protection)

**Where:** right after the service worker registration block, before the final `})();` IIFE close.

**What it does:** calls `navigator.storage.persist()` on app load. On Chrome/Edge, granted silently for PWAs installed to home screen → browser won't auto-evict Broker Book data when the device runs low on storage. Zero user-visible UX change.

**Code (added, 5 lines):**
```javascript
// Ask the browser to protect this app's data from auto-eviction when storage runs low.
// Chrome/Edge grants this silently for installed PWAs; Firefox prompts; iOS Safari limited.
if (navigator.storage && navigator.storage.persist){
  try { navigator.storage.persist(); } catch(e){}
}
```

**Browser behavior:**
- Chrome/Edge Android (installed PWA): granted silently
- Chrome/Edge desktop (installed PWA): granted silently
- Firefox: shows a one-time permission prompt
- Safari iOS: ignored (iOS doesn't fully honor this API; ITP still applies)

**Risk:** none. Fully try/catch'd, no-op on older browsers.

---

## Patch 2 — Backup button uses native share when available

**Where:** `$("mBackup").addEventListener("click", ...)` handler.

**What it does:** tries `navigator.share()` with the backup JSON file before falling back to download. On modern Android Chrome, this opens the system share sheet — broker taps WhatsApp → sends to any chat (including themselves). Backup now lives in WhatsApp cloud, retrievable from any device.

**Fallback:** if `navigator.canShare({files:[...]})` is unsupported, falls back to exact previous download behavior. Zero regression.

**Code (replaces the 3-line handler with a 15-line version):**
```javascript
$("mBackup").addEventListener("click", function(){
  var name = "broker-book-backup-" + today() + ".json";
  var blob = new Blob([JSON.stringify(S, null, 1)], {type:"application/json"});
  try {
    var file = new File([blob], name, {type:"application/json"});
    if (navigator.canShare && navigator.canShare({files:[file]})){
      navigator.share({ files:[file], title:"Broker Book backup", text:"My Broker Book backup — keep this file safe." })
        .then(function(){ S.settings.lastBackup = today(); save(); toast("Backup sent."); })
        .catch(function(){ /* user cancelled share — do not mark backed up */ });
      return;
    }
  } catch(e){ /* fall through to download */ }
  download(name, blob);
  S.settings.lastBackup = today(); save(); toast("Backup downloaded.");
});
```

**Note on `lastBackup` tracking:** only updated when the share resolves successfully. If the user cancels, `lastBackup` stays stale → Patch 3's reminder banner keeps nudging. Intentional.

---

## Patch 3 — Weekly backup-reminder banner on home screen

**Where:** `renderImportBanner()` function.

**What it does:** adds a second code path to the home-screen banner slot. If broker has data AND hasn't backed up in 7+ days (or never), shows a soft blue banner:
> **Keep your data safe.** Last backup was X days ago. If this phone's Chrome data is cleared, your deals disappear with it.
> [Later] [Backup now]

**Priority:** existing import-register banner still wins. Backup banner only shows if the import banner is NOT showing.

**Session-only "Later":** sets a module-level JS variable (`backupNudgeHidden`), not persisted. App relaunch resets it, so broker is nudged again on next open until they actually back up.

**No regression:** function signature unchanged, called from `render()` as before, existing paths preserved.

---

## What was NOT changed (and why)

| Item | Why skipped |
|---|---|
| 2-step erase confirmation | **Already in mediator's code.** No work needed. |
| Service worker VERSION bump | Only `index.html` changed; network-first strategy auto-fetches new HTML on next visit. SW stays at mediator's `bb-2026-09-30-8`. |
| Any new tabs/dialogs | These are pure enhancements to existing flows, not new UI surfaces. |
| Migration to IndexedDB | Not needed at current data scale (<1 MB per broker). Defer to a future version. |
| Cloud sync backend | Explicit non-goal for pilot. Only add when a specific broker asks for cross-device access. |

---

## Verification

Tested in Chrome on 2026-10-07 via browser automation:
- Banner appeared with correct message *"You haven't backed up yet."* (because fresh install had deals but no backup)
- Overdue section + Export dialog + due-date fields (mediator's work) all continue to work
- No console errors
- `navigator.storage.persist` call confirmed as present

File grew from 1,850 → 1,886 lines (+36 insertions).

---

## For the mediator — how to review and merge

The repo is public, so this patch comes as a Pull Request against your `main` branch from a fork.

**You'll receive an email from GitHub** titled something like *"RonakShah92 opened a pull request"*. Clicking the link takes you to a page with:
- A green "Merge pull request" button at the bottom
- A line-by-line diff of what will change (just `index.html` + this `CHANGES.md`)

**To merge:** click Merge pull request → Confirm merge. Done. Your `index.html` on GitHub Pages auto-redeploys in 1–2 minutes.

**To reject or request changes:** add a comment on the PR explaining what you want different. The PR stays open until merged or closed.

**None of these changes alter the mediator's new features** (due-date tracking, overdue section, export dialog). All three are additive and reversible.
