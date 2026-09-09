---
tags: [programming, people, media-system, journal, google-voice, caster, ahk]
created: 2026-09-04
related: ["[[MEDIA_SYSTEM]]", "[[JOURNAL_SYSTEM]]", "[[JOURNAL_SOURCES]]", "[[COMPLETION_LOG]]"]
---

# The person store — one record per human

Jamie's people lived in five places at once. As of 2026-09-04 they live in one,
and the other four are **generated views** of it.

## What it was

| Store | Count | Written by |
|---|---|---|
| `E:\Media\catalog\person.json` | **92** records / 103 names | `people.py` |
| `INIDATA\VoiceChoices\gv_contacts.json` | **77** name → phone | `AddGVNumber` ("number add") |
| `INIDATA\VoiceChoices\gv_people.json` | **69** harvested from mail | `mailwatch.py people-scan` |
| `INIDATA\VoiceChoices\completion_friends.json` | **1** (Cedar) | clog |
| `INIDATA\CopyPasteSlots\*.enc` | ~14 person-ish of 92 | CopyPasteManager |

183 distinct names between them, with no shared identity. `gv_contacts` and
`gv_people` overlapped on 36; 80 names existed **only** in a Google Voice store.

The journal and the media catalog were never rivals — `journal.py` already does
`import people as people_store`, so those two were one system all along.

## Why `person.json` won the merge

It is **thin in attributes but rich in links**. Only 3 records had tags, 10 had
aliases, **0 had notes** — but **74 of the 92 were referenced by journal
entries**, and every media item's `recommended_by_ids` points here.

Its **ids are load-bearing; its fields were not.** So the data came to the ids.
Rebuilding the store around the bigger contact list would have orphaned two
thirds of the journal's people links.

## The sidecars survive as views

`gv_contacts.json` and `gv_people.json` have real consumers with no reason to
learn about the person store: Caster's `text <contact>` Choice, mailwatch's
number matching in a daemon that must not shell out, and the `<gv_name>` Choice
behind every "notify <person>" watch.

So they are **regenerated** from `person.json`, exactly the way
`reading_books.json` is regenerated from the catalog. Nothing downstream
changed, and there is now one place a phone number is edited.

That also corrected a wrong assumption: `gv_contacts.json` looked like a dead
leaf with no writer, but `AddGVNumber` (voice **"number add"**) has been
appending to it all along. **Jamie caught that** — "I don't think I got 77
handwritten things in there." It is now a view, and `add_voice_choice.py` writes
through to the store. Without that write-through, "number add" would be a
**silent data-loss path**: the number would survive exactly until the next
regeneration.

## The record

```json
{
  "id": "person:cedar", "name": "Cedar",
  "aliases": ["Cedar <3"], "tags": ["story and steep"],
  "pronouns": "", "birthday": "", "how_we_met": "", "notes": "",

  "phones": [{"number": "+1…", "label": "", "src": "gv_contacts"}],
  "emails": [{"address": "…", "label": ""}],
  "addresses": [{"text": "…", "label": "home"}],
  "activities": ["walk"],
  "roles": ["contact"],
  "gv": {"count": 41, "last_seen": "…", "kinds": ["contact"]}
}
```

Contact fields are **absent until they hold something**, so a missing key means
"unknown" rather than "explicitly blank". `src` on a phone is the same
provenance rule tags and people links use: a harvested `auto` value is rewritten
freely, a `manual` one she typed is never touched.

Addresses are stored in the clear — she was asked and said she isn't concerned.
The encrypted `spit` slots are left exactly as they are and keep working.

## Roles, and the grammar explosion they prevent

**`read <person>` is generated from the store.** Folding the contacts in took it
from 92 records to 169, which would have put "Florida Blue" (an insurance
company in her contacts) and "Dad" behind a command meaning *show me what this
person recommended*.

Roles are the filter, and getting the rule right took two attempts:

1. **"Recommenders only"** — principled, and **wrong**. It dropped 62 people who
   had been sayable for months (cedar, charli, lina, kayla), because most of
   them have never formally recommended anything; they are in the store from
   journal links and manual adds. *Narrowing a grammar she already uses is a
   regression even when the new rule reads better.*
   (It also first looked like only 13 qualified, because most items name their
   recommender in the legacy `recommended_by` **string**, not
   `recommended_by_ids` — the string has to be resolved too.)
2. **"Everyone except a bare phone number"** — the minimum that solves the
   actual problem. A record whose *only* role is `contact`, created by the
   import and never touched since, stays out. Verified: **91 choices before,
   91 after, nothing lost.**

`roles_of()` is **derived plus manual**, which is what Jamie asked for: `contact`
from having a phone, `recommender` from a credited item, `correspondent` from
letters — computed every time so they cannot go stale — plus anything she sets
by hand.

## Six bugs the verification caught

**`rename` cannot recase.** It compares names case-insensitively and returns
early, deliberately, so "rename Lena to lena" is a no-op. That makes it
structurally unable to fix the store's seeded lower-case names — and it
**reported success while changing nothing** until the store was read back.
`_adopt_better_casing` sets `name` directly; the id never moves and `find` is
case-insensitive, so nothing relinks and the old spelling still resolves.

**Regenerating a sidecar destroys the capitalisation it was the only copy of.**
Running `sidecars` before the recasing persisted wrote the store's `jessie` back
over her `Jessie`. Hence `import-gv --from <file>`: point it at a pre-merge copy
to recover what she typed.

**`merge` lost a fact, not just a record.** It predated every contact field and
carried only tags and aliases, so folding `Rigo and Ten` into `Rigo` **silently
dropped the couple's shared phone number**, and folding `Cedar <3` into `Cedar`
dropped Cedar's Google Voice history. Nothing errored. Caught by reading the
merged records back against the backups; `_merge_contact_fields` now carries
phones, emails, addresses, activities, roles, scalars and a merged `gv` block
(bigger count, later sighting, union of kinds), and the damage was healed by
re-running the import from the pre-merge copies.

**A phone-book label is not a person, and the journal auto-linker could not
tell.** The Google Voice import brought in the number she calls to reach her
therapist under the name **`Therapy`**, her credit card under **`Discover`**,
and one contact simply called **`Summer`**. `_name_patterns` in `journal.py`
derives a word-boundary matcher from any name longer than four characters, so
each of those linked *a person* to every entry using the ordinary word:
`Therapy` alone matched **28 entries**, and the count only grows, because
journalling about therapy is the point of the journal. Nothing errors and
nothing is logged — the wrong links surface months later on a person's page as
writing that has nothing to do with them.

The escape hatch already existed for the opposite problem (a person called `T`
matching every "T-shirt") but only in its populated form. **An explicit empty
`match_patterns` list is now the third state: never auto-link this record by
name.** She can still link one by hand; only the automatic pass is off. Absent
vs empty is one falsy value apart in Python, and that was the actual bug:
`if person.get("match_patterns"):` had to become `is not None`.

Reviewing the backfill in context found three of these, not one. Every hit for
**`LASER`** is laser hair removal; **`Forest`** matched *"a forest walk gives
you"*; **`Sandy`** (the display name behind `Florida Blue`) matched a line of her
own poetry and the sandy floor of a Roman amphitheatre. LASER and Forest are
opted out entirely; Sandy keeps `Florida Blue` as its only matcher, which is
the half that works.

**The rule for reviewing one of these: read the matches, not the names.** `Mom`,
`Dad`, `Ash`, `Meg`, `Jess` and `Vic` all look like common words and every hit is
a real person. `Forest` looks like a name and no hit was.

**One known false positive is left in on purpose.** `person:wild` is forced
case-INsensitive because Jamie writes the name lower case, so *"let's give Icarus
10 minutes to go wild"* links her. A lookbehind would fix it, but "wild about"
is how she writes a real mention (*"I talked to wild about seeing Cedar"*) — and
losing a real link to a close friend silently is much worse than one spurious
one. 1 idiom in 32 occurrences.

The backfill then ran: **+67 links across 41 entries, 0 lost**, 74 → 102 people
with at least one link. Verified idempotent (a second pass reports +0/-0) and
diffed against a pre-scan copy of the store for lost links.

**An alias is data, and data reaches the GRAMMAR.** This is the one that cost
an outage rather than a wrong link. Cedar's stored alias `Cedar <3` flowed
through `people_sync` into `gv_contacts.json`, which is a Caster `Choice` map —
and a Choice key is a Dragonfly *spec*, not a string. `<3` reads as the start
of an extra reference, the parse raised inside the rule's class body, and
**every Google Voice command stopped existing at once**: `text Cedar`,
`text <textnv>`, `call <name>`, `number add`. Nothing was voiced; the only
trace was a traceback in `caster_messages.log`.

The same alias caused a second, quieter failure. Two stored names sanitising to
one spoken key overwrote each other by arrival order, so `notify Cedar` began
passing `"Cedar <3"` and armed a **second mail watch on Cedar's number** beside
the one running since August — two toasts per message, and an off switch that
reached neither.

Both are fixed where they belong rather than by scrubbing the store: the store
is *right* to know Cedar's alias, and the mail matcher needs the raw name to
compare against what Google Voice sends. The Choice loaders now repair and
verify their keys (`rules/voice_choice_safety.py`), and a contact watch is now
identified by its **phone number**, not the name it was armed under.

**The general lesson: a person's `aliases` are not free text.** They are
published into voice grammars, so anything that writes one is writing something
Dragon has to be able to say.

**A merge suggestion cost a destination.** The report's duplicate finder pairs
records whose names share a prefix, and it proposed folding **`Rigo and Ten`**
into **`Rigo`**. They are not the same thing: the first is the couple's own
joint Google Voice thread with its own number (`+18669854321`). The merge kept
that number as a second phone on Rigo's record — `_merge_contact_fields` did its
job — but `build_contacts` emits an alias with the record's PRIMARY number, so
`text Rigo and Ten` quietly started opening a one-on-one chat with Rigo.

Nothing errored, and it was the only name affected: a diff of the regenerated
sidecar against the pre-consolidation copy showed **one** number changed out of
95. That diff is now the standard check after any store surgery.

`Rigo and Ten` is a record again, with its own number, `match_patterns: []` (two
people are not a person to auto-link), and Rigo is back to one number and no
aliases. The suggester now **refuses to propose two records that hold different
numbers** — two numbers are two destinations, whatever the names look like.

**The rule this leaves behind: an alias is a second way to SAY a name, never a
second place to send.** `Charlie` for `Charli` is an alias. `Rigo and Ten` is
not.

## What the import did

169 people, 87 with a phone, 57 with Google Voice history. Nothing lost:
`gv_contacts` went 77 → 94 keys and `gv_people` 69 → 74 (aliases now contribute
keys), with **no name unresolvable and no number changed**. Journal links: 74 of
74 still resolve. Catalog recommenders unchanged.

She asked to import everything and check afterwards, so `people_sync.py report`
is the afterwards — it surfaces the three things a bulk import gets wrong:

- **probably not people** — found `Florida Blue`, who turned out to be a person
  after all: her insurance consultant, **Sandy**. Renamed, with `Florida Blue`
  kept as an alias so the contact she saved under that name still resolves. A
  good illustration of why the import offers rather than decides.
- **probably the same person twice** — 10 pairs (`Cedar` / `Cedar <3`,
  `Ally` / `Ally Lehman`, `Otter` / `Otter <3`, …). Reported, never merged: a
  wrong merge silently rewrites journal history. All ten were confirmed and
  folded, the longer name into the shorter one (the short id is what the
  journal's 74 links already point at). `Rigo and Ten` is a couple rather than a
  duplicate, and folding it in gave Rigo a second number — the joint line —
  which is right, and which is what exposed the `merge` bug above.
- **58 records with nothing but a name**

After the merges and the rename: **159 people, and the report is clean.**

## Commands

```
py Scripts\MediaCatalog\people_sync.py report        what it holds, what looks odd
py Scripts\MediaCatalog\people_sync.py sidecars      regenerate the two views
py Scripts\MediaCatalog\people_sync.py import-gv     fold the sidecars in (idempotent)
clog person-merge <src> <dst>                        fold two records together
```

## Files

| Layer | File |
|---|---|
| The store | `Scripts\MediaCatalog\people.py` |
| Views + import + review | `Scripts\MediaCatalog\people_sync.py` |
| Grammar filter | `clog.py` `_recommender_ids` / `_sync_people_choices` |
| Write-through | `Scripts\VoiceConfigManager\add_voice_choice.py` |
| Tests | `Scripts\codebase_tools\tests\test_people_store.py` |

## Letters — the first thing the store paid for

**`journal letter <person>`** opens an empty writing box titled *"Journal entry
— Letter to Cedar"*, and saves as an ordinary journal entry.

**No salutation.** Every other source gets a header because it carries something
she would otherwise have to go and fetch — a Kindle location, a video's url. A
letter's only context is who it is to, and that is already in the title bar; a
prefilled "Dear Cedar," is furniture to delete, in a register she does not write
in. (It was there for one build. It is not the 1800s.) A letter is a `source` of kind `letter`
carrying a `person_id` — so the numbering, the "By source" browser, the export
presets and the whole question loop work on letters the day the kind exists.
Same argument that made a prompt a quote rather than a fifth store.

**A letter's series is the PERSON**: `source_key` checks `person_id` first, so
"Letter to Cedar #3" counts what she has written *to Cedar*, and the title stem
is kind-aware (`Letter to Cedar #3: `, not `Cedar #3: ` — that would read like
the third thing Cedar said).

**A field missing from the arg map is dropped in silence.** `_JSourceArgs` in
JournalCapture translates a resolver's Map into `--source-*` flags, and
`person_id` was not in it when letters landed — so a letter would have saved
looking perfectly fine with no link to the person it was to, which is the only
thing that makes it a letter. Caught by probing what the resolver actually hands
the box rather than by opening the box and looking at it.

The `<letter_person>` Choice is `letter_people.json`, regenerated by
`people_sync.py sidecars` under **the same eligibility rule as `read <person>`**
— everyone except a record whose only role is `contact`, so the 77 imported
numbers stay out. 90 names. It is a third Choice in the journal rule and safe
because it sits behind the literal "letter": `journal letter <person>` cannot be
confused with `journal <book>` the way two Choices in one spec position would be.

## The hub — `open people`

One page per human. Ordered by **Google Voice message count**, not alphabetically:
159 people sorted A-Z buries Cedar under Aunt Linda.

```
Everyone              most-talked-to first
With a number         the ones you can text
Recommended you things
You write to          letters
+ New person
⚙ Store report

  Cedar                          +19412109550 · 75 msg
  Finnelius Mckellius            3525198898   · 53 msg
  Jamila Roth                                   51 msg
```

A person's page:

```
👤 Cedar
   also called   Cedar <3
   phone         +19412109550        Enter copies it
   Google Voice  75 messages, last Wed, 12 Aug …
   role          contact
✎  Edit                              add or change a detail
📓 Journal entries (36)              entries that mention them
✉  Letters                           nothing yet — Enter to write one
📚 Recommended you                   what they put you onto
```

**Journal entries ABOUT them and letters TO them are two rows, not one.** They
are different relations, and collapsing them would make a count mean nothing.

**One shell-out per page** (`people_sync.py card`) — a level that fetched a
phone, then an address, then the activities would be four Python starts on a
menu she is waiting through. Same reason journal.py's viewer emits its excerpt
inline.

`PeopleViewer.ahk` deliberately does **not** include JournalCapture or
JournalSources: they drag the whole Chrome/UIA/Kindle stack in, and doing so left
28 unresolved cross-file references, 23 of them dialog-class. Writing a letter
dispatches through `_JournalMainfun` instead — the rule JournalMenu's own leaves
already follow.

## Google Voice `journal this`

An open thread resolves to a person **by number first**. The heading gives a
display name, but a name is what drifts — "Cedar" / "Cedar <3" / "cedar_3" were
three records until the merge. The phone number is the identity, and
`find_by_phone` normalises formatting on both sides.

Both detectors already existed for "number add" (`_GV_AutoDetect*FromChrome`);
this is a third caller, not a second implementation. The entry is a `conversation`
source, numbered per person ("Talking with Cedar #3"), with the **tail** of the
thread captured into the box — the tail because a Google Voice thread runs for
years and what she is writing about is what was just said.
`journal.conversation_capture_chars` (1500) caps it; 0 records who it was with
and captures nothing.

## The merge hazard the hub exposed

`contact` is **derived** from having a phone (`roles_of`), so storing it only
ever marks a record as *created by the import* — which is what keeps it out of
`read <person>`. Merging an imported duplicate into an established person
therefore stamped that person as import-only and would have quietly dropped them
from the grammar; Jacob and Otter were saved only by happening to have a
recommendation. `_merge_contact_fields` no longer carries `contact`, and the
records that absorbed a merge were un-stamped.

Two of the ten merges also went the wrong way round — `Jacob Fiala` and
`Otter <3` were the *original* records and the short names were the imported
ones, so the surviving id is the import's. No damage: clog relinked the catalog
items, and all 74 journal people-links still resolve.

## Next

- Addresses: nobody has one yet. The `spit` slots still hold them
  (`paul_address`, `wild_address`, `ally_and_jam`) and were left alone; moving
  them onto records is a paste-per-person job, and the duplicate slot names
  (`address_paul` / `paul_address` / `paul`) could be tidied at the same time.
- `SearchProviders.ahk` carries three shadowed locals (`fnName`, `fnRef`,
  `type`). Not from this work — the file is untracked — but they entered the
  warning baseline when the new viewer's closure surfaced them.
