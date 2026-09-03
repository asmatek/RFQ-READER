# RFQ Reader — Outlook Add-in

Reads the currently open email (body + PDF/ZIP attachments), sends the text to
Claude to translate into English and extract a structured RFQ table, shows it
in a task pane, and lets you download it as an `.xlsx` file.

Columns produced: **Lot Number, Description, Part Number, Quantity, Budget,
Manufacturer, Additional Info, Tender End Date.**

## Files
- `manifest.xml` — the Outlook add-in manifest (defines the ribbon button/task pane)
- `taskpane.html` — the entire app (UI + logic), no build step needed
- `assets/` — icons referenced by the manifest

## 1. Host the files
Outlook add-ins must be served over **HTTPS**. Pick any static host:
- GitHub Pages, Netlify, Vercel, Azure Static Web Apps, or your own IIS/web server.

Upload `taskpane.html` and the `assets/` folder, note the final base URL
(e.g. `https://yourname.github.io/rfq-reader/`).

Then open `manifest.xml` and replace every occurrence of
`YOUR_HOSTING_DOMAIN` with your actual domain/path, e.g.:
```
https://yourname.github.io/rfq-reader/taskpane.html
https://yourname.github.io/rfq-reader/assets/icon-64.png
```

> For quick local testing you can also run a local HTTPS dev server
> (e.g. `npx office-addin-dev-certs install` + `npx http-server -S -p 3000`)
> and point the manifest to `https://localhost:3000/taskpane.html`.

## 2. Sideload it into Outlook
**Outlook on the web / new Outlook:**
Settings ⚙ → View all Outlook settings → General → Manage add-ins →
**My add-ins** → "Add a custom add-in" → "Add from file" → select `manifest.xml`.

**Outlook desktop (classic, Windows/Mac):**
Home tab → Get Add-ins → My add-ins → "Add a custom add-in" → "Add from file".

Once added, open any email → you'll see a **"Read RFQ"** button in the ribbon
that opens the task pane.

## 3. Get an Anthropic API key
Create one at https://console.anthropic.com. Paste it into the task pane's
API key field the first time — it's saved locally (Outlook roaming settings)
so you only enter it once. It is sent directly from your browser/Outlook
client to `api.anthropic.com` and nowhere else.

## 4. Use it
1. Open an RFQ/tender email (the kind you attached for testing — Samruk-Kazyna
   / QazTorg style PDFs work well since they're text-based, not scanned images).
2. Click **Read RFQ**.
3. Wait while it reads the body, extracts text from any PDF attachments (and
   PDFs inside ZIP attachments), and sends it to Claude for translation +
   extraction.
4. Review the table, then click **Download Excel (.xlsx)**.

## Supported file types
- **PDF** — text extracted directly. If a page has almost no embedded text
  (common for scanned drawings or nameplate photos saved as PDF), that page
  is instead rendered as an image and sent to Claude's vision so it can read
  it (part numbers, model/serial numbers, ratings, etc.).
- **Word (.docx)** — text extracted (legacy `.doc` is not supported by
  browser-side tools; ask the sender for `.docx` or PDF instead).
- **Excel (.xlsx / .xls / .csv)** — every sheet is converted to text/CSV.
- **Images (.jpg/.jpeg/.png/.gif/.bmp/.webp)** — sent directly to Claude's
  vision, e.g. photos of equipment nameplates or hand-marked drawings.
- **ZIP** — opened and every file inside is processed using the rules above
  (PDF/DOCX/XLSX/images), recursively through one level of nesting.
- **CAD files (.dwg/.dxf)** — cannot be read in-browser; the tool flags them
  so you know something was skipped. Ask the sender to also attach a PDF or
  image export of the drawing.

## Notes & limitations
- Up to **15 images** per email are sent to the model (attached photos +
  rendered "sparse-text" PDF pages) to keep request size reasonable; large
  images are automatically downscaled before sending.
- Very large documents are trimmed to stay within the model's context window;
  if a tender has many lots split across huge attachments, consider running
  it per-attachment/email if something gets cut off.
- The model can occasionally leave a field blank if the source document truly
  doesn't state it (e.g. missing part number) — that's expected, not a bug.
- Nameplate/drawing photos: the more legible the photo (good lighting, text
  in focus), the better the extraction — this relies on the model reading the
  image, not true OCR software.
