# Bitwarden Credential Setup — Design Doc

Deep dive for adding Bitwarden as the authoritative credential vault for
client work, alongside (not replacing) the existing CopyPasteSlotCrypt
system. This doc captures every decision, tradeoff, and integration point
discussed during the Apr 2026 conversation with Claude.

---

## TL;DR

- **Keep** `CopyPasteSlotCrypt` as-is. It is best-in-class for voice-first,
  fast, single-machine credential access. Don't migrate daily-use secrets.
- **Add** Bitwarden (free tier) as the *vault* for client credentials. It
  fills gaps the slot system was never designed for: metadata, TOTP/MFA,
  mobile access, recovery, sharing, and organization.
- **Bridge** the two with a one-way sync: Bitwarden is the source of
  truth, slots are a fast read-through cache for anything you want
  voice-pasteable on this machine.

---

## Why Bitwarden specifically (not 1Password, not KeePass)

| Option | Verdict | Reason |
|---|---|---|
| **Bitwarden** | Pick this | Free, open source, capable CLI, cross-device sync, mobile app, TOTP storage, recovery. ~90% of 1Password's value for $0. |
| **1Password** | Skip | Paid ($36/yr). Its biggest advantages (Windows Hello biometric unlock, `op run` subprocess injection, `op://` secret references) mostly evaporate once you're wrapping the CLI in an AHK helper anyway. |
| **KeePassXC** | Skip | Fully local, no built-in sync. You'd have to build sync yourself. Weaker CLI. Only worth it if you specifically want zero cloud involvement, which Bitwarden's self-host option covers better. |
| **Windows Credential Manager** | Skip as primary | Clunky UI, limited to Windows credential types, painful to script. Fine as a *helper* for caching Bitwarden session tokens (see below). |
| **Azure Key Vault / AWS Secrets Manager** | Skip | Wrong tool. Enterprise infrastructure secrets, not freelance credential management. |

---

## What the slot system already does well (do not rebuild)

| Capability | How the slots handle it |
|---|---|
| Strong encryption | AES-256-GCM, authenticated, same as any password manager |
| Voice-first access | `spit pass microsoft` — faster than any GUI can be |
| Local-first, no vendor | No account to breach, no subscription |
| Key/data separation option | README documents moving the keyfile off-machine |
| Zero friction for daily use | Already integrated into the Caster + AHK stack |

**Keep in slots:** OpenAI API key, GitHub tokens, Dragon license, Claude API
key, anything you paste into terminals/scripts repeatedly on this machine.
Migrating these to Bitwarden would be a downgrade.

---

## Where the slot system has gaps (what Bitwarden fills)

| Gap | Impact for client work |
|---|---|
| Slot is just a string | No username, URL, notes, security questions, expiry date, account recovery info. Client credentials often need all of these per entry. |
| No TOTP / 2FA seed storage | Can't generate 6-digit MFA codes. You fall back to phone for every MFA login — including Atlantic/McLeod/Microsoft which all require it. |
| No mobile access | If you need to log in to a client portal from your phone or away from your desk, slots are unreachable. |
| No secure sharing | Can't hand Rob a single McLeod credential without giving him the whole keyfile. |
| No recovery | Lose the keyfile = every encrypted slot is permanently gone. For Rob's business credentials, "permanently gone" is an unacceptable failure mode. |
| No breach monitoring | No HaveIBeenPwned integration, no "this password is 3 years old" warnings. |
| Flat namespace | Finding one credential among dozens requires remembering the exact slot name. No folders, tags, or metadata-based search. |
| No browser autofill | Not a huge deal for voice-first workflow, but still one more source of friction. |
| No audit log | No record of when a credential was last accessed or copied. |

---

## Division of responsibility

```
+----------------------------------+      +----------------------------------+
|  CopyPasteSlotCrypt (keep)       |      |  Bitwarden (add)                 |
+----------------------------------+      +----------------------------------+
|  Daily-use single-string secrets |      |  Full client credential records  |
|  API keys, personal tokens       |      |  Username + password + URL +     |
|  Dragon license                  |      |  TOTP + notes + security Qs      |
|  Voice-accessed 10+ times/day    |      |  Mobile access needed            |
|  Single machine OK               |      |  Recovery guaranteed             |
|  No metadata needed              |      |  Shareable with Rob/clients      |
|  No TOTP needed                  |      |  Organized by client             |
+----------------------------------+      +----------------------------------+
         ^                                           |
         |                                           |
         +------------ one-way sync -----------------+
         |  SyncSlotFromBitwarden(slot, bwItem)      |
         |  For credentials you want voice-pasteable |
         |  AND stored authoritatively in Bitwarden  |
         +-------------------------------------------+
```

Bitwarden is always the source of truth for anything stored in it. The
slot cache can get stale or be rebuilt from Bitwarden at any time.

---

## Installation and account setup

1. **Sign up** for a free Bitwarden account at bitwarden.com. Use the work
   Gmail (JamieHooperCode@gmail.com) since this is for client work. Set a
   long, memorable master password — this is the one secret that cannot
   live anywhere else.
2. **Enable 2FA** on the Bitwarden account itself. The free tier supports
   authenticator app (TOTP) and email. Pick authenticator app. Store the
   TOTP setup seed in your existing authenticator, and also print the
   recovery code.
3. **Print the recovery kit.** This is the "everything lost" escape hatch.
   Store it with the CopyPasteSlotCrypt keyfile backup — same "lose this
   and you're locked out forever" category, same backup locations (E:
   drive, planned Charli backup).
4. **Install the CLI:**
   ```powershell
   winget install Bitwarden.CLI
   # or: scoop install bitwarden-cli
   ```
   Verify with `bw --version`.
5. **Install the desktop app** for browsing/editing the vault. The CLI is
   for scripts; the desktop app is for when you need to see the big
   picture.
6. **Install the mobile app** on your phone. Enable biometric unlock
   there. This is the "I need to log into a client portal from the
   airport" escape hatch.

---

## Vault organization

Use **folders** (personal account) or **collections** (if you ever upgrade
to an organization). Suggested layout:

```
Personal/
    Email, banking, streaming, etc. (eventually migrate from slots if you want)
Atlantic Logistics/
    Rob's email credentials
    McLeod TMS login + TOTP
    Microsoft 365 admin
    Power Automate
    Any Atlantic-facing vendor portal
Client - <Name>/
    One folder per future client, same structure
Dev/
    GitHub machine tokens that need metadata
    Any API keys that need rotation tracking
    Anything you'd keep in an .env file
```

**Naming convention for items:** `<service> - <account identifier>`.
Example: `McLeod - atlantic admin`, `Microsoft 365 - rob@atlanticlogistics.com`.
The CLI matches on item name, so predictable names make `bw get` calls
unambiguous.

---

## CLI workflow

The one real friction with Bitwarden CLI: the session token dance.

```powershell
bw login              # once per device
bw unlock             # each time the session expires
# -> prints: export BW_SESSION="<token>"
$env:BW_SESSION = "<token>"
bw get password "McLeod - atlantic admin"
bw get totp "McLeod - atlantic admin"     # generates current 6-digit code
bw get item "McLeod - atlantic admin"     # full JSON
```

Session expires on lock (configurable). For script use, you want the
session token cached so you don't prompt on every call. Two options:

**Option A: Cache session token in Windows Credential Manager.**
`cmdkey` / PowerShell's `CredentialManager` module can store the session
token as a generic credential. AHK reads it when needed, falls back to
prompting for master password if the token is invalid. This is the
pattern 1Password gets "for free" via Windows Hello — we're replicating
it one layer up.

**Option B: Prompt once per boot.** Simpler. First call to
`GetCredential()` of the day prompts for the master password via AHK's
`InputBox`, unlocks, caches the session token in an AHK static variable,
never touches disk. When the machine sleeps/reboots, you prompt again.

Recommendation: **start with Option B** (simpler, no Windows Credential
Manager interaction), move to Option A only if the daily prompt becomes
annoying.

---

## AHK helper design — `Helpers/CredentialsFunctions.ahk`

New helper file, included from `MAINFUNCTIONS.ahk`. Follows the same
AHK-first pattern as `AlarmClockFunctions.ahk` and `VLCFunctions.ahk`.

**Public functions:**

```
GetCredential(itemName)              ; returns password string
GetCredentialTotp(itemName)          ; returns current 6-digit TOTP code
GetCredentialUsername(itemName)      ; returns username string
CopyCredential(itemName)             ; copies password to clipboard + auto-clear
CopyCredentialTotp(itemName)         ; copies TOTP to clipboard + auto-clear
TypeCredential(itemName)             ; types username + tab + password directly
SyncSlotFromBitwarden(slot, itemName); pulls password, re-encrypts into slot
BitwardenUnlock()                    ; explicit unlock prompt (rarely needed)
BitwardenLock()                      ; wipe cached session
```

**Internal:**

```
_BwSession                           ; static, cached session token
_BwEnsureSession()                   ; prompts for master pw if needed
_BwRun(args*)                        ; shell out to bw with session, capture stdout
_ClipboardAutoClear(seconds)         ; set a timer to clear clipboard
```

**Clipboard auto-clear is critical.** Copying a password to the clipboard
and leaving it there is how credentials leak into screenshots, clipboard
managers, and copy-paste history. Every `Copy*` function must schedule a
30-second clipboard wipe.

**TypeCredential pattern:** for forms where you want username + password
entered directly into the focused field without ever touching the
clipboard. Safer than copy-paste because there's no clipboard residue.

---

## Caster voice command ideas — `credentials_commands.py`

Pure mapping file, same pattern as `alarm_clock_commands.py`. All logic
in AHK.

```python
class CredentialsRule(MappingRule):
    mapping = {
        "Credential Commands": Text("Credential Commands Are Working"),

        # "get <client> password" -> copies to clipboard, auto-clears
        "get <client> password":
            R(named_mainfun_action_with_args("CopyCredential", "client"),
              rdescript="get <client> password -> CopyCredential(client)"),

        # "get <client> code" -> copies current TOTP code
        "get <client> code":
            R(named_mainfun_action_with_args("CopyCredentialTotp", "client"),
              rdescript="get <client> code -> CopyCredentialTotp(client)"),

        # "type <client> login" -> types username + tab + password
        "type <client> login":
            R(named_mainfun_action_with_args("TypeCredential", "client"),
              rdescript="type <client> login -> TypeCredential(client)"),

        # "sync client creds" -> refreshes all predefined slots from Bitwarden
        "sync client creds":
            R(mainfun_action("unused", function_name="SyncAllClientSlots"),
              rdescript="sync client creds -> SyncAllClientSlots()"),

        # "lock bit warden" -> wipes cached session
        "lock bit warden":
            R(mainfun_action("unused", function_name="BitwardenLock"),
              rdescript="lock bit warden -> BitwardenLock()"),
    }
    extras = [
        Choice("client", {
            "Atlantic McLeod": "McLeod - atlantic admin",
            "Atlantic Microsoft": "Microsoft 365 - rob@atlanticlogistics.com",
            "Atlantic email": "Email - rob@atlanticlogistics.com",
            # add per client
        }),
    ]
```

**Choice extras are important here** — they let Dragon match a short
spoken phrase ("Atlantic McLeod") to the full Bitwarden item name
("McLeod - atlantic admin"). This keeps voice commands natural while
still letting the vault use unambiguous internal names.

---

## The bridge pattern — `SyncSlotFromBitwarden`

For any credential you want *both* stored authoritatively in Bitwarden
*and* voice-pasteable via the existing `spit pass <name>` flow:

```
SyncSlotFromBitwarden(slotName, bwItemName) {
    pw := GetCredential(bwItemName)
    if (pw = "")
        return
    ; pipe into copy_paste_slot_crypt.py encrypt
    tmp := A_Temp "\_bw_sync_" A_TickCount ".txt"
    FileAppend(pw, tmp)
    try {
        RunWait('cmd /c type "' tmp '" | py -3 "' SlotCryptPy '" encrypt copy_paste_slots_passwords "' slotName '"', , "Hide")
    } finally {
        FileDelete(tmp)
    }
}

SyncAllClientSlots() {
    ; edit this list to define which Bitwarden items get mirrored to slots
    syncs := [
        ["mcleod",     "McLeod - atlantic admin"],
        ["atlanticms", "Microsoft 365 - rob@atlanticlogistics.com"],
    ]
    for s in syncs
        SyncSlotFromBitwarden(s[1], s[2])
    ToolTip("Client slots synced (" syncs.Length ")")
    SetTimer(() => ToolTip(), -2000)
}
```

**Why one-way only:** Bitwarden is the source of truth. If you edit a
credential, you edit it in Bitwarden, then run `sync client creds` to
refresh the slot cache. Never edit slot contents directly for synced
entries — you'd create divergence.

**Temp file hygiene:** same pattern as the existing slot crypt — plaintext
lands in a temp file briefly, gets piped in, temp file is deleted in a
`finally` block. Plaintext never sits on disk.

---

## Security notes

1. **Master password strength.** Bitwarden is only as secure as the
   master password. Use a long passphrase (5+ random words). Don't reuse
   it anywhere. Don't store it in slots or Bitwarden itself.

2. **2FA on the account.** Non-negotiable for client credentials. Use an
   authenticator app, not SMS.

3. **Recovery kit backup.** Print the Bitwarden recovery code and store
   it alongside the `copy_paste_slot_crypt.key` backup. Treat it as the
   same class of secret — "lose this and you're locked out forever."
   Same backup locations: E: drive, planned Charli backup.

4. **Session token hygiene.** If using Option A (Windows Credential
   Manager cache), make sure it's scoped to your user account. If using
   Option B (prompt per boot), the session lives only in AHK memory and
   is wiped on reload.

5. **Clipboard auto-clear is mandatory.** Every path that copies a
   credential to the clipboard must schedule a wipe. 30 seconds is the
   usual default; shorter is fine.

6. **Never log credentials.** Do not write passwords to
   `caster_messages.log`, `mainfun_calls.log`, or any debug output. The
   `_BwRun` internal helper should explicitly redact its args from any
   debug logging.

7. **Voice recognition risk.** Dragon logs recognition history. If you
   say "get McLeod password" ten times a day, "McLeod" appears in logs.
   That's fine — the item name is not the credential. But be aware that
   anything you dictate (including if you ever spoke a password aloud to
   "type it" via Dragon) would be in Dragon's history. Never dictate
   actual passwords. Use `TypeCredential` or `CopyCredential` instead.

---

## Threat model comparison

| Threat | Slot system | Bitwarden (free) | Bitwarden + master pw cached |
|---|---|---|---|
| Another user reading AppData | Protected (encrypted at rest) | Protected (encrypted at rest) | Protected |
| Malware scanning for plaintext | Protected | Protected | Protected |
| Cloud sync accidentally uploads | Protected if keyfile not also synced | Intentional (encrypted) | Intentional (encrypted) |
| GitHub repo leak of slot blobs | Protected (keyfile not in repo) | N/A (not in repo) | N/A |
| Lost keyfile / master password | **All data permanently lost** | Recovery code | Recovery code |
| Bitwarden server breach | N/A | Protected (E2E encrypted) | Protected |
| Malicious browser extension reading Bitwarden autofill | N/A | Real risk — review extension permissions | Real risk |
| Attacker with read access to your user account | Full compromise | Full compromise if session unlocked | Full compromise if cache valid |
| Phone theft | N/A | Protected by phone biometric + Bitwarden biometric | Same |

**Key insight:** the slot system and Bitwarden have similar on-machine
threat models when both are unlocked. The real differences are (1)
recovery (Bitwarden has it, slots don't), (2) cross-device access
(Bitwarden has it, slots don't), and (3) whether the encrypted data ever
touches the cloud (slots: no, Bitwarden: yes, but E2E encrypted).

---

## Implementation order

1. Create Bitwarden account + 2FA + print recovery kit + back up recovery
   kit to E:.
2. Install `bw` CLI and desktop app. Verify login + unlock + `bw get
   password <test item>` works from PowerShell.
3. Migrate the Atlantic Logistics credentials into the vault manually
   via the desktop app (faster than CLI). One folder, well-named items,
   TOTP seeds for anything that has MFA.
4. Write `Helpers/CredentialsFunctions.ahk` with Option B (prompt per
   boot) session handling. Just `GetCredential` and `CopyCredential` to
   start — add the others as needed.
5. Add `#Include` to `MAINFUNCTIONS.ahk`.
6. Create `credentials_commands.py` Caster rule file with 2 or 3 voice
   commands for one client. Test end-to-end.
7. Add more clients to the Choice extra as they come up.
8. Only after all the above is stable: add `SyncSlotFromBitwarden` and
   `SyncAllClientSlots` if you actually want the bridge. It's optional —
   most client credentials probably don't need voice-paste access, they
   need mobile access and metadata.

---

## Open questions to answer during implementation

- Does the Bitwarden free tier let you store TOTP seeds? **Yes** as of
  2024 — confirm on signup.
- What's the right default clipboard-clear duration? 30s is standard,
  but 10s is safer for real credentials on a screen-shared machine.
- Should `GetCredential` ever print the password to any output? **No.**
  Return it to the caller only.
- How to handle the "Bitwarden is offline" case? `bw` has an offline
  mode using the local encrypted vault cache. Works as long as you've
  unlocked at least once. Good enough.
- Does any existing rule file need to learn about Bitwarden, or is this
  purely additive? **Purely additive.** New helper, new Caster rule, no
  changes to existing code.

---

## References

- Bitwarden CLI docs: https://bitwarden.com/help/cli/
- Bitwarden free vs. paid feature comparison: https://bitwarden.com/pricing/
- CopyPasteSlotCrypt README: `Scripts/CopyPasteSlotCrypt/README.md`
- AHK-first architecture: `CLAUDE.md` in the caster rules repo
- Slot encryption memory: `.claude/projects/.../reference_slot_encryption.md`
