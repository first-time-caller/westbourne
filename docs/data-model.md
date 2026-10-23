# Data model spec (draft 1, 23 Oct 2026)

Working title: Westbourne. UK launch, iPhone first, built as a Claude artifact.

This spec covers how the app stores what it knows: wines, places, grapes, what you saw in shops and on lists, what you drank, and what it suggested. Screens and features are covered elsewhere; they are mentioned here only where they shape the data.

---

## 1. Principles

1. **Tall, not wide.** No record has a column for every possible fact. Entities are small; what we know about them is a list of claims (section 5). Adding a new kind of fact never needs a migration.
2. **Events are the asset.** Scans, sightings, tastings and suggestions are append-only. Nothing the user did is ever overwritten.
3. **Derived views are rebuilt, never edited.** The palate profile, passport, drink-next order and price summaries are computed from events and can be thrown away and rebuilt when the logic improves.
4. **Stable, readable ids.** Every entity has a slug id that means something (`grape:syrah`), so links survive exports, merges and a future move to a native app or a backend.
5. **Sources and confidence everywhere.** Every claim says where it came from (label, list, user, Claude, atlas) and how sure it is. User beats label beats Claude's inference.
6. **Hard limits are data, not prompt text.** They are stored as structured rules and checked in code against every suggestion (section 8).
7. **Versioned from day one.** Every stored document carries `schema`. The atlas carries its own version.
8. **Built to move.** Nothing here depends on the artifact runtime beyond the storage adapter. The same shapes port to SwiftData/CloudKit or a server later.

---

## 2. Where things live

| Layer | What | Where | Who writes |
|---|---|---|---|
| **Atlas** | Reference knowledge: places, grapes, styles, serving guides, UK merchants, glass sizes | Shipped inside the page as versioned JSON (like Bloom's `atlas.json`) | Us, at build time |
| **Personal graph** | Wines, vintages, producers, venues and people *you* have met, plus your own claims about atlas entities | `db`, under `data/users/<uid>/` | The app, from scans and the user |
| **Events** | Scans, sightings, tastings, suggestions, bottle movements | `db`, under `data/users/<uid>/`, grouped by month | The app |
| **Derived** | Palate profile, passport, price summaries, drink-next | `db`, one doc each, rebuildable | The app, recomputed |
| **Media** | Label photos, list photos | `assets`; only the asset id is stored in records | The app |
| **Per-device** | Last tab, draft notes, last weather tap | `localStorage` (wrapped in try/catch) | The app |

Everything under `data/users/<uid>/` is private to that person. Nothing is shared in v1.

### Collections

```
data/users/<uid>/profile/main          one doc: settings, limits, home, schema
data/users/<uid>/nodes/<id>            personal graph: one doc per entity
data/users/<uid>/events/<yyyy-mm>      one doc per month: scans, sightings, tastings, suggestions
data/users/<uid>/cellar/main           bottles you own (one doc, small)
data/users/<uid>/derived/<name>        palate, passport, prices, drinknext
```

Paths are `<collection>/<doc>`; slug ids use `:` and `/`, so a node's doc id is the slug with `/` replaced by `~` (`wine:jamet~cote-rotie`). The adapter does this both ways; nothing else sees it.

### Capacity budget

The artifact store holds 25,000 docs, 256 KiB per doc, and queries scan without indexes.

- Events grouped by month: ten years is 120 docs. A heavy month (40 tastings, 150 sightings, 60 scans, 100 suggestions) is roughly 120 KiB. If a month passes 200 KiB the adapter rolls over to `<yyyy-mm>-b`.
- Nodes: one per wine, vintage, producer, venue or person you've actually met. A keen drinker reaches 2,000–4,000 in several years. This is the only collection that is queried, and only by `type`.
- Atlas entities are never copied into `db`. Only an overlay doc is written when you add your own claim to one (section 4).

---

## 3. Ids

Format: `<type>:<path>`, lowercase ASCII, hyphens for spaces, accents folded (`côte-rôtie` → `cote-rotie`).

| Type | Example | Notes |
|---|---|---|
| `place` | `place:fr/rhone/northern/cote-rotie` | Hierarchical; parent is the id minus its last segment |
| `grape` | `grape:syrah` | Synonyms are aliases, not separate grapes (`shiraz` → `grape:syrah`) |
| `style` | `style:red/savoury-medium` | Small, opinionated set used for moods and suggestions |
| `producer` | `producer:jamet` | Disambiguated with a place suffix if needed: `producer:clape~cornas` |
| `wine` | `wine:jamet/cote-rotie` | Producer + cuvée, no vintage |
| `vintage` | `vintage:jamet/cote-rotie/2019` | `nv` for non-vintage; multi-vintage cuvées use the disgorgement or lot as a claim |
| `venue` | `venue:uk/london/noble-rot-lambs-conduit` | Shops, restaurants, bars, online merchants |
| `merchant` | `merchant:bbr` | A chain or online merchant; venues point to it |
| `person` | `person:sam` | People you drink with; for group mode and memory |
| `bottle` | `bottle:01J…` | A physical bottle you own; ulid, not a slug |
| `event` | `evt:01J…` | Ulid, sortable by time |

### Aliases and resolution

Every node carries `aliases: string[]` (folded forms). Resolution of a name read from a label or list:

1. Fold the text and look it up in the alias index (atlas aliases plus personal-node aliases, built in memory at load).
2. If exactly one hit, use it.
3. If several, or none, ask Claude to choose among the candidates (or propose a new node) in the same scan call, and mark the link `conf < 1`.
4. If the user corrects a match, add the corrected text as an alias on the right node and record a `merge` or `relink` event, so the same mistake is not made twice.

Merging two nodes keeps the surviving id, moves aliases and claims across, and leaves a tombstone `{ mergedInto: "<id>" }` at the old id so old events still resolve.

---

## 4. Entities (nodes)

Every node has the same envelope, whatever its type:

```js
{
  schema: 1,
  id: "wine:jamet/cote-rotie",
  type: "wine",
  name: "Côte-Rôtie",                 // display name, original accents
  aliases: ["jamet cote rotie", "cote rotie jamet"],
  links: {                            // typed edges to other nodes
    producer: "producer:jamet",
    place: "place:fr/rhone/northern/cote-rotie",
    grapes: ["grape:syrah", "grape:viognier"]
  },
  claims: [ /* section 5 */ ],
  createdAt: "2026-10-23T19:40:00Z",
  updatedAt: "2026-10-23T19:40:00Z"
}
```

`links` holds only structural edges that the app traverses (wine → producer → place). Everything descriptive is a claim.

### Types and their links

| Type | Links | Typical claims |
|---|---|---|
| `place` | `parent` | level (country, region, subregion, appellation, vineyard), climate, soils, known-for, typical styles, hemisphere |
| `grape` | `parent` (for mutations/clones) | colour, typical structure, aromatics, places known for it |
| `style` | none | colour, body, acid, tannin, sweetness, serving temperature |
| `producer` | `place`, `merchant` (UK importer) | farming (organic, biodynamic), founded, winemaker, size, house style |
| `wine` | `producer`, `place`, `grapes[]`, `style` | blend, oak regime, whole bunch, skin contact, closure, classification |
| `vintage` | `wine` | ABV, blend %, drinking window, vintage conditions, critic scores, release price |
| `venue` | `merchant`, `place` | kind (shop, restaurant, bar, online), postcode district, corkage, list style |
| `merchant` | none | website, price tier, delivery, UK duty-paid or in bond |
| `person` | none | likes, dislikes, limits (they carry their own `limits`, section 8) |

### Atlas overlay

Atlas entities (places, grapes, styles, merchants) are read-only in the page. When the user adds a claim about one ("I find Etna too volcanic"), the app writes a node doc with the same id holding only `claims` and `aliases`. At load, overlays are merged on top of the atlas entry.

---

## 5. Claims

A claim is one fact about one node.

```js
{ a: "oak", v: "30% new French barriques", src: "label", conf: 0.9, at: "2026-10-23", ev: "evt:01J…" }
```

| Field | Meaning |
|---|---|
| `a` | Attribute name. Free vocabulary, but the app knows a core set (below) and how to show them |
| `v` | Value: string, number, boolean, `[min,max]` range, or an id |
| `u` | Optional unit (`%`, `°C`, `g/l`, `GBP`) |
| `src` | `user`, `label`, `list`, `atlas`, `claude`, `merchant` |
| `conf` | 0–1. `user` claims are 1 |
| `at` | When it was learnt |
| `ev` | The event it came from, if any |

**Precedence when showing a value:** highest `src` rank (`user` > `label` > `list` > `merchant` > `atlas` > `claude`), then highest `conf`, then newest. Conflicting claims are all kept; the detail sheet can show "label says 13.5%, producer says 14%".

**Core attributes** (the app has display rules and units for these; anything else is shown as a plain key/value):

`abv`, `blend`, `oak`, `whole-bunch`, `skin-contact`, `closure`, `farming`, `sweetness`, `body`, `acid`, `tannin`, `drink-from`, `drink-to`, `serve-temp`, `decant`, `glass`, `soil`, `elevation`, `climate`, `classification`, `importer`, `release-price`, `critic`, `known-for`.

---

## 6. Events

Events are grouped by month in `events/<yyyy-mm>`:

```js
{
  schema: 1,
  month: "2026-10",
  items: [ { id: "evt:01J…", kind: "tasting", at: "…", … }, … ]
}
```

Every event has `id`, `kind`, `at` (ISO time, stored in UTC, shown in UK local time) and `ctx` (the context snapshot, section 7) where it makes sense. The kinds:

### `scan`
A photo (or several) sent to Claude.

```js
{ kind: "scan", mode: "label" | "compare" | "list",
  assets: ["<asset id>", …],
  venue: "venue:uk/london/…" | null,
  model: "quick" | "default",
  found: ["vintage:…", "vintage:…"],   // nodes it resolved or created
  raw: { … }                            // the structured result, trimmed
}
```

### `sighting`
A wine seen for sale at a price. Every shop scan and every line on a wine list makes one. This builds your own UK price history.

```js
{ kind: "sighting", node: "vintage:jamet/cote-rotie/2019",
  venue: "venue:uk/london/hedonism", channel: "shop" | "restaurant" | "online",
  price: 9500, cur: "GBP",               // pence; integers only
  size: 750,                              // ml: 750, 375, 1500; by the glass 125, 175, 250
  per: "bottle" | "glass" | "carafe",
  scan: "evt:01J…", conf: 0.95 }
```

UK specifics: wine by the glass is sold in 125, 175 and 250 ml measures, and lists often show more than one. Each size is its own sighting.

### `tasting`
You drank it.

```js
{ kind: "tasting", node: "vintage:…",
  rating: 4,                               // 1–5, half steps allowed
  structure: { body: 3, acid: 4, tannin: 3, sweetness: 1, fruit: "savoury" },
  notes: "Olive, violets, bacon fat. Better on day two.",
  tags: ["savoury", "floral"],
  again: true,                             // would buy again
  occasion: "weeknight" | "dinner-party" | "restaurant" | …,
  food: "lamb shoulder",
  with: ["person:sam"],
  bottle: "bottle:01J…" | null,
  venue: "venue:…" | null,
  paid: { price: 4200, cur: "GBP" } | null,
  suggestion: "evt:01J…" | null,           // if this came from a suggestion
  ctx: { … } }
```

### `suggestion`
What the app suggested, so it can learn from what you took.

```js
{ kind: "suggestion", moment: "tonight",
  ctx: { … },
  limits: "<hash of the limits in force>",
  picks: [ { node: "vintage:…" | "style:…", why: "…", rank: 1 }, … ],
  outcome: "taken" | "swapped" | "dismissed" | null,
  taken: "vintage:…" | null }
```

### `bottle`
Movements of bottles you own: `bought`, `opened`, `finished`, `gifted`, `moved`. The cellar doc (section 9) is rebuilt from these.

### `edit`
Corrections: `relink`, `merge`, `claim-removed`. Kept so the history explains itself.

---

## 7. Context snapshot (`ctx`)

Captured on every suggestion and tasting. It is what makes "Tuesday, rainy, 7pm, lamb" learnable.

```js
{ moment: "tonight",
  local: "2026-10-23T19:40", dow: 5, season: "autumn",
  weather: "cold-wet" | "mild" | "warm" | null,   // one tap; never fetched
  city: "london",
  food: "lamb shoulder" | null,
  mood: "comforting" | null,
  with: ["person:sam"] }
```

### Moment rules

| Moment | When | Shown as |
|---|---|---|
| `today` | 06:00–16:29 (weekends 06:00–11:59) | **Today** (buying for later, lunch) |
| `lunch` | 12:00–16:29 Saturday and Sunday | **This afternoon** |
| `tonight` | 16:30–22:29 | **Tonight** |
| `nightcap` | 22:30–05:59 | **Nightcap** |

The boundary times are settings, not constants. The user can always switch moment from the title.

Season uses the UK (northern hemisphere) calendar by default and follows `profile.home.hemisphere`.

---

## 8. Profile, limits and preferences

`profile/main`:

```js
{ schema: 1,
  home: { country: "uk", city: "london", hemisphere: "north", cur: "GBP" },
  moments: { tonightFrom: "16:30", nightcapFrom: "22:30" },
  limits: [ /* hard */ ],
  prefs:  [ /* soft */ ],
  explore: 0.5,                     // 0 = safe bets, 1 = surprise me
  updatedAt: "…" }
```

### Hard limits

Structured rules. A suggestion that breaks one is dropped in code, whatever the model returned.

```js
{ id: "lim1", on: true,
  field: "colour" | "price" | "place" | "grape" | "abv" | "sweetness" | "farming" | "closure" | "size",
  op: "in" | "not-in" | "max" | "min" | "between",
  value: ["red", "white"] | 3000 | ["place:fr"] | …,
  per: "bottle" | "glass" | null,   // for price
  moments: ["tonight"] | null }     // a limit can apply to one moment only
```

Examples: "red or white only", "under £30 a bottle, £12 a glass", "not from Australia", "no Sauvignon Blanc", "under 13.5% on weeknights".

Place limits match the whole subtree: `not-in place:au` excludes every Australian place.

### Soft preferences

Same shape with `weight` (−1 to 1) instead of hard rejection. Most are learnt from tastings and shown back to the user for confirmation, never applied silently.

---

## 9. Derived views

Each is one doc under `derived/`, carrying `schema`, `builtAt`, `fromEvents` (the last event id included) and a `logic` version. If `logic` is older than the page's, it is rebuilt.

| Doc | Built from | Holds |
|---|---|---|
| `palate` | tastings, structure, tags | Liked and disliked structure ranges, styles, grapes, places, with counts and confidence |
| `passport` | tastings + atlas | Coverage by grape, country, region, appellation, producer; "never tried" lists near what you like |
| `prices` | sightings | Per vintage: lowest, latest and median price, where, when; per venue: typical restaurant markup vs shop prices you've seen |
| `drinknext` | cellar + vintage drinking windows | Bottles ordered by urgency |
| `cellar` (in `cellar/main`) | bottle events | Bottles you own now: vintage, where bought, price, location |

---

## 10. Scan result contract

What a `sample.json` call returns before the app resolves and stores it. Claude fills only what is printed or clearly shown; unknowns are left out, never guessed.

```js
{ items: [
    { producer: "Domaine Jamet", cuvee: "Côte-Rôtie", vintage: "2019",
      place: "Côte-Rôtie", country: "France", grapes: ["Syrah"],
      abv: 13.5, size: 750,
      prices: [ { amount: 95.00, per: "bottle", size: 750 } ],
      raw: "DOMAINE JAMET · CÔTE-RÔTIE · 2019",
      conf: 0.92,
      candidates: ["vintage:jamet/cote-rotie/2019"] } ],
  venueHint: "Hedonism Wines" | null }
```

The app then: resolves each item to node ids (section 3), writes or updates nodes with `src: "label"` or `"list"` claims, writes one `scan` event and one `sighting` per price, and only then asks for the suggestion. Prices from Claude arrive in pounds and are stored in pence.

---

## 11. Versioning, export and moving later

- Every doc has `schema`. The adapter runs `migrate(doc)` on read and writes back only on the next real change.
- Unknown fields are kept, never stripped (the Bloom review's issue #6).
- Export writes one JSON file: profile, all nodes, all event months, cellar. Derived docs are left out (they rebuild). Import merges by id.
- The storage adapter is the only code that knows about `claude.use("db")`. Swapping in CloudKit or a server later means rewriting that one module.

---

## 12. Open questions

1. Should people you drink with ever see shared notes (group mode across two accounts), or stay private nodes in your own graph?
2. How much of the atlas ships in v1? Proposal: every UK-relevant wine country to region level, France/Italy/Spain/Germany/Austria/Portugal/NZ/Australia/South Africa/USA to appellation level for the main regions, ~150 grapes, ~30 styles, ~40 UK merchants.
3. Critic scores: store them as claims at all, or leave them out deliberately to keep the app about your own palate?
4. When the second app (UK prices and stock) exists, do its prices arrive as `sighting` events with `src: "merchant"`, or as a separate feed? Proposal: sightings, so one price history covers both.
