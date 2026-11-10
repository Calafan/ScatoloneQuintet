# ScatoloneQuintet

Card database for the **Scatolone**, the *Magic: The Gathering* cube played at
our club. Every card Scryfall knows about has an entry here: whether it made the
cube (a rating), whether it is set aside (a status), and what it does in play
(its effect tags). All of it is plain JSON tracked in git.

The images are not in the repository. They are downloaded from
[Scryfall](https://scryfall.com), kept locally and can be rebuilt at any time,
as can the cube report. Both the downloading and the tagging
are done by [ScatoloneDownloader](https://github.com/Calafan/ScatoloneDownloader).

## Repository layout

```
ScatoloneQuintet/
├── metadata/                 tracked: the source of truth
│   ├── pool.json             rating 3–5, the cube itself
│   ├── fringe.json           rating 1–2, evaluated and cut
│   └── unrated.json          rating 0, the rest of the library
├── Source/                   not tracked: card images, <year>/<set>/<card>.png
└── Views/                    not tracked: Cubo_Analysis.md, written by report
```

A card lives in exactly one tier file, chosen by its rating, so the file edited
day to day stays small and a diff shows only what changed. The tagger also
writes `metadata/review-log.jsonl`, a history of every save that records what
the automatic classifier proposed and what the reviewer kept.

**The metadata plus Scryfall is enough to rebuild everything.** If the images
are lost, `restore` downloads them again from the entries here, printing by
printing. No rating, status or tag exists only inside an image file.

## Ratings

| Rating | Meaning | File | In the report |
|---|---|---|---|
| 0 | not evaluated yet | `unrated.json` | no |
| 1 | rejected | `fringe.json` | no |
| 2 | cut, but only just: the bench | `fringe.json` | its own appendix, to promote from |
| 3–5 | in the cube | `pool.json` | every section |

## Statuses

A card can carry at most one status:

| Status | Meaning |
|---|---|
| **Banned** | On the Banned list: kept out of the cube whatever its rating. |
| **Token** | Not a card you play. It is a helper card printed to track an effect during the game, such as The Ring Tempts You, The Undercity or the Monarch. |
| **Jolly** | A wildcard. Everyone who chipped in for the cube picks one card to add, whatever its rating, and it is printed and goes in the box. |

A card with a status is set apart from the pool. At any rating it is left out of
the cube analysis, and `make-list` writes it in a section of its own.

## Card entries

Entries are keyed by Scryfall's `oracle_id`, so a card is one entry however many
times it has been printed.

```json
"646f9ad8-9d18-4b42-954a-e21a62e9f0e8": {
  "name": "Aang's Iceberg",
  "rating": 4,
  "scryfallId": "6cd25a10-…",
  "effects": ["Filter"],
  "reviewedAt": "2026-09-10T15:29:…"
}
```

| Field | Meaning |
|---|---|
| `name` | Card name, for reading only; the key is what identifies the card. |
| `rating` | 0–5, see above. It also decides which file the entry is in. |
| `status` | `Banned`, `Token` or `Jolly`; omitted for a normal card. |
| `scryfallId` | The exact printing (the art) the card is kept in. |
| `effects` | Effect tags from the ontology below, in the order of the table. |
| `reviewedAt` | When a person confirmed the entry in the tagger. Without it, the tags are only the classifier's proposal. |

## Effect ontology

Every card carries zero or more of the 25 tags below. They describe what the
card does **for the cube**: what it answers, what it gains, what it enables.
General rules:

- **A card can carry several tags.** An entering creature that also exiles a
  permanent is both what it makes and what it answers.
- **Tags describe what a card does to or for something else.** A creature that
  pumps, shields or mills only itself has a stat line, not an effect.
- **A tag has to be what the card is for.** A small rider on a card that does
  something else does not earn a tag of its own.
- **Card types are not effects.** Creature and Land are the card's type, which
  the tagger filters on.
- **Deliberately absent:** fogs (they buy a turn instead of answering a threat)
  and lifegain (a resource, and this cube has no lifegain archetype).
- **The classifier proposes and a person decides.** `classify` reads each
  card's rules text and fills in `effects`, but only a review in the tagger
  stamps `reviewedAt` and makes the tags final.

| Effect | What it is, and what it is not |
|---|---|
| **Tokens** | Puts CREATURE bodies on your side — earthbend, manifest, cloak, living weapon, an animated land, artifact, enchantment or token, a token copy of a creature or creature SPELL. A noncreature card or an Adventure's spell counts for one body; a CREATURE only for three at once or a repeating effect. A Treasure, Clue or Food is no body. |
| **Removal** | Answers ONE creature/planeswalker: destroy, exile, damage however counted, fight, an edict (one each, a punisher), INTO a library, made a Treasure, or ANY toughness malus ("+2/-1"). Not YOUR OWN, a creature TYPE, a BLINK, a GRAVEYARD card, "-2/-0" (Pacify), "each creature" (Wipe), a token's 1-point ping, or what only touches its blocker. |
| **Counter** | Somebody else's spell or ability taken off the stack — countered, exiled, back to hand or into the library — by an answer you time: a target, or a moment (flash, entering, face up, an instant that ends the turn). Counterspell. Not a trigger that counters whatever is cast, nor a counter for the card itself. |
| **RemovePermanent** | Somebody else's permanent of ANY type, chosen by you, gone for good: "destroy/exile target (nonland) permanent", a token, a colour, the O-ring family, shuffled away, made a Treasure. Vindicate. Not an edict (they choose), a list of types, your own, a blink, nor an exile its owner may play. |
| **Wipe** | Several of somebody else's creatures gone at once, taken by a rule and not target by target: all or each, damage or -X/-X to each, returned, shuffled away, two or more or half sacrificed, an edict every turn. Wrath of God. Not one creature type, what fights it or didn't attack, your own, lands or artifacts alone. |
| **Bounce** | A nonland permanent that can be somebody else's leaves the battlefield but stays its owner's, who plays it again: back in hand, on or near the top of the library, or exiled where they may play it. Unsummon. Not itself as a price, your own alone, lands, spells, cards in a graveyard, the bottom or a shuffle. |
| **Ramp** | Mana of yours beyond the land of the turn: one more land in play, a source adding mana or discounting other spells every turn, or a burst of mana that is the card's point. Rampant Growth. Not a land tapping for one, mana costing mana, a land for a land, mana for others, one Treasure per trigger, or mana and lands untapped by an attack. |
| **Disenchant** | An artifact or enchantment that can be somebody else's, named as artifact, enchantment or Aura (no other subtype), gone for good: destroyed, exiled, shuffled or sacrificed to an edict, one or all. Disenchant. An Aura on a creature or land counts. Not an artifact CREATURE (Removal) or a sweeper taking creatures too (Wipe). |
| **Discard** | Empties somebody ELSE'S hand: they discard, you pick, they pay to dodge a punisher, or it is exiled or put back on their library. Your own discard is a cost — madness, blitz, cycling, a loot, even one handed to a player — and so is "each player MAY discard" and a card that only WATCHES a discard. |
| **CardAdvantage** | Net count over zero (Ancestral Recall: -1 +3). The card itself and each card paid from HAND cost one; each card drawn, put in hand or played from a library, exile or graveyard gains one; a repeatable ability pays once. A permanent or graveyard card paid is free; only YOUR part counts. Net 0 is Filter if chosen; a sometimes gain is both. |
| **Filter** | YOU choose which cards you will have, not more: see more than you keep (one at most) or hand back as many as you take (Brainstorm). Only as the card's JOB: a spell or mode, an ability that comes AGAIN or costs the card, a land. Not a scry or loot made ONCE, a LAND kept, an OPPONENT's top. A gain only SOMETIMES is CardAdvantage too. |
| **Reanimate** | A nonland PERMANENT back from the dead onto the battlefield under YOUR control, its cost skipped (Reanimate): out of a graveyard, out of the exile this card put it in from there, or as a token copy; any mana value. Not a card bringing ITSELF back, nor one for a creature TYPE only. A land: Ramp only if this card filled the graveyard. |
| **Buff** | Raises POWER/TOUGHNESS for something else, when that is what the card is FOR: permanent, over an area, or bigger than +1/+1. A lone +1/+1 counts only on a pump spell or an engine; on a creature, only as counters or Roles. Not a bite (Removal), a self-pump, a pump only a tribe (colorless too) or tokens get, or a keyword but double strike. |
| **Protection** | Keeps another PERMANENT alive, chosen in response or over an unconditional AREA of yours: keyword shield, prevention, redirection, regeneration, return from death, saving blink or unsummon, shield counter, phasing. Not chosen at sorcery speed, a tribe's, a fog, its own, a spell's, nor damage a creature DEALS (Pacify). |
| **Burn** | An OPPONENT loses life to what the card does (Lightning Bolt): damage reaching a face, life lost or paid to stop the card, from two or one again and again; a hasty body gone at end step. Damage at a creature is Removal, "any target" both. Not your own life, a charge on a kill, a token's ability, nor life that buys cards or a target. |
| **Sacrifice** | An outlet you can feed your OWN creatures or artifacts AGAIN AND AGAIN — a cost, a repeating trigger, every upkeep, or a card you recast. Destroying or exiling your own counts. Not once: entering, as you cast it, kicker, exploit. Not a price for damage or lost life, not an edict, not a land, not an artifact token. |
| **Steal** | Takes control of a PERMANENT somebody else controls, for a turn (Ray of Command) or for good (Control Magic), an EXCHANGE included, and the PLAYER for a turn (Mindslaver). A card of theirs played is CardAdvantage at most; their graveyard is Reanimate, Bribery Cheat, a life swap Burn. Not what you own, nor giving one AWAY. |
| **Tutor** | A card the searcher CHOOSES from their whole library or the sideboard, by type, NAME or just "a card", taken anywhere (Demonic Tutor). A land that may be NONBASIC counts ("a land card"); not a basic land, a land type or a land exiled (Ramp, ManaFixing, Filter), not learn's Lesson, not what turns up at random, not the top N. |
| **ManaFixing** | The colour you lack, not more mana: a land of several colours or fetching a land, a land to HAND (also beside one put in play) or landcycling, mana PAID that comes back another colour, a Treasure of yours, a basic type added to lands. Evolving Wilds. Any colour for a tap, a life, a sacrifice or a trigger is Ramp (Birds), unless a land. |
| **Pacify** | Neutralises somebody else's creature without killing it: tapped, stunned, phased out, kept from untapping, can't attack or block, base power 0 or a -N/-0 that lasts, or the damage it DEALS prevented. Not a bare tap on your own turn, a LAND that won't untap, an artifact tap, or a price the card pays. |
| **LandDestruction** | Takes a land off the battlefield, however worded: in a type list, behind a count, beside a second target, by basic type ("destroy all Islands"), or in any sweep that takes them (Balance, Apocalypse). A land edict counts. Not a land in a GRAVEYARD, not an Aura ATTACHED to one, not your own, not mana denial. |
| **Mill** | Puts cards from ANOTHER player's library into their graveyard, or into exile when nobody plays them after, and rad counters handed to them. Filling your own graveyard is fuel for what the card does next, not an attack — dredge is a graveyard card, not a mill card. |
| **Regrowth** | A card of YOUR graveyard comes back as a card to play: into the HAND (lands too), onto the TOP of the library, or CAST from there or from the exile it was moved to, a copy too. Not the card ITSELF (flashback, unearth), a creature TYPE only, the BOTTOM of a library, an opponent's graveyard alone. Straight to the battlefield is Reanimate. |
| **Redirect** | A spell or ability ALREADY ON THE STACK, anybody's, made over without countering it: new targets, a new controller, or a copy — aimed, the next one, or each one you cast (Redirect, Fork). Not a card copying itself, a copied permanent spell (Tokens), an ability copied on every use, demonstrate, or a copy only on itself or yours. |
| **Cheat** | Skips the mana cost of a card YOU CHOSE: a NONLAND permanent from hand (Sneak Attack) or SEARCHED from any library (Natural Order, Bribery), or a standing free cast or fixed alternative cost (Omniscience). Nothing off the TOP of a library, under any mana-value cap, tribal, or sent back to hand. A graveyard is Reanimate, a land Ramp. |

These are the tagger's tooltips, word for word. The full rulings — each
boundary with the date it was decided, the cards that settled it and how many
reviewed cards it moved — are the comments on
[`CardEffect.cs`](https://github.com/Calafan/ScatoloneDownloader/blob/main/ScatoloneDownloader/Mtg/CardEffect.cs),
with the tooltips in
[`EffectGlossary.cs`](https://github.com/Calafan/ScatoloneDownloader/blob/main/ScatoloneDownloader/Mtg/EffectGlossary.cs).
When this table and the code disagree, the code is right.

## Working on the cube

All commands belong to
[ScatoloneDownloader](https://github.com/Calafan/ScatoloneDownloader) and take
`-m <metadata dir>`.

| Command | What it does |
|---|---|
| `register` | Adds the new images in `Source/` as unrated entries. It never touches an entry already here. |
| `tag` | Opens the web tagger: rating, status and effects, one card at a time, saved on every keystroke. It opens on the review queue. |
| `classify` | Proposes effect tags from each card's rules text. It never touches a reviewed entry. |
| `audit` | Lists reviewed cards that say the same thing but were tagged differently. |
| `report` | Writes the `Cubo_Analysis.md` report from the metadata. |
| `make-list` | Writes the pool as a download list, statuses in their own sections. |
| `restore` | Downloads again every image missing from `Source/`. |

## Banned mechanics

Cards with these mechanics are kept out of the cube altogether. They are not in
the Banned list either, so they need a fresh look if a mechanic is ever brought
back:

- The Ring tempts you
- Start your engines!
- The Initiative

---

Card data and images come from [Scryfall](https://scryfall.com). *Magic: The
Gathering* is a trademark of Wizards of the Coast; this is an unofficial fan
project.
