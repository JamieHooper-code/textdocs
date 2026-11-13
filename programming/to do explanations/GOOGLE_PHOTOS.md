---
tags: [programming, google-photos, google-voice, journal, reader, ahk]
---
# Google Photos + photos texted to me

Built 2026-10-30. Related: [[JOURNAL_SYSTEM]], [[GLOBAL_CLICK_HINTS]] (the Macro Bridge extension).

## What it does

- **"grab photos"** on a Google Voice thread: lists the photos in the conversation, newest first, each previewed on the right, low-quality ones flagged. Pick which ones, then Enter. The default sends them to Google Photos.
- **"grab chat"** on a Google Voice thread: after you pick the messages, any photos in them come up in the same pick screen. The ones marked "in the journal entry" are saved onto their messages, and the reader shows them inside the conversation.
- **Every Google Voice conversation entry now opens in the reader** ("grab read"), one page per day, with photos inline. Before, only AI chats did.
- Every photo is also kept on disk: `E:\Media\Photos\GV\<person>\<date_time>_<id>.jpg`.

## Account and sign-in

jamieeehooper@gmail.com (three e's). The OAuth client is the same one gdoc.py uses (Google Cloud project 526439768965). The Photos Library API and the Photos Picker API were enabled on it 2026-10-30.

While that Google app is in **Testing**, the sign-in dies 7 days after the Allow click. Moving the app to Production would end this for Photos and the poems doc together. That's still to do.

**The sign-in now does itself (2026-11-09).** `Scripts/google_auth/google_reauth.py` runs the OAuth flow in a small Chrome app window in the bottom-right corner and presses Google's screens through UIA: the account, "Continue" past the unverified-app warning, "Select all", "Continue". It takes about 10 seconds. Chrome only shows a page to UIA once its window has had real focus (the extension could read the DOM the whole time, while UIA saw only the frame), so it waits for a 2-second gap in her typing, focuses the window for the presses, then gives focus back. **Desk 3 doesn't work** for the same reason: a window there can't be focused without taking her there. A password or 2-step prompt is handed to her. Lock and 30-minute cooldown on a failed try; log in `INIDATA/GoogleAuth/reauth.log`. `py gphotos.py auth --by-hand` is the old click-it-yourself version.

**Nothing she grabs is lost.** `gphotos.upload_or_queue` (what the CLI `upload` and the download manager use) puts any file that didn't upload into `INIDATA/GooglePhotos/pending_uploads.json`, with its album key and description. An expired sign-in also starts `gphotos.py reauth --notify` in the background, which signs in, drains the queue, copies album photos into her real album, and shows a tooltip. Every later successful upload drains the queue too. A file is dropped from the queue after 6 failed tries (it stays listed with `gave_up`), or when it no longer exists on disk. `py gphotos.py pending` / `drain` / `status` (which shows the pending count). The poems Doc uses the same sign-in: `gdoc.py auth`, which `poem_doc_sync.py` starts by itself on an expired login. Its queue is the poems not yet in the Doc.

## Albums: why there are "(grabs)" albums

Since 2025-03-31 Google only lets an outside app add photos to albums **the app created**. Her own Cody / Charli / Cedar albums can't be touched through the API. So:

- Each album key has an app-made side album: "Cody (grabs)", etc. The map is `AutoHotkey\INIDATA\GooglePhotos\albums.json`, and `main_title` names her real album.
- The suggestion comes from the person (linked in albums.json) or from their GV label: Mika's contact is labelled "(Cody)", so Mika's photos suggest Cody.
- **The copy into her real album (built 2026-10-30):** `Scripts\google_photos\gphotos_sync.py` opens the side album in Chrome for a few seconds (in front: Google Photos ignores clicks in a background tab), ticks every photo, adds them to the real album, closes the tab and puts her back on hers. Google skips photos already there ("2 items already in album"), so re-copying is harmless. It runs after every album upload. The real album must match by EXACT title, chosen from her actual album list in `OpenGooglePhotosAlbums`, never typed. It refuses rather than guesses.
- As of 2026-10-30 there is no album named "Cody", "Charli" or "Cedar" in this account. The nearest are "Jamie - Charli" (and others) and "Cedar Fig Tree Poem", so each needs its real album picked.

## Pieces

| Piece | Where |
|---|---|
| Google Photos module (library + CLI) | `Scripts\google_photos\gphotos.py`: auth, upload (dedupes by file hash, ledger `INIDATA\GooglePhotos\uploads.jsonl`), albums, picker, quality |
| GV photos to disk | `Scripts\google_photos\gv_photos.py`, via the Macro Bridge ops `gv.photos` + `fetch.bytes` |
| AHK | `Helpers\GooglePhotos.ahk`: `GrabGVPhotos`, `GVPhotosPick`, `GVPhotosSave`, `GVPhotosIntoConversation` |
| Grab chat hook | `Helpers\JournalCapture.ahk` `GrabGVConversationToJournal` |
| Reader | `Scripts\reader\reader_collection.py` `resolve_gv_conversation`; `/api/photo` in read_server.py |
| Settings | `gphotos.default_dest`, `gphotos.journal_default` |
| Tests | `Scripts\codebase_tools\tests\test_google_photos.py` |

## Low quality

Texting squashes photos. A photo is flagged when it's under 1 MP, its short side is under 720 px, or it's very heavily compressed. Measured 2026-10-30: GV's largest size for a phone photo is 1200×1600. A 1024×768 one is what MMS left of it.

## Ideas for later

- The picker (`py gphotos.py pick`) lets you choose ANY photos from the whole library in Google's own page. That's the way in for "do stuff with my existing photos".
- Moving the OAuth app to Production.
