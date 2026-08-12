[READMEdomination.md](https://github.com/user-attachments/files/31001729/READMEdomination.md)
# Dominion — a solo Risk-style strategy game

1 human vs. 1–5 CPU opponents on the classic 42-territory map. A headless, UI-agnostic
rules engine plus a self-contained playable board.

**Play it:** open `play.html` in a browser. No server, no install, no build step needed.

```bash
node build.js          # rebuild play.html from src/ after any change
node test.js           # 137 engine tests — rules math, edge cases, 25 full games
node uitest/run.js     # 189 UI tests — drives real click handlers in a stubbed DOM
node bench.js 200      # AI tuning — measures whether difficulty tiers separate
npm run validate       # map graph + geography checks
npm run check          # validate, then both test suites
```

---

## Starting a game

Two ways to get territories, chosen at setup:

- **Draft** — players claim territories one at a time until all 42 are gone (one army
  each), then place their remaining armies. You pick first. You place your whole pool in
  one go; the CPUs rotate one army at a time, which is the classic rhythm and stops them
  simply mirroring a finished position. **Auto-assign the rest** hands the remainder to the
  computer at any point, so a draft you have tired of does not have to be finished by hand.
- **Auto-assign** — territories dealt at random and armies scattered, as before.

CPUs draft by difficulty. The value they place on a continent is *reward per unit of
defending* — bonus divided by how many of its territories touch the outside, discounted
when the continent is larger than one player's share of the map. Bonus-per-territory is the
tempting metric and the wrong one: it rates Europe and Asia top, and both are notoriously
hard to keep. See "One benchmark result worth keeping" below.

## Undo, hover and keyboard

**Placement undo.** Reinforcement is the one phase where a misclick is pure loss with no dice
to blame, and it is the most frequent interaction in the game. The engine mutates in place by
design, so rather than snapshotting whole game states it keeps a small per-player log of troop
placements and reverses them one at a time — `undoPlacement`, `undoableCount`. It covers the
initial deployment too, which is 30-odd clicks.

Two rules worth knowing:

- **Card trade-ins are not undoable.** They consume the set into the discard pile and advance
  the escalating trade-in counter; reversing one would either leak deck information or
  desynchronise the schedule.
- **An undo can never empty a territory.** If losses since the placement mean reversing it
  would drop the territory below one army — erasing ownership — it is refused.

**Hover** lights up a territory and every territory it borders, with a tooltip showing owner,
troop count, border count, continent and the card symbol it would earn. Without it you have to
commit to an attack origin before you can discover what it borders. It touches only CSS classes
and one overlay group, so moving the cursor never redraws the board.

**Keys:** `Space` advances the phase, `U` undoes a placement, `Escape` drops the current
selection. Ignored while an animation runs, while a CPU is moving, or while typing in a field.

## Ending a game

**The summary** is built from data the engine records as it goes, not reconstructed at the
end: per-player totals (battles, troops lost and killed, territories taken and lost, card sets
traded, peak holdings) plus a territory count snapshot at the close of every round. Keeping it
in the engine means a *resumed* game still knows what happened before it was saved.

The result sheet shows headline figures, a switchable chart, and a full stat grid.

**Six chart views**, because the measures diverge and the divergence is the point:

| View | Shows | Why it earns a tab |
|---|---|---|
| Territories | Land held per round | The default — who expanded, and when |
| Troops | Armies on the board | Diverges from land when someone stacks a border instead of spreading |
| Income | Reinforcements earned per turn | The number that *compounds*; a lead here arrives before a lead in territory |
| Continents | Whole continents held | Step changes — each rise is a lockdown, each fall a break-in |
| Casualties | Killed against lost, with the ratio | Who traded well, independent of who won |
| Rivalries | Conquests split by victim | Who actually fought whom — invisible in any total |

Income is the sharpest single read. In one recorded game the winner's income ran 3→7→8→14 by
round seven while territory counts still looked close; the chart makes that lead legible in
hindsight in a way the final score never could.

Rivalries needed a new engine field (`capturedFrom`, a per-victim tally) — everything else
came from measures already recorded. The chart is what makes a game legible in hindsight — you can see where
a runaway started, or the round your position collapsed.

Two bugs worth recording. The first was mine, introduced while adding the rivalry tally: I
wrote `const from = state.stats[player.id].capturedFrom` **inside** the capture block, where
`from` already meant the attacking territory. The shadowed name made `from.troopCount`
undefined, so `pendingOccupy` got `NaN` bounds — and because every comparison against `NaN` is
false, it sailed straight through `occupy()`'s range check and spread `NaN` troop counts across
the board, surfacing much later as an unrelated crash in the combat odds table. `occupy`,
`placeArmies` and `fortify` now check finiteness explicitly, and a test walks six full games
asserting every troop count stays a positive integer.

The second: `recordRound` originally ran *after* the winner check in
`endTurn`, so the deciding round never entered the history and the chart stopped one round
short of the conquest it was meant to show. It now snapshots before the check, and
`checkWinner` records too — because a game can end mid-assault, where `endTurn` never runs.

**Concede** hands the conceding player's territories to whoever currently leads and ends the
game. Passing them on rather than freezing the board mid-collapse keeps the final position
coherent, and the summary marks the ending as `concede` rather than pretending it was a
conquest. A late game against a runaway opponent can take many more rounds to actually finish
and there is no reason to make someone play it out.

## Saving and resuming

`serializeGame` captures the game as a plain object; `deserializeGame` rebuilds it. The only
part that needs care is the RNG, which is a closure — `rng.getState()` captures the
generator's position so a resumed game continues the same sequence instead of re-rolling from
the seed and diverging. Adjacency is rebuilt from the map data rather than stored, so a save
holds only what actually changes.

Saves are **files the player downloads**, not browser storage. This board is opened straight
from disk and sometimes inside a preview frame where storage APIs are unavailable or wiped; a
file survives both, and can be kept, copied, or handed to someone else. Loads that fail —
corrupt JSON, a save from a different `SAVE_VERSION` — report the reason rather than failing
silently.

## Dice

A single **Roll once** tumbles the dice over the contested territory, then settles them into
the panel: red for the attacker, bone for the defender, the printed-board convention. The
panel then shows the *pairing* — highest against highest, second against second, ties to the
defender — because that is the rule players most often get wrong and the board can simply
show it.

**Press the attack rolls no dice.** It can resolve a dozen rounds, and animating each would
turn a long assault into half a minute of waiting. The split is deliberate: `Roll once` is the
deliberate, watchable action; `Press` is the fast-forward. CPU attacks show no dice either,
only the territory-drain animation.

The tumble draws its faces from `Math.random`, never the game RNG — the real result comes from
`resolveCombat` before the animation starts, so the display cannot consume seeded randomness
and desynchronise a reproducible game.

**The tumble decelerates**, and it is one continuous loop. Both of those matter, and both were
wrong at first:

- A *fixed* face-swap interval reads as a slot machine however long it runs. Real dice lose
  energy, so the interval eases from `FACE_MIN_MS` to `FACE_MAX_MS` across the roll, with the
  wobble and hop decaying alongside. Duration alone does not fix the feel; the rate curve does.
- The first version tumbled all the dice to a stop and *then* assigned their true faces in a
  second loop. Dice waiting their turn sat motionless on a wrong number before snapping to the
  result. Now unsettled dice keep rolling while the earlier ones land, so **each die's last
  visible change is the moment it lands**. `uitest` covers this: it polls the pip counts through
  a real 2-second roll and fails if any die sits still for longer than one slow interval before
  settling. Reverting to the two-loop version reproduces a ~940ms freeze and the test catches it.

Three dials, next to the `SPEEDS` table:

- `DICE_PACE` — currently `2` (half speed). Raising it stretches a roll that already slows into
  its stop, rather than spinning for longer.
- `FACE_MIN_MS` / `FACE_MAX_MS` — currently 150 and 620. This pair, not the duration, is what
  decides whether the roll reads as dice or as a slot machine. Raise both to make faces linger.
- `SPEEDS[tier].dice` — the per-tier base. As configured a roll runs about 3.2s on slow with
  ~10 face changes, 2.2s on normal, 1.1s on fast, and is skipped on instant.

If the timing still feels off, the useful thing to identify is *which part*: faces flickering
too fast is the `FACE_*` pair; a roll that drags is `DICE_PACE`; dice landing too close together
is the `stagger` inside `rollDiceOnMap`.

## Pacing and animation

A CPU turn arrives as a sequence of events, not a finished result. `stepTurn()` is a
generator that yields one event per meaningful action — each battle round, each capture,
the fortify — and the UI drives it with a delay between steps. Headless callers drain it
and pay nothing, so tests and the benchmark stay instant; `takeTurn()` is exactly that
drain, which is why every earlier caller kept working when this landed.

That split is what makes a territory changing hands watchable. A losing defender's disc
shrinks in proportion to the troops still standing, and when it falls the attacker's colour
floods in. Same visual language whoever is attacking, at half the duration for your own
attacks so your turn stays quick.

Speed is a control in the top bar — **slow / normal / fast / instant** — rather than a
hard-coded pace. It governs the dice too; `instant` skips the tumble entirely.

Two separate dials sit on top of it, both in the UI script near the `SPEEDS` table:

- `CPU_ATTACK_PACE` — stretches NPC battle steps only. Currently `1 / 0.85`, a 15% speed
  reduction. Raise it to slow the opponents further.
- `HUMAN_BATTLE_SCALE` — your own attacks, currently `0.5` of the base pace.

They are deliberately independent, so tuning how long you watch an opponent fight never
changes how fast your own turn plays. At normal speed that works out to 353ms per NPC
battle step against 150ms for yours.

---

## Architecture

```
src/
  map/mapData.js          42 territories, 6 continents, adjacency graph (83 edges)
  map/validate.js         integrity checks — symmetry, connectivity, continent sizes
  state/gameState.js      state model, setup, derived-state queries
  state/rng.js            seeded RNG (reproducible games)
  cards/deck.js           deck, set detection, escalating trade-in values
  combat/resolveCombat.js SWAPPABLE combat + exact win-probability tables
  turns/engine.js         the phase machine (all rule enforcement lives here)
  turns/setup.js          the draft — claiming territories, deploying starting armies
  ai/difficulty.js        tier config — weights only, no logic
  ai/ai.js                shared AI engine
  ui/layout.js            board coordinates and display labels
  ui/geography.js         coastlines, terrain, sea labels, compass
  ui/validate-geography.js  keeps coastlines and coordinates in sync
  ui/index.template.html  the interface
  index.js                public API
build.js                  bundles src/ into play.html
uitest/                   DOM stub + UI integration tests
```

**Four rules everything follows:**

1. **Ownership and troop counts live only on territories.** Territory counts, continent
   control, borders, and reachability are all *derived* — the state can never contradict itself.
2. **No action throws on an illegal move.** Every engine action returns `{ ok: true, ... }`
   or `{ ok: false, error }`, so the UI calls optimistically and shows the error string.
3. **The UI re-implements no rules.** It drives the same public actions the CPU does.
   Anything the AI can do a human can do, and vice versa.
4. **`src/` is the single source of truth.** `play.html` is generated — never hand-edit it.

### Why play.html is bundled

Browsers block ES module imports over `file://`, so an unbundled build would only run
behind a local server. `build.js` flattens the modules into one file that opens by
double-clicking. It strips import/export syntax and rebuilds the `engine.` namespace
that `ai.js` calls through.

Flattening removes module scope, so two files may legally declare the same top-level name
and only collide once bundled — which happened twice during development. The build now
fails loudly on duplicate top-level declarations rather than shipping a blank page.

---

## Turn flow

```js
import { createGame, beginTurn, placeArmies, endReinforcement, attack, occupy,
         endAttack, fortify, endTurn, runCpuTurn, PHASES } from './src/index.js';

const state = createGame({
  humanName: 'You',
  cpuDifficulties: ['easy', 'medium', 'hard'],   // 1–5 CPUs
  seed: 42,                                       // optional, for reproducible games
  draft: true,                                    // false = deal at random
});
```

**Draft setup** (only when `draft: true` — `finishSetup` calls `beginTurn` for you):
```js
claimTerritory(state, 'brazil');                  // one per player, until all 42 are gone
deployInitial(state, 'brazil', 5);                // then place remaining armies
autoAssignRemaining(state, pickClaim, pickDeploy); // or hand the rest to the computer
```

With `draft: false`, call `beginTurn(state)` yourself and skip straight to the turn loop.

**Reinforce** — `state.turn.pendingReinforcements` holds the count.
```js
tradeInCards(state, [cardId, cardId, cardId]);   // required at 5+ cards
placeArmies(state, 'brazil', 3);
endReinforcement(state);                          // refused while troops remain
```

**Attack** — each `attack()` is *one round* of combat.
```js
const r = attack(state, 'brazil', 'northAfrica');
// r.attackerRolls, r.defenderRolls, r.attackerLosses, r.defenderLosses, r.territoryCaptured
if (state.turn.phase === PHASES.OCCUPY) occupy(state, 3);
endAttack(state);                                 // awards a card if anything was captured
```

**Fortify** — one move per turn, along a chain of connected owned territories.
```js
fortify(state, 'alaska', 'ontario', 5);
endTurn(state);                                   // advances and calls beginTurn
```

**CPU turns** — all at once, or step by step for animation:
```js
runCpuTurn(state);                                // plays the whole turn immediately
for (const ev of stepTurn(state)) { ... }         // or one event at a time
```

**Query helpers for a UI:** `territoriesOf`, `territoryCount`, `totalTroops`,
`continentsControlledBy`, `calculateReinforcements`, `borderTerritoriesOf`, `validAttacks`,
`connectedOwnedTerritories`, `activePlayers`, `attackWinProbability(a, d)`.

`validAttacks` and `connectedOwnedTerritories` exist so a UI can highlight legal targets
without reimplementing the rules.

---

## Rules implemented

| Rule | Implementation |
|---|---|
| Reinforcements | `max(3, floor(territories / 3))` + continent bonuses |
| Continent bonuses | NA +5, SA +2, EU +5, AF +3, AS +7, AU +2 |
| Starting armies | 40/35/30/25/20 by player count, territories dealt round-robin |
| Card sets | 3-alike or one-of-each; 2 wildcards in the deck |
| Trade-in values | 4, 6, 8, 10, 12, 15, then +5 per set |
| Forced trade-in | Required at 5+ cards held |
| Territory bonus | +2 troops on owned territories matching traded cards (configurable) |
| Card award | One per turn in which at least one territory was captured |
| Card counts | Opponent hand *sizes* are shown (public in Risk); faces stay hidden |
| Undo | Troop placements only, current phase only, never below one army |
| Fortify | One move, along any unbroken chain of owned territories |
| Elimination | Loser's cards transfer to the eliminator |
| Win condition | Own all 42 territories, or be the last player standing |

---

## Combat — locked, still swappable

**The model is classic Risk dice:** attacker rolls up to 3, defender up to 2, sorted and
compared pairwise, ties to the defender.

The engine calls exactly one function, so the mechanic can still be replaced without
touching turn logic, the AI, or the UI:

```js
resolveCombat(attackerTerritory, defenderTerritory, rng)
// → { attackerRolls, defenderRolls, attackerLosses, defenderLosses, territoryCaptured }
```

`attackWinProbability(a, d)` gives exact fight-to-the-finish odds, computed by enumerating
every dice matchup and recursing to absorption, memoised. The AI ranks attacks with it and
the board shows it on the odds bar.

**If you swap the combat model, update `attackWinProbability` to match** — it is the AI's
model of the world, and stale odds would make the AI play badly rather than fail loudly.

---

## AI difficulty

One engine; tiers are configuration in `ai/difficulty.js`. Adding a tier is adding a row.

| | Easy | Medium | Hard |
|---|---|---|---|
| Min win probability to attack | 0.35 | 0.60 | 0.60 |
| Randomness (chance of a non-optimal move) | 0.60 | 0.15 | 0.02 |
| Continent priority | 0.5 | 2.0 | 3.0 |
| Border defense | 0.2 | 1.0 | 2.0 |
| Target weak players | – | 0.5 | 1.5 |
| Position evaluation | – | – | 1-ply |
| Presses eliminations | – | – | yes |

Measured over 200 games per matchup (`node bench.js 200`):

| Matchup | Random start | Drafted start |
|---|---|---|
| hard vs easy | 95.8% / 4.2% | — |
| medium vs easy | 95.0% / 5.0% | — |
| hard vs medium | 72.5% / 27.5% | 78.0% / 22.0% |
| four-seat mixed | hard 48.7 / medium 33.3 / easy 18.0 | hard 50.0 / medium 35.0 / easy 15.0 |

### One benchmark result worth keeping

The first draft heuristic scored continents by bonus-per-territory. The benchmark caught it:
in a four-seat drafted game the tiers *inverted* — medium 40%, hard 32.5% — because that
metric ranks Europe and Asia top and sends the strongest AI after the two continents hardest
to hold. Rescoring on defensibility (bonus ÷ territories touching the outside, discounted
when the continent exceeds one player's share) restored the ordering and made hard stronger
than it is from a random start.

Head-to-head results looked fine throughout. Only the multi-seat game exposed it, which is
the argument for keeping `bench.js` around and running it after any weight change.

### One deviation from the plan, flagged

The plan specified *"2+ moves ahead, simple minimax"* for the hard tier. What's implemented
is a **1-ply static evaluation** of the position after a capture (continent completion,
stack overextension) rather than true multi-ply minimax.

With a 42-territory board and stochastic combat, real minimax must expand over both move
choices and combat outcome distributions, and the branching cost is large for little gain —
hard already wins 72% against medium. If it needs to be stronger, the cheap lever is tuning
the existing weights; the honest next step beyond that is Monte Carlo rollouts, not minimax.

---

## Interface

Mid-century atlas plate: navy ocean, slate chart lines, brass for instruments, sand for
lettering. Player colours are the loudest thing on the board, as on a real Risk box.
Barlow Condensed for cartographic lettering, IBM Plex Mono for every numeral — troop counts
and odds are instrument readouts and read as such.

**The signature element is the odds bar.** This engine computes exact fight-to-the-finish
probabilities, which most Risk implementations never surface. Selecting an attack draws a
tug-of-war between the two players' colours with the real number on it. Everything else
stays deliberately quiet.

### The map

Continents are hand-drawn coastlines in `ui/geography.js`, smoothed from point arrays into
closed curves, each backed by three widening ocean-depth bands stepping outward from the
shore — the cheapest trick on the board for making a coast read as a coast rather than a
shape floating on water, since it all happens in the background.

**Territory regions and biome fill.** The map only ever had nodes, so there was nothing to
fill. Each territory's region is a Voronoi cell — built by clipping the board rectangle with
the perpendicular bisector against every other territory on the same landmass — then clipped
to the coastline with an SVG `clipPath` rather than polygon-intersection maths. Filling those
cells by terrain type gives biome colour, and their shared edges give internal territory
borders for free, which is what turns the board from a network diagram into a political map.

Biome colours are deliberately dark and only moderately saturated. They sit *below* the
player colours in the visual hierarchy so a bright troop marker is still the loudest thing on
screen; raising their saturation is the fastest way to make ownership hard to scan.

**Continent cartouches.** Each continent gets a ruled plaque carrying its name and its
reinforcement bonus — `ASIA │ +7` — instead of the bonus living only in the side panel. A
printed board puts the reward on the territory it applies to, and it saves reading the panel
mid-turn to remember what Africa is worth.

Placement is a search, like the bare names it replaced, but a plaque is far larger than the
word inside it so the fit test uses the whole box, sizes step down until one fits, and each
placed plaque becomes an obstacle for the next. It also has to dodge the furniture already on
the chart: the first version knew only about territories and parked South America's plaque
squarely on top of "Atlantic Ocean".

**Holding a whole continent** lights its coastline, and its cartouche — plate, rules and name
take the owner's colour while the bonus stays brass — rather than washing the land. Once land carries biome colour there is nowhere left for an ownership
tint to go — a wash over terrain muddied both signals, so it moved to the coast and the
label, which carry it more clearly and leave the terrain intact.

Islands are separate shapes: Greenland, Iceland, Britain, Japan, Madagascar, Indonesia and
New Guinea. An earlier version wrapped each continent in a convex hull of its nodes, which
swallowed all seven and read as a blob.

Every territory carries a terrain type — mountain, forest, jungle, desert, ice, plains —
scattered as small glyphs around the node, kept on that territory's own landmass and clear
of every disc and label. Placement is seeded from the territory id, so the scatter is stable
across renders instead of reshuffling on every frame. Terrain is decoration; the engine
never reads it.

**Territory names are lettered across the land**, tracked out, rather than tagged under the
army marker. `letteringSpot` sweeps horizontal scan lines through each territory's own region,
measures the longest run clear of the marker, the card symbol and every name already placed,
then picks the largest size whose natural width fits — verifying the whole text *rectangle* is
clear, not just the baseline row, so a name cannot clip the top of a marker sitting above it.

Small islands cannot hold their own name, so a second pass lets the lettering run into
adjacent open water — beside the island, never across another landmass, and never more than
`waterReach` from its own marker. That is what an atlas does with Iceland or Madagascar.

Three numbers were tuned by looking at renders rather than by reasoning:

- **Tracking is capped** at 1.55× natural width. Uncapped, "SIAM" stretched across half a
  continent and adjacent names at the same height read as one long string.
- **A band must be wide enough to track out**, not merely to fit. Accepting a tighter one made
  `textLength` squeeze the letters *together* — compressed lettering, the opposite of intent.
- **`waterReach` is 52px.** At 78 a coastal name drifted until it belonged to nothing: "NW
  Territory" ended up in the Arctic. Tighter reach means one territory (Western United States)
  falls back to a tag under its marker, which is the better trade.

Lettering is placed *before* the terrain scatter and the cartouches, and both treat the name
boxes as obstacles — names matter more than decoration. As configured: 41 of 42 lettered on
the map, 25 at full size, 8 running into water, no name more than 49px from its marker.

**Card symbols** are printed above every territory, as on a classic Risk board. The deck has
always bound a card type to each territory (round-robin, 14 of each); naming that mapping
`TERRITORY_CARD_TYPE` lets the board show it. It is decoration *and* information — you can
see which symbols you are close to a set in before choosing where to attack.

**Sea routes.** Connections come in two kinds, and which is which is *derived*, not listed: a
connection is a sea route exactly when its two territories sit on different landmasses. On this
map that is exactly when they do not touch — the same rule a printed board uses to decide what
gets drawn as a line. It yields 30 sea routes and 53 land borders, and it picks out every
crossing the classic board draws: Alaska–Kamchatka, Brazil–North Africa, Greenland–Iceland,
Britain–Europe, Siam–Indonesia, and the rest.

Sea routes are dashed arcs with small port terminals. The arc bows to whichever side crosses
less of a landmass it *doesn't* connect — passing over its own two endpoints is expected and
fine, so scoring on raw distance from land just pushes short crossings out to sea for no
reason. Ties break toward open water, otherwise the bow direction would be arbitrary and the
arcs would zigzag.

Note this makes the Europe/Asia and Africa/Asia crossings sea routes. On a globe those are
isthmuses, but this board draws those continents as separate shapes with water between them,
so an arc is what the map actually shows.

**Links are authoritative; region edges are texture.** Territory regions are Voronoi cells,
which cannot be made to match an arbitrary graph — four pairs share an edge without being
connected (Congo–Egypt, Afghanistan–Siberia/Irkutsk/Mongolia) and four connected pairs do not
touch (China–Ural, China–Siberia, Mongolia–Siberia, East–North Africa). Rather than pretend
otherwise, region strokes are kept faint and the connection lines are drawn clearly: read
adjacency from the lines. `validate-geography.js` reports the count on every run so the gap
stays a known quantity.

**Frame and paper.** A printed border runs around the plate — heavy outer rule, light inner
rule, degree ticks along the margin, squared corner blocks — over a fine noise wash and a
corner vignette. The grain and vignette do most of the work: they unify flat vector fills into
something that reads as printed, at low enough opacity to be texture rather than dirt.

The frame lives in **extra viewBox space**, not inside the map. Content already ran to the
edges — the wraparound stubs reach x=6 and x=954, and the North America cartouche sits 4px
from the top — so insetting a frame would have meant moving all of it. `FRAME_MARGIN` grows
the plate around the existing 960×540 area instead, and every coordinate in `layout.js` stays
valid. A test asserts nothing crosses the inner rule; the tightest clearance is 21px.

Atlas furniture: a compass rose, italic open-water names, and the latitude/longitude
graticule underneath everything.

Three layout problems worth recording, since the fixes look arbitrary otherwise:

- **Continent names are placed by search, not at the centroid.** The centroid is exactly
  where territories cluster thickest, so every label landed on top of nodes. `watermarkSpot()`
  sweeps for the nearest position whose text box clears every node disc, label and terrain
  glyph, searching surrounding ocean too — which is what an atlas does when land is crowded.
- **Coastlines are validated against the node coordinates, not eyeballed.**
  `validate-geography.js` runs as part of the build. It checks every territory sits inside a
  landmass belonging to its own continent, that no two landmasses overlap, and — the check
  that matters — that there are at least 21px of coast around each node, because a 16px disc
  with a 20px selection ring hangs over the water otherwise. The first version of that file
  only checked the node *centre* was inside the shape, and passed a board where seven islands
  were smaller than their own markers.
- **Two territories moved to make room.** Great Britain and Quebec were close enough to the
  European and North American coasts that no shape could give both a clear marker. Coastlines
  and `layout.js` are coupled; when they fight, the validator says which one to move.
- **Voronoi cells are clipped per landmass, not globally.** A cell spans the whole board
  before clipping, so cells from different landmasses overlap freely; each landmass gets its
  own `clipPath`. Flattening them into one clip paints Africa over North America.
- **Territory labels and supply lines were re-tuned when biome fill landed.** Both were
  chosen against near-black land and lost contrast the moment the ground got lighter.

Accessibility: keyboard-focusable territories, visible focus rings, `prefers-reduced-motion`
respected, troop numerals pick their ink from the underlying colour's luminance, and the
layout stacks below 900px.

### Touch

Every primary action is a `click` handler, and `click` fires on tap, so the game is playable
on a touch device today. Three hardening fixes are in:

- **Hover styles sit behind `@media (hover: hover)`.** Touch browsers latch `:hover` after a
  tap and hold it until you tap elsewhere, which left the board looking like it had stale
  selection state. `build.js` fails on any ungated `:hover` — the rule is one CSS line away
  from being reintroduced and the test suite has no notion of pointer type.
- **`touch-action: manipulation`** on interactive elements, dropping the double-tap-to-zoom
  wait that otherwise delays every tap by up to 300ms.
- **Tap highlight and text selection suppressed on the board**, whose text is map lettering
  rather than content. Long-pressing a territory previously selected its name and raised the
  iOS callout menu.

Pinch-zoom is deliberately *not* disabled. Markers are well under the recommended tap size
(15px on a phone in portrait, 32px on a tablet, against a ~44px guideline), so zooming is
currently the main way to play on a small screen.

Still outstanding for touch, in rough priority order: no touch path to the adjacency
highlight and tooltip (bound to `mouseenter`, and the most consequential gap — it is the
feature that lets you see what a territory borders *without* committing to it as an attack
origin); tap targets below guideline; save via `a.click()` on a blob URL, which may open a tab
instead of downloading on iOS and would fail silently; and the 900px breakpoint, which stacks
the panel under the board on a portrait tablet that has room for both side by side.

---

## Testing

`test.js` (84) covers the rules in isolation — including the draft, the deploy rhythm, and
the shape of the events `stepTurn` yields. `uitest/run.js` (79) loads the real UI out of
`play.html` into a stubbed DOM and drives complete turns through the actual click handlers:
claiming territories, deploying by hand, attacking, and a full game played to a winner from
the human seat.

That second suite exists because bundling introduced a bug unit tests could not see: the UI
called `runCpuTurn`, a name that only existed as an alias in `index.js`, which the bundle
skips. It would have died on the first CPU move. A second collision (`fail` declared in both
`engine.js` and `setup.js`) later justified the build-time duplicate check.

The UI's actions are async now that they animate, so the suite runs at `instant` speed and
awaits them.

Two things about the stub that will bite again if forgotten: its `innerHTML` only reflects what
was *assigned*, so assertions about buttons added via `addRow` must read the button elements
rather than the markup; and the test helpers must hide the setup overlay the way the real start
handler does, or the keyboard handler correctly refuses to act and every key test fails.

**The stub models `SVGAnimatedString`, and that is not a detail.** On HTML elements `className`
is a string; on SVG elements it is an `SVGAnimatedString` object. The stub originally stored a
plain string for both, so `el.className.includes(...)` passed every test and threw on the first
hover in a real browser. The stub now returns the right shape per element kind, and `build.js`
additionally fails on any string method applied to `.className` in the template. Reach for
`getAttribute('class')` on SVG, or keep a direct reference and skip the lookup.

The general lesson, since it has now happened twice in different forms: **a stub that is more
forgiving than the browser converts real bugs into passing tests.** When the stub and the
browser disagree, fix the stub.

Dice landing times are recorded by the animation into `diceTimeline` rather than inferred from
the displayed faces — a tumbling die can show its final face by chance, which made the
ordering assertion flaky until it read the real timestamps.

---

## Deliberately not built

- **Secret mission cards** — conquer-the-world only for v1
- **Save/load** — state is plain serialisable data apart from the `rng` closure; persisting
  needs `rng.seed` plus a call count, or a serialisable RNG
- **Undo beyond placement** — attacks and fortifies stay final; only troop placement reverses
- **Autosave** — saving is a deliberate action; nothing is written without the player asking
- **Multi-human play** — seat 0 is the only human seat
- **Touch input** — partially addressed, see below
- **Rivers, relief shading, and named seas beyond the five labelled** — the next increments
  of realism if the map wants more
- **Regions that match adjacency exactly** — would need hand-drawn territory outlines instead
  of Voronoi cells
- **Curved lettering** — names run horizontally; following a region's spine would need
  text-on-path and per-territory curves

### Known edge cases

- A player pushed to 6+ cards by a mid-turn elimination trades at the start of their next
  reinforcement rather than immediately. Strict rules force it right away — change in
  `checkElimination`.
- Seat order carries a genuine first-player advantage (visible in the benchmark). Inherent
  to Risk, not a bug, but worth knowing when reading tuning numbers.
- The board is tuned for the 42-territory map; `ui/layout.js` would need new coordinates
  for a different map.
- In a draft, an uneven split is expected: with 4 players two seats get 11 territories and
  two get 10, since claims simply rotate until the map is full.
