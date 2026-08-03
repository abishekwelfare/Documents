# ARWA-Documents — Codebase Guide for Claude Code

Public document repository for the Abishek Residents' Welfare Association (ARWA),
68 East Coast Road, Thiruvanmiyur, Chennai 600041. Served as a static site via
GitHub Pages — no server, no build step, no dependencies.

**Repository**: https://github.com/abishekwelfare/Documents (public)
**Live site**: https://abishekwelfare.github.io/Documents/
**Local mirror**: `C:\ClaudeWorkspace\ARWA-Documents` — a separate git repo, **not**
a subfolder of `C:\ClaudeWorkspace\RWA-RMS`. See that repo's own `CLAUDE.md` for the
financial/receipt-automation system this one is unrelated to in code, though it
serves the same association.

---

## Git identity — read before committing anything here

This repo's local git config (`user.name` / `user.email`) is deliberately set to:

```
user.name  = Abishek Residents Welfare Association
user.email = abishekwelfare@gmail.com
```

**Not** the "3S Technologies Chennai" identity used in the sibling RWA-RMS repo —
this is a different GitHub account (`abishekwelfare`), and commits here should read
as coming from the association, not a third-party developer. This was set up wrong
once already (defaulted to RWA-RMS's identity) and corrected immediately — don't
re-introduce that mistake. Check `git config user.name` before committing if unsure;
it's local to this repo, not global, so it won't leak into other repos either.

Pushing also needs credentials for the `abishekwelfare` GitHub account specifically —
don't assume whatever's cached/authenticated for other repos on this machine applies
here.

---

## Repository layout

```
ARWA-Documents/
├── CLAUDE.md                        this file
├── README.md                        human-facing repo docs (GitHub repo page)
├── .nojekyll                        disables Jekyll — GitHub Pages serves files as-is
├── index.html                       homepage: association info, committee, key
│                                    document links, contact — self-contained,
│                                    no build step, edit directly
├── Bye-Laws/
│   ├── ARWA-Bye-Laws-Approved-2021-EGM.pdf     original
│   ├── ARWA-Bye-Laws-v1.1-Approved-2023-08-27-EGM.pdf   amendment (family
│   │                                member of owner now MC-eligible)
│   └── ARWA-Bye-Laws-Latest.pdf     git symlink -> v1.1 file above
├── Good-Living-Guidelines/
│   ├── ARWA-Good-Living-Guidelines.pdf
│   └── ARWA-Good-Living-Guidelines-Latest.pdf   git symlink -> file above
├── AGM-Minutes/
│   └── ARWA-AGM-Slides-2026-08-02.pdf   presentation slides, not minutes
│                                    (minutes themselves not yet finalized/uploaded)
├── Audited-Balance-Sheets/
│   ├── ARWA-Balance-Sheet-FY2022-23.pdf
│   ├── ARWA-Balance-Sheet-FY2023-24.pdf
│   └── ARWA-Balance-Sheet-FY2024-25.pdf
├── Maintenance-Charges/
│   └── Maintenance-Charge-Calculator-Oct2026.html   interactive per-flat
│                                    calculator, self-contained (Base64 JSON
│                                    blob of area/charge data, no server) —
│                                    see below
├── Circulars-Notices/                README.md only so far
├── Committee-Resolutions/            README.md only so far
└── Forms/                            README.md only so far
```

Flat categories, not deep nesting — GitHub Pages has no directory auto-listing, so
visitors navigate via `index.html`'s links, not by browsing folders. Every document
filename embeds its own date/FY so it's self-identifying even shared out of context.

---

## The "Latest" symlink mechanism — and why `git add -A` is dangerous here

Bye-Laws and Good Living Guidelines are linked from the homepage via a date-agnostic
`*-Latest.pdf` file that's a **git symlink** to whichever dated file is current —
so the homepage link never needs to change when a document is amended, only the
symlink's target does.

**This machine's git can't create real filesystem symlinks**: `core.symlinks` is
`false`, and Windows symlinks need admin/Developer Mode rights regardless. A plain
`ln -s` here just **copies the file's content** instead of linking — confirmed by
testing (`git config core.symlinks` → `false`; `ln -s X Y; file Y` reported Y as the
actual PDF, not a link, size matching X exactly).

The fix: create the symlink git **object** directly, bypassing the OS entirely —
GitHub stores and serves it correctly (mode `120000`, content = target filename)
regardless of how the local checkout represents it:

```bash
SHA=$(printf '%s' "NEW-FILENAME.pdf" | git hash-object -w --stdin)
git update-index --add --cacheinfo 120000,$SHA,Bye-Laws/ARWA-Bye-Laws-Latest.pdf
git commit -m "Update latest Bye-Laws to <date>"
git push
```

**Critical gotcha**: this only touches the git **index**, not the working tree —
there's no actual file at `Bye-Laws/ARWA-Bye-Laws-Latest.pdf` on disk, only an index
entry. `git status` will show it as "deleted" (comparing index to the nonexistent
working-tree file) even though it's staged correctly — that's expected, not a bug;
verify with `git ls-files -s | grep Latest` instead (should show `120000`, not
`100644`).

**Never run `git add -A` or `git add .` in this repo** after a symlink has been
staged — either will interpret "no working-tree file" as a real deletion and drop
the symlink from the index. Stage files explicitly instead:

```bash
git add index.html "Audited-Balance-Sheets/ARWA-Balance-Sheet-FY2025-26.pdf"
```

If a symlink does get dropped this way, just re-run the `hash-object`/`update-index`
pair above — the blob is still in git's object database (from the `-w` flag), so no
data is lost, but the index entry needs re-adding before the next commit.

---

## Audited Balance Sheets — standardized order (added 2026-07-29)

Every financial statement PDF follows the same page order: **Auditor's Report and
Notes to Accounts first**, then Balance Sheet, Income & Expenditure Statement,
Provision for Taxation, Depreciation schedule, and Annexure.

- **FY2022-23** — source file was already combined in this order; added as-is.
- **FY2023-24** — source was two separate PDFs (`audit report 2024.pdf`,
  `financial 23-24.pdf`); merged with `pypdf` (`PdfWriter`), auditor report first.
- **FY2024-25** — source was one combined PDF but in the *opposite* order (balance
  sheet first, auditor report last); reordered with `pypdf` before adding.

For a future year's statement, match this order — if given a combined PDF with the
balance sheet first, reorder it with `pypdf.PdfReader`/`PdfWriter` (`add_page()` in
the desired sequence) before committing; if given separate files, concatenate with
the auditor report first. Verify the result by rendering it (the `Read` tool renders
PDF pages as images) before committing — page-order mistakes are easy to make with
`pypdf` and hard to catch without a visual check.

---

## Committee roster on the homepage — time-bound, needs manual updates

`index.html`'s "Management Committee" table and its "in office until <date>" note
are only valid for the term shown. Update both the roster and the date after every
AGM (the association elects a new Management Committee at each one) — there's no
automation for this; it's a static table, edited by hand.

Current committee (set 2026-07-29, valid until the AGM on 02-Aug-2026): President
Asokamani (B3-6A), Vice President Sriram (B5-9A), Secretary Sankar (B5-3C),
Treasurer Chandrasekaran (B4-10A), Members Anantha Lakshmi (B4-5C), Murali (B2-7A),
Ashok (B5-4D).

**No personal contact info** (phone numbers, personal emails) belongs on this page —
only names, roles, and flat IDs for the committee, and the association's own email
(`abishekwelfare@gmail.com`) for contact. This was an explicit design constraint from
the association, not an oversight — don't add personal details even if asked for
"just this once."

---

## PDF links open in a new tab, with a "PDF ↗" badge (added 2026-07-29)

Every link to a PDF on `index.html` carries `target="_blank" rel="noopener"` (new
tab; `rel="noopener"` so the new tab can't reach back into this page via
`window.opener` — standard practice whenever `target="_blank"` is used) and a
small visible badge so visitors know before clicking that it opens/downloads a
file rather than navigating within the site.

- **"All Documents" list** (`.cat-list`) — the badge is CSS-generated
  (`.cat-list a[href$=".pdf"]::after`), so it appears automatically on any link
  ending in `.pdf` added there in the future. Only `target="_blank" rel="noopener"`
  needs adding by hand on a new link; the badge takes care of itself.
- **"Key Documents" cards** (`.key-doc-btn`) — these need the badge added
  explicitly, `<span class="pdf-badge">PDF &#8599;</span>` next to the title —
  the CSS `::after` trick would land after the whole card (title + subtitle),
  not next to the title where it reads best, so it isn't used there.

If a non-PDF link is ever added to `.cat-list` (unlikely, but e.g. a link to an
external page), don't give it `target="_blank"` by default — only PDFs get that,
since only PDFs are a "leaves the page to open/download a file" action; a normal
internal or informational link shouldn't force a new tab.

---

## Maintenance Charge Calculator (added 2026-08-02)

`Maintenance-Charges/Maintenance-Charge-Calculator-Oct2026.html` — a standalone,
self-contained page (same "no server, Base64 JSON blob embedded in the HTML"
pattern as the RWA-RMS dashboards in the sibling repo) letting a resident pick
Block / Floor / Wing and see their flat's maintenance charge, both old (till
Sep 2026, ₹3.25/sq.ft./month) and new (from Oct 2026, ₹4.25/sq.ft./month),
plus GST @18% where it applies.

- **Data source**: `rwa_rms.db` (`flats.flat_area`) in the sibling RWA-RMS
  repo — this repo has no DB access of its own, so the 148-flat blob was
  computed there and pasted in by hand. If the AGM approves another rate
  change, regenerate the blob from that DB with the same script logic (not
  currently saved as a standalone script — reconstruct from this page's own
  JS) and re-paste the `BLOB` constant; there's no automated sync between the
  two repos.
- **Base64, not real encryption**: deliberately not encrypted — for a static
  page, any client-side "encryption" key would have to ship in the same
  page's JS, so it wouldn't actually protect anything. The data itself (flat
  area + charge figures only, no names/emails/phone numbers) isn't sensitive
  enough to need more than this.
- **Cascading dropdowns are derived, not hardcoded**: Block → Floor → Wing
  options are built at runtime from the flat_id keys in the data blob itself
  (regex `^B(\d+)-(\d+)([A-Z]+)$`), so penthouse wing combos (`9AD`/`9BC` in
  Block 2, `9AB`/`9CD` in Block 3 — two flats merged into one, so a single
  two-letter wing rather than four separate ones) fall out naturally with no
  special-casing.
- **Rounding — treasurer's rule, corrected 2026-08-04**: the blob stores the
  *quarterly* charge directly (old, new-base, GST), computed as plain
  round-half-up to the nearest rupee (`x.50` rounds up) — **not** derived
  from a monthly figure. A quarter's total doesn't always split evenly into
  three equal monthly payments, so the page shows a payment-guide callout
  under the total instead (e.g. "₹X in 2 months + ₹X+1 in 1 month") — see
  `monthlyGuideText()` in the page's own JS. Earlier drafts of this page (i)
  stored monthly and derived quarterly as `monthly × 3`, and (ii) briefly
  used a `truncate(x + 0.49)` threshold (misreading the treasurer's ".50" as
  ".51") — both superseded; the companion review spreadsheet in the RWA-RMS
  repo (`Members-Database/Maintenance_Charge_Revision_Oct2026.xlsx`) and
  `rwa_rms.db` itself (`flat_balance.truncate_round()`,
  `update_maintenance_rate.py`) all use this same round-half-up rule now, so
  every figure across all three agrees.
- **GST threshold**: 18% GST applies where the *new* monthly maintenance
  exceeds ₹7,500 (CGST/SGST rule on society maintenance). At the new rate
  this affects exactly 4 flats — all penthouses — `B2-9AD`, `B2-9BC`,
  `B3-9AB`, `B3-9CD`; none crossed the threshold at the old rate.
- **`B4-10A`'s area** (a data-entry typo — was 1424 in the DB, corrected
  2026-08-03 to 1420, matching all 9 other B4 A-wing flats) is now read
  straight from `rwa_rms.db` like every other flat — no manual override in
  this page's generation script anymore. (An interim override existed
  2026-08-03/04 while the DB-side fix was pending; removed once
  `flats.flat_area` itself was corrected.)

---

## Adding a new document — checklist

1. Copy the PDF into the right category folder, named with a clear date/FY, e.g.
   `Audited-Balance-Sheets/ARWA-Balance-Sheet-FY2025-26.pdf`.
2. If it belongs in a category already listed on the homepage, add a link to it in
   `index.html`'s "All Documents" section, with `target="_blank" rel="noopener"`
   (see PDF link convention above — the badge appears on its own for this section).
3. If it supersedes a "latest"-linked document, repoint the `*-Latest.pdf` symlink
   (see mechanism above) — don't just add the new file and leave the old one linked.
4. Stage explicitly (never `git add -A`), commit, push. The live site updates within
   a minute or two of GitHub Pages picking up the push — no manual deploy step.

## GitHub Pages configuration

Settings → Pages → Source: "Deploy from a branch" → branch `main`, folder `/ (root)`.
This is a one-time manual setup step in the GitHub web UI, not something git-side —
if the live site ever 404s after a push that should have worked, check this setting
hasn't reverted rather than assuming a content problem.
