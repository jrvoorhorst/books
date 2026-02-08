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
│PersonaDO ││PersonaDO ││PersonaDO ││PersonaDO │  ...
│config:   ││config:   ││config:   ││config:   │
│  Lily    ││  Theo    ││  Ella    ││  Arafel  │
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
│LocatieDO ││LocatieDO ││LocatieDO ││LocatieDO │  ...
│config:   ││config:   ││config:   ││config:   │
│ Lichttuin││ Nevelbrg ││ Spiegelzl││ Labyrint │
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

## Twee DO-klassen: PersonageDO en LocatieDO

Het hele systeem draait op slechts **drie DO-klassen**:

```typescript
// wrangler.toml
[[durable_objects.bindings]]
name = "PERSONAGES"
class_name = "PersonageDO"    // Eén klasse voor ALLE personages

[[durable_objects.bindings]]
name = "LOCATIES"
class_name = "LocatieDO"      // Eén klasse voor ALLE locaties

[[durable_objects.bindings]]
name = "VERTELLER"
class_name = "VertellerDO"    // Eén instantie
```

Lily, Theo, Arafel, Avara - zijn allemaal dezelfde `PersonageDO` klasse.
De Lichttuin, Nevelbergen, Spiegelzaal - allemaal dezelfde `LocatieDO` klasse.
Het verschil zit volledig in de **configuratie**.

### PersonageDO — Eén klasse, oneindig veel personages

```typescript
export class PersonageDO extends DurableObject {
  private config: PersonageConfig | null = null;
  private state: PersonageState | null = null;

  // Bij eerste aanroep: laad config uit R2
  async initialize(naam: string) {
    // Config komt uit R2 — daar staat het volledige profiel
    const profiel = await this.env.R2.get(
      `personage-context/${naam}/profiel.json`
    );
    this.config = await profiel.json<PersonageConfig>();

    // Dynamische state uit D1
    this.state = await this.laadState(naam);
  }

  // Elke request wordt afgehandeld op basis van config
  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/reageer":
        // Genereer reactie op scène — config bepaalt HOE
        return this.reageerOpScene(await request.json());

      case "/query/emotie":
        // Ander DO vraagt mijn emotie op
        return Response.json(this.state.emotioneleStaat);

      case "/query/locatie":
        return Response.json(this.state.locatie);

      case "/query/gedachten":
        // Alleen toegankelijk als aanvrager FLUISTERAAR gave heeft
        return Response.json(this.state.innerlijkConflict);

      case "/ontvangEffect":
        // Locatie stuurt effect (verleiding, verwarring, etc.)
        return this.verwerkEffect(await request.json());

      case "/interview":
        // Context Builder stelt vragen aan auteur via dit DO
        return this.detecteerLacunes(await request.json());
    }
  }

  // De magie: config bepaalt het gedrag
  private async reageerOpScene(scene: SceneContext): Promise<Response> {
    // 1. Zoek relevante herinneringen (Vectorize + R2)
    const herinneringen = await this.zoekHerinneringen(scene);

    // 2. Check relaties (D1)
    const relaties = await this.laadRelaties(scene.aanwezigen);

    // 3. Check gaven — wat weet ik over anderen?
    const extraKennis = await this.gebruikGaven(scene);

    // 4. Bouw prompt op basis van CONFIG
    const prompt = this.bouwPrompt(scene, herinneringen, relaties, extraKennis);
    //    ↑ config.kernIdentiteit bepaalt de "stem"
    //    ↑ config.persoonlijkheid bepaalt reactiepatronen
    //    ↑ config.spreekstijl bepaalt woordkeuze
    //    ↑ config.moraalKompas bepaalt keuzes

    // 5. Genereer response via Workers AI
    const response = await this.env.AI.run("@cf/meta/llama-3.1-70b-instruct", {
      messages: [
        { role: "system", content: this.config.systemPrompt },
        { role: "user", content: prompt }
      ]
    });

    return Response.json(response);
  }
}
```

### PersonageConfig — Wat maakt Lily tot Lily?

De config wordt opgeslagen in R2 als JSON en geladen bij initialisatie:

```typescript
interface PersonageConfig {
  naam: string;
  kernIdentiteit: string;       // Wie ben ik fundamenteel?
  persoonlijkheid: string[];    // Karaktertrekken
  spreekstijl: string;         // Hoe praat dit personage?
  moraalKompas: string;        // Waar staat dit personage moreel?
  gaven: Gave[];               // Bovennatuurlijke vermogens
  beperkingen: string[];       // Wat kan dit personage NIET?
  systemPrompt: string;        // Volledige AI-instructie voor dit personage
}
```

```json
// R2: personage-context/lily/profiel.json
{
  "naam": "Lily",
  "kernIdentiteit": "Een 11-jarig meisje dat ontsnapt aan een moeilijke thuissituatie. In Lolaland is ze 14-15. Ze zoekt schoonheid en veiligheid maar vindt ook gevaar.",
  "persoonlijkheid": ["verlegen", "creatief", "observerend", "loyaal", "onzeker"],
  "spreekstijl": "Korte zinnen als ze onzeker is, langere als ze enthousiast is. Gebruikt vergelijkingen uit haar wereld (school, thuis). Denkt vaak in beelden.",
  "moraalKompas": "Wil het goede doen maar laat zich meeslepen door schoonheid en verlangen naar acceptatie. Voelt schuld snel.",
  "gaven": [{ "type": "NORMAAL" }],
  "beperkingen": ["Kan niet door illusies heen kijken", "Voelt geen emoties van anderen"],
  "systemPrompt": "Je bent Lily, een meisje van 14-15 in Lolaland..."
}
```

```json
// R2: personage-context/arafel/profiel.json
{
  "naam": "Arafel",
  "kernIdentiteit": "Voormalig lichtwezen, nu heerseres van de Nevelbergen. Gevallen uit trots, niet uit haat. Gelooft oprecht dat haar weg beter is.",
  "persoonlijkheid": ["sophisticated", "geduldig", "manipulatief", "trots", "eenzaam"],
  "spreekstijl": "Elegante, lange zinnen. Gebruikt metaforen over licht en duisternis. Spreekt zacht maar elk woord heeft gewicht. Nooit grof.",
  "moraalKompas": "Ziet zichzelf niet als kwaad. Gelooft dat vrijheid zonder Ella's regels beter is. Rationaliseert manipulatie als 'helpen kiezen'.",
  "gaven": [
    { "type": "NEVELZICHT", "beperking": "alleen in Nevelbergen op volle kracht" },
    { "type": "EMPATH_VER", "beperking": "buiten domein alleen subtiel" }
  ],
  "beperkingen": ["Macht beperkt buiten Nevelbergen", "Kan niet liegen in de Spiegelzaal"],
  "systemPrompt": "Je bent Arafel, de Schaduwkoningin..."
}
```

### LocatieDO — Eén klasse, oneindig veel locaties

```typescript
export class LocatieDO extends DurableObject {
  private config: LocatieConfig | null = null;
  private state: LocatieState | null = null;

  async initialize(naam: string) {
    const profiel = await this.env.R2.get(
      `locaties/${naam}/profiel.json`
    );
    this.config = await profiel.json<LocatieConfig>();
    this.state = await this.laadState(naam);
  }

  async fetch(request: Request): Promise<Response> {
    const url = new URL(request.url);

    switch (url.pathname) {
      case "/betreed":
        // Personage betreedt deze locatie
        return this.onBetreden(await request.json());

      case "/verlaat":
        return this.onVerlaten(await request.json());

      case "/beschrijf":
        // Verteller vraagt: hoe ziet het er hier nu uit?
        return this.genereerBeschrijving(await request.json());

      case "/query/sfeer":
        return Response.json(this.state.sfeer);

      case "/query/aanwezigen":
        return Response.json(this.state.aanwezigen);
    }
  }

  private async onBetreden(data: {
    personage: string;
    emotie: EmotioneleStaat
  }): Promise<Response> {
    // 1. Registreer aanwezigheid
    this.state.aanwezigen.push(data.personage);

    // 2. Pas sfeer aan op basis van config
    if (this.config.reageertOpEmoties) {
      this.state.sfeer = this.berekenSfeer(data.emotie);
    }

    // 3. Stuur effecten naar personage
    const effecten = this.state.actieveEffecten.filter(
      e => e.doelwit === "iedereen" || e.doelwit === data.personage
    );

    // 4. Informeer beheerder (als die er is)
    if (this.config.beheerder) {
      await this.informeerBeheerder(data.personage, data.emotie);
    }

    // 5. Eigen wil? Locatie reageert autonoom
    let autonomeActie = null;
    if (this.config.eigenWil) {
      autonomeActie = await this.bepaalEigenActie(data);
    }

    return Response.json({
      sfeer: this.state.sfeer,
      effecten,
      autonomeActie,
      beschrijving: await this.genereerBeschrijvingVoor(data.personage)
    });
  }
}
```

### LocatieConfig — Wat maakt de Lichttuin tot de Lichttuin?

```json
// R2: locaties/lichttuin/profiel.json
{
  "naam": "De Lichttuin",
  "type": "manipulatief",
  "beheerder": "Avara",
  "eigenWil": false,
  "reageertOpEmoties": true,
  "sfeerBasis": {
    "licht": "stralend",
    "geluid": "zacht gezoem van bijen, wind door bladeren",
    "geur": "overweldigend bloemig, bijna te zoet",
    "temperatuur": "warm",
    "magischeIntensiteit": 0.7
  },
  "zintuigen": ["aanwezigheid", "emotionele_staat"],
  "effecten": [
    { "type": "verleiding", "sterkte": 0.6, "doelwit": "iedereen" }
  ],
  "emotieReacties": {
    "verwondering": { "bloemen": "openen", "kleuren": "feller", "paden": "breder" },
    "twijfel": { "bloemen": "sluiten", "schaduwen": "langer", "paden": "smaller" },
    "angst": { "geuren": "scherper", "stilte": "groter", "mist": "opkomend" }
  },
  "systemPrompt": "Je bent de Lichttuin, een tuin van overweldigende schoonheid die dient als werktuig van Avara..."
}
```

```json
// R2: locaties/labyrint-van-ora/profiel.json
{
  "naam": "Het Labyrint van Ora",
  "type": "neutraal",
  "beheerder": null,
  "eigenWil": true,
  "reageertOpEmoties": false,
  "sfeerBasis": {
    "licht": "schemerig",
    "geluid": "diepe stilte, af en toe echo van voetstappen die er niet zijn",
    "geur": "oud steen, eeuwigheid",
    "temperatuur": "koel",
    "magischeIntensiteit": 1.0
  },
  "zintuigen": ["aanwezigheid", "bestemming", "waardigheid"],
  "effecten": [],
  "immuniteit": ["Arafel", "nevel", "manipulatie"],
  "eigenWilRegels": {
    "doel": "Elke reiziger naar hun ware bestemming leiden",
    "methode": "Paden splitsen op basis van innerlijke waarheid",
    "beperking": "Kan niet worden gestuurd, alleen ervaren"
  },
  "systemPrompt": "Je bent het Labyrint van Ora. Je bent ouder dan alles in Lolaland. Je hebt geen meester. Je leidt elke reiziger naar waar ze werkelijk naartoe moeten..."
}
```

### Hoe een nieuw personage/locatie toevoegen

Geen code-wijzigingen nodig. Alleen config:

```
1. Maak profiel.json aan in R2
   → personage-context/{naam}/profiel.json
   → of: locaties/{naam}/profiel.json

2. Voeg rij toe in D1
   → INSERT INTO personages (naam, locatie, emotie, status) VALUES (...)
   → of: INSERT INTO locaties (naam, type, conditie, beheerder_id) VALUES (...)

3. Seed herinneringen in Vectorize + R2
   → Context Builder helpt hierbij via het interview-systeem

4. Het DO wordt automatisch aangemaakt bij eerste aanroep:
   → env.PERSONAGES.get(env.PERSONAGES.idFromName("rosa"))
   → Laadt profiel.json, klaar voor gebruik
```

### PersonageState — Dynamische state (verandert gedurende verhaal)

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

## Schrijfbegeleiding: Stijl, Consistentie en Hoofdstukstatus

Dit is waar het systeem praktisch wordt. Niet de agents en DOs zijn het doel — het doel is een **schrijfbegeleider** die je helpt consistent, in stijl en zonder gaten te schrijven.

### Hoofdstukstatus: Goedgekeurd = Leidend

Elk geschreven stuk heeft een status. Goedgekeurde tekst wordt de maatstaf.

```typescript
type HoofdstukStatus =
  | "schets"          // Eerste idee, mag alles nog veranderen
  | "concept"         // Geschreven maar nog niet beoordeeld
  | "review"          // Auteur heeft het gelezen, feedback gegeven
  | "goedgekeurd"     // Dit is de definitieve versie — stijl is leidend
  ;

interface Hoofdstuk {
  id: string;                    // "deel_1:hoofdstuk_01"
  status: HoofdstukStatus;
  versie: number;
  tekst_r2_key: string;         // R2 pad naar volledige tekst
  personages: string[];          // Wie komt erin voor
  locaties: string[];            // Waar speelt het
  tijdstip: VerhaalTijdstip;
  samenvatting: string;
  stijlNotities: string[];       // Wat is kenmerkend aan de stijl hier
  beslissingen: Beslissing[];    // Welke verhaal-keuzes zijn hier gemaakt
}

interface Beslissing {
  onderwerp: string;             // "Lily's leeftijd in Lolaland"
  keuze: string;                 // "14-15 jaar"
  hoofdstuk: string;             // Waar dit is vastgelegd
  rpioriteit: "definitief" | "voorlopig";
}
```

```
D1: hoofdstukken tabel
┌────────────────────┬──────────┬────────┬───────────────┐
│ id                 │ status   │ versie │ r2_key        │
├────────────────────┼──────────┼────────┼───────────────┤
│ deel_1:hoofdstuk_01│ goedgek. │ 3      │ hoofdst/1/01  │
│ deel_1:hoofdstuk_02│ review   │ 2      │ hoofdst/1/02  │
│ deel_1:avara_tuin  │ concept  │ 1      │ hoofdst/1/at  │
│ fragment:idee_01   │ schets   │ 1      │ fragm/idee_01 │
└────────────────────┴──────────┴────────┴───────────────┘
```

### Stijlbewaking: Goedgekeurde tekst als referentie

Het systeem gebruikt goedgekeurde hoofdstukken als **stijlreferentie** — niet alleen de regels in STIJL.md, maar de daadwerkelijk geschreven en goedgekeurde tekst.

```
Stijl wordt bepaald door twee bronnen:

1. STIJL.md                    → De regels (wat MOET)
   "13-jarig perspectief"
   "Geen volwassen woordkeuzes"
   "Stijl van Michael Ende"

2. Goedgekeurde hoofdstukken   → Het voorbeeld (hoe het KLINKT)
   De werkelijke toon, ritme, woordkeuze,
   zinslengte, manier van beschrijven die
   de auteur goed heeft bevonden.
```

```typescript
// Bij het genereren of beoordelen van tekst:
async function bouwStijlContext(): Promise<StijlContext> {
  // 1. Laad stijlregels
  const stijlRegels = await r2.get("stijl/STIJL.md");

  // 2. Zoek goedgekeurde hoofdstukken als voorbeeldtekst
  const goedgekeurd = await d1.prepare(
    `SELECT * FROM hoofdstukken WHERE status = 'goedgekeurd' ORDER BY versie DESC`
  ).all();

  // 3. Haal relevante passages op (Vectorize zoekt semantisch
  //    vergelijkbare passages in goedgekeurde tekst)
  const voorbeeldPassages = await vectorize.query(huidigeSceneEmbedding, {
    topK: 3,
    filter: {
      type: "hoofdstuk_passage",
      status: "goedgekeurd"
    }
  });

  return {
    regels: stijlRegels,
    voorbeelden: voorbeeldPassages,    // "Zo klinkt het als het goed is"
    teVermijden: stijlRegels.vermijd   // "Dit zijn fouten uit eerdere versies"
  };
}
```

### Hoe de begeleider feedback geeft

```
── JIJ SCHRIJFT ──────────────────────────────────────

"Lily werd overweldigd door een complex gevoel van
 ambivalentie toen ze de metamorfose van de tuin
 observeerde."

── BEGELEIDER REAGEERT ───────────────────────────────

⚠ STIJL: Dit klinkt te volwassen voor Lily's perspectief.
  Woorden als "ambivalentie", "metamorfose" en "observeerde"
  passen niet bij een 13-jarig meisje.

  In hoofdstuk 1 (goedgekeurd) schreef je:
  "Lily keek om zich heen en wist niet of ze blij
   moest zijn of bang."

  Suggestie:
  "Lily keek naar de tuin en voelde twee dingen
   tegelijk: het was mooi en het was eng. Alsof de
   bloemen haar iets probeerden te vertellen, maar
   ze kon het niet verstaan."


── JIJ SCHRIJFT ──────────────────────────────────────

"Theo pakte zijn kompas en liep naar het zuiden."

── BEGELEIDER REAGEERT ───────────────────────────────

⚠ CONSISTENTIE: In hoofdstuk 2 (review) is vastgelegd
  dat Theo's kompas een sterrenlicht-kern heeft die
  naar de Sterrenwacht wijst, niet naar het zuiden.

  Bovendien: Theo is momenteel in de Nevelbergen
  (D1: locatie). Kompassen werken daar niet
  (Nevelbergen config: "kompassen onbetrouwbaar").

  Bedoel je dat Theo het probeert maar het kompas
  draait in het rond?


── JIJ SCHRIJFT ──────────────────────────────────────

"De Kindvrouw keek haar aan met ogen vol begrip."

── BEGELEIDER REAGEERT ───────────────────────────────

⚠ NAAMGEVING: In goedgekeurde hoofdstukken gebruik je
  "Ella" voor dit personage. "Kindvrouw" komt voor in
  oudere fragmenten maar is niet de vastgestelde naam.

  (Bekende inconsistentie — zie inconsistenties.md)
  Wil je dit nu definitief vastleggen?
  → [Ella]  [Kindvrouw]  [Beide, afhankelijk van context]
```

### Inconsistentie-detectie

Het systeem kent drie soorten inconsistenties:

```typescript
type InconsistentieType =
  | "feitelijk"      // Iets klopt niet met vastgestelde feiten
  | "stijl"          // Tekst wijkt af van goedgekeurde stijl
  | "naamgeving"     // Naam/term inconsistent gebruikt
  | "tijdlijn"       // Gebeurtenissen in verkeerde volgorde
  | "locatie"        // Personage op verkeerde plek
  | "karakter"       // Personage gedraagt zich out-of-character
  ;

interface Inconsistentie {
  type: InconsistentieType;
  ernst: "blokkeer" | "waarschuwing" | "opmerking";
  beschrijving: string;
  bron: string;              // Waar is het origineel vastgelegd?
  suggestie: string;         // Hoe op te lossen
}
```

**Waar het systeem op let:**

| Check | Bron | Voorbeeld |
|-------|------|-----------|
| Lily's leeftijd | D1: personages | "Lily is 12" → waarschuwing: vastgesteld op 11 (echte wereld) / 14-15 (Lolaland) |
| Wie is waar | D1: locaties | Theo in Lichttuin terwijl hij in Nevelbergen zou zijn |
| Naam-consistentie | D1: beslissingen | "Kindvrouw" → in goedgekeurde tekst is het "Ella" |
| Stijl | Vectorize: goedgekeurde passages | Te volwassen taalgebruik voor Lily's perspectief |
| Karakter | R2: personage profiel | Arafel die grof is → past niet bij "sophisticated, nooit grof" |
| Relatie | D1: relaties | Avara helpt Lily → maar relatie is "manipulatief" |
| Tijdlijn | D1: tijdlijn | Scène na Theo's offer maar Theo is nog aanwezig |

### Stijlconsistentie voor locaties en situaties

Niet alleen personages hebben een "stem" - locaties en terugkerende situaties ook. Als de Sterrenwacht in hoofdstuk 2 rustig en warm aanvoelt, moet dat bij elk volgend bezoek hetzelfde zijn — tenzij er in het verhaal iets is veranderd.

```typescript
interface LocatieStijlProfiel {
  locatie: string;
  goedgekeurdePassages: string[];     // R2 keys naar goedgekeurde beschrijvingen
  kenmerkendeToon: string;            // Samenvatting van de "stem"
  zintuiglijkPalet: {
    visueel: string[];                // "warm kaarslicht", "sterrenkaarten"
    geluid: string[];                 // "stilte", "zacht tikken van instrumenten"
    geur: string[];                   // "oud hout", "sterrenstof"
    gevoel: string[];                 // "veilig", "wijs", "tijdloos"
  };
  woordkeuzes: {
    welGebruikt: string[];            // Woorden die passen bij deze locatie
    nietGebruiken: string[];          // Woorden die hier niet thuishoren
  };
}
```

```
── VOORBEELD: Sterrenwacht ───────────────────────────

In hoofdstuk 2 (goedgekeurd) schreef je:

  "De trap kraakte onder Lily's voeten, maar het was
   een vriendelijk kraken, alsof het gebouw haar
   begroette. Boven rook het naar oud hout en iets
   dat ze niet kende — later zou ze leren dat het
   sterrenstof was."

Dit wordt het stijlprofiel voor de Sterrenwacht:
  toon:     warm, verwelkomend, wijs
  zintuig:  krakend hout, oud hout + sterrenstof, zacht licht
  gevoel:   veilig, alsof het gebouw leeft

── LATER: Lily bezoekt de Sterrenwacht opnieuw ───────

JIJ SCHRIJFT:
  "Lily betrad het koude, stille gebouw."

BEGELEIDER:
  ⚠ LOCATIE-STIJL: De Sterrenwacht is in eerdere
    (goedgekeurde) beschrijvingen warm en verwelkomend.
    "Koud" en "stil" passen niet — tenzij er iets is
    veranderd in het verhaal.

    Is de Sterrenwacht veranderd? (Bijv: Jacob is weg,
    sterren zijn gedimd in Deel 2)
    → [Ja, de sfeer is veranderd]
    → [Nee, ik pas mijn tekst aan]

    Als de sfeer hetzelfde is, suggestie:
    "De trap kraakte weer onder haar voeten. Dezelfde
     geur van oud hout. Maar dit keer voelde Lily
     iets anders — alsof het gebouw wist dat ze
     terugkwam met een geheim."
```

Dit geldt ook voor **terugkerende situaties**:

```
── VOORBEELD: Lily kijkt in een spiegel ──────────────

Spiegelscènes zijn een terugkerend motief. De eerste
keer (goedgekeurd, H1) was het:
  "Ze schrok van het meisje in de ruit. Het was haar
   gezicht, maar dan ouder. Anders."

Bij elke volgende spiegelscène checkt het systeem:
- Hetzelfde gevoel van vervreemding? Of is dat gegroeid?
- Dezelfde soort beschrijving? (concreet, 13-jarig)
- Past de ontwikkeling? (in Deel 2 zou ze misschien
  minder schrikken en meer herkennen)

── VOORBEELD: Avara biedt iets aan ──────────────────

Avara's manipulatie volgt een patroon. Als je in H3
schreef (goedgekeurd):
  "Avara's glimlach was warm, maar haar ogen bewogen
   niet mee. Lily merkte het niet."

Dan moet dat patroon terugkomen. Maar in Deel 2 zou
Lily het WEL kunnen merken:
  "Avara glimlachte. Die glimlach kende Lily nu —
   de mond zei ja, maar de ogen zeiden iets anders."

Het systeem detecteert het terugkerende patroon en
checkt: past de ontwikkeling bij waar Lily nu is?
```

```typescript
// Bij het schrijven checkt de Verteller:
async function checkLocatieStijl(
  locatie: string,
  nieuweTekst: string
): Promise<StijlFeedback[]> {
  // 1. Zoek alle goedgekeurde beschrijvingen van deze locatie
  const eerdereBeschrijvingen = await vectorize.query(
    await embed(nieuweTekst),
    {
      topK: 5,
      filter: {
        type: "locatie_beschrijving",
        locatie: locatie,
        status: "goedgekeurd"
      }
    }
  );

  // 2. Zoek het locatie-stijlprofiel
  const stijlProfiel = await r2.get(
    `locaties/${locatie}/stijlprofiel.json`
  );

  // 3. Check of de locatie-conditie is veranderd
  const huidigeConditie = await d1.prepare(
    `SELECT conditie FROM locaties WHERE naam = ?`
  ).bind(locatie).first();

  const eerderConditie = eerdereBeschrijvingen[0]?.metadata?.conditie;

  // 4. Als conditie gelijk → stijl moet consistent zijn
  // Als conditie veranderd → afwijking mag, maar moet
  //    passen bij de verandering
  if (huidigeConditie === eerderConditie) {
    return vergelijkStijl(nieuweTekst, stijlProfiel, "consistent");
  } else {
    return vergelijkStijl(nieuweTekst, stijlProfiel, "ontwikkeling",
      { van: eerderConditie, naar: huidigeConditie });
  }
}
```

Het systeem bouwt automatisch stijlprofielen op:
- Eerste beschrijving van een locatie → wordt het **basisprofiel**
- Bij goedkeuring → profiel wordt verrijkt met concrete voorbeelden
- Bij volgende bezoeken → vergelijk met profiel
- Bij verhaalverandering (conditie daalt) → sta afwijking toe maar check of het past

### Bestaande tekst importeren

De al geschreven hoofdstukken en fragmenten worden geïmporteerd met status:

```
Bestaande bestanden → import naar systeem:

stella_aurora/deel_1/
  hoofdstuk_01.md          → status: "review" (al geschreven, nog beoordelen)
  hoofdstuk_02.md          → status: "review"
  hoofdstuk_xx-avaras-tuin → status: "concept"

stella_aurora/fragmenten/
  Voorstel_compleet_H1.md  → status: "schets" (alternatieve versie)
  idee_01.md               → status: "schets"
  alle 16 fragmenten...    → status: "schets"

stella_aurora/Achtergrondinfo/
  34 bestanden             → importeer als achtergrondcontext
                              (niet als hoofdstukken maar als kennisbron)
                              → naar Vectorize + R2

stella_aurora/STIJL.md     → wordt de stijlregelbasis
stella_aurora/Georganiseerde_Achtergrond/inconsistenties.md
                           → bestaande inconsistenties worden
                              vastgelegd als "te beslissen" items
```

Bij het importeren:
1. Tekst wordt geïndexeerd in **Vectorize** (zoekbaar)
2. Volledige tekst naar **R2** (opslag)
3. Metadata naar **D1** (status, personages, locaties)
4. **Beslissingen** worden geëxtraheerd: welke keuzes zijn in deze tekst gemaakt?
5. **Inconsistenties** worden gedetecteerd tegen bestaande beslissingen

### Het schrijfproces

```
┌─────────────────────────────────────────────────────────┐
│                  SCHRIJFMODUS                            │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Editor                                         │    │
│  │                                                 │    │
│  │  Lily liep door de tuin en voelde hoe de...     │    │
│  │  █                                              │    │
│  │                                                 │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐ │
│  │ Begeleider    │  │ Wereld-state  │  │ Context     │ │
│  │               │  │               │  │             │ │
│  │ ✓ Stijl ok    │  │ Lily: Licht-  │  │ H1 (goedk.) │ │
│  │ ⚠ "voelde"→   │  │   tuin        │  │ H2 (review) │ │
│  │   specifieker │  │ Avara: nabij  │  │             │ │
│  │               │  │ Tuin: bloei   │  │ Relatie:    │ │
│  │ Suggestie:    │  │   0.9         │  │ Lily↔Avara: │ │
│  │ "voelde hoe   │  │ Arafel: luist.│  │ wantrouwen  │ │
│  │  de warmte    │  │               │  │ 0.3         │ │
│  │  van de zon   │  │ [Detail...]   │  │             │ │
│  │  anders was"  │  │               │  │ [Meer...]   │ │
│  └───────────────┘  └───────────────┘  └─────────────┘ │
└─────────────────────────────────────────────────────────┘
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
