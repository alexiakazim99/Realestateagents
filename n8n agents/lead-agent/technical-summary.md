# Lead Agent — Technical Summary

**n8n workflow name:** `Lead agent`
**Workflow ID:** `WDUPtrstUP9DT8cS`
**Status:** Active (production)
**Model:** GPT-4.1 (`@n8n/n8n-nodes-langchain.openAi`)

## Purpose

Receives leads submitted through the Tally form "Intresseanmälan" and automatically qualifies them by temperature (varmt/kallt) and priority (hög/låg), so the broker doesn't have to manually screen every incoming lead. The notification step to the broker (sending the qualified lead onward) is intentionally not built yet.

## Trigger

Tally posts to this workflow's webhook whenever a lead submits the form. Tally's payload shape (`body.data.fields[]`, an array of `{key, label, type, value}` objects) drives how the "Extract lead fields" node reads the data.

## Node-by-node flow

| Order | Node | Type | What it does |
|---|---|---|---|
| 1 | **Webhook** | `n8n-nodes-base.webhook` | Entry point. `POST /webhook/lead-agent`, authenticated via header auth (`x-api-key`, credential "Header Auth account"). `responseMode: onReceived` — acknowledges Tally immediately, the rest of the workflow keeps running in the background. |
| 2 | **Extract lead fields** | `n8n-nodes-base.code` (JS) | Reads `body.data.fields`, matches each field by its `label` (case-insensitive) to pull out `namn`, `epost`, `telefon`, `meddelande`, `objekt`. Matching by label rather than by Tally's field key makes it resilient to the form being edited/reordered in Tally. |
| 3 | **Categorize lead** | `@n8n/n8n-nodes-langchain.openAi` (GPT-4.1) | Sends `meddelande` + `objekt` to GPT-4.1 with a system prompt (see `system-prompts.md`) instructing it to return strict JSON with `temperatur`, `prioritet`, `motivering`. |
| 4 | **Format output** | `n8n-nodes-base.code` (JS) | Parses the model's JSON response (falls back to `kallt`/`låg`/"Kunde inte tolka AI-svaret." if parsing fails), merges it with the lead fields from step 2, and outputs the final structured object. This is the current end of the workflow. |

## Final output shape

```json
{
  "namn": "string",
  "e-post": "string",
  "telefon": "string | null",
  "objekt": "string | null",
  "temperatur": "varmt | kallt",
  "prioritet": "hög | låg",
  "motivering": "string"
}
```

## Credentials used

- **Header Auth account** (`fPkGUQWKqVMiFMON`) — validates the `x-api-key` header on the incoming webhook.
- **OpenAI account** (`lBbz6qPfrtWoFyLz`) — used by the "Categorize lead" node.

## Known gaps / not yet built

- No notification step to the broker (mäklare) — deferred on purpose, planned as the next node after "Format output".
- No persistence/CRM write of the lead — currently the workflow just produces the structured JSON and stops.
