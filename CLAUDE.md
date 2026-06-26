This is me testing that these changes get to GitHub! Fingers crossed ~ Dee Richards

# Kadince Import Playbook

You are helping prepare customer data for a Kadince import. Kadince is a CRA / community-engagement platform used by banks and credit unions. Customers typically deliver data in 2+ files from disparate sources (org master + donations log + sometimes hours, loans, investments). This playbook captures the workflow, gotchas, and user preferences from prior imports — read it before touching files.

> **⚠️ DO NOT TRIM OR SPLIT THIS FILE TO SILENCE THE "over the 40.0k-char limit" WARNING.** That startup warning is informational only — Claude Code loads CLAUDE.md **in full regardless of length** (verified against the official memory docs: "CLAUDE.md files are loaded in full regardless of length"). Nothing past 40k is truncated or dropped; the warning only signals higher context use / slightly weaker adherence. The user has explicitly decided to keep the ENTIRE playbook loaded every session — completeness over brevity is intentional. Splitting into `@import` files does **not** help: imported files also load in full at launch ("helps organization but does not reduce context"), so it changes nothing except adding moving parts. The only thing that reduces context is deleting content — which is **not** wanted here. Leave the file as one piece. If adherence ever genuinely degrades, the sanctioned move is path-scoped `.claude/rules/` (load per-file-pattern) — confirm with the user first; never silently trim.

> **🔒 GOLDEN RULE — NEVER ASSUME ANYTHING ABOUT THE CUSTOMER'S DATA. ASK.** This is sensitive financial-institution data. A wrong interpretation (a misclassified loan, a guessed CD-eligibility flag, an invented org, a mis-mapped field, an assumed date format, an unconfirmed dedupe/merge) can produce **CRA/regulatory exam findings, penalties, and fees** for the customer. Therefore:
> 1. **If a decision about the data is not EXPLICITLY covered by this playbook, STOP and ask the user clarifying questions before acting. Do NOT infer, default, guess, or "do something reasonable."** Surfacing an unknown is always cheaper than a wrong write to customer data.
> 2. This **overrides the "Be decisive" preference in Section 3.** Be decisive ONLY about mechanical/process steps this playbook already spells out (backups, file shaping, running a documented phase). For anything touching the **meaning, classification, eligibility, transformation, dedupe/merge, or field-mapping of the customer's actual data** that isn't explicitly written here, **ask first.**
> 3. **When in doubt about whether something is "covered," treat it as NOT covered and ask.** A playbook rule counts as explicit only if it directly addresses this exact situation — a loosely related rule is not permission to extrapolate.
> 4. **Fold every answer back into this playbook.** When you ask a clarifying question and the user answers, capture that answer here (as a new rule or a refinement of the relevant phase) so the decision is preserved for every future session — then proceed. Tie it to the existing section it relates to.

---

## 0. Before doing anything

1. **List the folder.** Identify input files (org master, donations log, hours, loans, etc.) and the Kadince template (usually in a `Data Tools Context/` subfolder or similar).
2. **Inspect the Kadince template FIRST.** The template `.xlsx` defines:
   - tab names (`USERS`, `ORGANIZATIONS`, `DONATIONS`, `HOURS`, `INVESTMENTS`, `LOANS`, plus `*-Contacts`, `*-Volunteer Invol`, `*-Payment Allocations`, `*-CRA Allocations`, `*-Book Values` sub-record tabs)
   - column order and exact column names
   - which columns are required vs recommended vs optional
   - which columns are "(formulated helper column)" — leave empty, Excel formulas compute them
   - **⚠️ A customer's own Kadince data extract OVERRIDES this template for mapping.** When the user provides Kadince extracts/exports for the customer (e.g. `Loans export.csv`, `Organizations export.csv`), THAT extract is the authoritative source for field mapping, exact column/header names, controlled-vocab values, and tenant-specific field renames — **NOT** the generic `EXAMPLE ... Template Spreadsheet.xlsx` in `_Context/`. The example template is only the fallback when no extract is provided. So: map every source column to the header **as it appears in the customer's extract** (their tenant display names), and reconcile controlled values to what the extract actually contains. Rationale: the extract reflects the customer's **live tenant schema** (renamed fields, their own vocab), whereas the example template is a generic baseline that can mismatch a configured tenant. (Ties into Phase K's tenant-display-name renaming layer and Phase I reconciliation — when an extract exists, you're always mapping/reconciling to it.)
3. **Note the customer type.** Banks need CRA fields; **credit unions don't adhere to CRA**, so `CRA Eligible`, `CRA Qualifying Rationale`, `Development Activities`, `Qualitative Factors` are optional.
4. **Backup every file before mutating.** Use numbered suffixes (`Organizations.backup.xlsx`, `Organizations.backup2.xlsx`, …). The user expects you to do this without asking.
5. **Read the status/handoff doc FIRST.** Look for a `<Customer>_Import_Status.md` (companion to the briefing). If it exists, read it before anything else — it's how you resume across sessions (the chat history does NOT persist; this doc does). If it doesn't exist, create one as soon as you make meaningful progress (see Section 2B).

---

## 1. Workflow phases

### Phase A — Enrichment (org data)

Common asks: Web Address, Tax ID/EIN, Phone, address.

- **EIN lookup (free):** ProPublica Nonprofits API — `https://projects.propublica.org/nonprofits/api/v2/organizations/{EIN}.json`. Returns name, address, financials, NTEE code. **Does NOT return websites.**
- **EIN BACKFILL (name → EIN, free) — the reverse of the lookup above, for orgs MISSING an EIN.** Match orgs against the **IRS Exempt Organizations Business Master File (EO BMF)** — bulk CSV by region from `https://www.irs.gov/pub/irs-soi/eo<N>.csv` (region 1 = Northeast, 2 = Mid-Atlantic + Great Lakes incl. WI, 3 = Gulf/Pacific, 4 = all other; plus `eo_pr`/`eo_xx`). Download the region, **filter to the customer's state**, then match on normalized name (+ city to disambiguate). **Gate by match QUALITY, not a score — a wrong EIN is worse than a blank one (CRA stakes):**
  - **Exact normalized-name (+ city) match → apply-ready.** Only exact matches auto-apply.
  - **Fuzzy / multiple-candidate → REVIEW list for the customer.** Never auto-apply — look-alikes are common ("American Cancer Society" ≠ "American Harp Society"; "Kettle Moraine YMCA" ≠ "Kettle Moraine ATV Association").
  - **No match is EXPECTED and fine** for schools, government departments, LLCs, and one-off events — they aren't registered 501(c)s, so they legitimately have no EIN. Don't force a match.
  - **Also RE-CHECK existing EINs** against the BMF — catches junk placeholders (`000000000`) and umbrella/parent mismatches (org is on file under a parent/DBA, e.g. "<X> Family – Kettle Moraine YMCA" → "YMCA Kettle Moraine"). Surface, don't auto-fix (umbrella-EIN problem, see Phase I).
  - Match on name + state; the BMF stores EINs dash-free (9-char, leading zeros preserved) — format to the customer's chosen style (see Tax ID/EIN format note in §2). (Kohler CU 2026-06-18: 61/211 missing EINs filled from exact IRS matches; 27 fuzzy held for review; re-check caught a `000000000` placeholder.)
- **Web Address lookup:** Web search per org is the only reliable path. For N ≥ 100 orgs, fan out to parallel general-purpose subagents (10 agents × ~25 orgs each). Each agent does `"<name>" <city> <state> nonprofit official website`, picks the org's own domain over directory listings (idealist, causeiq, guidestar, propublica, charitynavigator) and social (Facebook fallback OK if no own domain).
- **Subagent /tmp gotcha:** subagents are sandbox-blocked from writing `/tmp`. Tell them to return the JSON inline as the final message; you write the file in the main loop. Don't waste agent tokens trying to fix the write.

### Phase B — Cross-file reconciliation (donations refer to orgs)

Find donation `Organization Name` values not present in the org master. They fall into four buckets:

1. **Casing/punct variants of existing orgs** (e.g. `"City of X"` vs `"City Of X"`, `"4-H, Y County"` vs `"4H, Y County"`) — pick the cleaner canonical and rewrite both files to use it.
2. **Generic donation categories** — patterns like `"YYYY Scholarship Recipient"`, `"Education Sponsorship"`, `"Sport Sponsorship"`, `"Student Support"`, `"<Program Name>"` (when actually a sponsorship bucket). These are NOT real orgs. Consolidate under customer-prefixed grouping orgs:
   - `<Customer> Scholarships` for scholarship-type
   - `<Customer> Sponsorships` for individual/event sponsorships
3. **Person names used as the org** — sponsorship recipients (e.g. `"Jane Smith"` as Organization Name with title `"Sponsorship - Basketball game"`). Same treatment as bucket 2: org → `<Customer> Sponsorships`, splice name into title (`"Sponsorship - Jane Smith - Basketball game"`).
4. **True missing orgs** — add as new rows (name-only initially; can enrich later if needed).

**Person-name detection — DON'T use pure regex.** A 2-word `[Cap][lower]+ [Cap][lower]+` heuristic catches many false positives (`Morris Elementary`, `Trinity Players`, `Salvation Army`, `Discovery Museum`). Use a curated first-names list + 2–3 word pattern + org-noun blacklist. Even then, expect 1–2 false positives — manually review.

### Phase C — Canonical name selection

When deduping variants, the existing canonical row in the org master is often the WORSE version (trailing whitespace, mid-word typos, weird casing like `"X resource Center"`). Score variants for cleanliness:

- penalize leading/trailing whitespace (100)
- penalize mid-name lowercase word starts (20 per word, except `of`/`for`/`and`/`the`/`in`/`on`/`at`/`to`/`a`/`an`/`or`/`but`/`by`/`with`)
- penalize mid-word case anomalies like `"LIfe"` (30)
- penalize `"Of"`/`"For"`/`"And"`/`"The"` capitalized mid-name (5)

Pick the lowest-scoring variant as canonical. Rewrite both files.

### Phase D — Title-based org extraction

For donations consolidated into generic buckets in Phase B, the **donation title often names the real recipient org**. Build regex extractors like:

```
"State History Day"       → "<State> State History Day"
"Kinetic Sculpture|Race"  → "<Region> Kinetic Sculpture Race"
"Triple Crown.*Cheer"     → "Triple Crown Dynasty Cheer"
"AAU.*Basketball"         → "AAU Youth Basketball"
"FFA"                     → "FFA"
"<School>.*Senior Night"  → "<School>"
```

Apply across donations currently in: `<Customer> Sponsorships`, generic categories, anything else clearly mis-classified. If a specific org is extractable from the title, route there and add to org master. Otherwise leave grouped under `<Customer> Sponsorships` / `<Customer> Scholarships`.

### Phase E — Organization Type classification

Add an `Organization Type` column. Use the following type taxonomy (controlled — Kadince may have its own; verify with customer or Kadince support):

| Type | When to use |
|---|---|
| Non-profit | Foundation, Society, Association, Coalition, Trust, community/youth/family orgs |
| Sports/Recreation | Little League, Booster-less athletic clubs, Cheer, Martial arts, AAU, Race events |
| School | K-12, Elementary, High School, Middle School, Academy, Christian School |
| School Booster/PTO | Booster Club, PTSA, PTO, PTA, Safe & Sober, school-specific athletic program |
| School District | School District, USD, Office of Education, HCOE |
| Higher Education | University, College, Community College |
| Fire/EMS | Volunteer Fire, Fire Protection District, Rescue, CERT, EMS, SAR |
| Civic/Fraternal | Rotary, Kiwanis, Lions, Elks, Moose, Masons, Grange, VFW, American Legion, Scouts, Big Brothers Big Sisters |
| Arts/Culture | Theater, Playhouse, Opera, Symphony, Museum, Gallery, Festivals, Historical Society |
| Government | State/County/Federal departments, Treasurer, Office of Education, Fairgrounds |
| Municipal | City of *, Town of *, Community Services District (CSD), Public Utility District (PUD) |
| Religious | Church, Parish, Congregation, Fellowship (religious), Ministry |
| Healthcare | Hospital, Clinic, Hospice, Adult Day Health, Mental Health |
| Animal Welfare | Humane Society, Animal Rescue/Shelter, Wildlife, SPCA |
| Education | FFA, 4-H, Library, Literacy, Education Foundation |
| Chamber/Commerce | Chamber of Commerce, Realtors Association, Main Street, Economic Development |
| Media | Radio, TV, Broadcasting, Publishing, Communications |
| Tribal | Tribe, Tribal council, Indigenous orgs |
| Law Enforcement | Police, Sheriff |
| Business | LLC, Inc, Corp, Brewery, Casino, for-profit indicators |
| Individual | A person, not an org (use only if Phase B didn't catch it) |
| Internal | Customer-prefixed grouping orgs (`<Customer> Sponsorships`, `<Customer> Scholarships`) |
| Other | Truly unclassifiable — should be < 1% of rows |

**Process:** regex keyword pass first → produces ~5% "Other" → manual override map for stragglers. Don't try to be perfect with regex; the override list is cheaper.

### Phase F — Address enrichment on donations

For each donation row, copy the matching org's address into the donation's `Physical Address` columns (street, city, state, zip code). Match by exact `Organization Name`. Bare-name orgs (added in Phase B bucket 4) won't have an address — those donations will have blank address columns. That's OK; flag the count for the user.

**Don't put the address into `Mailing Address` on donations** — Kadince treats donation address as the event/recipient location, which goes in `Physical Address`.

### Phase G — Geographic validation (if donations have a county/region column)

If the donations file has a `County` (or similar) column, validate the org's city actually sits in that county. Build a city→county map for the relevant counties. Small unincorporated towns will be missing from your initial map — accept ~3–5 unknowns and patch them. Drop the validation column from the final deliverable (QA-only).

### Phase G2 — Derive `Donation Type` from title prefix

Kadince's `DONATIONS` tab has a `Donation Type` field (controlled vocab: `Sponsorship`, `Scholarship`, `Grant`, `Donation`, plus customer-specific additions). Source donation titles almost always lead with the type, making this trivial to derive — DON'T leave this column blank.

Standard pattern matches:

| Title pattern (case-insensitive) | Donation Type |
|---|---|
| starts with `Sponsorship` | `Sponsorship` |
| starts with `Scholarship` | `Scholarship` |
| contains `Grant` (often `"YYYY [Spring|Fall] Grant - ..."`) | `Grant` |
| no match | `Donation` (generic fallback) |

Insert `Donation Type` as a new column on the donations file (logical position: right after `Amount`). Report the per-type counts to the user. If `Donation` (fallback) > 0%, review those titles — they may need a custom rule.

**Watch for:** customers sometimes use `Sponsorship` titles for what are functionally individual scholarships or grants. Don't try to re-classify — the title prefix is what the source system used, and Kadince will accept it. Only intervene if the user explicitly wants reclassification.

### Phase H — Template alignment (deliverable shape)

| Source column | Kadince column |
|---|---|
| `Tax ID` | `Tax ID/EIN` (format as `NN-NNNNNNN` — 2 digits, dash, 7 digits) |
| `Website` | `Web Address` |
| `Type` (custom) | `Organization Type` |
| Single `Address`/`City`/`State`/`Zip` | Split into `Physical Address` + `Mailing Address` (8 cols total) |
| `Street`/`City`/`State`/`Zip` on donations | `Physical Address` / `Physical Address City` / `Physical Address State` / `Physical Address Zip code` |
| `County` on donations | `Impact Areas` with values like `"<County> County"` (Kadince auto-creates Impact Areas on import) |

**Address split logic on orgs:**
- Keep existing address as `Mailing Address`.
- If the address is NOT a PO Box (regex `^\s*p\.?\s*o\.?\s*box`), also copy it to `Physical Address` — a street address usually serves as both.
- If it IS a PO Box, leave `Physical Address` blank.

**Other prep (do automatically, don't ask):**
- **`Approval Status` → Title Case.** Convert lowercase values (`approved`, `declined`, `pending`, `withdrawn`) to Title Case. Kadince validates this and may reject lowercase rows. Don't ask first — just fix and report the count.
- **`Phone` → exactly 10 digits, else clear to blank.** Kadince validates phone as **exactly 10 digits** and rejects the **entire row** otherwise (`Phone: Phone numbers must have 10 numbers`). Strip to digits: if **11 digits with a leading `1`**, drop the `1` → 10. If it's **not 10** (truncated/partial like `(707) 578-77` = 8 digits, or has an extension), **clear to blank — do NOT pad or guess digits.** A blank phone imports fine; a malformed one fails the whole org. **Watch the silent-truncation trap:** source spreadsheets that store phones as *numbers* drop leading zeros AND trailing digits, so a column can look fine but have a cluster of 8–9-digit values. Validate every phone before delivery, report the count + list of cleared numbers (so the customer can re-enter the real ones in Kadince). (Confirmed Poppy 2026-06-12: 16/127 org phones were truncated to 8–9 digits → row rejections.) Same clear-to-blank principle as the Tax ID/EIN rule.
- Drop QA-only columns (`County Match`, etc.) before delivering.
- **🔑 The file you UPLOAD to Kadince MUST be a CSV (UTF-8), one object per file — NOT an `.xlsx`.** Kadince's Data Tools (Import / Update / Delete) **only accept CSV** (confirmed in the "Data Tools | Imports/Updates" help article: *"The data tools require a CSV file. Ensure the CSV file was exported using UTF-8 format."* and *"Excel and Numbers files need to be saved in a CSV UTF-8 format"*). The multi-tab `.xlsx` is a **working master only** — never the deliverable for upload. Export each object's tab to its own CSV (e.g. `<Customer>_1_Organizations.csv`, `<Customer>_2_Organization_Contacts.csv`), numbered in import order.
- **Header row position depends on the file's role — get this wrong and field-mapping silently breaks:**
  - **The upload CSV → headers MUST be on ROW 1, with NO blank row above them.** Kadince treats row 1 as the header row; if headers sit on row 2 (blank row 1), the importer doesn't recognize them and the **Field Mapping screen comes up blank** (help article: *"Field Mapping Blank: check that the CSV file does not have any blank rows above the column headers"* and *"if the column headers exist in any row beyond row one, they will not be recognized"*). So when exporting the working `.xlsx` → upload CSV, **drop the blank row 1** so headers land on row 1.
  - **The working `.xlsx` master may keep row 1 blank / headers on row 2** (the specialist's note row). That convention is ONLY for the internal xlsx and must NOT survive into the uploaded CSV.

### Phase I — Reconcile against an existing Kadince export (if customer is already in Kadince)

If the customer has an existing Kadince tenant and provides an `Organizations export.csv`, dedupe against it BEFORE delivering — otherwise the import will create duplicate org records.

**Kadince export schema (relevant columns):**
- `Organization Name` (exact spelling/casing as stored)
- `Tax ID/EIN` (formatted with dashes: `46-5092911`) and `Tax ID` (alternate format)
- `Organization Type` (controlled vocab — values look like `Non-Profit 501c3`, NOT my custom types)
- 49 columns total including phone, web address, addresses, CRA fields, geocoding fields

**Matching rules (CRITICAL — get this wrong and you'll mangle data):**

1. **Match by normalized name first** (case + punctuation stripped). If found, rename our org to Kadince's exact spelling. This is safe.
2. **EIN match alone is NOT safe.** If our org has the same EIN as a Kadince org but the names differ significantly, DO NOT auto-rename. This is the **umbrella-EIN problem**: booster clubs, school districts, and parent nonprofits register one EIN that covers multiple distinct sub-programs. Examples seen:
   - `Fortuna High Girls Wrestling` and `Fortuna High School Ladies Basketball Team` share an EIN (booster umbrella)
   - `Northern Humboldt Union High School District` and `Six Rivers High School` share an EIN (district covers schools)
   - `Community United of North America (CUNA)` and `Playhouse Arts` share an EIN (parent nonprofit)
3. **Safe EIN-match cases:** when the names are clearly the same org with minor variations (article/punct/casing), e.g. `The Trinity Players` ↔ `Trinity Players`, `Arcata Area Chamber Of Commerce` ↔ `Arcata Chamber of Commerce`. Surface these to the user for one-shot confirmation.

**Process:**
1. Normalize names + EINs in both files.
2. Build `rename_map` from name-normalized matches only.
3. Build `ein_only_review` list for EIN matches where the name doesn't match — surface to user for manual decision.
4. Apply renames to both Organizations file and Donations file (donation `Organization Name` references must also update).
5. Dedupe rows that became duplicates after rename (e.g., we had `City of Eureka` AND added `City Of Eureka` separately — after rename both become `City Of Eureka`).
6. Output match report CSV: `Kadince_match_report.csv` with columns `Our Org Name`, `Match Type` (`Name+EIN` / `Name only` / `EIN only (<EIN>)` / `No match`), `Kadince Name`.

**Why this matters:** Kadince's import deduper compares on EXACT name match. Any name difference (even casing or trailing space) creates a new duplicate org. After this phase, the count of orgs that exact-match Kadince should match the count from your reconciliation report.

### Phase I2 — Pre-upsert risk analysis (run when customer is upserting into an active Kadince tenant)

**Trigger:** customer says they're doing an **upsert** (not a fresh import). For matched orgs, every column in your file potentially overwrites the value in Kadince — including blank columns. If Kadince's upsert treats blanks as "set to null", every empty cell in your file wipes data that already exists.

This phase produces a per-cell preview of what would change so the customer can make informed cuts before importing. **Run this BEFORE writing the customer-facing summary (Phase J), since the upsert decisions affect what gets imported.**

**Method — categorize every cell on every name-matched org:**

| Category | Condition | Meaning |
|---|---|---|
| `NO_CHANGE` | ours == kadince's | Identical, no-op |
| `IMPROVEMENT` | ours has data, kadince blank | We're adding data Kadince was missing |
| `OVERWRITE` | both have data, values differ | Our value replaces Kadince's — risky if Kadince's was correct |
| `WIPE_RISK` | ours blank, kadince has data | Blank may erase Kadince's value depending on upsert null-handling |

Produce a summary table (per-field counts across all matched orgs) and a per-cell detail CSV (`Upsert_risk_report.csv` — columns: `Org Name`, `Field`, `Our Value`, `Kadince Value`, `Category`). Show samples to the user — especially OVERWRITE and WIPE_RISK rows.

**Common risk patterns to expect:**

| Pattern | What's happening | Mitigation |
|---|---|---|
| `Organization Type`: 100% OVERWRITE rate, ours = `Non-profit`, theirs = `Non-Profit 501c3` | Our taxonomy ≠ Kadince's controlled vocab — every match downgrades their better value to ours | **Strip the column from upsert entirely** unless you remap to Kadince's vocab first |
| Address formatting: `PO Box` vs `P.O. Box`, `Rd` vs `Road`, `Ste 26` vs `Suite 26` | Stylistic difference — neither is wrong | Leave Kadince's alone (don't include those addresses in upsert for matched orgs) |
| Zip `95524` vs `95524-0777` (ZIP+4 expansion) | Ours is more complete but Kadince may have entered 5-digit deliberately | Same — don't overwrite |
| `Web Address`: `https://example.org/` vs `https://example.org` (trailing slash, https, www variations) | Equivalent, no point overwriting | Same |
| Blank EIN where Kadince has one | Data Kadince already collected; we don't have it | **Fill our blanks from Kadince's data before upsert** OR confirm Kadince's null-handling first |
| Field appears with completely different content (e.g., Kadince's `Web Address` is `240 E St`) | Data entry error in Kadince | These are the rare cases where overwriting is correct — confirm case-by-case |

**Mitigation tooling — three approaches, pick based on customer's risk tolerance:**

1. **Safe-mode upsert file (recommended default):** for matched orgs, fill our blanks with Kadince's existing values (turns WIPE_RISK into NO_CHANGE) AND strip vocab-mismatch columns like `Organization Type`. For new orgs, send full row. This guarantees no Kadince data is lost regardless of how their upsert handles blanks.
2. **Strip only the worst column:** remove `Organization Type` (the always-overwrites field), send the rest as-is. Accept the address overwrites and wipe risk on other fields.
3. **Confirm Kadince's null-handling first:** ask Kadince support whether blank cells in upsert mean "leave existing" or "set to null". If "leave existing", the wipe risk goes away entirely and you only need to worry about OVERWRITE cells. **(CONFIRMED TD 2026-06-17: the import tool has an explicit "should blanks clear out data?" Yes/No prompt — null-handling is a per-import choice, not a mystery.** Set it **Yes ONLY for a deliberate field-clear** — a file whose only populated columns are the key + the column you intend to empty (verified it truly blanks the field). Keep it **No for every normal partial update**, or each blank cell wipes that field. To clear a field via import: include the column present-but-blank with the toggle ON, then confirm the result log echoes the field as blank — not just "Updated".)

**Default if no time to verify:** safe-mode upsert (option 1). Wipe risk is invisible and silent — you won't know data was lost until the customer notices in reports later.

**Output of this phase:**
- `Upsert_risk_report.csv` — per-cell detail of every OVERWRITE and WIPE_RISK
- Summary table in chat/customer doc showing per-field counts by category
- A "safe upsert" file variant if the customer picks option 1

### Phase I3 — Merging duplicate records in an active tenant (associations-safe)

When reconciliation finds duplicates that already exist **in the tenant** (not just in your upload), you can't just delete the extra — deleting an org drops nothing of its own but **orphans its linked records** (hours, donations, loans). Merge in this exact order:

1. **Pull the association exports** for every object that links to the record (Hours, Donations, etc.). Each links by the **Kadince record ID**, not the tax ID/name — so the link survives any field change but is **lost on delete**.
2. **Pick the survivor by the SOURCE-OF-TRUTH key — not by volume or completeness.** The survivor is the record whose key fields match the customer's authoritative source (the ID/EIN their validation joins on) — that's the record their checks will find. Do NOT pick the one with more activity or more populated fields: activity moves in step 4, and a richer-but-wrong-keyed record still fails the customer's join. (TD 2026-06-17: survivor = the record whose Bonterra ID + Tax ID match TD's source files. When BOTH candidates match the source — i.e. the source itself has the duplicate — park it for the customer's source-side dedup rather than guessing.)
3. **Salvage before delete (field-migration).** Diff the loser against the survivor field-by-field; if the loser holds any value the survivor lacks, build an UPDATE to copy it onto the survivor FIRST. (Often the dupes are bare records with nothing to salvage — verify across the substantive fields, don't assume.)
4. **Re-point the associations** to the survivor: an UPDATE keyed on each association's OWN record ID (`Hour ID`, `Donation ID`), setting its organization to the survivor's ID. Verify against the result log (all Updated, 0 errors, each now points to the survivor).
5. **Only THEN delete** the emptied losers. Verify the delete result log (Success=Yes, 0 errors).

**Never reverse 4 and 5** — re-point first, confirm, then delete. **Field-clears vs deletes:** because associations key on the record ID (not the tax ID), *clearing a field* (e.g. a misfiled tax ID) never detaches associations — only *deleting the record* does. So field-clears are safe to run independently; deletes require the re-point first. (Confirmed TD 2026-06-17: 261 bare duplicates merged this way — 1,152 hours re-pointed, then deleted, 0 loss.)

### Phase I4 — Backfill Org-level fields from transactional records (existing-tenant cleanup)

A common existing-tenant defect: a field that **org-level reports read from** (`Counties Impacted` / Impact Areas, Impact Focus, etc.) is **blank on the Organization records** but **populated on the customer's transactional records** (Donations, Hours) — because the data arrived on a sponsorship/donation or volunteer **upload** and was never propagated to the Org object. Reports keyed on the Org field then come up empty even though the data exists.

- **Recover by joining on `Organization ID`:** collect the field's values from Donations/Hours per Organization ID and backfill the matching Org records. It's a COPY, not data entry. Preserve multi-value sets (one org can span several counties — keep the full comma-separated set).
- **Scope to upload-origin records when the customer wants "only what the upload got wrong" (not records they've maintained live since launch).** Signal in a Kadince extract: **`Form Submitted` BLANK = imported / upload-origin; a form name present = created through live usage** (Sponsorship/Donation Request Form, Admin-entered, Volunteer Hours Form, New Organizations Form). Apply BOTH filters — backfill only upload-origin source values onto upload-origin org records — so you never overwrite data the customer entered post-launch. (Kohler CU 2026-06-18: 91 upload-origin orgs backfilled; 85 live-usage orgs left untouched; 0 exceptions.)
- **Watch for non-value NOISE in the source field.** A county/Impact field often holds non-county tokens — region names (`Northeast`), catch-alls (`Out of State`), or a city standing in for its county (`Mequon` ⊂ Ozaukee Co.). **Flag these for the customer's decision (keep / map / drop); do NOT silently scrub or propagate.** (Golden Rule — it's the meaning of their data.)
- **Deliver as an UPDATE keyed on `Organization ID`** (see §2 + the file-shape note), and validate post-import that the filled count moved as expected (Phase K).

### Phase J — Customer-facing change summary (deliverable)

**Before writing this doc, confirm with the user:** "Are you importing the files as-is, or do you want the customer to review/answer open questions first?" The answer determines the framing:

- **As-is** (most common): doc is a **post-import reference** that lives alongside the customer's Kadince reports. Reads "here's what got imported and why" — retrospective, no asks.
- **Pre-import review:** doc is an **action-required worksheet** the customer responds to before final import. Reads "here's what we did, here's what we need from you."

Save as `Import_Summary_for_Customer.md` in the customer folder.

**Tone:** professional but friendly, plain English. Avoid jargon (no "normalized", "regex", "canonical"). The reader is the customer's operations contact — assume zero technical background.

**Structure (as-is framing — default):**

1. **TL;DR** — three bullets max: starting row counts, ending row counts, biggest interpretive decision.
2. **What we added** — enrichment work (websites, organization types, addresses, etc.) with counts.
3. **What we changed and why** — every transformation that altered the customer's data. For each: what the source looked like, what it looks like now, and the rationale. Group by file (Organizations / Donations).
4. **Decisions we made (and why)** — the calls that weren't mechanical (e.g., person names grouped under "<Customer> Sponsorships", title-derived org names invented, classification choices). State the call and rationale; mention the alternative briefly so the customer understands the tradeoff.
5. **Things to be aware of in your Kadince data** — items that aren't bugs but where the imported data may need review/cleanup in Kadince later: ambiguous EIN matches, invented org names, classifications worth double-checking, donations missing addresses. Frame as observations, NOT to-do items. The customer can choose to act on them in Kadince.
6. **What's in the imported files** — final file names + row counts + expected dedupe vs. new behavior.

**Style rules:**
- Use tables for any list of ≥4 items.
- Lead each "why" with the user-relevant reason, not the technical one. ("Donations to individual people were grouped under one org so your reports don't show 30 individual sponsorship recipients as separate orgs you donated to" — not "person-name org rows were consolidated under a customer-prefixed grouping").
- Cite real examples (org name, donation title) — abstract claims aren't convincing.
- Flag every place you invented data, but frame it as informational not action-required ("We named this org `Ruth Community Dinner` based on the title `'Sponsorship - Ruth Community Dinner'`. You can rename in Kadince if there's a better label.").
- DON'T list every backup file you created or every script you ran — the customer doesn't care about your process.
- **No "please confirm" / "action needed" / "reply with section number" language** in the as-is version. The customer is reading this AFTER import — they can't reply to action items, they can only research and adjust in Kadince.

This document is part of the deliverable. It's how the customer translates what they originally sent into what they're now seeing in Kadince — without it, anomalies look like bugs instead of explained decisions.

### Phase K — Post-import reconciliation

After the customer runs the import, do a row-count sanity check AND inspect the Kadince results files. This catches silent merges, validation rejections, and donation-link failures that wouldn't show up as "failures" at the import-summary level.

**Kadince delivers results files** (named `results_<timestamp>.csv` per tab) that include extra outcome columns appended to the data:
- Orgs tab: `Created/Updated`, `Error(s)`, `New Option(s) Created`
- Donations tab: `Success` (Yes/No), `Error(s)`, `New Option(s) Created`

**Column-header note:** the result file's headers reflect the customer's **tenant display names**, NOT the template names you uploaded with. Kadince's renaming layer maps `Amount` → `Amount Approved`, `Date Needed By` → `Date Funds Are Needed By`, `Organization Name` → `Organization Name (Organization)`, etc. The data is unchanged — only the labels differ. Don't get confused looking for `Amount` and finding `Amount Approved`; they're the same field.

**Retry-import shortcut:** when building a retry file for failed rows, **reuse the result file's column headers directly** instead of re-mapping back to template names. Kadince accepts the customer's tenant-renamed columns on import (since that's what their tenant uses), so the cleanest retry workflow is:

1. Open the result file, filter to rows where `Success` = `No` (donations) or `Error(s)` is non-empty (orgs).
2. Drop the three outcome columns at the end (`Success`/`Created/Updated`, `Error(s)`, `New Option(s) Created`).
3. Fix the broken values in place (reroute orgs, trim whitespace, fix vocab, etc.).
4. Save as `<File>_retry.csv` — column headers stay as the customer's tenant names.

This skips the re-mapping step. The retry file is now ready to upload as-is.

**The math:**

```
Expected Kadince row count = (rows in upload file) - (silent within-file merges)
```

A "silent within-file merge" happens when two rows in your upload share the same canonical key (org name post-rename). Kadince treats them as one. This is intentional but easy to forget — record any in-file dedupes during Phase I/I2 so you can subtract them here.

**Check process:**

1. Customer reports the import result counts (e.g., "946 successful, 0 failed").
2. Compare against your delivered row count:
   - **Match exactly** → import is clean.
   - **Off by N, you have a record of N silent merges from earlier phases** → reconciled, no action.
   - **Off by N, no record of merges** → investigate. Likely causes: orgs your rename pointed to the same Kadince canonical name (Phase I dedupes that happened on Kadince's side, not yours), validation rejections that the import treated as "imported with warnings" rather than "failed", or data Kadince silently coalesced (e.g., two rows with the same EIN merged on their end).
3. Spot-check specific edge cases:
   - **Renamed orgs (Phase I):** confirm each appears under its Kadince-canonical name, not as a duplicate.
   - **Umbrella-EIN pairs:** confirm they exist as separate orgs (or merged, whichever you decided).
   - **Invented orgs from title extraction (Phase D):** confirm they got created.
   - **Customer-prefixed grouping orgs** (`<Customer> Sponsorships`/`Scholarships`): confirm the donation counts under each match your pre-import expectations.

**Failure modes worth flagging immediately to the customer:**

- **Rows imported with empty `Tax ID/EIN` where you sent a value** → format issue Kadince silently dropped; check format compliance.
- **Org Type appearing as `Other` or blank when you sent a specific type** → Kadince's controlled vocab rejected your value (or auto-created a new option — check the `New Option(s) Created` column).
- **Address fields blank where you sent data** → likely a Kadince field-name mismatch (sent `Mailing Address` to a column Kadince expects under a different name).

**Donation `Error(s)` patterns and how to interpret:**

| Error text | What it means | Fix |
|---|---|---|
| `Organization: value(s) not found: Ensure short text identifiers are valid` | Donation's `Organization Name` doesn't exactly match any org in Kadince. Usually caused by: (a) trailing/leading whitespace in donation org name (e.g., `"Jane Smith "` with trailing space doesn't match org row `"Jane Smith"`), or (b) the org was supposed to be rerouted (Phase B/E individual-grouping) but the rerouting was missed for late-flagged individuals. | Trim whitespace on `Organization Name` in donations before delivery; for late-flagged individuals, re-run the rerouting on their donation rows |
| `Organization: duplicates found: Ensure short text identifiers are unique` | Kadince's master has two or more orgs with the same name. Donation linker can't pick one. | Customer must merge the duplicates in Kadince first, then re-upload the failed donations |
| `Value not allowed for this field` (or similar) | Controlled vocab rejection (Donation Type, Approval Status, etc.) | Remap value to Kadince's allowed list |

**Whitespace gotcha — always trim org names in donations file before delivery.** Trailing or leading whitespace on `Organization Name` in donations is invisible to the eye but causes Kadince's exact-match linker to fail. Source systems leak whitespace into export columns silently. A simple `.strip()` pass on the donations file's `Organization Name` column prevents this entirely. Add this as a standard step at the end of Phase H.

**Late-flagging gotcha (Phase E manual override):** if you mark an org as `Individual` AFTER the main consolidation pass (Phase B routes individuals' donations to `<Customer> Sponsorships`), you MUST also reroute their donations in the same step. Otherwise the donations reference an org that's still in the file but conceptually shouldn't be — and even if the org row imports, the donation can fail (or worse, succeeds but inflates your individual-count instead of grouping under the umbrella org). Cross-reference: any org flagged `Individual` should have ZERO donations referencing it directly.

**Pre-import master-dupe check (run before donation upload):** scan the Kadince export for duplicate `Organization Name` values. Any duplicates cause `duplicates found` errors on any donation referencing that name. Flag these to the customer before donation upload so they can merge in Kadince first.

**Document the reconciliation result** in the customer summary doc as a footnote or appendix: "Import result: X imported, Y failed, Z silent merges accounted for, final Kadince count: W."

**Validating an UPDATE by diffing post-import vs pre-import export (the high-confidence check):** re-export the object after the load and diff it field-by-field against the pre-import baseline, keyed on the Kadince ID. This proves what *actually* changed (not just what the result log claims). Three gotchas (Kohler CU 2026-06-18):
- **Diff multi-value fields as SETS, not strings.** Kadince does NOT preserve the order of multi-value fields (Impact Areas / `Counties Impacted`, etc.) across exports, so a plain string compare shows **false-positive "changes"** that are just reordering (e.g. `Milwaukee, Waukesha` → `Waukesha, Milwaukee`). Compare order-independent; only flag a change when the *set* differs.
- **Confirm blanks did NOT wipe.** For a combined UPDATE run with blanks-overwrite=NO, verify the orgs whose cells were blank kept their prior values (e.g. an EIN-only row's existing county/ZIP survived). This is the real proof the toggle behaved — the result log alone won't show a silent wipe.
- **Explain the row-count delta.** A post-import export can have MORE rows than baseline from **new live-usage records created between exports** (a fresh form submission), NOT from the import — confirm each "new" ID is genuinely new (and that the result log showed 0 created) rather than a duplicate your update accidentally created.
- **Every substantive change should match your source-derived value.** Join the changed rows back to the correction file/source and confirm they match; any DIFF is a review item.

This phase takes 2 minutes if it goes well. If it doesn't, it's the only chance to catch silent data loss before the customer starts trusting their Kadince reports.

### Phase L — Post-import reports in Kadince (deliverable for every import)

Once an import is completed and reconciled, build a Kadince report so the customer can see and verify what landed. **Do this for EVERY object imported** — one report per object (e.g., one for Organizations, one for Loans; same pattern for Donations / Hours / Investments).

**Report columns — at minimum, include:**
- **Every column the customer provided** in their source file (so they can tie the report back to what they sent), PLUS these core Kadince-derived columns:
  - **Assessment Area (title)** — the geocoded AA the entry landed in
  - **Tract Income Level** — the income classification of the entry's census tract
  - **Geocoding Confidence** — so they can spot low-confidence/Unassigned entries

**⚠️ Before building anything — confirm the account:** make sure you're in the **correct customer's Kadince instance** (the right subdomain, e.g. `poppy.kadince.com`). Building reports in the wrong tenant exposes one customer's data in another's account. **If you don't know which subdomain to use, ASK the user before proceeding.**

**Where & naming:**
- Reports go in a **`Data Imports` folder** in Kadince Reports (consistent home across customers).
- **Folder BEFORE reports (ordering rule).** Always **create the `Data Imports` folder first, then create the reports inside it.** (Create the folder via any report's **Settings → Folder Location → `+`**, or the Save As dialog's folder `+`.)
- **If reports predate the folder** (you built reports before the folder existed), go back and **move each into `Data Imports` afterward**: open the report → **EDIT → Settings (pencil) → Folder Location → select `Data Imports` → Submit**. ⚠️ **Verify by opening the `Data Imports` folder** — the saved-reports list renders **flat** (a report can still show at root even when filed), so opening the folder is the only real confirmation. Don't "Discard edits" mid-move (it can revert the folder change); if the move doesn't stick via automation, it's a quick manual drag/move for the user.
- **Title each report `Imported <Object> (M/D/YYYY)`** with the import date — e.g., `Imported Organizations (6/8/2026)`, `Imported Loans (6/8/2026)`.

**Ownership & sharing:**
- Set the **import contact (customer MPOC) as the report owner**.
- **Share with any other stakeholders** who need access (e.g., secondary contacts cc'd on the project).

This makes the import auditable on the customer's side and pairs with the Phase J summary (the summary explains *what we did*; these reports show *what's now in Kadince*).

**Building it in the UI — fast path (learned TD 2026-06-15, Poppy):**
- **First confirm the account + folder (see above):** right subdomain, and a `Data Imports` folder exists (create it if not — see "Creating the folder & moving reports" below).
- **If a similar report already exists, duplicate it; otherwise build from scratch.** Often the instance is brand-new with **nothing built yet** — in that case use **"New Report"** → set Report Object → Edit to add filter + columns. When a prior report for that object *does* exist (it often already has the geocoding columns + an import-date filter), the faster path is **"..." menu → Save As** → rename → Edit the copy. Don't assume a template is there — check first.
- **⚠️ In the Save As dialog, leave Folder Location at the default (`Saved Reports`) — do NOT pick the target folder there.** Selecting a folder via the dialog's dropdown "SELECT" corrupted a clone (Poppy 6/16): the report got misfiled to a phantom folder and then errored on load ("Uh Oh / Something went wrong"). Instead: Save As into the default location, confirm the clone opens cleanly, configure it, then move it into the `Data Imports` folder afterward via the report's **Settings → Folder Location** (or have the user drag it in). If a Save-As clone ever errors on load, abandon/delete it and re-clone without touching the folder.
- **"Imported on date X" filter:** field = **`Date/Time Imported`**; operator **`equals`** → **`Specific Date`** → `MM/DD/YYYY`. `equals` on a date matches the whole day. ⚠️ For the **Loans** object there are two `Date/Time Imported` fields — use the one under **"Other"** (the loan's own import date), NOT "Details > Organization". Other objects have just the plain one.
- **One report spanning multiple import dates:** add one `Date/Time Imported equals <date>` condition **per date**, then set the Filter Group to **"Match Any Filter"** (OR) via the group **"..." menu**. Default "Match All" returns **0** for two different equals-dates. Title variant: `Imported <Object> — M/D & M/D/YYYY (with geocoding)`.
- **Where the geocoding fields live in the picker:** **Assessment Area Title** → "Assessment Area (Geocoded)" group (can also be used as a Rows grouping); **Geocoding Confidence** → Location > Physical Address; **Tract Income Level** → Location > Tract Details.
- **⚠️ Column multi-select gotcha:** the field picker's type-ahead **reverts to the full list the instant you click a checkbox**, so a coordinate-based click frequently lands on the wrong row (this repeatedly mis-checked "Nonprofit Classification" / "Notes"). **Reliable method:** after typing to filter, get the checkbox's **element ref** (find tool) and click **by ref**. Then **verify against the column chips** in the ROWS/COLUMNS panel — the chips are ground truth, not the in-dropdown count.
- **Preview & save:** **REFRESH** to preview; if the report is grouped, the **Summary > Count** row shows the row count — verify it against the known import counts (Phase K). Then **SAVE** (or **SAVE & REFRESH**); look for the "Victory!" / "Well Done!" toast.
- **Environment:** a password-manager (or other) browser extension can pop up on form dialogs and steal focus, blocking automation ("Cannot access a chrome-extension:// URL"). Ask the user to pause it before a long form session.

**Creating the `Data Imports` folder & moving reports into it (learned 6/16):** there's no folder-manager in the saved-reports list — folders are managed through a report's **Settings**.
- **Create the folder:** open any report → **EDIT** → in the right panel find **SETTINGS** → click its **pencil** (Edit Settings) → in the dialog, next to **Folder Location** click the **`+`** → enter Name = `Data Imports`, leave Folder Location = `Saved Reports` (top level) → **Submit**. This creates the folder AND sets it as that report's location in one step.
- **Move an existing report in:** open the report → **EDIT** → SETTINGS pencil → **Folder Location** dropdown → pick the folder (type to filter, then click the option's **SELECT** button — a plain click just previews) → **Submit**.
- **⚠️ Unsaved-changes guard:** the Settings submit persists the folder change immediately, but the report editor still thinks it's dirty. Navigating away throws a browser "Leave site?" block, then an in-app **"Are you sure? Edits will not reflect…"** modal — click **DISCARD EDITS** (the folder move already saved; there are no real view edits to lose).
- **Verifying:** the saved-reports list renders **flat** (reports show at root regardless of folder), so don't rely on nesting in that view — the report's **Folder Location** field in Settings is the source of truth.

---

## 2. Critical Kadince template gotchas

- **🔑 When UPDATING any existing object in Kadince (orgs, donations, hours, loans, investments, users), match on the Kadince record ID — never on a name or a source-system ID.** The Kadince internal ID (e.g. `Organization ID` like `05016`, the value in the export's `Organization ID (Information)` column) is the **only field guaranteed to be a true unique key** for that record. Source-system IDs are NOT safe keys: they **repeat** across a customer's files (confirmed TD 2026-06-12: the same Bonterra Organization ID recurs across the 3 org-report files, and is duplicated within the tenant), and names collide / carry whitespace. Matching an update on name or a source ID risks writing to the **wrong record** — a regulatory-exam-grade error. So: pull the Kadince ID from the customer's export, key the update file on it (header `Organization ID` for orgs, the analogous `<Object> ID` for other tabs), and include only the columns you intend to change so nothing else is overwritten. (Default template behavior matches orgs on `Organization Name` — for an *update* this is the wrong key; override it to the Kadince ID. Smoke-test a handful of rows in the customer's test tenant to confirm Kadince matches on the ID before running the full file. **If there's no test tenant, run a tiny keyed UPDATE in PRODUCTION first** and confirm the result log — matched on ID, 0 created, blanks ignored — before the full file. Kohler CU 2026-06-18.) This is **update**, not upsert — keying on an existing Kadince ID also guarantees no new records are created.
- **UPDATE file shape — combine data fields in ONE file; split by APPROVAL GATE, not by field.** Because the import's *"Should cells with blank data overwrite existing data?"* prompt can be set **NO** (blanks ignored — see Phase I2), one UPDATE file can safely carry **multiple data columns**, blank where a given row doesn't change that field. Do NOT split one-field-per-file out of blank-fear — unnecessary. Split files only when rows are on **different approval/timeline tracks** (e.g. pre-approved enrichment now vs. a field awaiting customer sign-off). **Exception — clearing a field needs blanks-overwrite = YES**, so a deliberate field-clear gets its OWN key+field-only file run with the toggle ON; never fold a clear into a NO-toggle file or it silently no-ops. **Caveat:** the "never ship a mapped column blank" rule (Phase 2E) is about **LOOKUP** columns (Organization/Volunteer/Owner — they error on an empty cell), NOT plain data fields (county, EIN, ZIP), which are fine blank under blanks-overwrite=NO. (Kohler CU 2026-06-18.)
- **Headers: row 1 in the upload CSV; row 2 (blank row 1) only in the working `.xlsx`.** In the working multi-tab `.xlsx`, row 1 is the blue note row and headers sit on row 2. **But the uploaded CSV must have headers on ROW 1 with no blank row above** — a blank first row makes Kadince's Field Mapping come up blank. Drop the blank row 1 when exporting to the upload CSV. (See the fuller rule in Phase H.)
- **"(formulated helper column)" headers** = leave empty, Excel computes them. Examples: `Submitted By on Users Tab?`, `Organization Already in Kadince?`, `Our Representative Already in Kadince?`, `Address & Tract Codes Present?`.
- **Donations Email columns** (`Email (Submitted By)`, `Email (Donation Owner)`, `Email (Requested By)`, `Email (Paid By)`) reference rows on `USERS` tab. Without at least one populated user, the email columns are dead. Ask the customer for an admin email to use as default if real per-row attribution isn't available.
- **USERS import requires `First Name` + `Last Name` as columns to process — even for an UPDATE keyed on `User ID`.** You can't ship a minimal `User ID, Email` update file; the importer needs the name columns present. To avoid wiping/overwriting (Phase I2 WIPE_RISK), **populate First/Last Name from the customer's Kadince Users extract verbatim** (existing values → NO_CHANGE) so the only real change is the field you actually intend to update. **General rule (confirmed Brady, Metro CU 2026-06-16): if we don't have new/updated names, use what's already in Kadince — backfill required name columns from the extract, never leave them blank and never guess.** Final minimal shape for an email-fix update: `User ID, First Name, Last Name, Email`.
- **Custom "Import ID"** column on `DONATIONS` is ONLY used when also importing sub-records (Payment Allocations, CRA Allocations). Skip otherwise.
- **Date format:** Kadince accepts US format (MM/DD/YYYY). Confirm with customer if source uses ISO or other.
- **⚠️ Excel serial-number dates — the silent partial-failure trap (confirmed Poppy 2026-06-16, 18/160 rows).** When reading a source `.xlsx`/`.xlsm`, a date column is often **mixed**: most cells are true date-typed (openpyxl/pandas read them as `datetime` → your `MM/DD/YYYY` formatter works), but **some cells are stored as raw numbers** (the underlying Excel **serial**, e.g. `46049`) because that cell was never formatted as a date. Those come through as a **bare integer string** (`"46049"`), pass silently through staging, and then **Kadince rejects only those rows** with `<DateField>: must be a date; format the date the same way that it is exported or as: ...`. Because most rows are fine, the file looks clean until the import partially fails.
  - **Detect:** after staging, scan every date column for **digit-only values** (`str.isdigit()`). Any hit = an unconverted serial. (Serials for modern dates are 5 digits, ~`40000`–`50000`; 2026 ≈ `46000`+.)
  - **Fix (serial → date):** `datetime.datetime(1899,12,30) + datetime.timedelta(days=int(serial))`, then format `MM/DD/YYYY`. (Base **1899-12-30** already accounts for Excel's 1900 leap-year bug for any modern date.) Bake it into your date formatter so it handles BOTH datetime cells and bare-int serials: `if isinstance(v,str) and v.isdigit(): v = serial_to_date(v)`.
  - **Verify before delivery:** re-read the staged CSV and assert **zero** digit-only values remain in any date column. Standard step for every `.xlsx`/`.xlsm` source going forward — don't trust that openpyxl returned datetimes for the whole column.
- **⚠️ Leading-zero IDs/TINs — the Excel-strip trap.** Values with significant leading zeros (9-digit EINs like `050377867`, zero-padded FIPS/codes) silently lose the leading zero if the import CSV is opened and re-saved in Excel (it retypes them as numbers → 8 digits, silently corrupting them). **Import such CSVs directly; never round-trip them through Excel.** If the file must be opened, format the column as Text first, or hand off the CSV untouched. Verify a known leading-zero value survived before importing. (Confirmed TD 2026-06-17: 836 leading-zero TINs in a 6,136-row file — CSV-direct upload preserved all; an Excel save would have corrupted ~14%.)
- **⚠️ Rebuild batch files from the LATEST export when ANY manual edits happened in between.** A full-file update built from a stale export carries the *old* value for any record changed manually since — and silently **reverts those manual fixes** on import. Before running a full-file update, re-pull the export if manual changes occurred, or exclude the manually-touched records. (Confirmed TD 2026-06-17: a dash-reformat batch built from a pre-edit export reverted a manual tax-ID fix made after that export was pulled.)
- **Tax ID/EIN format — match the customer's SOURCE, not a fixed style.** The template formats `NN-NNNNNNN` (with dash), but some customers want **dash-free 9-digit** to match their source files / downstream programs (SAS, their extract). Confirm which the customer uses and standardize to it — Kadince accepts either, and Candid/ProPublica validation works with or without the dash. Whatever the chosen style, keep all 9 digits (see leading-zero trap above). (Confirmed TD 2026-06-17: standardized all tax IDs to dash-free 9-digit per the customer's source format.)
- **Import is one-object-per-CSV — there is NO multi-tab-workbook upload.** Kadince imports a single object at a time from a single CSV (Organizations, then Organization Contacts, then Donations, etc., in dependency order). Build the multi-tab `.xlsx` as your working master, then export **one UTF-8 CSV per object**, named clearly and numbered in import order: `<Customer>_1_Organizations.csv`, `<Customer>_2_Organization_Contacts.csv`, `<Customer>_3_Donations.csv`. (Import-order reminder from Kadince's doc: **Users → Organizations → Hours/Donations → sub-objects**; associated records must exist before the records that reference them.)
- **Any CSV must be saved as UTF-8.** Kadince requires CSV files in UTF-8. Non-UTF-8 (Excel's default `Windows-1252`/ANSI, or Mac Roman) silently garbles non-ASCII characters — accented org names (`José`, `Peña`, `ñ`, `ü`), `&`, em/en dashes (`—`/`–`), curly quotes — and can reject rows on import. This bites the **CSV paths specifically**: the Phase K retry file, split/separate-object CSVs, and the Catch URL payload (`.xlsx` masters are unaffected). When you write a CSV, set the encoding explicitly. **For the upload CSV use plain `encoding='utf-8'` (NO BOM) — do NOT use `utf-8-sig`:** the BOM prefixes the first header (e.g. `﻿Organization Name`) and can break Kadince's auto field-mapping of that first column. Only use `utf-8-sig` for a file you know the customer will reopen in Excel and NOT upload. When the customer exports their own CSV, point them to Kadince's "Save As / Export a CSV in UTF-8" guidance. Verify by spot-checking a known accented value after export.
- **CRA fields:** banks need them, credit unions skip. Confirm customer type at the start.
- **Allowed values for `Organization Type`, `Donation Type`, `Approval Status`, `Loan Type`, `Affiliate Code`, etc.** are controlled vocabularies in an active tenant. Verify against the customer's data before locking in. Don't ship custom values into an *existing* tenant and assume they import — they may get rejected or dumped into `Other`.
- **Existing tenant rows may be TEST data slated for purge — confirm before treating them as production.** When the tenant already contains records AND a Kadince extract is provided, do **NOT** assume those rows are real data to preserve, update-in-place, or dedupe-around. They are often **test/sample data the customer loaded while evaluating Kadince**, to be **purged at the end** once the real import is verified. In that case: use the extract as the **mapping/schema authority** (Section 0 step 2) and **validate that your staged data matches it where it should** (spelling, IDs, vocab), but treat **YOUR import as the source of truth** — don't reconcile-to-avoid-touching or key updates onto test rows. **Plan an explicit purge step at the end** to clear the test data after the real import is confirmed. **Always confirm with the user whether existing tenant rows are test or production before deciding** — guessing wrong either wipes real data (purging production) or leaves duplicated test+real records (treating test as production). (Metro CU 2026-06-12: validated rather than assumed — tenant data turned out **MIXED**, not a clean purge: Users = real `@metrocu.org` roster (keep), Orgs = mostly real Kadince-onboarding data with a few test entries (`DG's Test Org`, fake EINs), Donations = likely a Kadince demo set (Utah orgs on a MA credit union). Lesson: validate per-object before any purge.)
- **New-customer default for controlled vocabularies.** If the customer **did NOT provide an export from their Kadince account**, assume this is a **new dataset for a customer who hasn't imported or used Kadince yet** — nothing is in use, so there's no existing controlled list to match. In that case **assume the values you derive are being ADDED as new options on import** (Kadince auto-creates them), and proceed without trying to reconcile against a non-existent list. Only when a Kadince export IS provided do you reconcile values to the tenant's existing vocabulary (and watch for new-option creation). **When in doubt, ask the user to confirm whether they're a brand-new customer with no existing data in Kadince** — that single answer decides whether you match an existing list or create new values.

---

## 2A. Loan field defaults (apply to all loan imports unless the briefing overrides)

These are general defaults for the `LOANS` tab. Apply them automatically; only deviate if the customer's briefing explicitly calls out different behavior for that loan type.

- **`Affiliate Code` default.** If the customer has no Affiliate Code but DOES have `Action Taken`, default `Affiliate Code` to **`Loan action taken by the institution`** (the controlled-vocab value; the alternative is `Loan action taken by an affiliate`). Rationale: if the institution itself took the action, the affiliate code should indicate the institution, not an affiliate.
- **`Action Taken` default.** If `Action Taken` is blank, default to **`Originated`**. (Allowed values: `Originated`, `Purchased`.)
- **Deriving `Loan Type` from a Call Report Code.** Core-system loan exports usually lack a `Loan Type` but include the bank's **FFIEC Call Report Code** (Schedule RC-C) — the classification they already maintain for regulatory reporting. Derive the loan type from it rather than guessing. **Call these "loan types," NOT "CRA loan types"** — the Call Report code reflects collateral/purpose, which is a *kind* of loan, not a CRA determination (CRA's small-business/small-farm definitions also depend on dollar amount; see CD eligibility above). Standard prefix mapping (adjust per customer if their codes differ):

  | Call Report Code | RC-C meaning | Loan Type |
  |---|---|---|
  | `6*` (6B/6C/6D) | Loans to individuals | Consumer |
  | `4` | Commercial & industrial | Small Business |
  | `1C*` (1C/1C1/1CM/1CB/1CO) | 1–4 family residential | Home Mortgage |
  | `1A*` (1A1/1A2) | Construction & land development | Construction |
  | `3`, `1B` | Ag production / farmland | Small Farm |
  | `1E*` (1E1/1E2) | Nonfarm nonresidential | Commercial Real Estate |
  | `1D` | Multifamily (5+) | Multifamily |
  | `8`, `9`, `9C` | Municipal obligations / other | Other |

  These labels are a controlled vocabulary — for a brand-new tenant they're created as new options on import; for an existing tenant, reconcile to the customer's Loan Type vocab first (see Section 2 controlled-vocab note).
- **Know the bank's CRA size category before defaulting CD eligibility.** Ask the customer (or derive from assets) whether they're a **small**, **intermediate small (ISB)**, or **large** bank — it changes how home-mortgage loans can be treated for CD (see next bullet). Note: the asset thresholds adjust annually and the CRA framework is in regulatory flux (2023 final rule rescission/transition), so confirm the current category/rule with the bank's compliance team rather than assuming.
- **CD eligibility when the source file doesn't state it.** Do NOT blanket-assume `No`. Only set **`CD Eligible` = `No`** + **`Percent Eligible` = `0.00`** where the loan clearly isn't CD-reportable; otherwise **leave both BLANK** for the customer to review. Specifically, assume `No` ONLY for:
  - **Consumer** (any amount)
  - **Mortgage / HMDA (1–4 family)** — assume `No` **only for LARGE banks**; for **intermediate small banks leave BLANK for review** (see HMDA-as-CD note below)
  - **Small Business** with amount **< $1,000,000**
  - **Small Farm** with amount **< $500,000**

  Leave CD Eligible/Percent Eligible **blank** for everything else — **Commercial Real Estate, Construction, Multifamily, Other, and Small Business ≥ $1M / Small Farm ≥ $500K**. Rationale: the SB/SF dollar thresholds are the CRA small-business/small-farm reporting limits; below them a loan is reported as SB/SF (not CD), but larger commercial/CRE/etc. loans CAN be community-development-eligible, so don't presume — let the customer flag genuine CD loans during review. If amount is missing on an SB/SF row, leave blank (can't confirm it's under the threshold). (This refines the older "set No for all SB/SF/HMDA" default — the thresholds matter.)
- **Loans from a dedicated CD-loan file → default `CD Eligible` = `Yes`, `Percent Eligible` = `100.00`.** When the source is the customer's own curated **Community Development loan file** — i.e. every row is a loan they already determined is CD-qualifying, typically with a per-loan CD "Hook"/category and a written CRA justification — treat the CD determination as **made by the customer**: set `CD Eligible` = `Yes` and `Percent Eligible` = `100.00` (the whole loan counts, absent a stated pro-rata split). This is the inverse of the SB default (No / 0.00) and does NOT contradict the "leave blank rather than guess" rule above — here the customer has explicitly classified the loan as CD, so it isn't a guess. (Confirmed by Brady, Poppy CD files, 2026-06-12. If a CD file ever states a partial CD percentage per loan, use that instead of 100.) **⚠️ CD percent lives on a SEPARATE data layer, not the LOANS tab — and Kadince AUTO-creates it (Poppy 2026-06-12, confirmed w/ Brady).** The `Percent Eligible` column you upload on the LOANS tab has **no loan-column home** in the tenant and is silently dropped — but this is **NOT data loss**. A loan's CD eligibility **allocation** (Assessment Area, Amount Eligible, Percent Eligible, Year) is stored on the **CRA Allocations layer**, which exports separately as the **"CRA Details" export** (one row per loan; fields: `ID`, `Loan ID`, `Assessment Area`, `Amount Eligible`, `Percent Eligible`, `Year`, `Geocoded Allocation`). **Kadince auto-creates a 100%-to-the-geocoded-AA allocation when a loan geocodes** (`Geocoded Allocation=true`) — so a CD loan *with an address* auto-gets `Percent Eligible=100` with no manual allocation import. **So `CD Eligible=Yes` (loan tab) + auto-geocoded 100% allocation = the customer's intent is captured; you do NOT need to upload percent.** To VALIDATE: pull the CRA Details export and join by `Loan ID` — ⚠️ **format mismatch**: the Loans export `Loan ID` is unpadded (`1`,`2`,`988`) while CRA Details zero-pads to 5 digits (`00001`); strip leading zeros to match. Loans with **no address** don't geocode → allocation `Percent=0`/AA blank until an address is added (auto re-geocodes on edit). Only build a manual CRA Allocations import if a loan needs a **non-100% / multi-AA split** that geocoding won't produce.
- **HMDA / home-mortgage loans CAN be CD-eligible for intermediate small banks.** Per the Interagency Q&A: *"Except in connection with intermediate small institutions, a housing-related loan is not evaluated as a community development loan if it has been reported/collected as a home mortgage loan, unless it is a multifamily dwelling loan."* So:
  - **Intermediate small bank** → a 1–4 family home-mortgage loan MAY count as a CD loan if it meets the CD primary-purpose test (e.g. affordable housing for LMI). **Do NOT auto-set Mortgage `CD Eligible = No`; leave it blank for review.**
  - **Large bank** → a HMDA-reported home loan is NOT also a CD loan, **except multifamily** affordable-housing loans (count as both). So 1–4 family Mortgage → `No` is safe; Multifamily → leave blank (already the default).
  - **Small bank** (smallest tier) → CD activity is optional/rating-enhancing; the home-mortgage-as-CD flexibility is specifically an ISB feature, so treat like large for the Mortgage default unless the customer says otherwise.
  - **Always:** whether a specific mortgage actually qualifies as CD is a **compliance/primary-purpose determination the bank makes — not derivable from the data.** When unsure of the bank's size category, leave Mortgage CD blank rather than guessing `No`.
- **Organization Name vs. Borrower (people are not organizations).** On the `LOANS` tab, **`Organization Name` is for actual organizations only.** When the borrower is an **individual/person**, leave `Organization Name` **blank** and put the name in **`Borrower`** only. **Confirmed by customer (FSSB, 2026-06-08): a loan imports fine with `Borrower` filled in and NO `Organization` attached** — `Organization Name` is NOT required for individual-borrower loans. Do not invent organizations for people just to fill the field. When the borrower is an **org** (business, farm entity, trust, church, municipality, etc.), put the name in **BOTH** `Organization Name` and `Borrower` — **BUT ONLY IF that org already exists in the customer's Kadince org master.** ⚠️ **The LOANS import VALIDATES `Organization Name` against existing org records and does NOT auto-create them** — any loan whose `Organization Name` isn't already an org fails with **`Organization: value(s) not found: Ensure short text identifiers are valid`** (the whole row rejects). So before populating `Organization Name` on loans, **check the borrower against the Kadince Organizations export/master**; if it's not there (and you're not also importing it as an org), **leave `Organization Name` BLANK and keep the name in `Borrower` only** — the loan imports fine that way. This matters most for **commercial/CD loan borrowers** (LLCs, trusts, hotels, etc.) — they are loan customers, NOT the bank's CRA *community* organizations, so they're almost never in the community-org master → Borrower-only. (Confirmed Poppy 2026-06-12: staging set `Organization Name` = borrower for 99 boarded + 98 CD business borrowers; none existed as orgs → 99/125 boarded loans failed `value(s) not found`. Fix = clear `Organization Name`, Borrower-only.) Core-system exports often jam everyone into one "Short Name" field, frequently **fixed-width / truncated** (e.g. `LASTNAME` + first 3 chars of first name). Classify org-vs-person from positive business signals only — `&` (partnerships), business suffixes incl. truncated forms in the trailing field (`LLC`, `INC`, `COR`, `CO`, `LP`, `PC`, `GIN`, `PRO`→Properties, `MOT`→Motors), and full org-word tokens (`TRUCKING`, `CONSTR*`, `SERVICES`, `FARM`, `CHURCH`, `CITY`, `MOTORS`, etc.) — and **default to PERSON** otherwise. Watch greedy tokens against surnames (`CONST*` matches *Constantine* — use `CONSTR*`). Truncation caps accuracy; report the org/person counts + a sample of detected orgs for the customer to eyeball.
- **Repeated `Loan Number` vs. an existing Kadince tenant (upsert vs. new).** A loan number can legitimately recur across years (renewals / re-bookings). When the same `Loan Number` already exists in the Kadince export, decide by **overlapping year, not loan-number match alone**: if the existing record is a **different year** than the one you're importing → import yours as a **NEW** record (each year is its own CRA-reportable event). If it's the **same year** → it's likely the same loan event → **upsert/update** rather than create a duplicate. Surface the overlap set to the user and confirm the call.
- **`Affiliate Code` exact value.** The controlled value is `Loan action taken by the institution` (not `Action taken by the institution`). Before locking it in, check the customer's Kadince export — the field is often loosely controlled and may already hold casing/wording variants. Match the existing dominant value so the import doesn't create a redundant new option.
- **HMDA loan fields (added to the LOANS template 2026-06-16).** Kadince's HMDA module (home-mortgage / residential RE) adds ~51 fields to the `LOANS` tab — now present in `_Context/EXAMPLE ... Template Spreadsheet.xlsx` (LOANS tab, after the base loan columns). Mapping notes for HMDA-LAR source files:
  - **ULI is READ-ONLY / auto-generated — do NOT map it to `Loan Number` (corrected 2026-06-16 per Kadince functionality-bot).** Kadince builds ULI = company **HMDA LEI**[20 chars] + **HMDA Loan Identification Number** (middle) + check digit[2]. The import tool **excludes read-only fields**, so ULI never appears as an importable column, and `Loan Number` does **NOT** feed it. To get a ULI generated: (1) the company's **HMDA LEI** must be set at the **company level** — **20 chars, uppercase letters + numbers**; ⚠️ **there is NO field for it in the customer-facing company-settings UI**, so it must be set **behind the scenes by Kadince (API / admin backend tool)** — do NOT tell the customer to enter it in Settings. You need the customer's actual 20-char LEI value FROM them, then have Kadince set it internally; the generated ULI only matches the customer's official ULI if this LEI matches theirs. And (2) populate the **`Loan Identification Number/Application Record ID`** field on each loan — this is a **separate field from `Loan Number`**, and Kadince does **NOT** auto-copy Loan Number into it (confirmed High); it must be imported/entered explicitly. If the source gives only a **full ULI** (≈23–45 chars), strip the first 20 (LEI) and last 2 (check digit) and import the middle into Loan Identification Number; if the source already has just the loan-id number (e.g. an 8–10-digit value, common — it's a NULI/loan-id, not a full ULI), import it directly. The loan's regular `Loan Number` is a separate field — fine to also hold the bank's loan number, but it does nothing for ULI. **On a FRESH HMDA import, map the loan-id into BOTH `Loan Number` (display identifier) AND `Loan Identification Number/Application Record ID` (ULI driver).** Since the importer maps one source column → one target, **duplicate the value into two columns** in the staged file (one mapped to each) so it lands in both and ULI generates on import — no separate update needed. For loans **already imported** with only `Loan Number` populated, don't re-import (creates dupes) — **backfill `Loan Identification Number` via an UPDATE** keyed on Loan Number + Activity Year. **HMDA Income** auto-derives from the loan's `Income` field (returned in **thousands**); **HMDA Loan Amount** from `Amount` (whole dollars, rounded to nearest $1,000) — both read-only too. ⚠️ Poppy 6/16: the 265 2025-Purchased loans had the 8–10-digit loan-id mapped to `Loan Number` only → ULI never generated (field blank). Fix folded into the batch-update: populate `HMDA Loan Identification Number` + confirm company LEI is set.
  - **Demographics are SINGLE fields (`Ethnicity`, `Race`, `Sex`, `Age`), not per-borrower.** Map BOTH the applicant and co-applicant source columns to the same field — the importer routes them to the loan's Borrower vs Co-Borrower sub-records (which export as `Borrower - Sex` / `Co-Borrower - Sex`). HMDA codes 2–5, "collected on basis of visual observation," and the free-form "Other" companions go to the matching `Other …` fields or are left out.
  - **Geography:** leave `State/County Code` + `Census Tract` unmapped — Kadince geocodes the address and fills tract/MSA/county itself (confirmed Poppy). 
  - **Purchased RE loans carry no origination "processing" data** (Total Loan Costs / Points & Fees / Origination Charges blank) — that's expected, not a mapping miss; those populate on *originated* files.
  - **⚠️ Known importer bug (open as of 2026-06-16, Poppy):** `Application Date`, `Action Taken Date`, and Borrower/Co-Borrower `Ethnicity` & `Race` are **accepted on import but not persisted** (blank in the saved record) — `Sex`/`Age` persist fine. Plan: import now, then **batch-update** those fields once devs fix it, keyed on **Loan Number + Activity Year**. Re-check whether this is fixed before relying on those fields in a new HMDA import.

---

## 2B. Running status / handoff doc (`<Customer>_Import_Status.md`)

**The chat history does NOT persist between sessions; this doc is how work survives.** Memory files and `CLAUDE.md` persist, but they don't hold per-project state. Maintain a `<Customer>_Import_Status.md` in the workspace root (companion to the briefing) and **update it after every meaningful step** so a brand-new chat can resume without re-deriving everything.

Keep it short and current (overwrite stale entries, don't just append forever). Suggested sections:

- **Last updated** (date) + one-line current state.
- **File board** — table: each input file → target tab → status (`done` / `ready` / `blocked` / `don't-touch`) → note.
- **Done** — what's been imported/delivered, with dates and row counts.
- **Waiting on** — who owes what (customer + internal), where it was asked (email/Slack), and what it unblocks.
- **Decisions made** — the non-mechanical calls and their rationale (so they aren't relitigated).
- **Open questions** — still-unanswered items.
- **Next step** — the single most useful thing to do next.

This is an INTERNAL doc (like the briefing), not the customer-facing Phase J summary. Don't link the customer to it.

---

## 2C. Geocoding & tract-code formatting (LOANS / INVESTMENTS / any geocoded tab)

Geocoding is **per-entry, not per-file**. Each row geocodes independently from EITHER a physical address OR a full set of tract codes. A single import can freely mix rows that have tract-codes-only, address-only, both, or neither — **you do NOT need to split them into separate imports**. (Source: Kadince "Geocoding" help article.)

- **Both a COMPLETE address AND tract codes present on a row → use the ADDRESS; clear the tract-code columns and move the tracts into `Notes`.** (Customer-preferred handling, FSSB 2026-06-08.) A "complete address" = street + city + state + valid zip. When you have one, blank out `State Code` / `County Code` / `Tract Code` and append a note like `Source tract codes (address used for mapping): State 01, County 071, Tract 9502.00` so the customer still has the codes for comparison. Rationale: the street address is the more precise/current locator, and keeping both populated would let the tract silently override the address (Kadince's default when both are present). Moving the tract to Notes makes the choice explicit and reversible.
  - **Only when the address is incomplete** do you keep the tract codes in the geocoding columns (that's the path for the ~70%+ of core-system loan rows that ship with tract codes but no street address).
  - This is the inverse of Kadince's raw default ("when both present, tract codes override the address") — the customer would rather geocode from the real address and retain the tracts as reference.
- **Neither present → the row still imports cleanly**, it just lands in `Assessment Area = Unassigned` until someone later adds an address or tract details (entries auto-re-geocode on that edit). Missing geo does NOT fail the row — report the count to the customer as a follow-up, don't treat it as an error.
- **The only thing that causes a geocoding MISS is an invalid/improperly-formatted code.** So formatting the tract codes correctly is the high-leverage step — get it wrong and you silently break geocoding for every row that relies on tracts.

**Exact required formats (must match or the row won't geocode):**

| Field | Format | Examples |
|---|---|---|
| State Code | `##` 2-digit FIPS | `01`, `02`, `06` (AL=01; map from USPS alpha) |
| County Code | `###` 3-digit FIPS, zero-padded | `049`, `012`, `099` |
| Tract Code | `####.##` 4-digit, 2-decimal, **keep the decimal** | `9604.02`, `0110.01`, `9601.00`, `0012.01` |

**Tract formatting gotchas (seen in real core-system exports):**
- Source tracts come in as bare/whole (`9601` → `9601.00`), 3-digit (`312` → `0312.00`), or short decimals. Zero-pad the whole part to 4 digits and the decimal to 2. Do NOT collapse to a 6-digit string (`960402` fails — the decimal point is required).
- **Trailing-zero truncation:** a tract stored numerically (e.g. `9509.10`) exports as `9509.1`. So a 1-digit decimal means the *right* zero was dropped → pad on the RIGHT (`.1` → `.10`, `.2` → `.20`), NOT the left.
- **`9999.99` is VALID** — FFIEC assigns it to small counties (≤30k pop) rolled up to county level. Keep it.
- **Malformed (≥5-digit whole, e.g. `96040`, `96033.03`) and `0`/blank tracts:** can't be safely reformatted — leave Tract Code blank and let the street address geocode the row if present; flag the no-address-no-tract rows to the customer.
- **County/State Code of `0`:** treat as missing → blank it (a `000` county breaks geocoding). Row falls back to address or Unassigned.

**Date driver for LOANS geocoding:** the year-specific census data is pulled using `Origination Date` (when `Loan Value Option` = Original Amount, the default) or `Renewal Date` (when Renewal Amount). If the driver date is blank, Kadince returns the most-current census year. On a CSV import, the date column chosen during mapping drives it.

**Kadince does NOT normalize/auto-correct addresses** — it geocodes the string as-is. Missing zip digits / wrong street abbreviations lower confidence but tract codes (when present) override anyway.

---

## 2D. Workspace foldering convention (one folder per customer)

Keep the workspace organized by **customer**, with a consistent lifecycle layout inside each. This is the standard going forward — set it up at the start of a new import and keep files in their lane as you work.

- **`_Context/` stays at the workspace root** — it holds the general, cross-customer material (the `CLAUDE.md` playbook, the EXAMPLE template, Kadince help PDFs). It is NOT customer-specific; never move it under a customer folder.
- **One folder per customer at the workspace root** (e.g. `Poppy Bank/`, `Metro CU/`). Everything for that customer lives under it.
- **Inside each customer folder, use this lifecycle layout:**

  ```
  <Customer>/
    Project Docs/        ← briefing, <Customer>_Import_Status.md, customer-facing summary
    Source Files/        ← customer-provided raw inputs, grouped by object/category
      <Category>/          (e.g. Small Business, Community Development, Organizations, Donations, Hours)
      Kadince Extracts/    ← exports the customer/user provided from their live tenant (authoritative for mapping — see Section 0 step 2)
    Working Files/       ← intermediate/in-progress files (combined, staged, masters)
      Backups/             (numbered .backup.xlsx, .backup2.xlsx …)
      QA/                  (sidecars, REVIEW/SUPERSEDED files)
    Deliverables/        ← final import-ready / imported files
  ```

- **Group `Source Files/` by object or category**, not by date — whatever matches how the customer thinks about the data (loan categories for a loan import, object tabs for a multi-object import).
- **`Kadince Extracts/` is a first-class source subfolder** — keep tenant exports here, separate from the customer's raw business data, since they play the distinct role of mapping authority + reconciliation target.
- **Confirm the exact subfolder taxonomy with the user when it's ambiguous** (2–3 options, recommended first) — the lifecycle skeleton above is the default, but the category breakdown under `Source Files/` is judgment-dependent.
- **Don't break references when moving files:** the status doc / briefing reference files by **name**, not path, so moves are safe — but say so when you reorganize, and never move files mid-task without flagging it.

---

## 2E. HOURS (volunteer-hours) import — field rules & gotchas (confirmed Metro CU 2026-06-16)

A first Hours import failed **52 of 53 rows** on four independent formatting issues — all preventable. Validate these BEFORE delivering any HOURS file:

- **Volunteer is OPTIONAL, but a mapped-yet-empty volunteer column FAILS the row.** Kadince links each hours row to a user via the volunteer-email column. If that column is **mapped during import but the cell is blank**, Kadince looks up user `""` and rejects the row: `user_id: value(s) not found: user Not Found`. Hours **can** import with no volunteer — but only if the volunteer column is **absent/unmapped** for those rows, not present-and-empty. **Fix for a mixed file: split it** — one file for rows that have a volunteer (column kept), and a **separate file for the no-volunteer rows with the volunteer column removed entirely**. (Generalizes: ANY mapped lookup column — volunteer, organization, owner — errors on an empty cell; omit the column rather than ship it blank.)
- **`CRA Eligible` controlled vocab = exactly `Yes` / `No` / `Unknown` / `Partial`.** Source forms commonly store `y`/`Y`/`n`/`N` → normalize (`y`→`Yes`, `n`→`No`). **Blank is accepted** (imports fine). This is the CRA Eligible vocabulary generally, not just on Hours.
- **`Number of People Benefited` must be an integer.** `.xlsx` sources read numeric cells as floats (`300.0`, `22.0`) → strip to int. Handle stray forms: `500+` → `500`; **non-integers like `2.5` are nonsensical as a headcount → blank them and flag** (the field is optional, so blank imports fine — do NOT guess a rounding).
- **`Organization Name` must EXACT-match a tenant org — casing included.** Hours link to orgs by exact name. `YouthBuild Boston` failed against the tenant's `Youthbuild Boston`. Prep-time normalized matching is NOT enough — **validate every hours `Organization Name` against the live Organizations export with exact casing** before delivery (one mismatch = one rejected row). Extends the Phase K whitespace rule: casing counts too.
- **Retry must EXCLUDE rows that already succeeded.** Hours have **no natural dedupe key**, so re-importing a successful row creates a **duplicate hours entry**. When building a retry from the results file, filter to `Success = No` and drop the already-imported rows (Phase K retry shortcut). Same caution applies to any object Kadince can't dedupe on re-import.

---

## 3. Work patterns the user prefers

- **Make backups automatically** before any destructive op (numbered `.backup.xlsx`, `.backup2.xlsx`...). Don't ask first.
- **Use TaskCreate/TaskUpdate** for multi-step work. Mark completed as you go.
- **Run parallel agents** for batch work (web lookups, classifications). Surface counts/summaries, not per-item logs.
- **Be decisive — but ONLY within the Golden Rule (see top of file).** For mechanical/process steps this playbook already spells out, propose a default approach with one-sentence rationale and proceed. For anything touching the **meaning, classification, eligibility, transformation, dedupe, or field-mapping of the customer's actual data** that isn't explicitly covered here, do NOT default or guess — **ask the user first**, then fold the answer back into the playbook. Regulatory-exam stakes mean a wrong assumption about the data is never worth the speed.
- **Filling a previously-BLANK field with a CONFIRMED value is a value-add, not an overwrite — apply it without the same sign-off as CHANGING an existing value** (you can't clobber data that wasn't there). Still: only HIGH-CONFIDENCE values (exact EIN match, validated typo fix); fuzzy/uncertain stays on the customer review list. Changing or clearing an EXISTING value always needs the normal overwrite review. (Kohler CU 2026-06-18: 61 confirmed EINs + 4 ZIP fixes auto-applied as a "nice surprise"; 27 fuzzy EINs held.)
- **Multi-choice questions:** when asking, give 2–3 options with the recommended one first. Don't bury the recommendation.
- **Terse responses.** No trailing "let me know if you have questions" summaries. State what changed and what's next.
- **Type/classification work:** keyword regex first → manual override list for stragglers. Don't iterate the regex past ~95% coverage; it's not worth the complexity.
- **Dedupe ambiguity:** flag candidate pairs and let user decide which are real dupes. Same-EIN doesn't always mean duplicate — booster programs and school districts often share an umbrella EIN across distinct entities.
- **Person-name "orgs":** when in doubt, include them and ask user to prune. Missing a real org is worse than including a real person.
- **Drop all-blank columns from the final sheets before delivering.** After populating each final sheet, remove any column whose data cells are entirely blank — **especially the formulated helper columns** (`Submitted By on Users Tab?`, `Organization Already in Kadince?`, `Address & Tract Codes Present?`, etc.), plus unused optional fields (Email/USERS columns, Guarantor, Collateral, Participation Amount, beneficiary/housing/jobs, Development Activities, Impact fields, etc.). Kadince matches by header name, so omitting an empty column changes nothing on import but makes the file far cleaner for the customer and the data specialist. Do this on the master FIRST, then split — so every chunk shares the same lean column set. (A blank required column shouldn't exist; if one does, that's a population bug, not a column to drop.)
- **Build the full dataset first; chunk only as the final import step.** Always produce and QA the complete single-file dataset first (that's the master of record). Do NOT pre-split. Then:
  1. **Ask the user when they're ready to import** — don't assume the deliverable build flows straight into importing.
  2. **If an object file exceeds 15,000 rows → suggest the Catch URL / import-template route instead of chunking** (see next bullet). For files **between ~5,000 and 15,000 rows**, ask whether to break into 5K-row chunks for the UI import. **≤5,000 rows: single file, normal UI import.**
  3. On the go-ahead to chunk, split into chunks of **≤5,000 DATA rows** each, keeping the master file intact. Each chunk preserves template shape (blank row 1, headers on row 2, data from row 3) and every column incl. custom fields. Name them clearly, e.g. `<File>_part1_of_N.xlsx`. Verify the concatenated chunks equal the master exactly (same row count, same order, no dupes/drops).
  - **Why 5K chunking exists:** it's purely a **front-end UI limitation workaround** — the UI import processor can fail randomly on large files. It is NOT a backend limit. The Catch URL route below bypasses the front end entirely, so chunking is unnecessary there.
- **Large objects (>15K rows): use a Kadince import template + Catch URL (background-job import).** For big datasets, recommend importing via Kadince's API instead of the UI loader. It runs as a **background job** (won't fail on large datasets the way the front end does) and **emails you when it's done**, so the data specialist isn't held hostage watching the loader finish each batch — and **no chunking is needed** (send the whole remainder in one go).
  - **Sequence:**
    1. **Run a small first import through the normal UI** — a first batch of ~**100 rows** is enough. You must do an import first in order to create the template from it.
    2. **Create the import template** from that import. This generates the template's **Catch URL** ("Catch URL" is Kadince's term — it's the API endpoint) **and a key**.
    3. **Key is per-template / per-URL** — each template+URL created has its own key (not a global account key).
    4. **Send the remaining rows to the Catch URL** using that key. Background job processes them; you get an email on completion.
  - **Net effect:** first 100 rows via UI → create template → push the rest via Catch URL. No 5K chunking, no babysitting the import screen.
  - *(Confirm the payload format the Catch URL expects — CSV vs JSON vs the same file we build — before the first run if unsure.)*
- **Customer emails — files live on the Kadince task, not the email.** When an email to a customer references deliverables/attachments (import files, summaries, appendices), tell them the files are **attached to the Kadince task**, not attached to the email itself. Phrase it like "the files are attached to your Kadince task" / "you'll find these on the task in Kadince." Don't attach them to the email and don't explain the reason to the customer — just point them to the task.

---

## 4. Common file-shape signals (pattern recognition)

| Signal | What it usually means |
|---|---|
| Donations `County` column blank for 80%+ of rows | The column is sparse / unreliable; treat as optional |
| `<Year> Scholarship Recipient` as Organization Name | Bucket placeholder for scholarship donations; consolidate under `<Customer> Scholarships` |
| `Sport Sponsorship` / `Education Sponsorship` / `Student Support` | Generic donation categories; consolidate or extract specific org from title |
| Person-pattern Organization Name with title `"Sponsorship - <description>"` | Individual sponsorship; consolidate under `<Customer> Sponsorships` |
| Same EIN, different names | Often legit umbrella (booster club covering multiple sports, district covering multiple schools) — confirm before merging. NEVER auto-rename based on EIN alone when reconciling against a Kadince export |
| Trailing whitespace, mixed case in org master names | Source system has weak validation; expect this and normalize during canonical selection |
| Address is `PO Box ...` | Mailing only; Physical Address should stay blank |
| `Tax ID/EIN` value is a single space, `"0"`, or partial digits | Source-system data quality issue; clear to blank — these WILL fail on Kadince import |
| `Phone` with fewer than 10 digits (e.g. `(707) 578-77`) | Source truncated trailing digits (often phones stored as numbers); unrecoverable → clear to blank (Kadince rejects the row otherwise). Flag the list for the customer to re-enter |
| `Organization Name` has trailing/leading whitespace | Invisible failure mode — donations referencing the org will fail at upload with `value(s) not found`. ALWAYS `.strip()` org names in donations before delivery |
| Two orgs in Kadince export with identical `Organization Name` | Master-side duplicate — every donation referencing that name will fail with `duplicates found`. Flag to customer for pre-import merge |

---

## 5. Deliverable checklist (final sanity pass before handing off)

**Org file:**
- [ ] Every donation's `Organization Name` exists in org file (exact match, post-dedupe)
- [ ] `Organization Type` populated for every row, `Other` ≤ 1%
- [ ] `Tax ID/EIN`, `Web Address`, `Organization Type` columns named exactly as template
- [ ] `Physical Address` + `Mailing Address` are 4 cols each, with appropriate population
- [ ] **`Tax ID/EIN` is either blank or formatted as `NN-NNNNNNN`.** Any other value (single space, `"0"`, partial digits, non-numeric) WILL FAIL on import — clear them to blank. Validate: anything that doesn't match `^\d{2}-\d{7}$` after formatting must be cleared.
- [ ] **`Phone` is either blank or exactly 10 digits.** Strip to digits, drop a leading `1` from 11-digit numbers; clear anything that isn't 10 digits (Kadince rejects the whole row). Report the cleared list.
- [ ] If a Kadince export was provided: org names exact-match Kadince spelling where the org already exists (name-normalized match auto-applied; EIN-only matches reviewed for umbrella-EIN false positives)

**Donations file:**
- [ ] Addresses pulled from matching org rows where available
- [ ] `Impact Areas` populated where source had a county (format as `"<County> County"`)
- [ ] `Approval Status` values in Title Case
- [ ] `Donation Type` populated from title prefix (Sponsorship / Scholarship / Grant / Donation)
- [ ] Donation `Organization Name` values updated to match any Kadince-export renames applied to the org file
- [ ] QA-only columns dropped (`County Match`, etc.)

**Hours file (HOURS tab) — see Section 2E:**
- [ ] `CRA Eligible` is blank or exactly `Yes`/`No`/`Unknown`/`Partial` (normalize `y`/`Y`→`Yes`, `n`/`N`→`No`)
- [ ] `Number of People Benefited` is blank or an integer (strip `.0` floats; `500+`→`500`; blank non-integers like `2.5` and flag)
- [ ] Every `Organization Name` exact-matches a tenant org **including casing** — validate against the live Organizations export
- [ ] No-volunteer rows are split into a separate file with the volunteer column **removed** (never mapped-and-blank)
- [ ] Retry files exclude rows that already imported (Hours have no dedupe key → re-import duplicates)

**Process:**
- [ ] Backups exist of every transformed file (numbered `.backup.xlsx`, `.backup2.xlsx`...)
- [ ] Open items for customer flagged (false-positive types, ambiguous extracted orgs, missing addresses, unresolved umbrella-EIN matches, etc.)
- [ ] `Kadince_match_report.csv` delivered alongside files (if Kadince export was provided)
- [ ] **If upsert into active Kadince tenant:** `Upsert_risk_report.csv` delivered AND customer chose a mitigation strategy (safe-mode file, strip Org Type only, or confirm Kadince null-handling). Without this step, blank cells in your upload can silently wipe existing Kadince data.
- [ ] `Import_Summary_for_Customer.md` delivered — customer-facing walkthrough of every change made and why, with open items flagged
- [ ] **Post-import:** row-count reconciled against expected (Phase K). Any deltas explained by silent within-file merges from Phase I, or investigated if not.

---

## 6. Tools to load early

These are deferred and need `ToolSearch` before first use:
- `TaskCreate`, `TaskUpdate`, `TaskList`
- `WebSearch`, `WebFetch`

---

## 7. Out of scope unless asked

- Phone number enrichment (slow, low payoff vs. cost)
- Mission Statement / Description fields (not derivable from data)
- CRA Eligible / Development Activities / Qualitative Factors (banks only)
- USERS tab population (needs customer-provided employee list)
- Geocoding lat/long + census-data lookup — Kadince does this automatically. BUT you DO still own formatting the State/County/Tract codes correctly when the source provides them (see Section 2C) — Kadince won't fix a malformed tract, it just fails to geocode that row.

---

## 8. What "complex" looks like (when this playbook isn't enough)

This playbook assumes the **simple** 2-file shape (org master + donations log). Real customers often deliver 4–10 files from disparate systems:
- Multiple donation source systems (different time periods, different schemas)
- Volunteer hours log (maps to `HOURS` tab)
- Loan portfolio (maps to `LOANS` tab)
- Community investments (maps to `INVESTMENTS` tab)
- Separate contact / volunteer involvement lists (sub-record tabs)
- HR roster (maps to `USERS` tab)

For these:
1. Map every input file to its target Kadince tab first
2. Normalize org names across ALL input files (every reference to "ABC Org" should resolve to one canonical name)
3. Build a single org master that's the union of every file's org references
4. Reconcile per-file, then do template alignment last
test push - it works
