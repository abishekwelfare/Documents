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
├── AGM-Minutes/                     README.md only so far — no minutes uploaded yet
├── Audited-Balance-Sheets/
│   ├── ARWA-Balance-Sheet-FY2022-23.pdf
│   ├── ARWA-Balance-Sheet-FY2023-24.pdf
│   └── ARWA-Balance-Sheet-FY2024-25.pdf
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

## Adding a new document — checklist

1. Copy the PDF into the right category folder, named with a clear date/FY, e.g.
   `Audited-Balance-Sheets/ARWA-Balance-Sheet-FY2025-26.pdf`.
2. If it belongs in a category already listed on the homepage, add a link to it in
   `index.html`'s "All Documents" section.
3. If it supersedes a "latest"-linked document, repoint the `*-Latest.pdf` symlink
   (see mechanism above) — don't just add the new file and leave the old one linked.
4. Stage explicitly (never `git add -A`), commit, push. The live site updates within
   a minute or two of GitHub Pages picking up the push — no manual deploy step.

## GitHub Pages configuration

Settings → Pages → Source: "Deploy from a branch" → branch `main`, folder `/ (root)`.
This is a one-time manual setup step in the GitHub web UI, not something git-side —
if the live site ever 404s after a push that should have worked, check this setting
hasn't reverted rather than assuming a content problem.
