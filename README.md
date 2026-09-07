# Shani Traders billing — v2

## Upload these to the same folder

```
index.html                 the whole app (replaces the three old pages)
sw.js                      offline support
manifest.webmanifest       install-on-home-screen settings
icon-192.png
icon-512.png
icon-maskable-512.png
apple-touch-icon.png
invoice-billing.html       tiny redirects, only so old bookmarks still work
master.html                (safe to delete later)
invoices.html
```

`Code.gs` goes into Apps Script, not the web folder.

## Two things to do once

1. **Apps Script** — open the Sheet → Extensions → Apps Script → delete the old
   code → paste `Code.gs` → Save → Deploy → Manage deployments → pencil →
   Version: **New version** → Deploy. Keep the same deployment so the `/exec`
   URL does not change.
2. **HTTPS** — the app must be served over `https://` (or `localhost`), or the
   service worker will not register and it cannot be installed. GitHub Pages,
   Netlify and Cloudflare Pages all do this for free.

Then open the site on the phone → Android: menu → *Install app*.
iPhone: Safari → Share → *Add to Home Screen*.

## Fixed

- **Duplicate bills after a dropped connection.** Every bill now carries a
  client ID; the server refuses to write the same ID twice, so a retry can
  never create a second copy of the same sale.
- **Numbering could repeat** if a row was ever deleted or sorted. The counter
  now lives in Script Properties and only moves forward.
- **Dates came back wrong** when Sheets silently converted `10-05-2026` into a
  real date. Date, mobile and invoice-number columns are now plain text, and
  the reader converts anything that slipped through.
- **Old bills could not be reprinted** — only item names were stored. Full line
  items are now saved as JSON in one extra column, so any bill can be re-opened
  and re-printed exactly.
- **PDF was a screenshot.** It was `html2canvas` at 780px scaled to fit, so text
  was fuzzy, files were large, and a long bill was squashed instead of paged.
  The PDF is now drawn directly in millimetres on A4 with real text — crisp at
  any zoom, roughly 30 KB, and it paginates with a repeated table header.
- **jsPDF was never actually loaded** as its own library; the code read
  `window.jspdf` while only the html2pdf bundle was on the page.
- **CGST/SGST split lost a paisa** on odd tax amounts. SGST now takes the
  remainder so the two halves always add back to the exact tax.
- **Three separate Apps Script calls on every page load** (next number, items,
  customers) × three pages. Now one `bootstrap` call per app launch, and the
  offline queue goes up in a single `syncBatch` call.
- **A half-finished bill was lost** if the phone locked or the browser reloaded.
  The form now autosaves and restores.
- Item picking was a `<select>` that gets unusable past ~20 items; it is now a
  searchable sheet.

## Nothing is lost, by design

Every bill is written to the phone *before* the network is touched, and the
queue survives a reload, a crash and a flat battery. If a bill is created
offline it gets a provisional number; when it reaches the sheet the server keeps
that number if it is still free, and if it is not, the app tells you which bills
were renumbered so you can re-download those PDFs from History.

## Optional: lock the sheet

The web app is deployed as "Anyone", so anyone with the URL can read your
invoice list. Set `SHARED_TOKEN` in `Code.gs` to any word, put the same word in
More → Access word, and redeploy. This is obfuscation, not real security — the
word sits in the page source — but it stops casual access.
