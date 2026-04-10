# Atlantic Logistics — Project Notes

## Client Context

- **Company:** Atlantic Logistics LLC — 3PL / freight broker, Jacksonville FL
- **Founded:** 2001, woman-owned. CEO: Robert Hooper Jr., Ph.D. (Rob, Jamie's dad)
- **Scale:** ~121 employees, 8,500+ carrier partners, ~$47M freight value/year (2024 Inc. 5000)
- **HQ:** 3003 Claire Ln, Jacksonville FL; second branch in Keystone Heights, FL
- **Website:** shipatlantic.com

### Key Contacts
| Name | Role | Email | Phone |
|------|------|-------|-------|
| Robert Hooper Jr. | CEO | rob@shipatlantic.com | 904.886.1110 / 904.477.4762 |
| Alex Rodriguez | (CC'd on AI project) | alex@shipatlantic.com | — |
| Nate Hooper | (intermediary) | natehoop@gmail.com | — |

---

## Tech Stack

### Confirmed
| System | Purpose | Notes |
|--------|---------|-------|
| McLeod PowerBroker | Primary TMS | Deployed 2010. Has REST API (availability depends on license/IT) |
| McLeod IQ | Analytics/reporting | Added 2018. Data lives here for custom reports |
| Microsoft 365 | Email, docs | Outlook, Word, Excel confirmed. Teams/SharePoint likely |
| Load Pay | Carrier payments | Added 2018 |
| Project44 | Shipment tracking | Partnership 2023. Has documented REST API |
| Paylocity | HR/payroll | Confirmed via hiring portal |

### Key Unknowns (clarify before scoping new work)
- What M365 license tier? (determines Power Automate and Purview access)
- Is McLeod API accessible? What version — PowerBroker or LoadMaster?
- Is there an IT department or self-managed?
- Do they use SharePoint / Teams internally?

---

## Completed

### Email Cleanup Flow
Power Automate (cloud) + Excel config on OneDrive. Two flows:
- **Flow 1** — "Email Cleanup: Apply Rules" — runs on schedule, moves emails matching rules to folders or staging
- **Flow 2** — "Email Cleanup: Hard Delete Staged" — runs daily, processes staging folder

Config is entirely Excel-driven — client only touches the spreadsheet to add/change rules.
Safety features: DryRunMode, NeverHardDelete, staging folder, run log.
Files: `Email Cleanup Flow/` folder — build guide, setup guide, config generator script.

---

## In Progress

### AI Load Email to McLeod
**Status:** Scoped, not started. Waiting on McLeod API access from client.

**What it does:** Dedicated email inbox receives forwarded customer load emails → LLM parses them → posts loads to McLeod as Subject Orders → keeps orders current as new emails arrive.

**Phases:**
1. MVP — single customer (Optimus Steel pilot), Gmail API polling, Claude extracts load data, manual review before posting
2. Full automation — multi-customer profiles, automatic list diffing, auto-post high-confidence, vision API for PDFs
3. Pricing integration — call pricing API before posting, only post if margin threshold met

**Critical dependency:** McLeod API access + docs + sandbox. Do not commit to timeline until confirmed.

**Key technical challenges:**
- Idempotency / list diffing — is this a replacement list or a single additive load?
- Email format variability — Optimus Steel is a clean HTML table; others may be PDFs, Word docs, free text
- Error handling — misread rate or wrong origin could cause real business problems; confidence threshold + human review queue recommended

**Questions still open for Rob:**
1. McLeod API access + docs + sandbox environment?
2. McLeod version (PowerBroker? LoadMaster?)
3. How many customer distros initially?
4. Are all emails as clean as Optimus Steel?
5. Who posts loads manually today? (auth/user context)
6. Is human review acceptable or must it be fully automated?
7. Expected email volume per day?
8. Pricing API — when available, build for it now or add later?

---

## Ideas / Considering

<!-- dump things here — no structure required -->


---

## Reference

### MCP (Model Context Protocol)
A standardized way to connect external systems to Claude so it can read data and take actions directly in a conversation — without copy-pasting. Already in use: Gmail, Google Calendar, Playwright browser control are all MCPs.

**Why it's relevant here:**
- A custom McLeod MCP would let Claude query load data, customer profiles, or order status directly in a conversation — no manual data export required
- An M365/Graph API MCP could let Claude read Outlook or SharePoint data the same way
- Builds naturally on top of the existing AI load email project — same API access, more flexible interface

Building one is a Python project (weekend-scale for something useful). Requires API access on the McLeod/M365 side first.

---

### M365 Automation Options (quick reference)

| Need | Best tool | Notes |
|------|-----------|-------|
| Move/delete on arrival | Outlook Rules (built-in) | No time delay |
| Time-delayed delete/move | Power Automate recurrence flow | Requires PA license (usually included in M365) |
| Bulk sender suppression | Sweep (built-in) | Sender-only, no subject/keyword matching |
| Signatures by context | Built-in signatures | Already works natively |
| Static reusable snippets | My Templates add-in or AppSource | Quick Parts not available in new Outlook |
| Context-aware compose | Office JS add-in | Best path; requires JS/TS + hosted manifest |
| Post-send logging | Power Automate | Works well for this |
| Complex logic / bulk data | Python | PA for glue, Python for real logic |

**Power Automate note:** Can be HTTP-triggered from Python (`requests.post()`), making it callable from any script or AHK. Use PA as the M365 integration layer, Python as the logic layer.

---

### Project44
Shipment tracking/visibility platform (partnership 2023). Has a well-documented REST API — potential source for real-time tracking data if that becomes relevant.
