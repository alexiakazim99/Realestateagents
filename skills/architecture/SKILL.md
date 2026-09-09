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

Nod-för-nod-detaljerna för varje agent ligger i `docs/`, inte här. Denna fil
beskriver vad agenterna gör och hur de hänger ihop.

| Agent | n8n-workflow | Trigger | Flöde i korthet | Detaljer |
|---|---|---|---|---|
| **Lead Agent** | `Lead agent`, 16 noder | Webhook (Tally) + schema var 2:e min | Tar emot lead → GPT sätter temperatur/prioritet → slår upp mäklaren i Objekt-fliken → bekräftelse till spekulanten, notis till mäklaren (eller admin vid utebliven träff) → loggar raden i Leads-fliken. Den schemalagda kedjan fyller i formulärlänkar för nya objekt. | `docs/lead-agent.md` |
| **Text Content Agent** | `Text Content Generator`, 3 noder | n8n-formulär | Mäklaren fyller i objektdata, ton, språk och plattform → GPT skriver annonstexten → konverteras till HTML. | `docs/text-content-agent.md` |
| **Document Analysis Agent** | `Document summary Agent`, 17 noder | n8n-formulär med filuppladdning | Dokument laddas upp → routas på filtyp (PDF/TXT/CSV/XLSX) → text eller tabelldata extraheras → GPT sammanfattar, sätter risknivå 🟢🟡🔴 och listar varningar → HTML. Egna felgrenar för fel filtyp och tomt dokument. | `docs/document-analysis-agent.md` |

**Att känna till:**
- Document Analysis Agent heter `Document summary Agent` i n8n — namnen matchar
  alltså inte. Den har också två kända, ej åtgärdade buggar (formulär utan fil
  ger 500-fel, och MIME-matchningen är för strikt) och kör GPT-4 i sin
  CSV/XLSX-gren medan resten av systemet kör GPT-4.1.
- Lead Agent är den enda agenten som skriver till Google Sheet. De andra två
  returnerar bara sitt resultat i formuläret.

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
