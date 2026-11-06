---
tags: [programming, design-doc, downloads, music, bandcamp, google-photos, google-voice, local-viewer, chrome-extension]
created: 2026-11-02
updated: 2026-11-02
status: built 2026-11-02 — page on :8301, 5 actions, Bandcamp auto rule, Bandcamp smart folder, jump N on Bandcamp
---

# Download manager

One place to pick what's in Downloads and do something to it. Not built per source: the sources (Bandcamp, Takeout, shared photo albums...) all land in `E:\Downloads`, and what changes is what she *does* with each kind of file.

Say **"download manager"**, or use the Downloads link in the page header.

## The three pieces

| Piece | File | Job |
|---|---|---|
| Items | `Scripts/downloads/dm_items.py` | What's waiting (finished downloads from `INIDATA/downloads.jsonl` + loose files in `E:\Downloads`) and what **type** each is: album zip, photos folder, image, book... |
| Actions | `Scripts/downloads/dm_actions/*.py` | One file per action. Each says which types it takes. **New action = new file**; the page picks it up without a restart. |
| Runner | `Scripts/downloads/dm.py` | Runs an action, writes one line per item to `INIDATA/download_actions.jsonl`, applies the auto rules. |

Page: `dm_server.py` + `dm_page.html` (`http://127.0.0.1:8301`). AHK: `Helpers/DownloadManager.ahk`.

## Actions (2026-11-02)

1. **Add to music library**: unzip, strip Bandcamp's `Artist - Album - ` track prefix, split multi-album downloads (discographies by folder; an album + its demos sharing one folder by album tag, 2+ tracks per group), file with `acquire/library.py sort --all` (one Artist/Album per album, CD1/CD2 folded), index for the local player, zip to the Recycle Bin, remember the source.
2. **Upload to Google Photos**: via `gphotos.py`; picks a side album or none.
3. **Text to someone**: opens the Google Voice thread and pastes the images; she presses send.
4. **Show in Explorer**.
5. **Move to Recycle Bin**.

An item is *handled* (hidden unless "show handled") once an action with `handles: True` succeeds on it, or its file is gone.

## Numbers (voice and numpad are one set)

Every control has a speakable number drawn into the page (kit.js inline jump numbers, new-tab-page look): rows on screen read 1, 2, 3… in their gutter; actions are pinned at **11–15**; "show handled" is **16** (pins stay in 11–29, easy to say; kit skips them, so rows read 1–10 and carry on at 17). The picker covers the page and numbers only its choices. In the box, `N` Enter = jump N (several: `3 5 9`), bare Enter = first action that fits, `0` = clear / close the picker. Saying "four five" picks both.

## Automatic rules

`INIDATA/download_rules.json`. The Chrome bridge consumer (`chrome_bridge/consumers/downloads_consumer.py`) runs `dm.py auto` on every finished Chrome download; the first matching rule runs its action and a tooltip says what happened. The download log now records each download's URL so rules can match on the source.

Starting rule: album zips from `bandcamp.com` / `bcbits.com` → music library.

## Bandcamp

- `jump N` on the collection page (`bandcamp.com/jamiemaybe`) clicks album N's download link, makes sure the format is FLAC, and clicks Download. The rule above files the zip. Contexts: `bandcamp_collection`, `bandcamp_download`.
- Music Folders has a **Bandcamp** smart folder (under Local albums), read from `MusicData\catalog\album_sources.json`.

## Related

[[GAINESVILLE_EVENTS]] (same local-page pattern) · [[LOCAL_APP_HUB]] · [[LOCAL_PLAYER]]
