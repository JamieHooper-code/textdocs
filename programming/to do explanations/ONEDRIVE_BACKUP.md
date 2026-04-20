# OneDrive cloud mirror — design + gotchas

## Goal
Every folder currently backed up to `E:\Important Backups` should also be
mirrored to OneDrive as a third copy (live disk -> E: -> cloud). OneDrive
acts as off-site redundancy in case both the SSD and E: die.

## Architecture (how to add it)
Add a parallel mirror destination to `Scripts/git_push_config.json`:
a `oneDriveRoot` alongside `externalBackupRoot`, and per-mirror a flag
for whether to copy to OneDrive in addition to E:. `GitPushAll.ps1`
robocopies to both. OneDrive's sync agent handles the upload on its own
schedule.

Destination path would be e.g.
`C:\Users\jamie\OneDrive\Important Backups\<name>\`.

## DO NOT sync to OneDrive
These go to E: only (or encrypted before syncing):

- `.atlantic_graph_certs` — Microsoft Graph client cert. Cloud-provider
  cert on same-provider cloud = circular compromise risk. Keep local
  + E: only.
- Any `.env`, `.aws/`, `.ssh/`, `.gcloud/`, `.azure/` creds.
- The `CopyPasteSlotCrypt` keyfile (AutoHotkey/Scripts/CopyPasteSlotCrypt/).
  The encrypted ciphertext is fine on OneDrive; the key is not. Key +
  ciphertext must never share a storage provider.
- API keys / tokens / OAuth refresh tokens in plaintext.
- Code-signing / GPG private keys.
- Database dumps with user PII.

**Rule of thumb:** if a file grants access to something, assume OneDrive
= Microsoft can read it + anyone who phishes my Microsoft account can
read it. Either encrypt first (7z AES-256 with a strong passphrase, or
age/gpg) or keep it off the cloud.

## The "two desktops" problem — KEEP KFM OFF
Known Folder Move (KFM) is the OneDrive setting that redirects
`C:\Users\jamie\Desktop` to `C:\Users\jamie\OneDrive\Desktop`. If it's
ever on, every hardcoded path across AHK / Caster / Stream Deck /
batch files breaks silently.

**To fix:** OneDrive tray -> Settings -> Sync and backup -> Manage backup
-> turn OFF Desktop (and Documents/Pictures if on). It asks where to
put existing files — choose "back to original location." Clean up any
stragglers manually.

After this, the OneDrive folder at `C:\Users\jamie\OneDrive\` exists but
is a normal folder, not the Desktop. Copies dropped there sync. Nothing
moves out of `Desktop\Important`.

## Storage math sanity check
- Microsoft 365 Personal: 1 TB OneDrive.
- Per-file cap: 250 GB.
- Current E: mirror targets: AutoHotkey, Atlantic Logistics, certs,
  CredentialFiles, Important (whole desktop).
- `Important` contains `projects/poem-poster/clipart/` at ~27 GB.
  **Exclude `clipart/` from the OneDrive mirror** (and other bulk media
  from item 16b). Cloud backup of clipart is wasteful — it's not data
  I'd ever restore from cloud, and it eats quota fast.

## Mirror destination structure (proposed)
```
C:\Users\jamie\OneDrive\Important Backups\
  AutoHotkey\
  Atlantic Logistics\          (excludes _certs)
  CredentialFiles\             (encrypted blobs only; keyfile stays local)
  Important\                   (excludes big media: clipart/out/etc.)
  ... (deliberately NO _certs, NO raw .env, NO keyfile)
```

## Implementation TODO (when ready)
1. Turn KFM off first, verify nothing broke.
2. Extend `Scripts/git_push_config.json` schema to support OneDrive
   mirror with per-target `toOneDrive: true/false` and `excludeSubpaths`.
3. Update `GitPushAll.ps1` Phase 2 to do E: + OneDrive in sequence.
4. Audit the first few runs: watch OneDrive quota, watch for filename
   collisions (OneDrive has stricter char rules than NTFS — `#`, `%`
   historically, plus trailing spaces/dots).
