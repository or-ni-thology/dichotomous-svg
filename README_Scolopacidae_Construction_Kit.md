# The Scolopacidae Construction Kit

*A dichotomous morphology-assembler for the sandpipers, snipes and godwits of the North Pennines, with a determinative engine that adjudicates each constructed bird along a gradient from **catalogued** to **wholly imaginary**, and lays its egg accordingly.*

> The **O** of the wordmark is not a letter. It is a nest. A fresh snipe egg is laid into it on every page load. *Quid ni?*

---

## Argument

The Kit is a **combinatorial aviary**. The operator selects, from a small set of discrete morphological primitives — a body, a leg length, a bill, a head, a tail, a plumage tone, a habitat, and sundry extras — and the apparatus assembles the corresponding wader as a line drawing in SVG. Nothing is fixed in advance; the bird is a **coordinate in a discrete morphospace**, and the great majority of coordinates in that space correspond to no bird that has ever been ringed on Widdybank Fell.

This is the entire point. The Kit does not merely draw the eight or so Scolopacidae a diligent naturalist might expect. It draws the **spaces between them**, and it is scrupulously honest about what it has done. A bird assembled from an internally contradictory set of parts is not rejected — it is **determined**, graded, provided with a plausible museum provenance, and issued an egg. The instrument's temperament is that of a Victorian cataloguer who has been handed a specimen that cannot exist and resolves, with no change of expression, to catalogue it anyway.

The rendering doctrine is stated once, in the source, and adhered to without exception:

> *A rufous-chestnut woodcock is a bird whose hatch-lines and accent-strokes are inked rufous — not a brown-bodied shape.*

That is: **colour is carried by line, never by fill.** The plumage tone is the *ink* the colour-bearing marks take, not a wash poured into a silhouette. A bird with `color: null` is rendered as a clean mid-sepia blueprint — a draughtsman's skeleton, uncoloured because uncommitted.

---

## The determinative engine

This is the hazy part, so it is set out here in full and without mercy. The determination proceeds in four movements: **measure, rank, grade, drift.**

### 1. Measure — `partDist`

Every catalogue bird carries an **expectation vector** in `EXPECT`: the parts it *ought* to have. When the operator assembles a bird, the engine computes a **weighted morphological distance** between the assembled bird and each expectation, one axis at a time, in the function `partDist`.

Four of the axes are **structural** and heavily weighted; three are **cosmetic** and function only as tie-breakers. The doctrine, stated in the comment above the function, is unambiguous:

> *Structural parts (body, leg, bill, tail) verify a bird; colour & habitat are gentle tie-breakers only. One wrong structural part → cf.; two → aberrans.*

The axes and their normalisers:

| Axis | Source | How the difference is measured | Normaliser | Weight |
|---|---|---|---|---|
| **Body** | `BODY_SIZE` — an ordinal mass scale (`large: 9` … `small: 3`, with `rotund` and `pot_bellied` interposed) | absolute difference of the two mass ranks | ÷ 6 | **1.6** |
| **Leg** | `LEG_ORD` — the ladder `short → medium → long → very_long` | absolute difference of **ladder positions** (indices), so *very-long-vs-short* is maximally distant | ÷ 3 | **1.1** |
| **Bill length** | `BILL_CH[bill].len` — a length in notional millimetres (`short_straight: 20` … `very_long_decurved: 82`) | absolute difference in length | ÷ 62 | **1.1** |
| **Bill curve** | `BILL_CH[bill].curve` — a signed curvature (`-0.6` upturned … `+1` savagely decurved, `0` straight) | absolute difference in curvature | ÷ 2 | **1.0** |
| **Tail** | categorical | 0 if identical, else 1 | — | **0.9** |
| **Tone** | categorical | 0 if identical, else 1 | — | **0.3** |
| **Habitat** | categorical | 0 if identical, else 1 | — | **0.28** |

The normalisers exist to bring each raw axis onto a **commensurable scale** (roughly 0–1 per axis) *before* the weights are applied, so that a millimetre of bill and a rung of leg-ladder can be summed without one silently dominating the other. The weights then re-assert the hierarchy: a wrong **body** (1.6) hurts far more than a wrong **habitat** (0.28). Bill length and bill curve are deliberately **decomposed into two independent axes**, because the diagnostic distinction between a Whimbrel and a Curlew is not *how long* but *how bent* — the Whimbrel's bill has a bend at the midpoint, the Curlew's a smooth continuous decurve — and a single conflated "bill" axis would be blind to exactly the character that separates them.

The seven weighted, normalised terms are summed into a single scalar. Call it **`best`** once minimised (below). It is the bird's distance, in weighted morphospace, from the nearest thing that actually exists.

### 2. Rank — nearest catalogue landmark

`birdToEgg` iterates every expectation in `EXPECT`, computes `partDist` against each, and retains the **argument of the minimum** — the catalogue key `bk` whose expectation the assembled bird most nearly satisfies, and the distance `best` at which it does so. This is an unadorned **nearest-neighbour classification**: the bird is provisionally *this* species, because no other catalogue species lies closer in the weighted space.

### 3. Grade — the four tiers

The scalar `best` is then guillotined at three thresholds into four **determination tiers**:

| Condition | Tier | Label emitted | Sense |
|---|---|---|---|
| `best < 0.8` | `real` | the species' own name | a genuine, catalogued wader |
| `0.8 ≤ best < 1.6` | `cf` | ***Ovum incognitum*** · "cf. *name*" | close; the egg recognises the family, not quite the species |
| `1.6 ≤ best < 2.3` | `aberrans` | ***Ovum aberrans*** | an anomaly; determinable only by its nearest relative |
| `best ≥ 2.3` | `hypothetical` | ***Ovum hypotheticum*** | no bird precedes it; the egg precedes the bird |

The nomenclature is deliberate. *Incognitum* is the diffident **cf.** of the working taxonomist — *conferatur*, "compare with" — the honest admission that the specimen resembles a named form without being confidently assignable to it. *Aberrans* is the specimen that has left the principal series. *Hypotheticum* is the **hypothetical species** of nineteenth-century ornithology: a bird catalogued on the strength of a single dubious skin, a traveller's description, or a draughtsman's licence, and never satisfactorily rediscovered. Here the licence is the operator's, and the rediscovery is indefinitely deferred.

The three cosmetic tie-breakers earn their existence at exactly these thresholds. A bird one structural part away from a real species may fall either side of the `cf.`/`aberrans` line depending on whether its tone and habitat are *right*; a matching habitat can, at the margin, redeem an otherwise doubtful determination. They cannot rescue a structurally impossible bird — the weights forbid it — but they adjudicate the borderline, which is what a tie-breaker is for.

### 4. Drift — the widening of the egg

Here is the mechanism that makes the gradient *visible in the shell itself*. The engine computes:

```
drift = max(0, best − 0.8)
```

— the bird's distance **beyond the real-tier threshold**. For a genuine species, `drift` is zero. For everything else it grows with departure from the catalogue.

Each morphological parameter of the egg is then set as the catalogue **anchor** value plus a **signed, seeded perturbation scaled by `drift`**:

```
len     = anchor.len     ± (drift × 9)      mm
wid     = anchor.wid     ± (drift × 6)      mm
d       = anchor.d       ± (drift × 0.16)   [clamped 0 … 0.46]
capward = anchor.capward ± (drift × 0.3)    [clamped 0.1 … 1]
coarse  = anchor.coarse  ± (drift × 0.3)    [clamped 0.25 … 1.1]
density = anchor.density ± (drift × 10)     [clamped 16 … 56]
```

The `± ` in each line is `(rr() × 2 − 1)` — a seeded random draw on the interval [−1, +1]. The consequence is a strict and legible correspondence: **a real egg sits exactly on its catalogue anchor; a *cf.* egg trembles slightly around it; an *aberrans* egg wanders; a *hypotheticum* egg may be grotesquely elongated, savagely pointed, densely or sparsely freckled** — the geometry straying in proportion to how far the bird has left the family. The clamps (`Math.max`/`Math.min` on every line) are the apparatus's last concession to physical possibility: however imaginary the bird, its egg may not have negative asymmetry or a coarseness beyond the renderable range. The egg is permitted to be impossible, but not incoherent.

Colour does **not** drift. `ground`, `shade`, `blot` and `grey` are inherited verbatim from the anchor. An imaginary North Pennine wader still lays a recognisably Scolopacid egg; only its *form* betrays that it never existed.

---

## Determinism: seeds and hashes

Nothing here is left to ambient chance. Identical inputs yield byte-identical eggs, always. Two hashing regimes cooperate:

- **`hashStr` — the bird's identity.** A variant of the **Fowler–Noll–Vo (FNV-1a)** non-cryptographic hash: each character is XORed into the accumulator and the accumulator multiplied by the FNV prime `0x01000193`, all forced through `>>> 0` to stay an unsigned 32-bit integer. The offset basis, however, is not FNV's canonical `2166136261` — it has been quietly reset to **`1817`**, the year of the type description of a certain wader, so that the empty string hashes to the snipe rather than to nothing. `birdSeed` feeds the bird's entire part-list (sorted extras included, so ordering is irrelevant) through `hashStr` to produce the single integer that seeds that bird's every subsequent random draw.

- **`mulberry32` — the freckle scatter.** A compact, fast, seedable **pseudo-random number generator** (PRNG) with a full 2³² period. Given a seed it returns a deterministic stream of floats on [0, 1). It is used for the speckle placement, the drift perturbations, and the provenance selection — each seeded from a XOR-fold of the bird's seed with its own dimensions, so that a change in *any* measured quantity reshuffles the freckles in a stable, reproducible way.

The provenance lines are themselves seeded selections. `provLine` draws from one of two curated pools — `PROV.aberrant` and `PROV.hypothetical` — indexed by a `mulberry32` stream seeded from the egg's dimensions. Thus *"specimen labelled in error; no matching skin"* or *"the egg of a wader that declined to exist"* is not sprinkled on at random each render, but is a **stable property of that particular impossible bird**, as fixed as its length and as reproducible as its freckles.

---

## The clinamen

One control violates the determinism, deliberately and by name. The button **LAY THE EGG** increments a `layNonce`, and this nonce perturbs the **mottle scatter alone** — never the identity, dimensions, tier, or code. The comment names the manoeuvre exactly:

> *this swerves the speckle scatter alone — the **clinamen** of the particular egg.*

The **clinamen** is Lucretius' term — borrowed from the Epicurean physics of the *De rerum natura* and enthusiastically repurposed by the 'Pataphysicians — for the minimal, uncaused **swerve** of an atom from its determined path, the infinitesimal deviation that makes a cosmos possible rather than a sterile rain of parallel lines. Here it is the swerve of the freckles: two eggs of the identical bird, laid one after the other, are the same egg in every catalogued respect and yet are freckled differently, because each laying introduces its own small ungoverned deviation. The bird is fixed. The egg is fixed. The scatter of pigment across the shell is the one place the apparatus permits contingency to enter.

---

## Transmission — the `ov1~` protocol

The Kit is one of two instruments; its companion is the **[Ova Transmissor](./README_Ova_Transmissor.md)**. An egg determined here can be **transmitted** there, exactly, and rebuilt without loss. The mechanism honours a distinction of taxonomic principle:

- **A real (or *cf.*) egg travels as its name.** The transmit code is simply `species·seed` — e.g. `snipe·1817`. The catalogue is a **shared dictionary** held identically by both instruments; the word is sufficient, because the receiver can look the species up and reconstruct the anchor. This is nominalism made practical: the name suffices *because both parties already possess the thing it names.*

- **An imaginary egg has no name, so it travels as its body.** A *'patovum* — a 'pataphysical ovum — cannot be transmitted by reference, because no shared catalogue contains it. It is therefore **serialised whole** by `encodeRichCode`: the tier, every morphological parameter at full precision, all four colours packed as raw six-digit hex, the exact mottle seed in base-36, the nearest-species whisper, and the provenance line itself — URI-encoded — riding last after a `|`. The whole is prefixed with the marker **`ov1~`**, by which the Transmissor recognises a body-code on sight.

The governing comment states the ethic:

> *the imaginary egg arrives intact — still imaginary, never demoted in transit. Same egg in, same egg out.*

The refusal to *demote in transit* is the moral centre of the whole contraption. A lesser apparatus would round the imaginary egg toward the nearest real one for convenience of storage. This one insists that a hypothetical wader's egg has exactly as much right to faithful reproduction as a Curlew's, and expends the extra bytes to guarantee it. The parameters must round-trip **to the bit** — the comment says so — because the receiving engine reads `capward`, `coarse` and `density` *exactly*, and a rounding error would silently reshape the freckle scatter. The imaginary is held to the same standard of reproducibility as the real, which is the highest compliment a deterministic system can pay to a thing that does not exist.

---

## Glossary of the deliberately obscure

| Term | Meaning as used here |
|---|---|
| **Scolopacidae** | the sandpiper family — snipes, godwits, curlews, dunlins, the woodcock and their kin |
| **morphospace** | the abstract multidimensional space whose axes are morphological characters; every possible form is a point in it, most points being occupied by no real organism |
| **dichotomous** | proceeding by successive either/or branchings, as a traditional identification key does |
| **`cf.` / *conferatur*** | "compare with"; the taxonomist's mark of a resemblance falling short of confident identification |
| ***incognitum*** | unknown, unrecognised |
| ***aberrans*** | departing from the type; anomalous |
| ***hypotheticum*** | a species posited but never securely demonstrated |
| **'patovum** | coinage: a 'pataphysical ovum; the egg of a bird that is an imaginary solution |
| **'Pataphysics** | Jarry's science of imaginary solutions and of the laws governing exceptions; the presiding spirit of the whole determination |
| **clinamen** | the Lucretian atomic swerve; here, the ungoverned deviation of the freckle scatter at each laying |
| **Hügelschäffer** | the egg-outline model whose asymmetry term `d` renders one pole broader than the other — the reason a real egg is egg-shaped and not elliptical |
| **capward** | non-standard parameter: the degree to which pigment crowds toward the broad (cap) pole, as it does in real wader eggs |
| **FNV-1a** | the Fowler–Noll–Vo hash regime underlying `hashStr`, here with its offset basis reset to `1817` |
| **mulberry32** | a compact seedable 32-bit PRNG; the source of every reproducible random draw |
| **Herbst's corpuscles** | mechanoreceptors clustered at a wader's bill-tip, permitting the detection of prey by pressure-wave through wet substrate (recorded in the anatomy notes, and the reason the bill primitives matter) |

---

*or·ni·thology · Quid ni?*
