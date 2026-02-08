# Stella Aurora - AI Agent Architectuur

## Concept

Ieder personage in Stella Aurora wordt een autonome AI-agent, gehost als Cloudflare Durable Object. Elk personage heeft eigen herinneringen, doelen, persoonlijkheid en beperkingen. Een **Verteller-agent** orkestreert het verhaal samen met de auteur.

De kern van het idee: **bovennatuurlijke gaven worden permissiemodellen** voor inter-agent communicatie.

---

## Architectuuroverzicht

```
┌─────────────────────────────────────────────────────┐
│                    AUTEUR (UI)                       │
│         Geeft richting, maakt keuzes,                │
│         schrijft mee met de Verteller                │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│              VERTELLER DO (Orchestrator)             │
│  - Beheert scène-context en voortgang               │
│  - Coördineert personage-interacties                │
│  - Bewaakt narratieve consistentie                  │
│  - Genereert verhaaltekst samen met auteur          │
│  - Bepaalt welke personages "in scène" zijn         │
└──────┬──────────┬──────────┬──────────┬─────────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
┌──────────┐┌──────────┐┌──────────┐┌──────────┐
│ Lily DO  ││ Theo DO  ││ Ella DO  ││ Arafel DO│  ...
│          ││          ││          ││          │
│ Emotie   ││ Emotie   ││ Emotie   ││ Emotie   │
│ Locatie  ││ Locatie  ││ Locatie  ││ Locatie  │
│ Doelen   ││ Doelen   ││ Doelen   ││ Doelen   │
│ Gaven    ││ Gaven    ││ Gaven    ││ Gaven    │
└──────────┘└──────────┘└──────────┘└──────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
┌─────────────────────────────────────────────────────┐
│                   VECTORIZE                          │
│  Herinneringen per personage (semantisch zoekbaar)  │
│  - Ervaringen & gebeurtenissen                      │
│  - Relatie-geschiedenis                             │
│  - Emotionele indrukken                             │
└─────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│                   D1 / R2                            │
│  D1: Gestructureerde data                           │
│  - Tijdlijn, locaties, relatie-status               │
│  - Scene-metadata, hoofdstukindeling                │
│  R2: Opslag                                         │
│  - Geschreven hoofdstukken                          │
│  - Gegenereerde verhaaltekst                        │
└─────────────────────────────────────────────────────┘
```

---

## Personage Durable Object

Elk personage-DO bevat een **system prompt** die het karakter definieert, plus dynamische state.

### Vaste Configuratie (per personage)

```typescript
interface PersonageConfig {
  naam: string;
  kernIdentiteit: string;       // Wie ben ik fundamenteel?
  persoonlijkheid: string[];    // Karaktertrekken
  spreekstijl: string;         // Hoe praat dit personage?
  moraalKompas: string;        // Waar staat dit personage moreel?
  gaven: Gave[];               // Bovennatuurlijke vermogens
  beperkingen: string[];       // Wat kan dit personage NIET?
}
```

### Dynamische State (verandert gedurende verhaal)

```typescript
interface PersonageState {
  locatie: Locatie;
  emotioneleStaat: EmotioneleStaat;
  actieveDoelen: Doel[];
  relaties: Map<string, RelatieStatus>;
  geheimen: string[];          // Wat weet dit personage dat anderen niet weten?
  innerlijkConflict: string;   // Huidige worsteling
}
```

---

## Bovennatuurlijke Gaven als Permissiemodel

Dit is de kern van het systeem. Elke gave bepaalt welke informatie een personage-DO mag opvragen bij andere DOs.

### Gave-types

```typescript
type GaveType =
  | "ALWETEND"          // Kan alle state van alle DOs lezen
  | "ZIENER"            // Kan locatie van specifieke DOs opvragen
  | "EMPATH_NABIJ"      // Kan emotie lezen van DOs op zelfde locatie
  | "EMPATH_VER"        // Kan emotie lezen ongeacht locatie
  | "FLUISTERAAR"       // Kan gedachten lezen op korte afstand
  | "NEVELZICHT"        // Kan alleen zien in eigen domein (Arafel)
  | "SPIEGELZICHT"      // Kan waarheid zien door illusies heen (Ella)
  | "NORMAAL"           // Alleen directe interactie, geen extra waarneming
  ;

interface Gave {
  type: GaveType;
  bereik?: number;        // Optioneel: maximale afstand
  beperking?: string;     // Bijv: "alleen in de Nevelbergen"
  kosten?: string;        // Bijv: "kost energie", "veroorzaakt hoofdpijn"
}
```

### Mapping naar Personages

| Personage | Gave | Bereik | Beperking |
|-----------|------|--------|-----------|
| **Ella** | `ALWETEND` + `SPIEGELZICHT` | Onbeperkt | Wordt zwakker als zij ziek wordt |
| **Arafel** | `NEVELZICHT` + `EMPATH_VER` | Nevelbergen (vol), daarbuiten (beperkt) | Buiten eigen domein alleen subtiel |
| **Avara** | `EMPATH_NABIJ` + `FLUISTERAAR` | Lichttuin en directe omgeving | Alleen effectief bij emotioneel kwetsbare personen |
| **Theo** | `EMPATH_NABIJ` | Korte afstand | Voelt stemmingen maar kan ze niet duiden |
| **Lily** | `NORMAAL` → ontwikkelt | Groeit gedurende verhaal | Begint zonder gaven, ontwikkelt intuïtie |
| **Jacob** | `ZIENER` (via sterrenkaarten) | Beperkt tot wat sterren tonen | Niet real-time, meer profetisch |

### Technische Implementatie

```typescript
// In het Personage DO
async function queryAnderPersonage(
  doelPersonage: string,
  informatieType: "locatie" | "emotie" | "gedachten" | "geheimen"
): Promise<QueryResultaat | null> {

  const gave = this.config.gaven.find(g =>
    this.kanOpvragen(g, informatieType, doelPersonage)
  );

  if (!gave) {
    return null; // Geen toestemming - dit personage weet het simpelweg niet
  }

  // Check bereik
  if (gave.bereik && !this.binnenBereik(doelPersonage, gave.bereik)) {
    return null; // Te ver weg
  }

  // Check beperkingen
  if (gave.beperking && !this.voldoetAanBeperking(gave)) {
    return null;
  }

  // Vraag het andere DO op
  const anderDO = this.env.PERSONAGES.get(
    this.env.PERSONAGES.idFromName(doelPersonage)
  );

  return await anderDO.fetch(`/query/${informatieType}`);
}
```

---

## Herinneringen in Vectorize

Elk personage heeft een eigen namespace in Vectorize. Herinneringen worden opgeslagen als embeddings zodat het personage **semantisch relevante herinneringen** kan ophalen.

### Herinnering Structuur

```typescript
interface Herinnering {
  id: string;
  personage: string;           // Eigenaar van de herinnering
  type: "ervaring" | "observatie" | "emotie" | "kennis" | "geheim";
  inhoud: string;              // De herinnering zelf
  tijdstip: VerhaalTijdstip;   // Wanneer in het verhaal
  locatie: string;             // Waar het plaatsvond
  betrokkenPersonages: string[];
  emotioneleLading: number;    // -1 (pijnlijk) tot 1 (vreugdevol)
  belangrijkheid: number;      // 0-1, hoe belangrijk voor dit personage
}
```

### Voorbeeld: Lily's herinneringen

```
Query: "Avara" + "bloemen" + "ongemakkelijk gevoel"
→ Herinnering: "Avara gaf me de mooiste bloem die ik ooit had gezien.
   Maar toen ik haar aannam voelde ik iets... alsof er iets van me
   werd weggenomen. Iets kleins. Ik weet niet wat."

Query: "Theo" + "vertrouwen"
→ Herinnering: "Theo keek me aan bij de sterrenwacht en zei dat
   sommige sterren alleen zichtbaar zijn als het echt donker is.
   Ik begreep niet wat hij bedoelde, maar het voelde belangrijk."
```

### Hoe personages herinneringen gebruiken

Wanneer een personage moet reageren in een scène:
1. De Verteller geeft de huidige scène-context
2. Het personage-DO zoekt in Vectorize naar relevante herinneringen
3. Deze herinneringen kleuren de reactie (een personage met pijnlijke herinneringen aan een plek reageert anders)
4. Nieuwe ervaringen worden als herinneringen opgeslagen

---

## De Verteller DO

De Verteller is het hart van het systeem - de orkestrator die samenwerkt met de auteur.

### Verantwoordelijkheden

```typescript
interface VertellerState {
  huidigHoofdstuk: number;
  huidigeScene: Scene;
  actievePersonages: string[];     // Wie is "in scène"?
  narratieveDraad: string;         // Waar gaat het verhaal naartoe?
  stijlContext: string;            // Uit STIJL.md
  auteurInstructies: string;       // Wat wil de auteur bereiken?
}

interface Scene {
  locatie: Locatie;
  tijdstip: VerhaalTijdstip;
  sfeer: string;
  spanning: number;                // 0-1
  actieveConflicten: string[];
  onthullingen: string[];          // Wat wordt in deze scène onthuld?
}
```

### Workflow: Een scène schrijven

```
1. AUTEUR → VERTELLER
   "Ik wil een scène waarin Lily voor het eerst de Lichttuin betreedt
    en Avara ontmoet. Lily voelt zich aangetrokken maar ook ongemakkelijk."

2. VERTELLER analyseert:
   - Welke personages zijn betrokken? → Lily, Avara
   - Waar speelt het? → Lichttuin
   - Welke herinneringen zijn relevant?
   - Wat zijn de doelen van elk personage?

3. VERTELLER → LILY DO
   "Je betreedt de Lichttuin. Wat zie je? Wat voel je?"
   → Lily DO zoekt herinneringen, checkt emotionele staat
   → Antwoord: verwondering, maar ook vaag ongemak

4. VERTELLER → AVARA DO
   "Lily betreedt jouw tuin. Wat is je strategie?"
   → Avara DO checkt: EMPATH_NABIJ → leest Lily's emotionele staat
   → Avara weet: Lily voelt zich alleen, zoekt schoonheid
   → Strategie: gebruik dit verlangen, bied bloemen aan

5. VERTELLER genereert scènetekst
   - Gebruikt Lily's perspectief (STIJL.md: 13-jarig perspectief)
   - Weeft beide personage-reacties samen
   - Laat Avara's manipulatie subtiel doorschemeren
   - Presenteert aan auteur voor feedback

6. AUTEUR geeft feedback → VERTELLER past aan → herhaal
```

---

## Scène-interactie: Gaven in Actie

### Voorbeeld 1: Arafel bespioneert vanuit de Nevelbergen

```
VERTELLER: Lily is in de Lichttuin bij Avara.

ARAFEL DO ontvangt update van Verteller.
→ Arafel heeft NEVELZICHT + EMPATH_VER
→ Query naar Lily DO: emotionele staat?
  → EMPATH_VER: ✓ (mag op afstand)
  → Resultaat: "onzekerheid, verlangen, begin van vertrouwen in Avara"
→ Query naar Lily DO: exacte locatie?
  → NEVELZICHT: ✗ (buiten Nevelbergen, beperkt zicht)
  → Resultaat: vaag beeld, weet dat ze in de Lichttuin is maar niet precies waar
→ Arafel's interne reactie: "Het meisje opent zich. Goed. Avara doet haar werk."

Dit kleurt Arafel's gedrag in latere scènes zonder dat Lily
weet dat ze wordt bespioneerd.
```

### Voorbeeld 2: Theo voelt dat er iets mis is

```
VERTELLER: Lily en Theo lopen samen door het Fluisterbos.

THEO DO:
→ Heeft EMPATH_NABIJ
→ Query naar Lily DO: emotionele staat?
  → EMPATH_NABIJ: ✓ (zelfde locatie)
  → Resultaat: "verdriet vermengd met schuldgevoel, iets dat ze verbergt"
→ Theo weet niet WAT ze verbergt (geen FLUISTERAAR gave)
→ Theo's reactie: "Er is iets. Ik voel het. Maar ik kan haar niet dwingen
   het te vertellen. Ik kan alleen... er zijn."

→ Dit genereert dialoog die past bij Theo's karakter:
  empathisch maar respectvol, bezorgd maar niet opdringerig.
```

### Voorbeeld 3: Ella's alwetendheid vs. genade

```
ELLA DO:
→ Heeft ALWETEND + SPIEGELZICHT
→ Kan ALLES opvragen van alle personages
→ Weet dat Lily naar Avara gaat
→ Weet dat Arafel via Avara manipuleert
→ Ziet door alle illusies heen

MAAR: Ella's karakter definieert dat zij niet ingrijpt
tenzij erom gevraagd wordt (thema: vrije wil en genade).

→ Dit creëert de theologische spanning:
  Ella WEET alles maar HANDELT niet altijd.
  Haar gaven zijn niet beperkt door het systeem,
  maar door haar eigen morele keuze.
```

---

## Verhaalontwikkeling: Lily's Groeiende Gaven

Een bijzonder aspect: Lily's gaven ontwikkelen zich gedurende het verhaal.

```typescript
// Lily's gave-evolutie
const lilyGavenPerDeel = {
  deel_1: {
    // Stella Crepuscula - nog geen gaven
    gaven: [{ type: "NORMAAL" }],
    beschrijving: "Lily heeft geen bijzondere waarneming. Ze merkt dingen
                   op als gewoon meisje - soms een vaag gevoel dat iets
                   niet klopt, maar ze kan het niet duiden."
  },

  deel_2: {
    // Stella Noctis - begin van donker bewustzijn
    gaven: [{
      type: "EMPATH_NABIJ",
      beperking: "alleen negatieve emoties",
      kosten: "veroorzaakt angst en verwarring"
    }],
    beschrijving: "In de Nevelbergen begint Lily de duisternis in anderen
                   te voelen. Maar het is geen gave - het voelt als een
                   vloek. Ze voelt Arafel's invloed maar kan er niet
                   aan ontsnappen."
  },

  deel_3: {
    // Stella Aurora - getransformeerde waarneming
    gaven: [{
      type: "SPIEGELZICHT",
      beperking: "alleen na Ella's aanraking",
      kosten: "confronterend - ziet ook eigen waarheid"
    }],
    beschrijving: "Na de verlossing kan Lily door illusies heen kijken.
                   Maar de gave is tweesnijdend: ze ziet ook haar eigen
                   fouten en gebrokenheid. Het is genade die pijn doet."
  }
};
```

---

## Technische Stack

### Cloudflare Services

| Service | Gebruik |
|---------|---------|
| **Durable Objects** | Personage-agents + Verteller |
| **Vectorize** | Herinneringen (semantisch zoekbaar) |
| **Workers AI** | LLM-aanroepen voor personage-responses |
| **D1** | Tijdlijn, relaties, locaties, metadata |
| **R2** | Geschreven hoofdstukken, gegenereerde tekst |
| **Workers** | API-laag, routing, auteur-interface |

### Data Flow

```
Auteur schrijft instructie
    → Workers API
    → Verteller DO
    → Verteller vraagt relevante personage-DOs
    → Personage-DOs raadplegen Vectorize (herinneringen)
    → Personage-DOs raadplegen elkaar (via gaven/permissies)
    → Personage-DOs geven response aan Verteller
    → Verteller combineert tot verhaaltekst
    → Workers AI genereert proza
    → Auteur ontvangt tekst + kan feedback geven
    → Nieuwe herinneringen worden opgeslagen in Vectorize
    → State-updates in D1
    → Definitieve tekst naar R2
```

---

## Toekomstige Mogelijkheden

### "Wat als" scenarios
Omdat elk personage een autonome agent is, kun je scenario's simuleren:
- "Wat als Lily WEL naar Theo had geluisterd in hoofdstuk 3?"
- Fork de state, laat de agents opnieuw reageren met andere input

### Personage-dagboeken
Elk personage-DO kan een "dagboek" bijhouden - hun eigen perspectief op de gebeurtenissen. Dit geeft de auteur inzicht in hoe personages de wereld ervaren.

### Consistentie-bewaking
Het systeem kan automatisch waarschuwen bij inconsistenties:
- "Lily refereert aan een herinnering die ze niet heeft"
- "Arafel gebruikt een gave buiten haar bereik"
- "Theo is op twee locaties tegelijk"

### Meerdere schrijvers
Het permissiemodel maakt het mogelijk dat meerdere auteurs aan het verhaal werken, elk vanuit een ander personage-perspectief.

---

## Volgende Stappen

1. **Proof of Concept**: Begin met 2 personages (Lily + Avara) en de Verteller
2. **Herinnering-systeem**: Seed Vectorize met bestaande verhaalfragmenten
3. **Eerste scène**: Laat het systeem de Lichttuin-scène genereren
4. **Iteratie**: Voeg personages toe, verfijn gave-permissies
5. **Integratie**: Koppel aan bestaande hoofdstukken in `deel_1/`
