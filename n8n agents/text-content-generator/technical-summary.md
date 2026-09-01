# Text Content Generator (Listing Description) — Technical Summary

**n8n workflow name:** `Text Content Generator`
**Workflow ID:** `CBzRhFUBOOljJqOS`
**Status:** Active (production)
**Model:** GPT-4.1 (`@n8n/n8n-nodes-langchain.openAi`)

## Purpose

Generates ready-to-use property listing descriptions ("objektbeskrivningar") for a broker, in one of three formats (social media, web, prospect/brochure) and in a chosen tone/language, from a small structured intake form. No LLM-based lead qualification or document analysis — purely a content-generation agent.

## Trigger

An n8n **Form Trigger** ("On form submission1") titled "Objektbeskrivning" — the broker fills in a structured form directly (not via an external webhook/Tally). Fields:

| Field | Type | Required | Notes |
|---|---|---|---|
| adress | text | yes | |
| bostadstyp | dropdown | yes | lägenhet / villa / fritidshus / Radhus |
| rum | number | yes | |
| boarea | number | yes | |
| pris | number | yes | |
| valuta | dropdown | yes | SEK / EUR / USD |
| vaning | text | no | |
| avgift | number | no | |
| byggar | number | no | |
| detaljer | textarea | no | free-text extra details |
| ton | dropdown | yes | Modern / Klassisk / Lyxig / Saklig |
| sprak | dropdown | yes | Svenska / Engelska / Spanska |
| plattform | dropdown | yes | sociala medier / webb / prospekt / Alla |
| emoji | dropdown | no | ja / nej |

## Node-by-node flow

| Order | Node | Type | What it does |
|---|---|---|---|
| 1 | **On form submission1** | `n8n-nodes-base.formTrigger` | Entry point; collects the property data form described above. |
| 2 | **Message a model** | `@n8n/n8n-nodes-langchain.openAi` (GPT-4.1) | Sends the form data to GPT-4.1 with a detailed system prompt (see `system-prompts.md`) that enforces: no invented facts, no mention of price, no value judgement on the fee, correct output length/format per selected platform, emoji only if requested, and language matching the "sprak" field. |
| 3 | **Markdown** | `n8n-nodes-base.markdown` | Converts the model's Markdown output (`$json.output[0].content[0].text`) to HTML. This is the current end of the workflow. |

## Credentials used

- **OpenAI account** (`lBbz6qPfrtWoFyLz`) — used by the "Message a model" node.

## Notes

- The system prompt is fact-grounding heavy: it explicitly forbids fabricating details not present in the submitted data, and forbids stating the price or judging the fee as high/low — these are hard behavioral constraints baked into the prompt, not something enforced by the workflow logic itself.
- If "plattform" = "Alla", the model is instructed to produce all three variants (social/web/prospect) in one response, each under its own heading.
