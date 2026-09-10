---
name: document-analysis-agent
description: >
  Läs denna skill innan du ändrar Document Analysis Agent. Den tekniska nod-för-
  nod-beskrivningen finns i docs/document-analysis-agent.md — läs den för att
  förstå flödet. Denna fil samlar bara vad du måste tänka på när du rör agenten,
  så inget går sönder.
---

# Document Analysis Agent – att tänka på vid ändringar

Teknisk beskrivning: se `docs/document-analysis-agent.md`. Denna fil är
beteenderegler, inte en flödesbeskrivning.

## Innan du ändrar
- Agenten har **fyra parallella grenar** (PDF, TXT, CSV, XLSX) plus två
  felgrenar. En ändring i en gren gäller inte automatiskt de andra — läs
  `docs/document-analysis-agent.md` så du vet vilka som berörs.
- Två `If`-noder heter nästan exakt lika ("If statment" och "If statment " med
  avslutande mellanslag). Kontrollera att du redigerar rätt nod innan du sparar.

## Testa alltid detta
- **Minst en textgren och en datagren.** PDF/TXT och CSV/XLSX går till olika
  analysnoder med olika prompter. Testar du bara en vet du inget om den andra.
- **Tomt dokument.** En fil som går att läsa men saknar innehåll ska landa i
  "Tomt dokument" med felmeddelande — inte skickas vidare till modellen.
- **Fel filtyp.** Något som inte är PDF/TXT/CSV/XLSX ska landa i "Fel filtyp".
- **Ingen fil alls.** Ska ge felmeddelande, inte krascha. Detta var tidigare en
  bugg och är värt att verifiera igen efter varje ändring i Switch-noden.

## Rör inte utan att testa outputen
- **Filtyps-routingen i Switch.** Varje gren matchar på **antingen** MIME-typ
  **eller** filändelse. Fallbacken på filändelse finns för att uppladdningar inte
  alltid sätter korrekt MIME-typ — en riktig `.txt` kan komma in som
  `application/octet-stream`. Tar du bort den börjar giltiga filer avvisas som
  "fel filtyp". Noden läser också filen med säker åtkomst så att en saknad fil
  inte kastar fel; behåll det.
- **Modellvalet.** Alla analysgrenar kör GPT-4.1. Tidigare låg CSV/XLSX-grenen
  kvar på GPT-4, vilket gav inkonsekvent kvalitet. Byter du modell någonstans,
  byt på båda ställena så systemet förblir enhetligt.
- **Ansvarsreglerna i prompterna.** Båda prompterna förbjuder uttryckligen
  specifika juridiska och skatterättsliga råd, och hänvisar istället till rätt
  expert. Det är en medveten avgränsning, inte utfyllnad — ändrar du prompten,
  verifiera att den fortfarande håller sig inom den ramen.
- **Risknivåerna.** Analysen ska alltid returnera en flagga (🟢/🟡/🔴) med
  motivering. Ändrar du promptstrukturen, kontrollera att flaggan finns kvar i
  ett faktiskt svar.

## Känsligt
- **Uppladdade dokument kan innehålla personuppgifter och avtalsdata.** Klistra
  aldrig in verkligt dokumentinnehåll i commits, dokumentation eller testfall —
  använd påhittade exempel.
- **Inga hemligheter** i prompt, workflow-export eller commit.
- Följ `done-check` innan du kallar en ändring klar.
