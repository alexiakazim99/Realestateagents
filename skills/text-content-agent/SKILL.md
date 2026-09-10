---
name: text-content-agent
description: >
  Läs denna skill innan du ändrar Text Content Agent. Den tekniska nod-för-nod-
  beskrivningen finns i docs/text-content-agent.md — läs den för att förstå
  flödet. Denna fil samlar bara vad du måste tänka på när du rör agenten, så
  inget går sönder.
---

# Text Content Agent – att tänka på vid ändringar

Teknisk beskrivning: se `docs/text-content-agent.md`. Denna fil är beteenderegler,
inte en flödesbeskrivning.

## Innan du ändrar
- Agenten är liten — nästan allt värde sitter i **systemprompten**, inte i
  noderna. Läs `prompts/text-content-agent.md` innan du rör något.
- Formulärfälten och prompten hänger ihop. Byter du namn på ett fält i
  formuläret slutar motsvarande uttryck i prompten att fyllas i, tyst och utan
  felmeddelande. Byt alltid på båda ställena samtidigt.

## Testa alltid detta
- **Varje plattformsval.** "Sociala medier", "Webb" och "Prospekt" ger olika
  format och längd, och "Alla" ska ge alla tre under egna rubriker. Testar du
  bara ett val ser du inte om de andra gått sönder.
- **Ett objekt med luckor.** Lämna avgift, våning och detaljer tomma. Tomma fält
  ska hoppas över helt, inte skrivas ut som tomma rader eller hittas på.
- **Språkvalet.** Prompten ska följa fältet, inte formulärets språk.

## Rör inte utan att testa outputen
- **Faktareglerna i prompten.** Prompten förbjuder uttryckligen att hitta på
  fakta, att nämna priset och att värdera avgiften som hög eller låg. Det är
  inte stilistiska önskemål — det är sådant en mäklare kan bli ansvarig för.
  Ändrar du prompten, verifiera att reglerna fortfarande efterlevs i ett
  faktiskt genererat resultat.
- **Markdown-noden.** Den läser modellsvaret från en bestämd sökväg i JSON:en.
  Byter du modell-nod eller nodversion kan svarets struktur ändras, och då blir
  outputen tom utan att något kraschar. Kolla alltid att HTML:en faktiskt
  innehåller text.

## Känsligt
- **Inga hemligheter** i prompt, workflow-export eller commit.
- Följ `done-check` innan du kallar en ändring klar.
