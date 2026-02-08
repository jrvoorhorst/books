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
│  - Coördineert personage- EN locatie-interacties    │
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
└─────┬────┘└────┬─────┘└────┬─────┘└────┬─────┘
      │          │           │           │
      ▼          ▼           ▼           ▼
      ┌──────────────────────────────────┐
      │    ◄── betreden / verlaten ──►   │
      └──────────────────────────────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
┌──────────┐┌──────────┐┌──────────┐┌──────────┐
│Lichttuin ││Nevelberg.││Spiegelzl.││Labyrint  │  ...
│  DO      ││  DO      ││  DO      ││  DO      │
│          ││          ││          ││          │
│ Conditie ││ Conditie ││ Conditie ││ Conditie │
│ Sfeer    ││ Sfeer    ││ Sfeer    ││ Sfeer    │
│ Effecten ││ Effecten ││ Effecten ││ Effecten │
│ Beheerder││ Beheerder││ Beheerder││ Eigen wil│
└──────────┘└──────────┘└──────────┘└──────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
┌─────────────────────────────────────────────────────┐
│                   VECTORIZE                          │
│              Semantische zoekindex                   │
│  - Embedding + compacte metadata + r2_key           │
│  - "Vind wat relevant is"                           │
└──────────────────────┬──────────────────────────────┘
                       │ r2_key
                       ▼
┌─────────────────────────────────────────────────────┐
│                      R2                              │
│              Volledige inhoud                        │
│  - herinneringen/{personage}/{id}.json              │
│  - hoofdstukken/deel_{n}/hoofdstuk_{nn}.md          │
│  - locaties/{locatie}/geschiedenis.json             │
│  - personage-context/{personage}/profiel.json       │
└─────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────┐
│                      D1                              │
│              Gestructureerde relaties (SQL)          │
│  - personages: locatie, emotie, status              │
│  - relaties: type, sterkte, richting                │
│  - tijdlijn: gebeurtenissen, volgorde               │
│  - gave_permissies: type, bereik, actief            │
│  - locatie_effecten: type, sterkte, doelwit         │
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

## Opslagarchitectuur: Vectorize + R2 + D1

De drie opslaglagen hebben elk een eigen rol. Ze werken samen als een gelaagd systeem:

```
┌─────────────────────────────────────────────────────────┐
│                    VECTORIZE                             │
│              Semantische zoekindex                       │
│                                                         │
│  Wat:  Embeddings + compacte metadata                   │
│  Doel: "Vind relevante herinneringen/context"           │
│  Limiet: ~10KB metadata per vector                      │
│                                                         │
│  Bevat per vector:                                      │
│  - embedding (semantische representatie)                │
│  - metadata: { personage, type, locatie, tijdstip,      │
│    emotioneleLading, r2_key, samenvatting }              │
│                                                         │
│  De samenvatting is kort genoeg voor metadata,          │
│  de volledige inhoud staat in R2.                        │
└───────────────────────┬─────────────────────────────────┘
                        │ r2_key
                        ▼
┌─────────────────────────────────────────────────────────┐
│                       R2                                 │
│              Volledige inhoud (object storage)            │
│                                                         │
│  Wat:  Ongelimiteerde opslag per object                 │
│  Doel: "Haal de complete tekst/context op"              │
│                                                         │
│  Buckets:                                               │
│  herinneringen/                                         │
│    {personage}/{id}.json     → volledige herinnering    │
│  hoofdstukken/                                          │
│    deel_{n}/hoofdstuk_{nn}.md → geschreven tekst         │
│  locaties/                                              │
│    {locatie}/geschiedenis.json → wat hier is gebeurd    │
│  fragmenten/                                            │
│    {id}.md                   → ruwe ideeën, notities    │
│  personage-context/                                     │
│    {personage}/profiel.json  → volledig system prompt   │
│    {personage}/dagboek.json  → intern perspectief       │
└─────────────────────────────────────────────────────────┘
                        │
                        │ gestructureerde queries
                        ▼
┌─────────────────────────────────────────────────────────┐
│                       D1                                 │
│              Gestructureerde relaties (SQL)               │
│                                                         │
│  Wat:  Relationele data, exacte queries                 │
│  Doel: "Wie is waar? Wat is de status? Wat was de       │
│         volgorde van gebeurtenissen?"                    │
│                                                         │
│  Tabellen:                                              │
│  personages        → naam, locatie_id, emotie, status   │
│  locaties          → naam, type, conditie, beheerder_id │
│  relaties          → personage_a, personage_b, type,    │
│                      sterkte, status                    │
│  tijdlijn          → hoofdstuk, scene, gebeurtenis,     │
│                      tijdstip, betrokkenen              │
│  gave_permissies   → personage_id, gave_type, bereik,   │
│                      actief, beperking                  │
│  locatie_effecten   → locatie_id, effect_type, sterkte, │
│                      doelwit, actief                    │
└─────────────────────────────────────────────────────────┘
```

### Hoe de drie lagen samenwerken

```typescript
// Voorbeeld: Lily betreedt de Lichttuin en het systeem zoekt relevante context

async function bereidSceneVoor(personage: string, locatie: string) {

  // STAP 1: D1 — Exacte state ophalen
  const personageState = await d1.prepare(
    `SELECT * FROM personages WHERE naam = ?`
  ).bind(personage).first();

  const locatieState = await d1.prepare(
    `SELECT * FROM locaties WHERE naam = ?`
  ).bind(locatie).first();

  const relaties = await d1.prepare(
    `SELECT * FROM relaties WHERE personage_a = ?`
  ).bind(personage).all();

  // STAP 2: Vectorize — Semantisch relevante herinneringen zoeken
  const sceneContext = `${personage} betreedt ${locatie}`;
  const embedding = await ai.run("@cf/baai/bge-base-en-v1.5", {
    text: sceneContext
  });

  const relevantHerinneringen = await vectorize.query(embedding.data[0], {
    topK: 5,
    filter: { personage: personage },
    returnMetadata: true
  });

  // STAP 3: R2 — Volledige inhoud ophalen voor de top-resultaten
  const volledigeHerinneringen = await Promise.all(
    relevantHerinneringen.matches.map(async (match) => {
      const r2Key = match.metadata.r2_key;
      const object = await r2.get(r2Key);
      return {
        samenvatting: match.metadata.samenvatting,    // Uit Vectorize
        volledigeInhoud: await object.json(),          // Uit R2
        score: match.score                             // Relevantie
      };
    })
  );

  // Ook locatie-geschiedenis ophalen
  const locatieGeschiedenis = await vectorize.query(embedding.data[0], {
    topK: 3,
    filter: { type: "locatie_gebeurtenis", locatie: locatie },
    returnMetadata: true
  });

  return {
    personage: personageState,
    locatie: locatieState,
    relaties: relaties.results,
    herinneringen: volledigeHerinneringen,
    locatieGeschiedenis: locatieGeschiedenis.matches
  };
}
```

### Wat gaat waar? Vuistregels

| Data | Opslag | Waarom |
|------|--------|--------|
| "Avara gaf me een bloem en ik voelde iets wegglijden..." (500 woorden) | **R2** (inhoud) + **Vectorize** (index) | Te lang voor Vectorize metadata, moet semantisch zoekbaar zijn |
| `{ personage: "Lily", locatie: "Lichttuin", emotie: "twijfel" }` | **D1** | Exacte state, vaak gequeried, moet consistent zijn |
| Volledig geschreven hoofdstuk (5000+ woorden) | **R2** | Grote tekst, opslag |
| "Lily voelt twijfel bij Avara's bloem" (korte samenvatting) | **Vectorize** metadata | Kort genoeg, maakt snelle scanning mogelijk zonder R2-roundtrip |
| Relatie Lily↔Theo: "vriendschap, sterkte 0.8" | **D1** | Gestructureerd, verandert vaak, moet queryable zijn |
| Locatie-conditie Lichttuin: 0.9 → 0.4 → 0.1 | **D1** | Numeriek, per scene bijwerken |
| Arafel's volledige system prompt + backstory | **R2** | Te groot voor DO storage of D1 |
| Gave-permissies per personage | **D1** | Gestructureerd, snel opvraagbaar bij elke interactie |

### Herinnering: Vectorize Entry + R2 Object

```typescript
// Wat in Vectorize staat (compact, zoekbaar)
interface VectorizeEntry {
  id: string;                          // "lily_mem_042"
  values: number[];                    // embedding vector
  metadata: {
    personage: string;                 // "Lily"
    type: "ervaring" | "observatie" | "emotie" | "kennis" | "geheim";
    locatie: string;                   // "Lichttuin"
    tijdstip: string;                  // "deel_1:hoofdstuk_03:scene_2"
    emotioneleLading: number;          // -0.3
    betrokkenen: string;               // "Avara,Lily" (string want metadata)
    samenvatting: string;              // "Avara's bloem voelde verkeerd"
    r2_key: string;                    // "herinneringen/lily/mem_042.json"
  };
}

// Wat in R2 staat (volledig, rijk)
interface R2Herinnering {
  id: string;
  personage: string;
  volledigeInhoud: string;             // De complete herinnering, onbeperkt
  // "Avara gaf me de mooiste bloem die ik ooit had gezien.
  //  De blaadjes waren zo dun dat je het licht erdoorheen kon zien,
  //  als de vleugels van de Lichtlingen in het Fluisterbos. Maar toen
  //  ik haar aannam voelde ik iets... alsof er iets van me werd
  //  weggenomen. Iets kleins. Ik weet niet wat. Avara glimlachte
  //  en ik glimlachte terug, maar mijn vingers trilden een beetje."
  context: {
    watGebeurdeErvoor: string;
    watGebeurdeErna: string;
    innerlijkeMonoloog: string;        // Lily's gedachten op dat moment
  };
  verpijtVanBelangrijkheid: number[];  // Hoe belangrijk was dit over tijd?
  gekoppeldeHerinneringen: string[];   // Links naar gerelateerde herinneringen
}
```

### Voorbeeld: Zoek + Ophaal flow

```
Lily moet reageren in een nieuwe scène bij de Lichttuin:

1. VECTORIZE QUERY
   Zoek: "bloemen" + "Lichttuin" + emotie < 0
   Filter: personage = "Lily"
   Top 3 resultaten:
     → { score: 0.92, samenvatting: "Avara's bloem voelde verkeerd",
         r2_key: "herinneringen/lily/mem_042.json" }
     → { score: 0.85, samenvatting: "Paden in tuin werden smaller",
         r2_key: "herinneringen/lily/mem_058.json" }
     → { score: 0.71, samenvatting: "Mooie jurk maar ongemakkelijk",
         r2_key: "herinneringen/lily/mem_023.json" }

2. R2 OPHAAL (alleen top 1-2, bespaar latency)
   → Volledige tekst van mem_042 en mem_058
   → Nu heeft Lily's DO de rijke context om mee te reageren

3. D1 CHECK
   → Huidige relatie Lily↔Avara: "wantrouwen, sterkte 0.3"
   → Locatie Lichttuin conditie: 0.4 (verwelkend)

4. PERSONAGE-DO REAGEERT
   Met: volledige herinneringen + huidige relatie + locatie-staat
   → Genereert een response vanuit Lily's perspectief die
      gekleurd is door haar eerdere ervaringen
```

### Hoe personages herinneringen gebruiken

Wanneer een personage moet reageren in een scène:
1. De Verteller geeft de huidige scène-context
2. Het personage-DO zoekt in **Vectorize** naar relevante herinneringen (semantisch)
3. De beste matches worden opgehaald uit **R2** (volledige inhoud)
4. Huidige state komt uit **D1** (relaties, locatie, emotie)
5. Deze drie bronnen samen kleuren de reactie
6. Na de scène: nieuwe herinneringen worden geschreven naar **R2** + geïndexeerd in **Vectorize** + state-updates in **D1**

---

## Locatie Durable Objects

Locaties zijn niet alleen decor - ze zijn **levende entiteiten** met eigen staat, gedrag en invloed. Elke belangrijke locatie krijgt een eigen DO.

### Locatie Configuratie

```typescript
interface LocatieConfig {
  naam: string;
  type: "neutraal" | "beschermend" | "manipulatief" | "heilig" | "corrupt";
  beheerder?: string;              // Welk personage-DO heeft invloed?
  eigenWil: boolean;               // Heeft de locatie eigen agency?
  sfeerBasis: string;              // Standaard atmosfeer
  zintuigen: string[];             // Wat kan de locatie waarnemen?
}

interface LocatieState {
  seizoen: string;
  conditie: number;                // 0 (vervallen) tot 1 (bloeiend)
  sfeer: Sfeer;                    // Huidige atmosfeer
  aanwezigen: string[];            // Welke personage-DOs zijn hier?
  actieveEffecten: Effect[];       // Lopende magische effecten
  verborgenElementen: string[];    // Wat is er te ontdekken?
  geschiedenisLaag: number;        // Hoe veel geschiedenis is "zichtbaar"?
}

interface Sfeer {
  licht: "stralend" | "zacht" | "schemerig" | "duister";
  geluid: string;                  // Wat hoor je?
  geur: string;                    // Wat ruik je?
  temperatuur: "warm" | "aangenaam" | "koel" | "koud";
  magischeIntensiteit: number;     // 0-1
}

interface Effect {
  type: "bescherming" | "verleiding" | "verwarring" | "openbaring" | "onderdrukking";
  sterkte: number;
  doelwit?: string;                // Specifiek personage of "iedereen"
  bron: string;                    // Waar komt het effect vandaan?
}
```

### Locatie-DOs voor Stella Aurora

#### De Lichttuin

```typescript
const lichttuin: LocatieConfig = {
  naam: "De Lichttuin",
  type: "manipulatief",
  beheerder: "Avara",
  eigenWil: false,                 // Reageert op Avara's wil
  sfeerBasis: "Overweldigend mooi, bijna te perfect",
  zintuigen: ["aanwezigheid", "emotionele_staat"]
};
```

**Dynamisch gedrag:**
- **Bloemen bloeien** als Avara een doelwit manipuleert - de tuin *helpt* haar
- **Kleuren vervagen** als een bezoeker wantrouwen voelt
- **Geuren worden intenser** naarmate iemand dieper de tuin ingaat
- **Paden verschuiven subtiel** - je kunt altijd dieper in, maar de weg terug is langer
- De tuin heeft een `conditie` die daalt als Avara's macht afneemt

```
LICHTTUIN DO ontvangt: Lily betreedt de tuin
→ Check Lily's emotionele staat (via Verteller)
→ Lily voelt verwondering → bloemen openen zich, kleuren worden feller
→ Lily voelt twijfel → paden worden smaller, schaduwen langer
→ Avara DO ontvangt: "tuin signaleert twijfel bij bezoeker"
→ Avara past strategie aan
```

#### De Nevelbergen

```typescript
const nevelbergen: LocatieConfig = {
  naam: "De Nevelbergen",
  type: "corrupt",
  beheerder: "Arafel",
  eigenWil: true,                  // De nevel heeft EIGEN intentie
  sfeerBasis: "Beklemmend, desoriënterend, verleidelijk fluisterend",
  zintuigen: ["aanwezigheid", "emotionele_staat", "herinneringen", "angsten"]
};
```

**Dynamisch gedrag:**
- **Nevel wordt dichter** naarmate reiziger dieper gaat - niet lineair maar in golven
- **Nevel fluistert** - trekt herinneringen uit personage-DOs en vervormt ze
- **Beschermende mantels verzwakken** - locatie-DO stuurt `onderdrukking`-effect naar personage-DOs
- **Paden veranderen** - de bergen hebben eigen agency, los van Arafel
- **Arafel's macht versterkt** - haar gave-permissies upgraden binnen dit domein

```
NEVELBERGEN DO ontvangt: Lily en Theo betreden de bergen
→ Query naar beide personage-DOs: angsten? herinneringen?
→ Lily's angst: "alleen gelaten worden" → nevel fluistert: "Theo gaat je verlaten"
→ Theo's angst: "niet sterk genoeg zijn" → nevel fluistert: "je kunt haar niet redden"
→ Nevel scheidt hen geleidelijk (eigen agency)
→ Arafel DO ontvangt: "twee reizigers in domein, worden gescheiden"
→ Arafel's gaven upgraden: NEVELZICHT nu volledig actief
```

#### Het Labyrint van Ora

```typescript
const labyrintVanOra: LocatieConfig = {
  naam: "Het Labyrint van Ora",
  type: "neutraal",               // Niet goed, niet kwaad
  beheerder: undefined,            // Niemand beheerst het Labyrint
  eigenWil: true,                  // Sterkste eigen wil van alle locaties
  sfeerBasis: "Oud, onpeilbaar, eigen logica",
  zintuigen: ["aanwezigheid", "bestemming", "waardigheid"]
};
```

**Dynamisch gedrag:**
- **Immuun voor Arafel** - het Labyrint luistert niet naar haar bevelen
- **Kiest eigen paden** voor elke reiziger op basis van hun bestemming
- **Scheidt Lily en Theo** - niet uit kwaadaardigheid maar uit noodzaak
- **Twee paden**: "Het Pad der Doden" (Theo) en "De Oude Reisweg" (Lily)
- **Kompassen werken niet** - het Labyrint overschrijft alle navigatie

```
LABYRINT DO ontvangt: Lily en Theo betreden het labyrint
→ Labyrint heeft EIGEN logica (geen beheerder)
→ Query: wat is de bestemming van elk personage?
  → Lily: moet naar Arafel's toren (haar keuze, haar pad)
  → Theo: moet naar het Schemerland (zijn offer, zijn pad)
→ Paden splitsen - niet te stoppen, niet te manipuleren
→ Arafel DO probeert invloed: GEWEIGERD (labyrint is immuun)
→ Ella DO observeert: kent de uitkomst maar grijpt niet in
```

#### De Spiegelzaal

```typescript
const spiegelzaal: LocatieConfig = {
  naam: "De Spiegelzaal",
  type: "heilig",
  beheerder: "Ella",
  eigenWil: false,                 // Reflecteert Ella's wil
  sfeerBasis: "Ontzagwekkend, eerlijk, troostend en confronterend tegelijk",
  zintuigen: ["alles"]             // De spiegels zien alles
};
```

**Dynamisch gedrag:**
- **Spiegels tonen waarheid** - ongeacht wat een personage probeert te verbergen
- **Licht concentreert** zich op belangrijke momenten
- **Sterren aan het plafond** dimmen of flikkeren bij emotionele intensiteit
- **Echo's versterken** woorden met gewicht en betekenis
- **Wordt kwetsbaar** als Ella ziek wordt - spiegels worden troebel

```
SPIEGELZAAL DO ontvangt: Lily betreedt de zaal voor haar oordeel
→ Query naar Lily DO: ALLE state (geheimen, schuld, herinneringen)
→ Spiegels tonen: elke leugen, elke verkeerde keuze, maar ook...
→ Spiegels tonen ook: elk moment van moed, elk sprankje hoop
→ Ella DO ontvangt: volledige context voor genade-moment
→ Licht in de zaal verandert: van confronterend naar warm
```

#### De Sterrenwacht

```typescript
const sterrenwacht: LocatieConfig = {
  naam: "De Sterrenwacht",
  type: "beschermend",
  beheerder: "Jacob",
  eigenWil: false,
  sfeerBasis: "Rustig, wijs, verbinding tussen aarde en hemel",
  zintuigen: ["sterrenpatronen", "profetische_signalen"]
};
```

**Dynamisch gedrag:**
- **Sterrenkaarten updaten** zich op basis van verhaalgebeurtenissen
- **Kompassen met sterrenlicht-kern** worden hier gemaakt
- **Waarschuwingssignalen** als er gevaar dreigt voor bekende personages
- **Beschermende aura** - manipulatieve effecten van andere locaties werken hier niet

### Locatie-Personage Interactie

Locaties beïnvloeden personages die er binnenkomen, en personages beïnvloeden locaties:

```typescript
// Wanneer een personage een locatie betreedt
async function betreedt(personageDO: PersonageDO, locatieDO: LocatieDO) {
  // 1. Locatie registreert aanwezigheid
  locatieDO.state.aanwezigen.push(personageDO.naam);

  // 2. Locatie past sfeer aan op basis van personage
  const emotie = await personageDO.getEmotioneleStaat();
  const nieuweSfeer = locatieDO.berekenSfeer(emotie);
  locatieDO.state.sfeer = nieuweSfeer;

  // 3. Locatie stuurt effecten naar personage
  for (const effect of locatieDO.state.actieveEffecten) {
    if (effect.doelwit === "iedereen" || effect.doelwit === personageDO.naam) {
      await personageDO.ontvangEffect(effect);
    }
  }

  // 4. Beheerder wordt geïnformeerd (als die er is)
  if (locatieDO.config.beheerder) {
    const beheerder = getBeheerderDO(locatieDO.config.beheerder);
    await beheerder.onBezoeker(personageDO.naam, emotie);
  }

  // 5. Verteller ontvangt locatiebeschrijving voor de scène
  return {
    beschrijving: locatieDO.genereerBeschrijving(personageDO),
    effecten: locatieDO.state.actieveEffecten,
    sfeer: nieuweSfeer
  };
}
```

### Locatie-Evolutie door het Verhaal

Locaties veranderen mee met het verhaal:

```
DEEL 1 (Stella Crepuscula):
  Lichttuin:    conditie 0.9 → bloeiend, verleidelijk
  Nevelbergen:  conditie 0.7 → dreigend maar nog op afstand
  Spiegelzaal:  conditie 1.0 → op volle kracht
  Sterrenwacht: conditie 0.8 → actief, waarschuwend

DEEL 2 (Stella Noctis):
  Lichttuin:    conditie 0.4 → verwelkend, Avara verliest grip
  Nevelbergen:  conditie 0.9 → op volle macht, Arafel triomfeert
  Spiegelzaal:  conditie 0.5 → spiegels worden troebel (Ella ziek)
  Sterrenwacht: conditie 0.6 → sterren dimmen, Jacob bezorgd

DEEL 3 (Stella Aurora):
  Lichttuin:    conditie 0.1 → bijna dood, maar één bloem bloeit nog
  Nevelbergen:  conditie 0.3 → nevel trekt op na Arafel's val
  Spiegelzaal:  conditie 1.0 → hersteld, helderder dan ooit
  Sterrenwacht: conditie 0.9 → sterren stralen, nieuwe sterrenbeelden
```

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
| **Durable Objects** | Personage-agents + Locatie-agents + Verteller |
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
    → Verteller activeert locatie-DO (sfeer, effecten, conditie)
    → Verteller vraagt relevante personage-DOs
    → Personage-DOs "betreden" locatie-DO → ontvangen effecten
    → Locatie-DO reageert op personage-emoties (bloemen bloeien, nevel dikt in)
    → Personage-DOs raadplegen Vectorize (herinneringen)
    → Personage-DOs raadplegen elkaar (via gaven/permissies)
    → Locatie-DO informeert beheerder-DO (Avara weet dat Lily twijfelt)
    → Personage-DOs geven response aan Verteller
    → Locatie-DO geeft sfeer-beschrijving aan Verteller
    → Verteller combineert tot verhaaltekst
    → Workers AI genereert proza
    → Auteur ontvangt tekst + kan feedback geven
    → Nieuwe herinneringen worden opgeslagen in Vectorize
    → Locatie-geschiedenis wordt opgeslagen in Vectorize
    → State-updates in D1 (incl. locatie-conditie)
    → Definitieve tekst naar R2
```

---

## Context Builder: Het Interview-systeem

De Context Builder is de brug tussen de auteur en het systeem. Het werkt in twee modi: **Onboarding** (nieuw personage/locatie opzetten) en **Doorlopend** (kennislacunes detecteren tijdens het schrijven).

### Modus 1: Onboarding — Een personage tot leven brengen

Wanneer je een nieuw personage aanmaakt, start de Context Builder een gesprek dat steeds dieper graaft. Het begint breed en wordt specifieker op basis van je antwoorden.

```
┌─────────────────────────────────────────────────────┐
│              CONTEXT BUILDER                         │
│                                                     │
│  AUTEUR ◄──── gesprek ────► AI INTERVIEWER          │
│                                                     │
│  Antwoorden worden automatisch verwerkt:            │
│                                                     │
│  "ouders maken ruzie"                               │
│    → D1: relatie Lily↔ouders: "gespannen"           │
│    → R2: achtergrond/lily/gezin.json (volledig)     │
│    → Vectorize: "Lily's thuissituatie is onveilig"  │
│                                                     │
│  "ze heeft een zusje"                               │
│    → D1: relatie Lily↔zusje: "beschermend"          │
│    → D1: nieuw personage "zusje" (stub)             │
│    → AI: "Hoe heet haar zusje? Hoe oud is ze?"      │
│                                                     │
│  "ze heet Rosa, 7 jaar"                             │
│    → D1: personage Rosa, leeftijd 7                 │
│    → AI: "Zorgt Lily vaak voor Rosa? Is ze meer     │
│       moeder of vriendin voor haar?"                │
│                                                     │
│  "meer moeder, ze leest haar voor en brengt         │
│   haar naar bed als mama dat niet doet"             │
│    → R2: achtergrond/lily/rol_als_zus.json          │
│    → Vectorize: "Lily neemt moederrol voor Rosa"    │
│    → D1: relatie Lily↔Rosa: "beschermend/moederlijk"│
│    → AI detecteert thematische link:                │
│      "Dit verklaart waarom Lily in Lolaland         │
│       zowel kind wil zijn als verantwoordelijkheid  │
│       voelt. Wil je dat dit conflict doorwerkt      │
│       in hoe ze reageert op Ella's zorg?"           │
└─────────────────────────────────────────────────────┘
```

### Interview-strategieën

De AI kiest vragen op basis van wat het systeem nodig heeft:

```typescript
interface InterviewStrategie {
  type: "breed" | "verdiepend" | "relatie" | "conflict" | "thematisch";
  prioriteit: number;
  context: string;  // Waarom wordt deze vraag gesteld?
}

// De AI kiest uit verschillende vraagtypes:
const vraagStrategieen = {

  // BREED — Begin, grote lijnen
  breed: [
    "Vertel me over de achtergrond van {personage}.",
    "Waar komt {personage} vandaan?",
    "Wat is het eerste dat je denkt bij {personage}?"
  ],

  // VERDIEPEND — Inzoomen op een antwoord
  verdiepend: [
    "Je zei dat {detail}. Kun je daar meer over vertellen?",
    "Hoe voelt {personage} zich daarbij?",
    "Wanneer begon dat?"
  ],

  // RELATIE — Connecties tussen personages
  relatie: [
    "Hoe staat {personage_a} tegenover {personage_b}?",
    "Weet {personage_b} hoe {personage_a} zich voelt?",
    "Wat zou {personage_a} opofferen voor {personage_b}?"
  ],

  // CONFLICT — Innerlijke spanning
  conflict: [
    "Wat wil {personage} het liefst maar durft niet?",
    "Waar schaamt {personage} zich voor?",
    "Wat is de leugen die {personage} zichzelf vertelt?"
  ],

  // THEMATISCH — Koppeling aan verhaalthema's
  thematisch: [
    "Hoe raakt {detail} aan het thema van {thema}?",
    "Wil je dat dit doorwerkt in {verhaallijn}?",
    "Ik zie een parallel met {ander_personage}. Is dat bewust?"
  ]
};
```

### Modus 2: Doorlopend — Kennislacunes detecteren

Dit is de kern: **het systeem weet wat het niet weet**. Tijdens het schrijven van een scène detecteert het DO wanneer het onvoldoende context heeft.

```typescript
interface KennisLacune {
  personage: string;
  onderwerp: string;
  waaromNodig: string;          // Welke scène triggert dit?
  urgentie: "blokkerend" | "verrijkend" | "optioneel";
  suggestieVraag: string;       // Voorgestelde vraag aan auteur
}

// Het personage-DO detecteert lacunes bij het genereren van een response
async function detecteerLacunes(
  personage: string,
  sceneContext: Scene
): Promise<KennisLacune[]> {

  const lacunes: KennisLacune[] = [];

  // Check: heeft dit personage relevante herinneringen voor deze scène?
  const relevanteHerinneringen = await zoekHerinneringen(personage, sceneContext);

  if (relevanteHerinneringen.length === 0) {
    lacunes.push({
      personage,
      onderwerp: sceneContext.thema,
      waaromNodig: `Scène in ${sceneContext.locatie}`,
      urgentie: "blokkerend",
      suggestieVraag: `Ik heb nog geen context over hoe ${personage} zich voelt bij ${sceneContext.thema}. Kun je me daar meer over vertellen?`
    });
  }

  // Check: zijn er onbekende relaties die in deze scène relevant zijn?
  for (const anderPersonage of sceneContext.aanwezigen) {
    const relatie = await d1.prepare(
      `SELECT * FROM relaties WHERE personage_a = ? AND personage_b = ?`
    ).bind(personage, anderPersonage).first();

    if (!relatie) {
      lacunes.push({
        personage,
        onderwerp: `relatie met ${anderPersonage}`,
        waaromNodig: `${personage} en ${anderPersonage} zijn samen in scène`,
        urgentie: "blokkerend",
        suggestieVraag: `Hoe kent ${personage} ${anderPersonage}? Wat voelt ${personage} bij ${anderPersonage}?`
      });
    }
  }

  return lacunes;
}
```

### Voorbeelden: Doorlopende Context-vragen

```
── VOORBEELD 1: Ontbrekende familiecontext ──────────

AUTEUR: "Schrijf een scène waarin Lily in Lolaland een klein meisje
         ziet dat verloren is, en Lily denkt aan haar zusje."

SYSTEEM detecteert:
  → Lily DO zoekt herinneringen over "zusje"
  → Vectorize: 0 resultaten
  → D1: relatie Lily↔zusje: NIET GEVONDEN

CONTEXT BUILDER:
  "Voor deze scène heb ik nog onvoldoende kennis van Lily
   als grote zus. Een paar vragen:
   - Heeft Lily een zusje? Hoe heet ze en hoe oud is ze?
   - Zorgt Lily voor haar? Op welke manier?
   - Mist Lily haar zusje in Lolaland, of is het juist een
     opluchting om even niet verantwoordelijk te zijn?"

AUTEUR: "Ze heet Rosa, 7 jaar. Lily brengt haar naar bed als
         mama dat niet doet. Ze mist haar heel erg maar voelt
         zich ook schuldig dat ze soms blij is even weg te zijn."

SYSTEEM verwerkt:
  → D1: INSERT personage Rosa (7 jaar, Lily's zusje)
  → D1: INSERT relatie Lily↔Rosa (beschermend/moederlijk)
  → R2: achtergrond/lily/gezin.json (volledige context)
  → Vectorize: 3 nieuwe entries
    - "Lily zorgt voor Rosa als moederfiguur"
    - "Lily mist Rosa in Lolaland"
    - "Lily voelt schuld over opluchting weg van thuis"

→ NU kan de scène geschreven worden met echte emotionele diepte.


── VOORBEELD 2: Ontbrekende locatiedetails ──────────

AUTEUR: "Lily loopt door de Lichttuin in de herfst."

LICHTTUIN DO detecteert:
  → Seizoensinformatie: NIET GEVONDEN
  → Heeft de tuin seizoenen? Of is het altijd zomer?

CONTEXT BUILDER:
  "De Lichttuin kent in mijn huidige informatie geen seizoenen.
   Heeft de tuin seizoenen, of is het er altijd hetzelfde?
   - Als er seizoenen zijn: wat gebeurt er in de herfst?
     Vallen de blaadjes, of verwelken de bloemen anders?
   - Of is de tuin altijd in bloei - en is dat juist
     onderdeel van de onnatuurlijke perfectie?"

AUTEUR: "De tuin is altijd in bloei - maar als Avara's macht
         afneemt beginnen er voor het eerst blaadjes te vallen.
         Dat is een teken dat er iets verandert."

SYSTEEM verwerkt:
  → R2: locaties/lichttuin/seizoenen.json
  → Vectorize: "Lichttuin kent geen seizoenen, altijd in bloei,
                bladval = teken van Avara's verzwakking"
  → D1: UPDATE locaties SET seizoenen = false WHERE naam = 'Lichttuin'


── VOORBEELD 3: Thematische verdieping ──────────────

AUTEUR: "Schrijf de scène waarin Theo Lily's schuld overneemt."

THEO DO + VERTELLER detecteren:
  → Dit is het kernmoment van het hele verhaal
  → Christus-typologie is vastgelegd maar...
  → Hoe precies? Wat zegt Theo? Wat voelt hij?

CONTEXT BUILDER:
  "Dit is het hart van het verlossingsthema. Ik heb de
   theologische achtergrond, maar voor deze scène heb ik
   jouw visie nodig op een paar cruciale punten:

   - Weet Theo vooraf wat er gaat gebeuren, of is het
     een spontane keuze in het moment?
   - Wat zijn zijn exacte woorden? Of wil je dat ik
     voorstellen doe?
   - Voelt Theo angst, of is er vrede?
   - Ziet Lily wat er gebeurt, of begrijpt ze het pas later?
   - Hoe fysiek is de schuldovername? Is het zichtbaar,
     of innerlijk?"

→ De vragen worden steeds dieper en specifieker
  naarmate het moment belangrijker is.
```

### Completeness Score

Het systeem houdt per personage/locatie bij hoe "compleet" de context is:

```typescript
interface CompletenessScore {
  entiteit: string;
  scores: {
    achtergrond: number;       // 0-1: familie, verleden, motivatie
    relaties: number;          // 0-1: connecties met andere personages
    innerlijkLeven: number;    // 0-1: angsten, verlangens, conflicten
    spraak: number;            // 0-1: hoe praat dit personage?
    thematisch: number;        // 0-1: hoe past dit in de verhaalthema's
    sensorisch: number;        // 0-1: (voor locaties) hoe voelt/ruikt/klinkt het?
  };
  totaal: number;
  lacunes: string[];           // Concrete ontbrekende onderdelen
}

// Voorbeeld
const lilyCompleteness: CompletenessScore = {
  entiteit: "Lily",
  scores: {
    achtergrond: 0.7,          // Gezinssituatie nu bekend, school nog niet
    relaties: 0.8,             // Theo, Ella, Avara goed, Rosa nieuw
    innerlijkLeven: 0.6,       // Schuld en verlangen bekend, angsten deels
    spraak: 0.4,               // Weten we hoe ze praat? Dialect? Stijl?
    thematisch: 0.9,           // Goed gekoppeld aan verhaanthema's
    sensorisch: 0.0            // n.v.t. voor personages
  },
  totaal: 0.68,
  lacunes: [
    "Hoe praat Lily? Gebruikt ze lange zinnen of korte?",
    "Heeft Lily vriendinnen op school of is ze echt alleen?",
    "Wat zijn Lily's specifieke angsten? (niet alleen 'alleen zijn')"
  ]
};
```

### UI Concept

```
┌─────────────────────────────────────────────────────┐
│  STELLA AURORA — Context Builder                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Personages          Locaties         Verhaallijnen │
│  ├─ Lily     [68%]   ├─ Lichttuin [82%]            │
│  ├─ Theo     [55%]   ├─ Nevelbergen [71%]          │
│  ├─ Ella     [73%]   ├─ Spiegelzaal [60%]          │
│  ├─ Arafel   [61%]   ├─ Labyrint [45%]             │
│  ├─ Avara    [50%]   └─ Sterrenwacht [38%]         │
│  ├─ Jacob    [30%]                                  │
│  └─ Rosa     [15%] ← NIEUW                        │
│                                                     │
├─────────────────────────────────────────────────────┤
│  💬 Interview: Rosa                                 │
│                                                     │
│  AI: Je vertelde dat Rosa 7 jaar is en dat Lily     │
│      voor haar zorgt. Ik wil Rosa beter begrijpen   │
│      als eigenstandig personage:                    │
│                                                     │
│      - Weet Rosa dat de thuissituatie niet normaal  │
│        is? Of beschermt Lily haar daartegen?        │
│      - Heeft Rosa een eigen fantasiewereld, of is   │
│        het juist een nuchter kind?                  │
│      - Komt Rosa ooit in Lolaland?                  │
│                                                     │
│  Jij: Rosa weet het wel maar ze doet alsof ze het   │
│       niet weet. Ze doet dat voor Lily, zodat Lily  │
│       niet nog meer zorgen heeft. ze is eigenlijk... │
│                                                     │
│  [Verstuur]                                         │
└─────────────────────────────────────────────────────┘
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

1. **Proof of Concept**: Begin met 2 personages (Lily + Avara), 1 locatie (Lichttuin) en de Verteller
2. **Herinnering-systeem**: Seed Vectorize met bestaande verhaalfragmenten
3. **Eerste scène**: Laat het systeem de Lichttuin-scène genereren - de tuin reageert op Lily's emoties
4. **Locatie-interactie**: Test hoe de Lichttuin Avara informeert over Lily's twijfels
5. **Uitbreiding**: Voeg Nevelbergen + Arafel toe, test domein-gebonden gaven
6. **Labyrint**: Implementeer het Labyrint van Ora als locatie met eigen wil
7. **Integratie**: Koppel aan bestaande hoofdstukken in `deel_1/`
