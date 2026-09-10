---
name: error-handling
description: >
  Läs och följ denna skill när du bygger eller ändrar ett workflow, och när du
  hanterar fel. Varje agent ska fånga sina fel och skriva dem till en gemensam
  logg i samma format — krascha aldrig tyst. Detta är förutsättningen för att
  kunna se och åtgärda fel när systemet är live.
---

# Error-handling – fånga fel, logga enhetligt

Mål: när något går fel ska det synas, inte försvinna. Alla agenter loggar fel
till samma ställe i samma format, så felen går att överblicka och åtgärda.

## Grundregel
- **Krascha aldrig tyst.** Varje workflow ska ha en felgren som fångar fel.
- I n8n: använd en **Error Trigger**-nod som fångar när flödet kraschar, kopplad
  till en nod som skriver felet till loggen.

## Logga till ett gemensamt ställe
- Alla fel skrivs till **Errors-fliken** i projektets Google Sheet.
- Samma format för alla agenter, en rad per fel:
  `Tidpunkt | Agent | Vad som gick fel`
- Sprid inte fel till olika ställen (mejl här, konsol där) — allt till Errors-fliken,
  så en dashboard kan läsa den senare.

## Skriv begripliga fel
- Skriv *vad* som gick fel och *var* (vilken agent, vilket steg) — inte bara "Error".
- Framtida-du ska förstå raden utan att gräva i n8n.

## Känsligt
- **Logga aldrig hemligheter** — inga API-nycklar, webhook-secrets eller
  credentials i loggen.
- **Logga aldrig känslig data i klartext** — särskilt uppladdade dokument och
  lead-uppgifter kan innehålla personuppgifter. Logga *att* något gick fel och
  vilket steg, inte hela innehållet.

## Redan befintlig felhantering
- Vissa agenter har egen felhantering sedan tidigare (t.ex. Lead Agents
  fallback-mejl till admin, Document Analysis Agents grenar för tomt dokument
  och fel filtyp). Behåll dem, men se till att de *också* skriver till
  Errors-fliken så allt samlas på ett ställe.
