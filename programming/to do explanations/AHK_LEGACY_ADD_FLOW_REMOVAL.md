---
tags: [todo, ahk, voice, cleanup, refactor]
related: ["[[VOICE_COMMAND_SYSTEM]]"]
status: site-doc-bodies-deleted; adddirectorybyvoice-pending-keyboard-test
updated: 2026-06-23
---

## DONE 2026-06-23 — AddHardcodedSite + AddGoogleDoc bodies DELETED

779+ lines removed from `Helpers/VoiceConfigManager.ahk` (validate-clean, closure resolves).
Deleted (verified transitive closure, callers only within the delete set): `AddHardcodedSite`,
`AddGoogleDoc`, `_VoiceConfigSiteDecisionTree`, `_VoiceConfigTitleOrSkipPrompt`,
`_VoiceConfigPeekDoc`, `_VoiceConfigOpenPrefixed`, `_VoiceConfigPeekSite`,
`_VoiceConfigReverseLookupSite`, `_VoiceConfigReverseLookupSiteAll`,
`_VoiceConfigEditOrRemovePrompt`, `_VoiceConfigNormalizeUrl`, and `Scripts/SiteEditor.ahk` (whole file).
KEPT (shared with the live smart-add): `_VoiceConfigAliasPrompt`, `_AddSiteContextBridge`, +
all widely-shared utils (`_VoiceConfigInputBox/Tooltip/RunHelper/ShortUrl/SnapTokens`).
Stale comments scrubbed (RegistryEditorMenu false "stays reachable", `_DIRECTORY_DOCUMENTATION` add-fn list).

**STILL PENDING here:**
- `AddDirectoryByVoice` body — gated on Jamie keyboard-testing `AddDirectorySmart` (wired as
  `add directory legacy`). Delete its body + any then-orphaned helpers once confirmed.
- The context-LINKS write cluster (`_VoiceConfigSiteCtxWrite` / `_VoiceConfigAddLinkHere` /
  `_VoiceConfigAddSubContext`, now zero-caller orphans) belongs to the **dormant context-links
  deletion** (VOICE_COMMAND_SYSTEM §18.3 step 4) — delete alongside `_iter_context_links` /
  `site_contexts.py` / `SiteBrowserMenu` / `_DestAddLinkQuiet`.
- NOTE: the quick orphan-detector script was unreliable (skipped statement-style calls). Trust
  `ahk.py validate` + closure + per-name grep, not an auto-orphan sweep.

# Retire the bespoke add-flow AHK bodies (AddHardcodedSite / AddGoogleDoc / AddDirectoryByVoice)

The three old hand-rolled add wizards are **superseded** by the quiet shared add
(`AddSmartDestination` / `AddDocSmart` / `AddDirectorySmart` in `RegistryEditorMenu.ahk`,
VOICE_COMMAND_SYSTEM §17.5d–e). Voice wiring is the safe part and is **done** (2026-06-23):

- `AddHardcodedSite` — already unwired (no phrase maps to it; "add site" → `AddSmartDestination`).
- `AddGoogleDoc` — "add doc legacy" phrase **removed**; now unwired.
- `AddDirectoryByVoice` — still wired as **"add directory legacy"** (safety net) until
  `AddDirectorySmart` is keyboard-verified. Delete that phrase + body once confirmed.

## Why the BODIES weren't deleted yet (the landmine)

The wizard bodies live in `Helpers/VoiceConfigManager.ahk` and lean on a web of `_VoiceConfig*`
helpers — and **at least one is SHARED with the live smart-add**: the macro-context-bridge
offer (`VoiceConfigManager.ahk:480`, the `_AddSiteContextBridge` path) is also called by the
new flow via `_RegMaybeContextBridge` (`RegistryEditorMenu.ahk:1279-1282`). Ripping a body out
blind would orphan-or-break shared helpers. `Scripts/SiteEditor.ahk` is also spawned by
`AddHardcodedSite` and needs a reachability check.

## The careful deletion pass (do as its own focused task)

1. For each of `AddHardcodedSite` / `AddGoogleDoc` / `AddDirectoryByVoice`, list every helper it
   calls (`ahk.py deps` + grep), then for each helper grep the WHOLE codebase for other callers.
2. Delete only helpers with **zero** non-legacy callers. Keep shared ones (context-bridge,
   position-token validator, `_VoiceConfigGet*Path` detectors — the path detectors are now reused
   by `AddDirectorySmart`, so they STAY).
3. Decide `Scripts/SiteEditor.ahk`'s fate (was it only AddHardcodedSite's row-editor? the Registry
   Editor replaces it).
4. Remove the bodies, scrub stale comments (`web_opener_commands.py` header + the `__no_search__`
   sentinel notes still name `AddHardcodedSite`), update `_DIRECTORY_DOCUMENTATION.ahk`'s add-fn list.
5. `ahk.py validate` + include-closure + a live "add site/doc/directory" smoke test.

## How to apply

Low urgency — the legacy bodies are now dead-but-harmless (unwired). Do the trace-and-excise in
one sitting when touching VoiceConfigManager next, so a half-deletion never strands a shared helper.
Gate `AddDirectoryByVoice`'s removal on Jamie confirming `AddDirectorySmart` works live.
