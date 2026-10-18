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
| **Tokens** | Puts CREATURE bodies on your side, however worded — earthbend, manifest, cloak, living weapon, an animated land, artifact or enchantment, a token copy of a creature or creature SPELL. A noncreature card counts for one body; a CREATURE only for three at once or a repeating effect. A Treasure, Clue or Food is a resource, not a body. |
| **Removal** | Answers ONE creature: destroy, exile, damage however counted, fight, an edict (one each, a punisher), INTO a library, made a Treasure, or ANY toughness malus ("+2/-1"). Not YOUR OWN, a creature TYPE, a BLINK, a GRAVEYARD card, "-2/-0" (Pacify), "each creature" (Wipe), a token's 1-point ping, or what only touches its blocker. |
| **Counter** | Answers a spell on the stack — countered, or taken off it to its owner's hand, to exile or into the library — or an activated or triggered ability. Not one of your own spells, nor one aimed at the card itself. |
| **RemovePermanent** | Answers ANY permanent — "destroy/exile target (nonland) permanent", a token, the O-ring family, and a kill narrowed only by COLOUR ("target red permanent") or by what it is NOT ("noncreature"). The breadth is the point, even if you would usually aim it at a creature. Not your own, nor a blink. |
| **Wipe** | Mass removal: the board emptied — destroyed, exiled, damaged (players too), shrunk, airbent, returned, two or more sacrificed each, or an edict every turn. An adjective does not narrow it; a creature TYPE does. Not what fights it or didn't attack, your own, a graveyard, lands alone (Armageddon is LandDestruction). |
| **Bounce** | Returns a NAMED permanent to its owner's hand ("return target …", or a mass "return each/all"), or puts it on top of, or near the top of, its owner's library. Returning itself as a cost or a drawback is a price, not an answer; so is returning your OWN permanent, and a land returned sets a mana base back and is not a bounce. |
| **Ramp** | More mana, or sooner: a mana ability on a creature or rock, a land onto the battlefield, a ritual, a cost reduction, an extra land drop, untapping lands, several Treasures. Mana that COSTS mana fixes colour instead. A land tapping for one is just a land, its own mana or one GRANTED to it. |
| **Disenchant** | Answers an artifact or enchantment, at any count — "up to one", "X target", "another", "all" — and an edict aimed at one counts too. A bare Aura counts; an Aura ATTACHED to a named thing does not. An artifact CREATURE is Removal, a sweeper that takes the creatures too is Wipe, and your own is never an answer. |
| **Discard** | Empties somebody ELSE'S hand: they discard, you pick, they pay to dodge a punisher, or it is exiled or put back on their library. Your own discard is a cost — madness, blitz, cycling, a loot, even one handed to a player — and so is "each player MAY discard" and a card that only WATCHES a discard. |
| **CardAdvantage** | A card the opponent does not get. COUNT IT — the card ITSELF counts: Ponder nets 0. A one-shot must draw two more than it pays; a REPEATABLE ability needs one, even paid with a card or a permanent, even THEIR card. Two off and ONE played is Filter; both only on two abilities. Not THEIR draw, a tribe's graveyard, an unreachable trigger. |
| **Filter** | Cards changing places at NO net gain: keep one of a few looked at, milled or revealed, rearrange ANY player's top, every rummage or wheel, hideaway, a scry LAND, a scry, Blood or Map that comes AGAIN. Not one made ONCE in passing, nor a LAND kept (ManaFixing), nor a CLOAK. With CardAdvantage only on its own repeating ability, or hideaway. |
| **Reanimate** | A creature back from the dead and straight onto the BATTLEFIELD, cheating its cost. A card this card EXILED counts, and so does a token copy of one in a graveyard. Not a card bringing ITSELF back, one put back for somebody ELSE, or one only a creature TYPE can use. Blinking your OWN creature is protection; a land back is Ramp. |
| **Buff** | Raises POWER/TOUGHNESS for something else, when that is what the card is FOR: permanent, over an area, or bigger than +1/+1. A lone +1/+1 counts only on a pump spell or an engine; on a creature, only as counters or Roles. Not a bite (Removal), a self-pump, a pump only a tribe (colorless too) or tokens get, or a keyword but double strike. |
| **Protection** | Keeps something ELSE alive, held up in response or over an AREA: a keyword shield, prevention, redirection, regeneration, a saving blink or unsummon, a shield counter, phasing. Not one creature's static, conditional or Aura/Equipment shield, a tribe's, a fog, its own, nor damage a creature DEALS or aimed at YOU (Pacify). |
| **Burn** | Damage or life loss aimed at a FACE, however counted and however slow: "damage equal to the number of Swamps", an upkeep tax, life PAID to stop the card, two or more life, one point AGAIN and again, a hasty token gone at end step. Damage at a creature is Removal; "any target" is both. Not damage to YOURSELF, nor a TOKEN's own ability. |
| **Sacrifice** | An outlet you can feed your OWN creatures or artifacts AGAIN AND AGAIN — a cost, a repeating trigger, every upkeep, or a card you recast. Destroying or exiling your own counts. Not once: entering, as you cast it, kicker, exploit. Not a price for damage or lost life, not an edict, not a land, not an artifact token. |
| **Steal** | Takes what is somebody else's: control of a permanent, an EXCHANGE of two, the player, their life total, an AURA moved off their permanent, or a CARD out of their library, hand or graveyard — played, or reanimated onto your side. Giving one AWAY is the mirror image, and "from A graveyard" names no victim: that is Reanimate alone. |
| **Tutor** | Searches a library for a card you CHOOSE — by type, subtype, colour, count, by NAME, or just "a card" — to hand, battlefield, top, graveyard or exile. A land search is Ramp or ManaFixing. Revealing until a type turns up is not this: that picks for you. |
| **ManaFixing** | Fixes colours: a choice paid in MANA, any landcycling, a land with two abilities, a land fetched to HAND or kept from a few milled, a Treasure of YOURS. Free for a tap, a life, a sacrifice or a trigger is just Ramp (Birds), unless it's a land. Not a land put onto the battlefield, not restricted mana unless a land makes it. |
| **Pacify** | Neutralises somebody else's creature without killing it: tapped, stunned, phased out, kept from untapping, can't attack or block, base power 0 or a -N/-0 that lasts, or the damage it DEALS prevented. A tap need not last. Not a LAND that won't untap, not an artifact tap, not a tap on your own attack, not a price the card pays. |
| **LandDestruction** | Takes a land off the battlefield, however worded: in a type list, behind a count, beside a second target, by basic type ("destroy all Islands"), or in any sweep that takes them (Balance, Apocalypse). A land edict counts. Not a land in a GRAVEYARD, not an Aura ATTACHED to one, not your own, not mana denial. |
| **Mill** | Puts cards from ANOTHER player's library into their graveyard, or into exile when nobody plays them after, and rad counters handed to them. Filling your own graveyard is fuel for what the card does next, not an attack — dredge is a graveyard card, not a mill card. |
| **Regrowth** | Buys a card back out of a graveyard: to HAND, to the TOP of a library, or by CASTING it there or out of the exile it was moved to — reanimation for spells. A card coming back ITSELF (to hand, flashback, escape, unearth) is not. Nor is the BOTTOM of a library, or an opponent's graveyard. Straight to the battlefield is Reanimate. |
| **Redirect** | Acts on a spell without countering it: changes its targets, takes control of it, or copies it — anybody's ("copy TARGET spell") or YOUR OWN ("when you next cast… copy that spell", or storm or replicate given to your spells). Not a card copying only itself, and not a copied CREATURE spell, which is Tokens. |
| **Cheat** | Puts a NONLAND permanent onto the battlefield without casting it — from hand (Sneak Attack, Elvish Piper) or library (Natural Order) — or lets you cast free STANDING (Omniscience). Not one only a creature TYPE can use. Out of a graveyard is Reanimate; a land is Ramp; one free exiled card is CardAdvantage. |

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
