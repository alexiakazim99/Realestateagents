# Lead Agent — Technical Summary

**n8n workflow name:** `Lead agent`
**Workflow ID:** `WDUPtrstUP9DT8cS`
**Status:** Active (production)
**Model:** GPT-4.1 (`@n8n/n8n-nodes-langchain.openAi`)

## Purpose

Receives leads submitted through the Tally form "Intresseanmälan", qualifies them with GPT-4.1 (temperature + priority), confirms receipt to the lead, routes the lead to the correct broker, and logs every lead to a spreadsheet. Also generates a per-property Tally form link so each property gets its own pre-filled intake link.

The workflow contains **two independent chains** with separate triggers.

---

## Chain A — Lead intake (webhook-triggered)

Tally posts to the webhook whenever someone submits the interest form.

| Order | Node | Type | What it does |
|---|---|---|---|
| 1 | **Webhook** | `webhook` | `POST /webhook/lead-agent`, header auth (`x-api-key`, credential "Header Auth account"). `responseMode: onReceived` — acknowledges Tally immediately. |
| 2 | **Extract lead fields** | `code` | Reads `body.data.fields`, matches each field by its `label` (case-insensitive) to pull out `namn`, `epost`, `telefon`, `meddelande`, `objekt`. |
| 3 | **Categorize lead** | `openAi` (GPT-4.1) | Sends the message + objekt to GPT-4.1; returns strict JSON with `temperatur`, `prioritet`, `motivering`. See `system-prompts.md`. |
| 4 | **Format output** | `code` | Parses the model's JSON (falls back to `kallt`/`låg` if unparseable) and merges it with the lead fields, including the raw `meddelande`. |
| 5 | **Lookup broker in sheet** | `googleSheets` (read) | Reads **all rows** of the Objekt tab. `alwaysOutputData: true` — required, see "Design notes". |
| 6 | **Merge broker lookup** | `code` | Finds the row whose `Objekt` matches the lead's objekt, using normalized comparison (trim + lowercase) on both values *and* column names. Outputs `brokerFound` plus `maklare`, `maklarEmail`, `byra`, `annonslank` (all null when no match). |
| 7a | **Send confirmation to lead** | `gmail` | Plain-text email to the lead, signed with the matched broker + agency (falls back to a neutral signature when no match). |
| 7b | **IF broker found** | `if` | Branches on `brokerFound`. |
| 7b-true | **Send email to broker** | `gmail` | Full lead data to the matched broker — contact details, the lead's own message, and the AI assessment. |
| 7b-false | **Send fallback email to admin** | `gmail` | Same content to the admin address, flagged as unmatched. Doubles as the error log. |
| 7c | **Build lead row** | `code` | Shapes the Leads-tab row. Key order here defines the column order. Broker/ad-link are blank in the fallback case. |
| 7d | **Log lead to sheet** | `googleSheets` (append) | Appends the row to the **Leads** tab. Runs for every lead, matched or not. |

Steps 7a, 7b and 7c run in parallel from "Merge broker lookup".

### Final lead shape

```json
{
  "namn": "string",
  "e-post": "string",
  "telefon": "string | null",
  "objekt": "string",
  "meddelande": "string | null",
  "temperatur": "varmt | kallt",
  "prioritet": "hög | låg",
  "motivering": "string",
  "brokerFound": true,
  "maklare": "string | null",
  "maklarEmail": "string | null",
  "byra": "string | null",
  "annonslank": "string | null"
}
```

---

## Chain B — Form-link generation (schedule-triggered)

Runs every 2 minutes and fills in the "Formulärlänk" column for any property row that lacks one.

| Order | Node | Type | What it does |
|---|---|---|---|
| 1 | **Check for new objects** | `scheduleTrigger` | Every 2 minutes. |
| 2 | **Read objects sheet** | `googleSheets` (read) | Reads all rows of the Objekt tab. |
| 3 | **Build missing links** | `code` | Skips empty rows and rows that already have a link. For the rest, builds `https://tally.so/r/kdjp16?objekt=<objekt>` (spaces encoded as `+`). Writes back the objekt value **exactly as stored**, including any stray whitespace — see "Design notes". |
| 4 | **Write links to sheet** | `googleSheets` (appendOrUpdate) | Upserts on the `Objekt` column, filling in `Formulärlänk`. |

A Google Sheets **Trigger** node was the obvious choice here but needs its own credential type (`googleSheetsTriggerOAuth2Api`), separate from the regular Sheets credential. The schedule-based approach avoids that and has a side benefit: it also picks up rows added while n8n was offline.

---

## Google Sheet structure

Document ID `1gvs2jwwD-WFNNEioheNklxUJmgkOlnifCFwTdE7KET0`, two tabs.

**Objekt tab** (gid `0`) — the property/broker directory, maintained by hand:

| Objekt | Mäklare | Mäklarmejl | Formulärlänk | Byrå | Annonslänk |
|---|---|---|---|---|---|
| property address (match key) | broker name | broker email | auto-generated | agency name | broker pastes their listing URL here |

`Annonslänk` is deliberately platform-agnostic — Hemnet, Idealista, Zillow, an own website, anything. No logic assumes a particular platform; the value is only read and passed through.

Note: the live headers are `Mäklarmejl` (not "Mäklarens mejl") and `Byrå ` (with a trailing space). The code normalizes column names, so neither matters.

**Leads tab** — append-only log, one row per incoming lead:

| Datum | Namn | E-post | Telefon | Objekt | Meddelande | Temperatur | Prioritet | Motivering | Mäklare | Annonslänk |
|---|---|---|---|---|---|---|---|---|---|---|

`Mäklare` and `Annonslänk` are blank when the objekt wasn't found. `Annonslänk` comes from the same lookup that finds the broker — no second read.

---

## Credentials

- **Header Auth account** (`fPkGUQWKqVMiFMON`) — validates `x-api-key` on the webhook.
- **OpenAI account** (`lBbz6qPfrtWoFyLz`) — GPT-4.1 categorization.
- **Google Sheets account** (`hBzYYWrAg76ck0Ue`) — all sheet reads and writes.
- **Gmail account** (`ajeltaiTK7zd8Ffm`) — all three outgoing emails.

---

## Design notes (things learned the hard way)

- **`alwaysOutputData: true` on "Lookup broker in sheet" is load-bearing.** A node that outputs zero items stops its whole downstream branch — even a Code node set to "Run Once for All Items" simply never executes. Without this flag, an unmatched objekt silently produced no fallback email at all. The downstream Code node therefore distinguishes a real match from the empty placeholder item by checking for an actual `Objekt` field, not by array length.
- **Matching is normalized, writes are not.** Comparison trims and lowercases both values and column names, so `"Frejgatan 6 "` matches `"frejgatan 6"`. But Chain B writes the objekt back *verbatim*: an earlier version trimmed it first, `appendOrUpdate` then found no matching row, and appended a duplicate.
- **The Gmail node cannot change the From address** — mail is always sent from the connected account. `emailType` must be set to `text`; the default is `html`, which collapses newlines into one paragraph. `appendAttribution: false` removes n8n's "sent automatically with n8n" footer.
- **The Google Sheets node's `columns` resourceMapper is fragile when built via the API.** Declaring a column in `schema` that doesn't exist in the sheet is rejected outright ("Column names were updated after the node's setup"). To *add* a new column, leave it out of `schema` and let `options.handlingExtraData: "insertInNewColumn"` create it. For writes generally, `mappingMode: "autoMapInputData"` with a preceding Code node that shapes the keys is the reliable pattern — `defineBelow` silently fell back to auto-mapping and once wrote junk into the header row.
- **Phone numbers need `options.cellFormat: "RAW"` on the Leads append.** The default (`USER_ENTERED`) makes Sheets interpret each value as if it had been typed by hand, so `0709876543` was stored as the number `709876543` and lost its leading zero. `RAW` writes values verbatim. The Telefon column is additionally set to a TEXT number format (see below) as a second line of defence. Rows written before this fix keep the broken value — it is not corrected retroactively.

## Sheet presentation (one-time setup, not part of the workflow)

The Leads tab's formatting was applied once by calling the Google Sheets API's `spreadsheets.batchUpdate` directly — n8n's Google Sheets node only reads and writes values, it has no formatting operations. It was done with an HTTP Request node using `authentication: predefinedCredentialType` + `nodeCredentialType: googleSheetsOAuth2Api`, so it reuses the same Google Sheets credential without any extra setup. The throwaway workflow has since been deleted; recreate it from these requests if the sheet ever needs rebuilding:

- `updateSheetProperties` → `frozenRowCount: 1` (header row stays visible while scrolling)
- `repeatCell` on row 0 → bold text, light grey background
- `repeatCell` on all columns → `wrapStrategy: WRAP`, `verticalAlignment: TOP`
- `updateDimensionProperties` per column → explicit pixel widths (Datum 110, Namn 180, E-post 240, Telefon 120, Objekt 180, Meddelande 340, Temperatur 120, Prioritet 110, Motivering 340, Mäklare 170, Annonslänk 240)
- `repeatCell` on column D → `numberFormat: { type: "TEXT" }` so phone numbers keep their leading zero

## Known limitations

- Emails come from the owner's personal Gmail address. Fixing this properly needs an own domain plus a transactional email service (the Gmail node can't override the sender).
- Reachability depends on the local n8n instance being up and the ngrok tunnel running; ngrok's free tier issues a new URL on every restart, which then has to be updated in Tally.
- The Objekt tab is a single shared table. Once there is more than one agency, they would all see each other's rows — that needs either a sheet per agency or a real database.
