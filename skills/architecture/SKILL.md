---
name: architecture
description: >
  Kartan över hela Real Estate AI Agents-systemet — hur subagenterna hänger
  ihop och hur delarna kopplas. Läs denna skill i början av varje session som
  rör systemets helhet eller flera agenter. Uppdatera filen när du ändrar
  arkitekturen (nya agenter, ändrad routing, nya kopplingar).
---

# Arkitektur – Real Estate AI Agents

## Vad systemet är

Ett multi-agent AI-system byggt i **n8n** som automatiserar återkommande
arbete för fastighetsmäklare. Riktar sig till mäklarbyråer brett, inte en
enskild marknad — därför används generiska fält (t.ex. "Annonslänk" istället
för en specifik plattform).

Systemet består av flera **subagenter** som byggs var för sig. En **huvudagent
(orchestrator)** som tar emot arbete och dirigerar det till rätt subagent
byggs sist, varefter subagenterna kopplas in under den.

## Subagenter

### Klara

#### Lead Agent
Tar emot leads via webhook, kategoriserar efter temperatur och prioritet med
GPT, slår upp rätt mäklare i Google Sheet och skickar notis. Fallback till
admin om objektet inte hittas.

n8n-workflow: `Lead agent` (id `WDUPtrstUP9DT8cS`), 16 noder, aktivt.
Innehåller **två oberoende kedjor** med var sin trigger.

**Kedja A – lead som kommer in (webhook)**

| # | Nod | Typ | Gör |
|---|---|---|---|
| 1 | Webhook | webhook | `POST /webhook/lead-agent`, header-auth via `x-api-key` |
| 2 | Extract lead fields | code | Plockar namn, epost, telefon, meddelande, objekt ur Tallys `fields[]` genom att matcha på fältets label |
| 3 | Categorize lead | openAi (GPT-4.1) | Returnerar strikt JSON: temperatur, prioritet, motivering |
| 4 | Format output | code | Tolkar AI-svaret, slår ihop med lead-fälten (inkl. råa meddelandet) |
| 5 | Lookup broker in sheet | googleSheets (read) | Läser **alla** rader i Objekt-fliken. `alwaysOutputData: true` |
| 6 | Merge broker lookup | code | Matchar objektet normaliserat (trim + gemener, även på kolumnnamn). Sätter `brokerFound` + maklare, maklarEmail, byra, annonslank |
| 7a | Send confirmation to lead | gmail | Bekräftelse till spekulanten, signerad med matchad mäklare + byrå |
| 7b | IF broker found | if | Grenar på `brokerFound` |
| 7b→sant | Send email to broker | gmail | Full lead-data till mäklaren, inkl. spekulantens eget meddelande |
| 7b→falskt | Send fallback email to admin | gmail | Samma innehåll till admin, flaggat som omatchat. Fungerar även som fellogg |
| 7c | Build lead row | code | Formar raden till Leads-fliken. Nyckelordningen styr kolumnordningen |
| 7d | Log lead to sheet | googleSheets (append) | Skriver raden till Leads-fliken. Körs för **varje** lead, matchat eller ej |

Steg 7a, 7b och 7c körs parallellt från "Merge broker lookup".

**Kedja B – automatisk formulärlänk (schema)**

| # | Nod | Typ | Gör |
|---|---|---|---|
| 1 | Check for new objects | scheduleTrigger | Var 2:e minut |
| 2 | Read objects sheet | googleSheets (read) | Läser alla rader i Objekt-fliken |
| 3 | Build missing links | code | Hoppar över tomma rader och rader som redan har länk. Bygger `https://tally.so/r/<formId>?objekt=<objekt>` |
| 4 | Write links to sheet | googleSheets (appendOrUpdate) | Upsert på `Objekt`, fyller i `Formulärlänk` |

En Google Sheets **Trigger**-nod hade varit det naturliga valet men kräver en
egen credential-typ (`googleSheetsTriggerOAuth2Api`). Schemat undviker det och
fångar dessutom rader som lagts till medan n8n varit nere.

#### Text Content Agent
Genererar annonsbeskrivningar anpassade per plattform.

n8n-workflow: `Text Content Generator` (id `CBzRhFUBOOljJqOS`), 3 noder, aktivt.

| # | Nod | Typ | Gör |
|---|---|---|---|
| 1 | On form submission1 | formTrigger | n8n-formulär "Objektbeskrivning" — adress, bostadstyp, rum, boarea, pris, valuta, våning, avgift, byggår, detaljer, ton, språk, plattform, emoji |
| 2 | Message a model | openAi (GPT-4.1) | Skriver texten enligt vald plattform (sociala medier / webb / prospekt / alla), ton och språk |
| 3 | Markdown | markdown | Konverterar svaret till HTML |

#### Document Analysis Agent
Analyserar dokument och flaggar risknivåer.

n8n-workflow: heter `Document summary Agent` i n8n (id `y4PQC8qh4QLQOhvM`),
17 noder, aktivt. Namnet i n8n matchar alltså inte agentnamnet.

| # | Nod | Typ | Gör |
|---|---|---|---|
| 1 | On form submission | formTrigger | Formulär "ladda upp dokument": nyckelord + filuppladdning |
| 2 | Switch | switch | Routar på MIME-typ: PDF / TXT / CSV / XLSX, med fallback-utgång |
| 3 | PDF-file / TEXT-file / CSV-file / XLSX-file | extractFromFile | Extraherar text respektive tabelldata |
| 4 | If statment1 / If statment / If statment / If statment2 | if | Kollar att innehållet inte är tomt innan analys |
| 5a | textanalysis(PDF/TXT) | openAi (GPT-4.1) | Sammanfattning, risknivå 🟢🟡🔴, varningar, nyckeldata, åtgärdsförslag |
| 5b | Dataanalysis(CSV/XLSX) | openAi (**GPT-4**) | Motsvarande analys för tabelldata |
| 6 | Markdown / Markdown1 | markdown | Konverterar analysen till HTML |
| 7 | Merge → Tomt dokument | merge + set | Samlar de tomma fallen och sätter felmeddelande |
| 8 | Fel filtyp | set | Felmeddelande för filtyp som inte stöds |

Två kända buggar, ej åtgärdade: formulär utan fil ger ett ohanterat 500-fel
(Switch läser `dokument[0].mimetype` utan att kolla att filen finns), och
MIME-matchningen är exakt, så en giltig fil med avvikande MIME-typ hamnar i
"Fel filtyp". CSV/XLSX-grenen kör dessutom GPT-4 medan resten av systemet
kör GPT-4.1.

### Kommande
- **Scheduling Agent** – håller koll på mäklarens kalender, föreslår/bokar
  visningstider automatiskt utifrån förfrågningar, och skickar
  bekräftelse/påminnelse till spekulant och mäklare.
- **Follow-up Agent** – håller koll på kontakter som redan varit på visning
  eller haft kontakt, skickar automatiska påminnelser till spekulanter
  ("Är du fortfarande intresserad av…"), och flaggar "kalla" leads till
  mäklaren för manuell uppföljning.
- **Broadcast Agent (utskick till egna kontakter)** – skickar meddelanden till
  mäklarens befintliga kontakter (t.ex. via WhatsApp) för nytt objekt, öppet
  hus eller prissänkning. Mäklaren väljer vilka som får utskicket (t.ex. område
  eller prisklass). Personalisering + möjlighet för mottagare att avregistrera sig.

## Byggs sist
- **Huvudagent (orchestrator)** – tar emot inkommande arbete och dirigerar det
  till rätt subagent. Byggs efter att subagenterna är på plats.
- **Dashboard** – samlad översiktsvy för mäklaren. Innehåll bestäms senare.

## Planerad utökning
- **Kalender i Lead Agent** – efter att Scheduling Agent är byggd ska Lead Agent
  få en kalenderfunktion där spekulanten själv kan boka visningstid med mäklaren.
  Mäklaren lägger in sin tillgänglighet, och användaren bokar en ledig tid.
  Byggs **efter** Scheduling Agent.

## Delad infrastruktur

- **Google Sheet** – ett dokument med två flikar:
  - *Objekt-fliken* (routing-tabell som Lead Agent slår upp mot):
    `Objekt | Mäklare | Mäklarmejl | Formulärlänk | Byrå | Annonslänk`
    Objekt är matchningsnyckeln. Formulärlänk fylls i automatiskt av Lead
    Agents kedja B. Annonslänk är plattformsoberoende — mäklaren klistrar
    själv in sin annons-URL oavsett tjänst.
  - *Leads-fliken* (append-only logg, en rad per inkommet lead):
    `Datum | Namn | E-post | Telefon | Objekt | Meddelande | Temperatur | Prioritet | Motivering | Mäklare | Annonslänk`
    Mäklare och Annonslänk är tomma när objektet inte hittades.
- **Tally** – formulär som skickar leads till webhooken. Objektet följer med
  som dold URL-parameter.
- **Gmail** – skickar bekräftelse till spekulanten och notis till mäklaren.
  Kan inte sätta egen avsändaradress; mejlen går från det kopplade kontot.
- **OpenAI (GPT-4.1)** – kategorisering och textgenerering.
- **ngrok** – tunnel som släpper in externa webhook-anrop till lokal n8n
  (port 5678). Gratisversionen ger ny URL vid varje omstart, vilket kräver
  att webhook-URL:en uppdateras i Tally.

---

## Instruktion till agenten

Du har MCP-tillgång till n8n. Använd den så här:

1. **Fyll i de faktiska flödena.** Läs de riktiga workflow-definitionerna från
   n8n och komplettera varje klar subagents nod-för-nod-flöde under respektive
   rubrik ovan. Beskriv verkligheten som den ser ut i n8n — gissa inte.

2. **Håll filen levande.** När du gör en ändring i något workflow, uppdatera
   motsvarande avsnitt här och notera kort vad som ändrades och varför.

3. **Committa aldrig hemligheter.** API-nycklar, webhook-secrets, ngrok-URL:er
   och Google-credentials hör inte hemma i denna fil eller i git.
