---
name: spec-first
description: >
  Läs och följ denna skill innan du bygger något nytt (en agent, ett workflow,
  en funktion). Bygg inte förrän kraven är utredda och en kort spec är godkänd.
  Syftet är att gå från lös idé till samsyn till en tydlig plan innan arbetet börjar.
---

# Spec-first – reda ut innan du bygger

Innan du bygger något nytt: bygg inte direkt. Ställ först frågor tills det är
tydligt vad som ska göras. Gissa inte kraven — fråga.

## Fråga om detta först

1. **Syfte** – vad ska det här lösa? Vem är det för?
2. **Trigger** – vad startar det? (webhook, formulär, schema, manuellt…)
3. **Input** – vad kommer in, och varifrån? Vilka fält är obligatoriska?
4. **Output** – vad ska hända till slut? (mejl, rad i ett sheet, ett svar…)
5. **Edge cases** – vad kan gå fel? Vad händer vid tom input, ingen träff,
   dubblett, eller när något externt inte svarar?
6. **Klart när…** – hur vet vi att det funkar? Beskriv det så konkret att det
   går att testa.

Ställ en fråga i taget om det blir tydligare så. Anta inget som användaren
inte har sagt.

## Skriv sedan en kort spec

När frågorna är besvarade, sammanfatta i en kort spec:

- **Vad som ska byggas** (en mening)
- **Trigger → input → output**
- **Edge cases som hanteras**
- **Definition of done** (testbar)

Visa specen för användaren och vänta på **godkännande innan du börjar bygga**.
Om något ändras under bygget, uppdatera specen.
