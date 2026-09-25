# ScatoloneQuintet

Card database for the **Scatolone**, the *Magic: The Gathering* cube played at
our club. Every card Scryfall knows about has an entry here: whether it made the
cube (a rating), whether it is set aside (a status), and what it does in play
(its effect tags). All of it is plain JSON tracked in git.

The images are not in the repository. They are downloaded from
[Scryfall](https://scryfall.com), kept locally and can be rebuilt at any time,
as can the folders used to browse the cube. Both the downloading and the tagging
are done by [ScatoloneDownloader](https://github.com/Calafan/ScatoloneDownloader).

## Repository layout

```
ScatoloneQuintet/
├── metadata/                 tracked: the source of truth
│   ├── pool.json             rating 3–5, the cube itself
│   ├── fringe.json           rating 1–2, evaluated and cut
│   ├── unrated.json          rating 0, the rest of the library
│   └── import-state.json     watermark for incremental imports from Adobe Bridge
├── Source/                   not tracked: card images, <year>/<set>/<card>.png
└── Views/                    not tracked: browse folders and Cubo_Analysis.md,
                              rebuilt from scratch by build-views
```

A card lives in exactly one tier file, chosen by its rating, so the file edited
day to day stays small and a diff shows only what changed. The tagger also
writes `metadata/review-log.jsonl`, a history of every save that records what
the automatic classifier proposed and what the reviewer kept.

**The metadata plus Scryfall is enough to rebuild everything.** If the images
are lost, `restore` downloads them again from the entries here, printing by
printing. No rating, status or tag exists only inside an image file.

## Ratings

| Rating | Meaning | File | Browse folder |
|---|---|---|---|
| 0 | not evaluated yet | `unrated.json` | `Views/0_Unrated/<year>/<set>/` |
| 1 | rejected | `fringe.json` | none |
| 2 | cut, but only just: the bench | `fringe.json` | `Views/6_Bench/` |
| 3–5 | in the cube | `pool.json` | every other folder under `Views/` |

## Statuses

A card can carry at most one status:

| Status | Meaning |
|---|---|
| **Banned** | On the Banned list: kept out of the cube whatever its rating. |
| **Token** | Not a card you play. It is a helper card printed to track an effect during the game, such as The Ring Tempts You, The Undercity or the Monarch. |
| **Jolly** | A wildcard. Everyone who chipped in for the cube picks one card to add, whatever its rating, and it is printed and goes in the box. |

A card with a status is set apart from the pool. It appears only in its own
folder (`Views/0_Banned/`, `Views/0_Token/`, `Views/0_Jolly/`) at any rating, it
is left out of the cube analysis, and `make-list` writes it in a section of its
own.

## Card entries

Entries are keyed by Scryfall's `oracle_id`, so a card is one entry however many
times it has been printed.

```json
"646f9ad8-9d18-4b42-954a-e21a62e9f0e8": {
  "name": "Aang's Iceberg",
  "rating": 4,
  "label": "",
  "scryfallId": "6cd25a10-…",
  "effects": ["Filter"],
  "reviewedAt": "2026-09-10T15:29:…"
}
```

| Field | Meaning |
|---|---|
| `name` | Card name, for reading only; the key is what identifies the card. |
| `rating` | 0–5, see above. It also decides which file the entry is in. |
| `label` | Adobe Bridge colour label from the original import. Nothing reads it any more. |
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
- **Card types are not effects.** Creature and Land are covered by the type
  buckets in `Views/`.
- **Deliberately absent:** fogs (they buy a turn instead of answering a threat)
  and lifegain (a resource, and this cube has no lifegain archetype).
- **The classifier proposes and a person decides.** `classify` reads each
  card's rules text and fills in `effects`, but only a review in the tagger
  stamps `reviewedAt` and makes the tags final.

| Effect | What it is, and what it is not |
|---|---|
| **Tokens** | Puts CREATURE bodies on your side, however worded — earthbend, cloak, manifest dread, living weapon, a token copy of a creature. A noncreature card counts for a single body; a CREATURE only when it makes three at once or the effect repeats. A Treasure, Clue or Food is a resource, not a body. |
| **Removal** | Answers ONE creature: destroy, exile, damage however counted, fight, an edict aimed at THEM, the BOTTOM of a library, or ANY toughness malus — "+2/-1" counts, an Aura counts. Not YOUR OWN, not a card in a GRAVEYARD, not "-2/-0" (Pacify), not "each creature" (Wipe), not what can only touch what it is already blocking. |
| **Counter** | Answers a spell on the stack by countering it. |
| **RemovePermanent** | Answers ANY permanent — "destroy/exile target (nonland) permanent", the O-ring family, and a kill narrowed only by COLOUR ("destroy target red permanent"). The breadth is the point, so it stays this tag even when you would usually aim it at a creature. |
| **Wipe** | Mass removal: the board is emptied, by any verb — destroyed, exiled, damaged, shrunk, or all returned to hand. An adjective does not narrow it: "all white permanents" still counts. "Destroy all creatures blocking or blocked by this creature" is a combat trick. |
| **Bounce** | Returns a NAMED permanent to its owner's hand ("return target …", or a mass "return each/all"). Returning itself as a cost or an end-step drawback is a price the card pays, not an answer. |
| **Ramp** | More mana, or sooner: a mana ability on a creature or rock, a land onto the battlefield, a ritual, a cost reduction, an extra land drop, untapping lands, several Treasures. Mana that COSTS mana fixes colour instead. A land tapping for its own one mana is just a land. |
| **Disenchant** | Answers an artifact or enchantment, at any count — "up to one", "X target", "another", "all" — and an edict aimed at one counts too. A bare Aura counts; an Aura ATTACHED to a named thing does not. An artifact CREATURE is Removal, a sweeper that takes the creatures too is Wipe, and your own is never an answer. |
| **Discard** | Empties somebody ELSE'S hand. A bare "discard a card" is a cost you pay — madness, blitz, cycling, the back half of a loot — and paying it attacks nobody. |
| **CardAdvantage** | A card the opponent does not get. COUNT IT — the card ITSELF counts, so Ponder nets 0. A one-shot must draw two more than it pays; a REPEATABLE ability bought the card once, so one is enough — even when it costs a card, itself or another permanent. Two off and ONE played is Filter. Not a draw for THEM, nor a trigger you can't reach. |
| **Filter** | Cards changing places at NO net gain: scry, surveil, hideaway, look at the top few and take one, exile two and play one, every rummage either way round. Not a card you CLOAK or manifest (Tokens), not a LAND off the top (Ramp), not somebody else's loot. One that also comes out ahead, or repeats, keeps this AND CardAdvantage. |
| **Reanimate** | A creature back from the dead and straight onto the BATTLEFIELD, cheating its cost. A card this card EXILED counts, and so does a token copy of a creature in a graveyard. Blinking your OWN creature does not — that is protection. A land coming back is Ramp. |
| **Buff** | Raises POWER/TOUGHNESS for something else, when that is what the card is FOR: permanent, over an area, or bigger than +1/+1. A lone +1/+1 or one counter counts only on a pump spell or a noncreature engine. Not a bite (Removal), a self-pump, a pump only a tribe (colorless too) or tokens get, or a keyword but double strike. |
| **Protection** | Keeps something ELSE alive, and can be held up in response. A granted keyword shield (hexproof, indestructible, ward), damage prevention, or phasing it out. Not a fog, not a shield the card puts on itself, not preventing the damage a creature DEALS, and not prevention aimed at YOU alone — those are Pacify. |
| **Burn** | Damage or life loss aimed at a FACE, however counted and however slow: "damage equal to the number of Swamps", an upkeep tax, life PAID to stop the card, and losing TWO or more (one point is a rider). Damage at a creature is Removal; "any target" is both. Not damage to YOURSELF, and not what a TOKEN you made deals. |
| **Sacrifice** | An outlet you can feed your OWN creatures or artifacts AGAIN AND AGAIN — a cost, a repeating trigger, every upkeep, or a card you recast. Destroying or exiling your own counts. Not once: entering, as you cast it, kicker, exploit. Not a price for damage or lost life, not an edict, not a land, not an artifact token. |
| **Steal** | Takes what is somebody else's: control of a permanent, an EXCHANGE of two, the player, their life total, an AURA moved off their permanent, or a CARD out of their library, hand or graveyard — played, or reanimated onto your side. Giving one AWAY is the mirror image, and "from A graveyard" names no victim: that is Reanimate alone. |
| **Tutor** | Searches a library for a card you CHOOSE — by type, subtype, colour, count, by NAME, or just "a card" — to hand, battlefield, top, graveyard or exile. A land search is Ramp or ManaFixing. Revealing until a type turns up is not this: that picks for you. |
| **ManaFixing** | Fixes colours: a choice that COSTS you something, any landcycling, a land with two abilities, a land fetched to HAND, a Treasure. Free for a tap is just Ramp (Birds), unless it's a land. Not a land put onto the battlefield, not mana you may only spend on one thing. |
| **Pacify** | Neutralises somebody else's creature without killing it: tapped, stunned, phased out, locked from untapping, can't attack or block, shrunk to base power 0, or the damage it DEALS prevented. HOW LONG does not matter. Not a LAND that won't untap, not an artifact tap, not a tap on your own attack, not a price the card pays itself. |
| **LandDestruction** | Takes a land off the battlefield, however worded: in a type list, behind a count, beside a second target, in a sweeper, or named by basic type ("destroy all Islands"). A land edict counts. Not a land in a GRAVEYARD, not an Aura ATTACHED to one (that protects it), not your own, and not the wider mana-denial family. |
| **Mill** | Puts cards from ANOTHER player's library into their graveyard. Filling your own graveyard is fuel for what the card does next, not an attack — dredge is a graveyard card, not a mill card. |
| **Regrowth** | Buys a card back out of a graveyard: to HAND, to the TOP of a library, or by CASTING it there — that last is reanimation for spells. A card that gives ITSELF a second cast (flashback, escape, unearth) is not. Nor is the BOTTOM of a library, or an opponent's graveyard. Straight to the battlefield is Reanimate. |
| **Redirect** | Acts on SOMEBODY ELSE'S spell, already on the stack, without countering it: changes its target ("the target of", or "that spell's target"), or copies it by aiming at it ("copy TARGET instant or sorcery spell"). Doubling your own next spell is not this, and neither is Storm spelling out its own reminder text. |
| **Cheat** | Puts a NONLAND permanent onto the battlefield without casting it — from hand (Sneak Attack, Elvish Piper) or library (Natural Order) — or lets you cast free STANDING (Omniscience). Out of a graveyard is Reanimate; a land is Ramp; one free exiled card is CardAdvantage. |

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
| `tag` | Opens the web tagger: rating, status and effects, one card at a time, saved on every keystroke. It opens on the review queue. |
| `classify` | Proposes effect tags from each card's rules text. It never touches a reviewed entry. |
| `audit` | Lists reviewed cards that say the same thing but were tagged differently. |
| `build-views` | Rebuilds `Views/` and the `Cubo_Analysis.md` report from the metadata. |
| `make-list` | Writes the pool as a download list, statuses in their own sections. |
| `restore` | Downloads again every image missing from `Source/`. |
| `import` | Brings in ratings and labels set in Adobe Bridge. |

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
