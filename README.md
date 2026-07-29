# ARWA Documents

Public document repository for the **Abishek Residents' Welfare Association (ARWA)**,
68 East Coast Road, Thiruvanmiyur, Chennai 600041.

Live site: https://abishekwelfare.github.io/Documents/

## What's here

| Folder | Contents |
|---|---|
| `Bye-Laws/` | Approved Bye-Laws |
| `Good-Living-Guidelines/` | Guidelines for residents |
| `AGM-Minutes/` | Annual General Body Meeting minutes, by financial year |
| `Audited-Balance-Sheets/` | Audited balance sheets, by financial year |
| `Circulars-Notices/` | General notices and circulars |
| `Committee-Resolutions/` | Management committee resolutions |
| `Forms/` | Contact update and other resident forms |

## Adding a new document

1. Drop the PDF into the right category folder, named clearly with a date/FY, e.g.
   `Audited-Balance-Sheets/ARWA-Balance-Sheet-FY2025-26.pdf`.
2. If it belongs in a category listed on the homepage (`index.html`), add a link
   to it in the "All Documents" section, with `target="_blank" rel="noopener"` —
   every PDF link should open in a new tab. The small "PDF ↗" badge next to it
   appears automatically (CSS, matches any link ending in `.pdf` in that list) —
   no extra markup needed for that part. The "Key Documents" cards at the top
   need the badge added by hand instead (`<span class="pdf-badge">PDF ↗</span>`
   next to the title) since they're outside that auto-styled list.
3. If it's a new version of a "latest"-linked document (Bye-Laws, Good Living
   Guidelines), repoint the `-Latest.pdf` symlink to the new file (see below).
4. `git add`, `git commit`, `git push` — the live site updates within a minute
   or two.

## Repointing a "latest" symlink

Bye-Laws and Good Living Guidelines are linked from the homepage via a
date-agnostic `*-Latest.pdf` file that's a git symlink to the current version,
so the homepage link never needs to change when a document is updated — only
the symlink's target does.

This machine's git isn't set up for real filesystem symlinks (`core.symlinks`
is `false`, and Windows symlinks need admin/Developer Mode rights anyway), so
`ln -s` won't create one — instead, create the symlink git object directly:

```bash
SHA=$(printf '%s' "NEW-FILENAME.pdf" | git hash-object -w --stdin)
git update-index --add --cacheinfo 120000,$SHA,Bye-Laws/ARWA-Bye-Laws-Latest.pdf
git commit -m "Update latest Bye-Laws to <date>"
git push
```

GitHub stores and serves this correctly regardless of how the local checkout
represents it (as a real symlink, or — on this machine — a plain text file
containing the target filename).

## Committee roster on the homepage

`index.html`'s committee list is only valid for the term shown — update it
(and the "in office until" date) after each AGM.
