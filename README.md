# Shani Traders billing — v2

## Upload these to the same folder

```
index.html                 welcome page: install button + all settings
app.html                   the billing app itself
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

## How the two pages fit together

`index.html` is what you open in a browser: the shop name in large type, an
**Install app** button, and a **Settings** button in the top right. Settings
live only here — shop details, the Google Sheet URL, and the starting bill
number. The app itself has no settings tab.

`app.html` is the billing app: Bill, History, Items, Parties. Opening it shows
the logo and the shop name for a moment, then the new-bill screen. The shop name
sits across the top in large type with the current screen named underneath.

The installed icon opens `app.html` directly. If you ever open `index.html`
from the installed app it jumps straight to billing, so the welcome screen only
appears in a browser. To reach settings from the installed app, long-press its
home-screen icon and pick **Settings**, or open the site in a browser. If the
sheet URL is ever missing, the app shows a banner on the bill screen with a link
straight to settings.

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

## Second pass — mobile UI

- Boxes ran off the right edge of the phone. Grid and flex children default to
  `min-width:auto`, so the GSTIN and mobile fields refused to shrink below their
  placeholder width and pushed the page sideways.
- The Save button kept disappearing because the total bar was `position:sticky`
  — sticky still scrolls away once the page is taller than its section. It is
  now `position:fixed` above the tab bar, so it is on screen at all times.
- Pinch and double-tap zoom are off (`user-scalable=no` plus
  `touch-action:manipulation`), which is what stops a mis-tap from zooming the
  form mid-bill. Input font stays at 16px so iOS does not zoom on focus either.
- "Rate ₹ (incl. GST)" wrapped to two lines and knocked the Qty / Rate / Amount
  boxes out of alignment. The label is now just "Rate ₹" with one note above
  the lines, and the three columns sit on a shared baseline.
- Checked at 320, 360 and 412 px: nothing overflows and the page scrolls.

## Bill numbering

Settings → Bill numbering → enter the number the next bill should carry. If the
shop already issued 430 bills in the old software, enter 431 and the next PDF
comes out as ST00431. The app warns you if the sheet already holds a higher
number, since that would repeat.

## PDF typography

Body text went from about 8 pt to 9.4 pt, the shop name to 19 pt and the total
to 13.5 pt. The item table now stretches down to a fixed line on the page, so a
two-item bill fills the sheet instead of floating at the top. Twelve items fit
on one page; beyond that it pages properly with a repeated table header.

## GST and non-GST bills

**Save & make PDF** now asks which bill to make.

*GST Bill* is unchanged: ST00431, HSN, GST%, CGST/SGST or IGST, the rate-wise
tax summary, terms and signature.

*Non-GST Bill* runs its own series numbered 0001, 0002, and prints the plain
format: shop name, address and mobile only, then Party name / Address / Mob no.
/ Bill no. / Date, then S.No, Description, Qty, Rate, Amount with blank ruled
rows filling the page, TOTAL at the bottom, and the amount in words. No GSTIN,
no HSN, no tax anywhere.

The rate you type is used as-is on a non-GST bill — nothing is backed out of it,
so a rate of 250 prints as 250 and totals as 250.

Both kinds land in the same Invoices tab, told apart by a new **Bill Type**
column, and both appear in History with a tag, so either can be re-opened and
re-printed. The two counters are completely independent: setting one never
touches the other. Both are set from Settings → Bill numbering.

## Deleting old bills

Delete them in the sheet by selecting the row numbers, right-clicking and
choosing **Delete rows**. Do not just press Delete to clear the cells — a
cleared row still counts as the last row, so the next bill is written below the
blank gap.

The counter does not roll back on its own. If the bills you removed were the
last ones and you want those numbers reused, open Apps Script, pick
`resetInvoiceCounter` in the function dropdown and Run it once. It re-points
both series at the highest number still in the sheet. Do not run it after
deleting rows from the middle — that would hand out numbers that already exist
further down.

Then open History in the app and tap **Refresh**. Refresh is authoritative: any
bill the server no longer has is cleared from the phone too. Bills still waiting
to go up are never touched, and if the reply is capped at 300 rows the older
tail is left alone rather than wrongly deleted.

## Extra charges and credit sales

Below the items there are three optional boxes: **Transport charges**,
**Labour charges** and **Credit amount**.

Transport and labour are added to the bill total exactly as typed, with no GST
on them, so the taxable value and the rate-wise tax summary stay untouched. Both
appear as their own lines just above the total — inside the totals block on a GST
invoice, and as table lines on a non-GST bill. Zero means they are left off the
bill entirely.

Credit is the amount left unpaid; the rest counts as received. A bill with
nothing in that box prints as before. With a figure in it, the PDF shows
**Received** and **Balance due** beside the total, and the bill is added to the
debtors record.

The debtors record lives in the sheet, in the **Credit** column. There is no
outstanding view in the app — the Parties tab is just customers, as before.
History tiles do carry a red "₹X due" tag so an unpaid bill is easy to spot, and
opening one gives a **Record payment** button that lowers the Credit figure on
that row. Edit the column in the sheet directly if you prefer; the app picks it
up on the next Refresh. Payments queue like everything else, so they survive
being recorded with no signal.

**Clear all** now wipes the whole bill — buyer name, mobile, address, GSTIN,
items and all three charge boxes — and asks first, since it cannot be undone.

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
