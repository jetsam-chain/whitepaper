# Jetsam: A Proof-Carrying Layer 1

**Whitepaper v3.1 — 15 October 2026** · supersedes v3.0 (12 October 2026) and v2.0 (24 September 2026)

> **Disclaimer.** This document describes a live network and a system still under
> active development. It is not an offer to sell securities, an investment
> recommendation, or a promise of future performance. It quotes no price, no
> valuation and no holder count. Parameters marked *open* or *provisional* are
> not final and will change.
>
> **What changed since v2.0.** Two consensus upgrades, both crossed. The CPU
> proof of work, TowerWalk, described in v2.0 as a plan, has been **live on the
> public network since block 24 846 (29 September 2026)**. The v1.5 rules —
> 180-second blocks, the client proof slot and a shortened
> development-allocation window — have been **in force since block 38 400,
> crossed on 13 October 2026 at 20:48:25 UTC**. That crossing was not clean:
> about 70 % of the network had not updated and kept mining the old rules, and
> the branch it produced stalled the synchronisation of up-to-date nodes. The
> v1.5.1 release of 14 October 2026 fixed that fault and pinned blocks 38 399
> and 38 400 (sections 2 and 8). Every number that moved is retagged below;
> section 12 lists the differences. The code reference for every **[code]** tag
> is the public repository at tag `v1.5.1` (commit `dce63e6`).
>
> **How to read the numbers.** Every figure in this document carries a tag that
> says where it comes from, because they do not all come from the same place:
>
> | tag | meaning |
> |---|---|
> | **[code]** | a constant in the consensus source. Anyone can read it in the public repository and it is enforced by the network. |
> | **[chain]** | read from the public chain through a node or the explorer, with the date of reading. It will have moved by the time you read this. |
> | **[measured]** | our own benchmark. The method is stated next to it. |
> | **[derived]** | arithmetic on figures that are themselves tagged. |
> | **[estimate]** | a projection or an order of magnitude. Not measured, and we say so rather than rounding it into a fact. |
>
> Where a claim is inherited from published research rather than proven by us,
> the text says whose result it is.

---

## Contents

0. [Abstract](#0-abstract)
1. [Why Jetsam exists](#1-why-jetsam-exists)
2. [What is live today](#2-what-is-live-today)
3. [Design principles](#3-design-principles)
4. [Architecture](#4-architecture)
5. [Proof of work](#5-proof-of-work)
6. [The proof market](#6-the-proof-market)
7. [Token economics](#7-token-economics)
8. [Roadmap](#8-roadmap)
9. [What we do not claim](#9-what-we-do-not-claim)
10. [Where Jetsam sits](#10-where-jetsam-sits)
11. [Governance, licensing and provenance](#11-governance-licensing-and-provenance)
12. [Glossary, references and links](#12-glossary-references-and-links)

---

## 0. Abstract

Jetsam (ticker **JTM**) is a proof-of-work Layer 1 whose every block carries a
recursive proof. Each proof covers the history from the root of its relation
generation; below that root, the chain rests on checkpoints published in the
binary (section 4.2). A joining node authenticates the current state, verifies
one proof, checks the last eight blocks and the proof of work of the headers.
It never replays transactions, so the cost of checking proofs does not grow with
the age of the chain; checking headers does grow, linearly, but lightly.

The public network has been producing blocks since 3 September 2026 **[chain]**
and has crossed five consensus upgrades, the latest at block 38 400 on
13 October 2026 **[chain]**. The chain has stopped twice: for 6 146 s on
7 September 2026, for a reason unrelated to an upgrade, and for about
49 minutes on 13 October 2026, in the hours after the fifth upgrade, when most
of the hashrate stayed on the old rules (section 2). Supply is hard-capped at
21 000 000 JTM **[code]**.
The genesis block carries no coinbase and no coins existed before mining
**[code]**; a **development allocation of 10 % of the block subsidy**, paid by
consensus on a published schedule that ends, runs for the first two target-time
years (section 7.3). We state both facts together, because "no premine" alone
would be the flattering half.

Building that machinery gives Jetsam something most chains have to buy: recursive
proving capacity, a verifier already inside consensus, and a settlement layer.
The product Jetsam is being built towards is the sale of that capacity — a
registry where an application can prove, in public and permanently, that it
followed a rule it said it would follow. Getting there required freeing the GPUs
that used to mine, which is why the proof of work **has been moved** onto a
memory-latency-bound function on which a server CPU socket does work of the same
order as a datacentre accelerator, for an estimated three to five times less
power **[estimate — extrapolated from a single measured core; see section 5.4]**.
That move happened at block 24 846. The next step, the client proof slot, opened
at block 38 400 with an **empty catalogue**: the machinery shipped before the first
customer, and this document says so rather than implying a market exists.

Nothing in the consensus rules depends on an elliptic curve. Spending is a proof
of a hash preimage, not a signature.

---

## 1. Why Jetsam exists

### 1.1 History is a tax that compounds

On a conventional chain, the price of being able to disbelieve everyone is that
you must replay everything. A new node downloads every block ever mined and
re-executes it before it can say one true thing about the present. That cost
rises forever. In practice it does not stop people from running nodes — it stops
them from running *new* ones, and the set of parties who can independently check
the chain quietly narrows to the ones who started early or bought a server.

Jetsam's answer is the one thing recursive proofs are genuinely good at. Each
block ships a proof of exactly one statement: *the previous proof was valid.*
Induction does the rest — back to the root of the block's relation generation,
not further. When the rules change the constraint system, a new generation
starts from a new root, and the history below that root is held by checkpoints
published in the binary rather than by the proof (section 4.2). A node that
verifies the newest proof has verified the chain down to that root, without
replaying any transaction. At height 3 000 000 the proof check costs what it
costs at height 10. The headers are another matter: they are kept permanently
and a fresh node checks the proof of work of each one, which grows linearly with
height — about 1.85 ms per header on one server core **[measured, stated in the
source]**, so on the order of an hour and a half of one core at height
3 000 000 **[derived]**.

The state follows the same rule. Coins live in a fixed accumulator of slots;
spending an output erases its slot in place and the next transaction reuses the
seat. There is no append-only set of spent outputs growing forever. State is
proportional to live coins, not to everything that ever happened.

### 1.2 Proving is a cost centre almost everywhere

Every chain that has adopted zero-knowledge proofs has had to answer the same
question: who pays for the proving hardware? The answers are always one of three
— subsidise it with inflation, pass it to users as fees, or concentrate it into
the handful of operators who can afford the machines.

Jetsam starts from the opposite premise. **The proving capacity a chain builds
for itself is an asset it can sell.** A network that already runs a recursive
proof system for its own blocks has, in place and paid for, the three things a
proof market needs: hardware, a verifier that is already in consensus, and
settlement. The marginal cost of proving somebody else's computation is the slot
in the schedule it occupies, not the capital.

### 1.3 The customer is not another blockchain

The market of blockchains that need proofs is small and crowded with
well-funded competitors, all of them fighting over rollups. The market of
**applications that have to prove they do what they claim** is not small, and it
is not served by the systems fighting over rollups.

That market is growing for reasons that have nothing to do with cryptography.
Software agents now take actions with money, data and third-party systems; the
people who deploy them are being asked, by customers and by regulators, to show
what the agent did and did not do. The usual answer is a log file and a
promise — a log file the party under scrutiny wrote about itself.

The value of a registry has never come from its technology. It comes from the
fact that somebody *requires* you to be in it. TLS certificates and compliance
attestations are the model: nobody buys the cryptography, they buy the badge —
and the badge is worth something because a counterparty asks to see it. Jetsam's
long-term bet is that "verified on JTM" can be such a badge, and that a chain
which already verifies proofs inside its own consensus is the cheapest possible
place to hold that registry.

### 1.4 What a proof guarantees — and what it does not

This is the sentence most projects in this field leave out, so it goes early.

**A proof guarantees that a computation was performed correctly. It says nothing
about whether the inputs were true.** A service can prove, impeccably, that it
applied the correct rule to data it invented. For a proof to be worth anything
to a third party, the *input* has to be committed by somebody other than the
party being checked.

That single constraint decides which uses hold up and which are marketing. It is
why section 6 proposes the uses it proposes: cases where the inputs are
naturally witnessed by others — signed API calls, transactions, writes into
somebody else's system.

A second limit follows from the first. **We never prove the quality of a
judgement.** Not that an agent reasoned well, not that an answer was correct, not
that a model is good. We prove compliance with rules that were defined in
advance and are checkable by a machine. The failure mode that kills projects in
this field is selling a vague guarantee and delivering a narrow one. We would
rather state the narrow one first.

---

## 2. What is live today

| | | as of |
|---|---|---|
| first block mined | **3 September 2026, 16:48 UTC** **[chain]** | |
| height | **39 234** **[chain]** | 15 Oct 2026, 14:54 UTC |
| mean block interval | **≈ 90.4 s** against a 90 s target over blocks 1 → 38 399; **≈ 181.7 s** against a 180 s target over blocks 38 400 → 39 234 **[derived]** — header timestamps at both ends of each range, divided by the number of blocks between them | 15 Oct 2026 |
| circulating supply | **1 697 535 JTM** — 8.1 % of the cap **[chain]** | 15 Oct 2026 |
| genesis coinbase | **none** — emission starts at height 1 **[code]** | |
| development allocation | **10 % of the subsidy**, by consensus, inside a window that ends (section 7.3) **[code]** | |
| live slots in the state accumulator | **29 823** **[chain]** | 15 Oct 2026 |
| proof of work in force | **TowerWalk** (CPU, cache-resident) since block **24 846** — 29 Sep 2026, 18:21 UTC **[chain]** | |
| consensus upgrades crossed on the public chain | **five** — heights 2 000, 8 450, 17 750, 24 846, 38 400 **[chain]** | 13 Oct 2026 |
| rules in force | **v1.5** since block **38 400** — 13 Oct 2026, 20:48:25 UTC **[chain]**: 180 s blocks, client proof slot, shortened allocation window | |
| current release | **v1.5.1** (14 Oct 2026): the v1.5.0 rules unchanged, plus hard checkpoints at blocks 38 399 and 38 400 and a synchronisation fix (section 8) **[code]** | |

The five upgrades were crossed on 5, 12, 22 and 29 September and on 13 October
2026 **[chain]**. Each was published with its activation height already announced
and a notice period before it; only one of them, v1.3 (height 17 750), was first
published as a dormant binary — a build that contains the new rules but does not
activate them — and armed later. The notice was sometimes short: the first
upgrade (height 2 000) was released about two hours before its activation block
**[chain]**. None of the first four crossings stopped the chain; the fifth is
described below, because it did not go cleanly.

The chain has stopped twice, and both should be said here rather than found by
a reader. On 7 September 2026 no block was produced between heights 3 575 and
3 576 for **6 146 s** (about 1 h 42 min) **[chain]**: a block template had grown
into a proof class larger than the consensus size limit, so nodes in that state
rejected their own blocks. It was not an upgrade crossing, and the fix shipped
in v1.2.0 without a hard fork. The fourth upgrade replaced the mining
function itself, which invalidated every existing miner at a known height; the
chain crossed it with zero proof errors and the activation block is now pinned as
a hard checkpoint **[code]**.

The fifth upgrade, v1.5 at block 38 400, was crossed on 13 October 2026 at
20:48:25 UTC **[chain]** — but about 70 % of the network had not updated
**[estimate, from the hashrate that kept mining the old rules; stated in the
v1.5.1 release notes]**. Those nodes went on mining their own branch at the old
90-second target, invalid under v1.5 but growing taller than the real chain.
Two things followed. The v1.5 chain, left with little hashrate, produced no
block for **about 49 minutes** between heights 38 410 and 38 411 — header
timestamps 21:19 and 22:08 UTC **[chain]**. And a fault in v1.5.0 made an
up-to-date node that saw the taller old branch try to synchronise from it; that
attempt could never complete, and while it ran the node held back the real
chain, so running nodes stalled for up to about 30 minutes and a node starting
from scratch could fail to join at all. v1.5.1, published on 14 October 2026,
pins blocks 38 399 and 38 400 so that the old branch is refused by its block id
whatever its height or work, remembers a peer caught on a refused branch,
judges synchronisation candidates by work and against the pins, and never holds
the next valid block behind a pending synchronisation **[code]**. The consensus
rules did not change. Those are the project's real evidence of execution —
including the part that went wrong — and it is why they are stated with heights
and times rather than as ticks in a box.

The v1.5 rules were crossed on the third public test network at its own
activation height on 8 October 2026, with a build that preceded the final
`v1.5.0` tag, and the final build then ran on that network for 24 hours before
being armed on the public chain **[chain]**. What the test network did not
rehearse was a majority of the hashrate staying on the old rules after the
height; that is what happened on the public chain, and section 8 records the
lesson.

Two properties are worth pointing at because they are checkable by a stranger in
a few minutes:

- **The live state of the whole network is small enough to audit from a
  browser tab.** It holds 29 823 live slots at the time of reading **[chain]**;
  v2.0 measured the encoded form at 747 KB for 15 306 slots, which at the same
  size per slot puts today's state at about 1.5 MB **[derived, not re-measured]**.
  The state grows with live coins, not with chain age. The slot count is a live
  reading anyone can repeat against the explorer.
- **The emission schedule sums to the cap exactly**, and a consensus test checks
  it height by height rather than by tracking a running counter **[code]**.

### What is *not* live

Two things in this document are not on the public network at the time of
writing, and no sentence here should be read as suggesting otherwise.

- **The proof market** of section 6 has, since block 38 400, its on-chain
  carrier: a block can hold one client proof. What it does not have is a single
  registrable computation: the public network's catalogue is **empty** **[code]**,
  so no registration and no client proof can appear until a later release lists
  one. The SDK and the first catalogue entries are not shipped.
- **Zero-knowledge client proofs and recursive batching of client proofs** are
  an intended v1.6 line of work for the *test* network. Only early preparatory
  work exists; they have no date and no public activation height, and this
  document promises none (section 8).

---

## 3. Design principles

Five choices explain most of the rest of this document. Each is stated with the
reason, because a principle without a reason is a slogan.

### 3.1 No curves in consensus — there is nothing for Shor to break

Jetsam has **no signature scheme**. An address is a Poseidon2b digest of a
256-bit secret, and spending is a proof of knowledge of that preimage, bound to
the transaction body and folded into the block's recursive proof **[code]**. The
spending proof is **witness-hiding by construction**: it is built so as not to
reveal the secret, and the source certifies the finite-field lemmas this relies
on — but no complete hiding proof has been written, so we do not call it
zero-knowledge. There are no pairings and no elliptic curves in the consensus
rules, in the proof system, or in the proof of work.

That is a claim somebody will try to break, so here is exactly where the
boundary runs. No elliptic-curve or signature library is reachable from any
consensus, proof-system or proof-of-work crate. Curves *do* appear in the
dependency graph, and it would be dishonest to imply otherwise: the
peer-to-peer layer brings in an Ed25519 peer identity and an X25519 Noise
handshake, with the usual supporting crates. Every one of them traces back to
the networking library. None can authorise a transaction, create value, or
satisfy any ownership rule in consensus — they identify and encrypt a **link
between two machines**.

**The consequence, stated narrowly.** There is no elliptic-curve scheme in this
chain, so there is nothing here for Shor's algorithm to attack and nothing to
migrate when large quantum computers arrive. Grover's algorithm still applies to
preimage search and to the proof system, here as everywhere; it is a square-root
speed-up. The only quantum figure the source certifies is modest: about **64.7
bits** for the production profile in an idealised, sequential quantum-random-oracle
model **[code]** — see the table below.

**The consequence, stated honestly.** Removing signatures does not remove the
question, it moves it: security now rests on Poseidon2b and on the soundness of
the proof system. That is a real dependency, and here is where it actually
stands. The production profile targets **128-bit soundness**, and two different
statements sit behind that target.

| statement | value | status |
|---|---|---|
| Fiat–Shamir FRI security of the block-history proof (rate 1/4, 133 queries), by Block and Tiwari's method | **127 bits** provable, **127** conjectured | the **method** is their published result, not ours; the numbers are our profile put through it. Computed for the launch-generation proof classes; the v1.5 classes are not yet put through the certificate (section 4.2) |
| compact FRI profile documented in the proving stack (64 queries) | **128 bits** under the Reed–Solomon **capacity conjecture**; **≈ 64 bits** provable under the Johnson bound | the lowest provable figure documented anywhere in the stack |
| production profile against a quantum adversary | **≈ 64.7 bits** in an idealised, sequential quantum-random-oracle model **[code]** | a model, not a theorem about real quantum machines |

We take the **lowest** of these figures, about **64 bits provable**, as the floor
we are prepared to defend, and we do not average it upwards. Exactly which
component of the stack each figure governs is a point we are still documenting;
until that is published, read the 64-bit figure as the one that applies. Only
the conjectured figures reach 128 bits. The capacity conjecture is relied on by
every deployed FRI system we know of and it remains a conjecture; for the compact
profile, lifting the *provable* figure to 128 bits by query count alone would
take roughly 148 queries and about double the opening size, which is a trade we
have not made.

We keep proven, inherited and conjectured apart deliberately and everywhere in
this document. Merging them is the single most common way a project in this
field is taken apart, and averaging them into one confident number would buy a
better sentence at the cost of the document.

What "signatureless" does **not** mean: it does not mean private. Addresses,
amounts and fees are public (section 4.4).

### 3.2 The state forgets, and the proofs make forgetting safe

An append-only ledger is a promise to store everything forever. Jetsam erases
spent slots and reuses them. What makes that safe is that the proof, not the
archive, is what carries the guarantee that the erased history was valid — back
to the root of its relation generation, with the checkpoints of section 4.2
below it. Net-new
state growth is priced by a consensus burn, so the party that makes every node
carry more state is the party that pays for it (section 7.4).

### 3.3 The proof of work lives outside the circuit

The nonce enters neither the constraint system nor the HistoryStep relation
**[code]**. This is a deliberate architectural choice with a concrete
consequence: **the proof-of-work function can be replaced without regenerating a
single matrix.** On a chain that proves its own proof of work, changing the
mining function is a multi-month relation migration. Here it is a consensus
parameter and a new digest function.

That property is not an elegance. It is the only real mitigation Jetsam has
against a hardware monoculture forming around its mining function (section 9.1).

### 3.4 Relations have generations, and a node that cannot honour one refuses to start

When the rules change in a way that changes the constraint system, the matrices
change with them, and a block proved against the old matrices is only ever
verifiable against the old matrices. Jetsam makes this explicit: a binary carries
every relation generation it needs and selects by **the block's own height**,
never by the tip. A node whose activation height is armed but which carries no
matching parameter pack **refuses to start**, rather than running happily until
it reaches the fork and then stopping the chain for everybody **[code]**.

Generations are deliberately rare. After five consensus upgrades on the public
network there are exactly **three** of them **[code]**: the third, v1.5, moved
the two proof classes from circuit sizes 2^22 and 2^24 to **2^23 and 2^25**
rows, because the client arm is carried inside the relation. Consensus changes
that do not touch the constraint system — a wire-format change, or a new
proof-of-work digest — are *not* generations: the chain crosses them on the
parameters it already has. The proof-of-work replacement of section 5 added
none, which is section 3.3 paying for itself. A reader will meet the figure
"m = 25" in the release notes: it is the **size of the large circuit**, not a
count of anything per block.

### 3.5 No general-purpose virtual machine

Jetsam will not ship an EVM or a WASM VM inside its circuit, and this is a
permanent choice rather than a missing feature. Every opcode of a general VM
becomes constraint rows, and the budget does not survive the comparison. The row
budget of a block would stretch to a few thousand RISC-V cycles; **a single
token transfer on such a VM takes tens of thousands** **[estimate]**. The gap is
one to two orders of magnitude, before any application logic runs at all.

The consequence is section 6.3: clients get a **catalogue** of parameterised
computations, not a programming language. That is a smaller promise, and it is
one the circuit can actually keep.

---

## 4. Architecture

### 4.1 One permutation: Poseidon2b over GF(2^128)

Every hash in Jetsam — Merkle trees, transaction roots, state commitments, the
proof-of-work seed — is built from a single permutation, **Poseidon2b**, over the
binary tower field GF(2^128). We did not invent it: it is published work (Grassi
et al., ePrint 2025/1893) and it is named in our references and in the
repository's `NOTICE`.

A binary field is chosen over the prime fields common in ZK systems for one
practical reason. On a binary field the arithmetic is XOR and carry-less
multiplication rather than modular reduction — the same algebra the proof system
natively speaks, and an operation that maps directly onto the `PCLMULQDQ`
instruction, present on mainstream x86 CPUs since 2010–2011, and onto `PMULL`
on ARM. The
hash and the circuit are not two worlds that have to be bridged.

| permutation parameter | value |
|---|---:|
| state width | **4** field elements (rate 2, capacity 2) **[code]** |
| rounds | **8 full + 58 partial = 66** **[code]** |
| S-box | x^7 **[code]** |

Domain separation is explicit rather than implied: each use of the permutation
absorbs a distinct 8-byte tag, so a digest computed for one purpose can never be
replayed as a digest for another **[code]**.

Carry-less multiplication is the one hard hardware requirement in the whole
system. A node needs SSE4.1 and `PCLMULQDQ` on x86-64, or NEON and `PMULL` on
ARM64, and refuses to start otherwise rather than silently falling back to a path
that would make it useless **[code]**.

### 4.2 HistoryStep: the block that proves the block before it

<div class="fig">
<svg viewBox="0 0 720 150" role="img" aria-label="Each block's proof verifies the previous block's terminal commitment, chaining back to the root of its relation generation.">
  <g fill="none" stroke="currentColor" stroke-width="1.3">
    <rect x="18" y="44" width="150" height="62" rx="4"/>
    <rect x="248" y="44" width="150" height="62" rx="4"/>
    <rect x="478" y="44" width="150" height="62" rx="4"/>
    <path d="M168 75 H243" stroke-width="1.6"/>
    <path d="M398 75 H473" stroke-width="1.6"/>
    <path d="M232 70 l11 5 -11 5 z" fill="currentColor" stroke="none"/>
    <path d="M462 70 l11 5 -11 5 z" fill="currentColor" stroke="none"/>
    <path d="M655 75 H690" stroke-dasharray="4 4"/>
  </g>
  <g fill="currentColor" font-family="ui-sans-serif,system-ui,sans-serif" font-size="12">
    <text x="32" y="66">block n−2</text><text x="32" y="86" opacity=".65">proof π(n−2)</text><text x="32" y="100" opacity=".65">terminal τ(n−2)</text>
    <text x="262" y="66">block n−1</text><text x="262" y="86" opacity=".65">proof π(n−1)</text><text x="262" y="100" opacity=".65">terminal τ(n−1)</text>
    <text x="492" y="66">block n</text><text x="492" y="86" opacity=".65">proof π(n)</text><text x="492" y="100" opacity=".65">terminal τ(n)</text>
    <text x="176" y="36" font-size="11" opacity=".75">π(n−1) proves τ(n−2) was valid</text>
    <text x="406" y="36" font-size="11" opacity=".75">π(n) proves τ(n−1) was valid</text>
    <text x="18" y="134" font-size="11" opacity=".75">A verifier reads only the rightmost box. Induction covers everything back to the generation's root.</text>
  </g>
</svg>
</div>

Each block carries a proof of one relation, **HistoryStep**, stating that the
block's state transition follows the consensus rules *and* that the previous
block's terminal commitment was valid. The proof for block *n* never re-opens
history — the proof for block *n−1* already vouched for it.

**How far back the induction reaches.** Not to genesis. Each relation generation
(section 3.4) roots its recursion at the block just below its activation height,
and a proof at a given height covers the blocks above that root and nothing at or
below it **[code]**. Each proof therefore covers the history from the root of its
generation; below that, the chain rests on the permanent headers, whose proof of
work every node checks, and on checkpoints published in the binary — today the
hashes of blocks 24 846, 38 399 and 38 400 **[code]**. Under the v1.4.3 rules a
proof covered blocks 17 750 onward; since block 38 400 it covers 38 400 onward,
and v1.5.1 pins 38 399, the last block both rule sets share, and 38 400, the
activation block itself (section 8).

The proving stack is assembled from published components:

| component | what it is |
|---|---|
| constraint system | **R1CS over GF(2)**, block-diagonal, with its witness generation |
| polynomial commitment | multilinear, over binary fields, in the Binius line — GF(2) packing, additive NTT, Merkle commitment, ring-switching and a BaseFold-style folding |
| low-degree test | **FRI**, in a compact production profile |
| structured sub-relations | **GKR / sumcheck** arguments, used where they are cheaper than a generic circuit — including the witness-hiding spend authorisation of section 3.1 |
| recursion | the layer that folds block *n−1* into block *n*; it is self-recursive, verifying the parent's complete proof envelope |

The workspace also carries an executable soundness certificate: it loads the real
parameters from the crates that define them, returns an error and **fails its
tests** if any of them drifts from the profile it is supposed to be **[code]**.
It is a test-time check, not a build-time one, and it currently loads the
launch-generation proof classes (2^22 and 2^24 rows) only: the v1.5 classes
(2^23 and 2^25) are not yet put through it **[code]**.

### 4.3 State: a slot accumulator, not a growing set

Coins live in a fixed-size accumulator addressed by slot. Spending erases a slot
and returns it to the pool; the next output takes the seat. The accumulator
expands only when occupancy crosses a threshold, and the expansion is a protocol
event, not a per-transaction cost.

The practical figure is in section 2: the whole live state of the public network
encodes to about **1.5 MB** at today's slot count **[derived from the v2.0
measurement of 747 KB for 15 306 slots; not re-measured]**. That is the number
behind the claim that anyone can audit the live state from a browser tab.

### 4.4 Transactions

| | |
|---|---|
| model | transparent **UTXO** — amounts and owners are public. **Privacy is not claimed.** |
| shape | a fixed body carrying **up to 8 inputs and up to 2 outputs**; unused places are padded and are not counted. Conservation is `inputs = outputs + fee` **[code]** |
| addresses | **bech32m**, human-readable part `j` on the public chain, `tj` on the test chain — so `j1…` and `tj1…` **[code]** |
| spend authorisation | proof of knowledge of a Poseidon2b preimage, bound to the transaction body, witness-hiding by construction (no complete hiding proof) — **no signature scheme, no curve** **[code]** |
| fees | an explicit field in μJTM, on a deterministic schedule (section 7.4); miner-claimable except for the burned part |
| precision | 1 JTM = 1 000 000 μJTM **[code]** |

A developer who wants to build on Jetsam today is building on a transparent UTXO
chain with hash-based spend authorisation and a public JSON-RPC interface. The
proof market of section 6 does not exist yet, and nothing in this document should
be read as an SDK announcement.

### 4.5 Block cadence and finality

| parameter | below block 38 400 | from block 38 400 (v1.5, in force since 13 Oct 2026) | source |
|---|---:|---:|---|
| target block interval | **90 s** | **180 s** | `BLOCK_TIME`, `BLOCK_TIME_V1_5`, read through `block_time_at(height)` **[code]** |
| blocks per target-time day | 960 | 480 | **[derived]** |
| difficulty algorithm | ASERT, anchored on the **parent's** timestamp | unchanged | **[code]** |
| epoch length | **6 blocks** | **6 blocks** | **[code]** |
| ASERT halflife | **540 s** = 6 × 90 s | **540 s** = 3 × 180 s — unchanged **in seconds** | **[code]** |
| consensus finality depth (a local refusal to reorganise — see below) | **8 blocks** ≈ 12 min | **8 blocks** ≈ 24 min | **[code]** |
| genesis target | 2^238 — the **easiest target the protocol will ever issue** | unchanged | **[code]** |

**What "finality depth" means here, exactly.** It is a rule each node applies to
itself: a node will not reorganise more than 8 blocks below its own tip
**[code]**. It is not a guarantee that the network cannot fork. If two branches
ever diverge by more than 8 blocks, the nodes on each side refuse the other side
permanently, and the split is resolved by operators, not by the protocol. A node
that does not hold the prefix — a fresh sync, for instance — chooses by work
alone, and the depth rule does not protect it; the source says so itself
**[code]**. On a young network with few miners (section 9.8) a transaction should
be treated as settled when it is 8 blocks deep *and* the network has not reported
a split, not before.

Anchoring on the parent rather than on the block's own timestamp removes a
grinding vector: dating a block earlier can no longer make it cheaper. It also
means ASERT absorbs a **loss** of hashrate gracefully and does **not** absorb a
**stall** — a block that never arrives never makes the target easier. Section 5.7
explains why that distinction drives a design decision.

**Why 180 s, and why only now.** v2.0 left the interval open and said the
proof-slot work might raise it. It did. A block that carries a client proof
needs proving-time headroom inside the interval, and the decision was taken
together with the larger circuit class, as one hardfork rather than two. The
interval is a function of the block's own height **[code]**: every block the
chain already holds keeps the 90 s rule it was judged by. The halflife was kept
at 540 s in seconds on purpose — the mining population is concentrated, and the
scenario to absorb quickly is the departure of one large miner, at the price of a
somewhat noisier difficulty at steady state **[code, from the rationale recorded
beside the constant]**. The activation block's own target is derived as half the
90-second target at that height, so the first 180 s block is not asked to be
found at a 90 s difficulty **[code]**. On the public chain block 38 400 was
found 519 s after block 38 399, and the mean interval over the first 834 blocks
of the 180 s era is about 181.7 s **[chain, header timestamps read on
15 October 2026]**.

Nothing about the 180 s interval changes the supply: halvings are heights
(section 7.2). What it changes is the calendar — every "per day" figure in this
document halves at block 38 400 — and the finality depth in minutes, which
doubles to about 24 minutes (a local refusal to reorganise, as defined above,
not a guarantee).

### 4.6 Running a node

These are operational starting points, not consensus minima. The only hard
requirement is the instruction floor of section 4.1.

| role | CPU | RAM | disk |
|---|---|---|---|
| wallet / ordinary node | ≥ 2 vCPU | 4 GiB | SSD, 20 GiB free |
| public full or seed node | ≥ 4 vCPU | 8 GiB | SSD or NVMe |
| miner, small proof class | benchmark the host | ≥ 8 GiB | SSD or NVMe |
| miner, large proof class | benchmark the host | ≥ 16 GiB below block 38 400; 32 GiB from it | NVMe preferred |

There is no honest fixed disk figure to quote: the store grows with live state,
a bounded recent window and the permanent headers, not with the full history of
blocks. From block 38 400 the large proof class is the larger 2^25 class, and the
recommendation for a mining node on that class rises to **32 GiB**
**[estimate]**.

The node mines on CPU out of the box — there is no separate miner to install and
no pool account required. The work protocol is public JSON-RPC, so an external
miner is anybody's to write. The project also publishes **towerminer**, a free,
open-source CPU miner for TowerWalk (Linux and Windows) in the same GitHub
organisation; measured against the node's built-in miner on the same machine it
was faster, by a margin that depends on the CPU family **[measured, our own
machines; figures deliberately not generalised]**. The GPU miner repository that
served the pre-24 846 TowerHash era is of historical interest only: a GPU has no
useful place on this network's proof of work any more (section 5.4). One known
rough edge: at every start-up the node scans its own parameter sets to work out
which relation generation it can serve, which is slow on small instances. Its
RPC stays silent while that runs, so the node looks broken when it is not.
Reducing it remains a task in section 8.

---

## 5. Proof of work

### 5.1 What the public network mined until block 24 845: TowerHash

TowerHash is a sponge over the Poseidon2b permutation on GF(2^128). It was the
public network's proof of work from genesis to block 24 845, and it remains the
**seed** of the current one: TowerWalk takes the TowerHash digest of the header
as its input **[code]**. Its
arithmetic is XOR and carry-less multiplication — the same algebra as the proof
system. We did not invent the permutation and do not claim to; TowerHash is a
construction over it: our round constants from a documented seed, our domain
tags, our sponge schedule, and the **nonce placed in lane 0**. That last choice
is worth a sentence, because it is checkable: the header absorbs 16 field
elements, two per permutation, so eight permutations per attempt. With the nonce
further down the schedule, everything before it is fixed across attempts and can
be precomputed once as a midstate — an optimised miner would replay three
permutations per attempt instead of eight. A sponge cannot be run backwards, so
putting the nonce in the *first* absorbed pair removes that shortcut for
everybody: **every attempt pays for all eight permutations** **[code]**.

TowerHash is honest, cheap to verify, and **good on GPUs**. It is compute-bound
finite-field arithmetic over a few kilobytes of state, which is the shape of
work a GPU's thousands of arithmetic units exist to do. For the first 24 845
blocks a miner with a graphics card was at a substantial advantage on this
network, and we did not describe that as a feature.

### 5.2 Why we replaced it

Not to resist ASICs, and not to decentralise mining. Section 9 says plainly that
neither of those is on offer. The reason is narrower and it is the reason the
whole roadmap has:

**If the GPUs mine, they are not proving.** Section 6 needs GPU capacity pointed
at client proofs. As long as the most profitable thing a GPU could do on this
network was search for nonces, the proof market competed with the chain's own
security budget for the same silicon. Moving mining onto a function that a CPU
runs about as well separates the two markets: proving work goes to the hardware
that is good at it and gets paid for it, and mining goes to hardware that is
abundant, already owned, and otherwise idle.

### 5.3 TowerWalk — live on the public network since block 24 846

| property | value |
|---|---:|
| scratchpad per searching thread | **512 KiB** — 65 536 cells of 8 bytes **[code]** |
| rounds × lanes | **131 072 × 4 = 524 288 dependent reads** per digest **[code]** |
| Poseidon2b folds per digest | **33** — one every 4 096 **cells** while filling the pad (16), one every 8 192 **rounds** while walking it (16), plus a final one **[code]** |

Each step reads an 8-byte cell whose address depends on the accumulator, mixes
it, and writes it back. The reads are **dependent**: the address of step *n+1*
cannot be computed until step *n* has returned. No amount of parallelism removes
that chain; only the latency of the innermost cache that holds the pad does.

The four lanes of a round are written sequentially in the reference
implementation — lane 1 may read the cell lane 0 has just written. A miner may
issue them concurrently and fall back to the sequential path on the rare
collision, which is bit-exact and is what any serious implementation will do; it
raises the number of requests in flight, and it does not shorten the chain.

512 KiB is chosen to sit inside a CPU core's private L2 cache, and outside what a
GPU can give each of the thousands of concurrent threads it needs in flight to
saturate its memory system.

### 5.4 What the measurements say

These are our own benchmarks. The method is stated so that the numbers can be
disputed on their method rather than on their size.

**GPU.** Measured on rented cards, against a kernel we wrote and then tried to
beat — including the variant of section 5.3 that issues a round's four lane reads
concurrently instead of in sequence.

| hardware | rate | note |
|---|---:|---|
| NVIDIA H100 SXM 80 GB HBM3 | **22 554 H/s** **[measured]** | best of a thread-count sweep |
| NVIDIA A100 PCIe 40 GB | **9 951 H/s** **[measured]** | |
| NVIDIA RTX 3090 | **5 558 H/s** **[measured]** | |

**CPU.** Measured on one exclusive physical core of a 64-core server socket,
running the real digest including its 33 permutations, with the machine
otherwise under load. That last detail matters: wall-clock time under load is
noisy, so the **cycle counts are the primary measurement** and the rate is
derived from them.

| quantity | value |
|---|---:|
| **cycles per digest** | **6.3–6.5 M**, stable to ±3 % across runs **[measured]** |
| **cycles per dependent step** | **≈ 12** **[measured]** |
| wall-clock rate per exclusive core, one walk at a time | **465–554 H/s** across runs, ≈ 537 H/s typical **[measured]** |

The wall-clock spread is the machine's load, not the algorithm's variance: the
cycle count moves by 3 % while the time moves by 20 %. Every rate in this
document should be read with that spread attached.

The second line is the one that matters. Twelve cycles per dependent step *is*
the load-to-use latency of that core's L2 cache: a **single** walk is running at
the cache's floor, and no code makes one dependent chain faster. That floor
bounds one chain, not the throughput of a core. A core can keep more than one
walk in flight — several nonces, or two hardware threads — and overlap their
memory accesses; our own open-source miner, towerminer, does exactly that and
gets more out of a core than the figure above, by a margin that depends on the
CPU family **[measured, our own machines; not generalised here]**. The 537 H/s
figure is therefore a single-walk reference, not a ceiling, for anybody —
including us.

**One important caveat on scaling, because it also limits what we can claim.**
Measured on a loaded dual-socket machine (128 cores), oversubscribed with an
untuned thread count, the real rate was
**32 200 H/s for the whole machine** **[measured]** — roughly half the ~68 700
H/s that multiplying 128 × 537 predicts.

That gap is a configuration effect, not a ceiling. The best thread layout
depends on the CPU family — on some, one thread per physical core is best; on
others, two threads per core keep more walks in flight and do better — and
towerminer picks a layout per family. But we have not published a full-machine
measurement in a tuned configuration, so every whole-machine and per-watt figure
below is a **linear extrapolation from a measured single walk on one core**, and
it is labelled as one.

### 5.5 Why this is a ceiling and not a bad kernel

The H100 result deserves an explanation, because on its own "22 554 H/s" is a
number somebody will assume we failed to optimise. Two different limits are at
work, and they are worth keeping apart because conflating them is easy and
wrong.

**Limit one: random access does not reach peak bandwidth.** That card advertises
3 350 GB/s. Our kernel sustains about **755 GB/s of DRAM sector traffic —
roughly 22 % of peak** **[derived]**, which is what dependent random access
costs on any card. The four-lane variant, which is exact to the bit, changes
nothing at saturation: both kernels land within 0.4 % of each other.

**Limit two: most of that traffic is thrown away.** This one is not a
measurement at all, it is arithmetic.

<div class="fig">
<svg viewBox="0 0 720 150" role="img" aria-label="One walk step uses sixteen bytes but moves two thirty-two-byte DRAM sectors, so exactly one quarter of the transferred bytes are useful.">
  <g stroke="currentColor" stroke-width="1.2">
    <rect x="18" y="52" width="128" height="32" fill="none"/>
    <rect x="18" y="52" width="64" height="32" fill="currentColor" fill-opacity=".3" stroke="none"/>
    <path d="M82 52 v32"/>
    <rect x="330" y="52" width="180" height="32" fill="none"/>
    <rect x="330" y="52" width="180" height="32" fill="currentColor" fill-opacity=".3" stroke="none"/>
    <rect x="520" y="52" width="180" height="32" fill="none"/>
    <rect x="520" y="52" width="180" height="32" fill="currentColor" fill-opacity=".3" stroke="none"/>
  </g>
  <g fill="currentColor" font-family="ui-sans-serif,system-ui,sans-serif" font-size="11">
    <text x="18" y="42" font-size="12">what one step uses</text>
    <text x="330" y="42" font-size="12">what DRAM moves for it</text>
    <text x="18" y="102" opacity=".7">8 read + 8 written = 16 B</text>
    <text x="330" y="102" opacity=".7">read sector 32 B</text>
    <text x="520" y="102" opacity=".7">write sector 32 B</text>
    <text x="18" y="130" opacity=".8">16 / 64 = 25 %. This is arithmetic, not a benchmark: no kernel changes it.</text>
    <text x="18" y="146" opacity=".65" font-size="10">Separately, dependent random access reaches only ~22 % of the card's peak bandwidth — a different limit.</text>
  </g>
</svg>
</div>

**A GPU's DRAM granularity is a 32-byte sector.** One step reads 8 bytes at a
random address and writes 8 back, so it moves two sectors — 64 bytes — for 16
useful ones. The ratio of useful to transferred bytes is therefore **exactly
25 %**, by construction and not by measurement, and the kernel sits at that
ceiling because there is nowhere else to sit. Of the 755 GB/s the card moves,
**189 GB/s is data the algorithm actually uses** **[derived]**.

No kernel improves that ratio. The four addresses of a round are uniformly
distributed and, collisions aside, independent, so they cannot be made to share
a sector; and two threads' pads are different address spaces entirely.

The other regimes were checked rather than assumed. Keeping pads resident in the
GPU's large L2 trades bandwidth for latency and lands well below the DRAM regime;
so does placing part of the pad in shared memory, because the duration of a step
is set by its *worst* access and at least one of the four misses almost always
**[measured, on a smaller card; derived for the H100]**. Combining L2-resident
and DRAM-resident pads on the same card could plausibly add 15–25 % **[estimate]**
— which moves the number, not the conclusion.

So the honest version of the claim is: **on an H100-class card, nobody is going
to be several times faster than this, because the limit is the memory system and
not the code.** That is a stronger statement than "our kernel is fast", and it is
the one we are willing to defend. It is also bounded on purpose. A future card
with several times the random-access throughput will produce several times the
hashrate — the ratio of hashes to achieved random-access bandwidth is roughly
constant — and section 9 treats that as the open risk it is rather than as an
exception.

### 5.6 What this buys, and what it does not

The defensible statement, in one sentence:

> **Watt for watt, an ordinary CPU core does three to five times more of this
> work than a GPU does — and no accelerator delivers the order-of-magnitude
> advantage that a pure permutation hash hands them.**

| unit of comparison | CPU advantage | how solid |
|---|---|---|
| **per watt, from measured throughput and published TDPs** | **≈ ×2.2** vs. an H100 at its TDP **[derived]** | Throughput measured, power **not**: both sides use the manufacturers' TDP. The CPU side is the whole-machine figure above, taken in an untuned configuration. It is a floor, not the design point. |
| **per watt, tuned thread layout** | **×3 to ×5** **[estimate]** | The design point, and the direction is a property of the memory system. It is an estimate for two reasons we will not hide: the CPU side extrapolates a measured core to a socket, and the GPU side compares against a power figure we did **not** measure — a memory-bound kernel typically draws well under a card's TDP. Neither gap has been closed by a measurement yet. |
| per unit of capital | large against datacentre accelerators, **about ×2** against a current consumer card **[estimate]** | **Thin, and the weakest figure here.** No current-generation consumer card has been measured — the table above stops at a 2020 part. If a modern consumer GPU beats our projection, this row is wrong before any of the others. |
| **per unit of rented capacity** | **not reliable** | See section 9.4. Renting general-purpose compute is cheap, and a rational attacker rents rather than buys. |

It does **not** make GPU mining impossible; it makes buying a GPU *for this
purpose* a poor trade against buying CPU cores. It does **not** make the
algorithm ASIC-proof — section 9.1 is explicit about how far from ASIC-proof it
is. And it does **not** make the chain unattackable by rented hardware: the
algorithm is not listed on any hashrate marketplace, which raises the friction of
acquiring hashrate, but general-purpose compute is rentable by the hour from
dozens of providers and we are not going to pretend otherwise.

### 5.7 Crossing the fork

Replacing a proof-of-work digest changes the cost of a hash, and therefore the
hashrate, discontinuously. Carrying the pre-fork difficulty target across the
boundary would make the first post-fork block take a multiple of its intended
time, on the one day the chain is being watched.

Jetsam therefore pins the target of **exactly one height** — the first block
under the new proof of work — and lets ASERT resume from the next block on. The
anchor is deliberately set on the easy side, because the two errors do not cost
the same:

- **too easy** — a burst of fast blocks, which ASERT tightens away within an
  hour;
- **too hard** — the chain stops, and because ASERT anchors on the parent, a
  block that never arrives never makes the next target easier. Restarting
  requires a new binary.

The procedure was rehearsed on the test network first, and then **executed on
the public network at block 24 846 on 29 September 2026, 18:21 UTC**. The anchor
was set at exactly **2^235** **[code]** — chosen on the easy side, as the
reasoning above requires. What happened is on the chain for anyone to read
**[chain]**: the boundary block was found at the pinned target, the next blocks
came fast (ASERT had already tightened the target four-fold within about a dozen
blocks), and the chain continued with no proof errors and no reorganisation
reaching back across the boundary block. The release that followed pinned the hash of block 24 846
as a hard checkpoint, so no node of that release or later can be fed an
alternative history below it **[code]**.

Two things v2.0 listed as open are therefore closed. The method itself was reused
by v1.5: the activation block at 38 400 again carried a derived target — this
time half the 90-second target, because the interval doubles rather than the
hash cost **[code]** — and ASERT resumed from the next block. The target did its
job; what the method does not address is a majority of the hashrate staying on
the old rules after the height, which is what happened at 38 400 (section 2).

---

## 6. The proof market

Sections 1 and 4 describe a network with recursive proving capacity and a
verifier already inside consensus. This section is what that is *for*. Since
v2.0 the on-chain part of it has been written and has been live since block 38 400
(section 6.7); the catalogue it serves is **empty**, the SDK is not shipped, and
there is no customer. Where the text below says what the project "will" do, read
it as an intention with no date attached. The design is unchanged; what has moved is the line
between "designed" and "in the binary", and section 6.7 draws it precisely.

### 6.1 The shape of the product

A client submits a computation and its committed inputs; a prover produces a
proof; the proof is registered on-chain where anyone can verify it cheaply and
permanently. What the client buys is not the cryptography. It is a public,
durable, third-party-checkable statement that a rule was followed.

### 6.2 Three uses, written out

**A. An agent that stayed inside its fence.** A company runs a software agent
that calls tools, spends money and touches customer data. It wants to be able to
show a customer — or an auditor — a set of statements about a period of
operation: *the agent never called a tool outside its authorised list; it never
spent more than the configured budget; it only ever read data belonging to the
customer it was acting for; every irreversible action was preceded by the
required approval; and the trace was not edited afterwards.*

Each of those is a deterministic predicate over a sequence of events. Proving
them is cheap — it is ordinary computation over a list, not model inference. The
reason this case is first is section 1.4: an agent's actions are naturally
witnessed by other parties. Signed API calls, transactions, writes into somebody
else's system. The input is committed by people who have no interest in the
answer. The piece that must be built well is an **incremental commitment** over
the trace, anchored block by block, so that history cannot be rewritten after the
fact.

We do not prove that the agent reasoned well. We prove it stayed inside the
fence.

**B. The model you were billed for is the model that ran.** Customers in the
inference market have no way to check that the model they were charged for is
the one that served their request; silent routing to a cheaper model is a
concern they raise and cannot currently settle either way. The
computation is deterministic, the model is identified by a commitment to its
weights, and the input is committed by the customer. What is proven is a
correspondence — *the responses in this billing period were produced under the
commitment you were charged for* — not that the model is good.

**C. Attestation of a computation.** A scoring decision, a prize draw, a
commission calculation, a rebate allocation: cases where a party computes
something that affects other people and is currently trusted to have computed it
correctly. Here the rule is public, the inputs are committed in advance, and the
proof replaces "trust our spreadsheet". This is the broadest family and the most
boring, which is usually a good sign.

### 6.3 A catalogue, not a language

The client does not write a circuit. The intention — with no date — is a
**catalogue** of ten to fifteen standard computations that a client
parameterises with their own data, and an SDK that picks one, supplies the data,
obtains the proof and registers it. Neither exists on the public network today.

This is not a convenience decision, it is a constraint. The recursion geometry
imposes a fixed *shape* — the proof's dimensions and its query count are frozen
in consensus. A client's relation is free, its shape is not. A catalogue is what
a fixed shape allows you to promise honestly.

Identity proofs — proving an attribute of a passport or a certificate without
disclosing it — have been **removed** from this catalogue and will not appear in
it. The client form is pinned at **2^22 = 4 194 304 constraint rows**
**[code]**. RSA and ECDSA verification inside this circuit cost on the order of
**10^8 to 10^9 rows** **[estimate]** — roughly twenty-five to two hundred and
fifty times that. It
is not a matter of optimisation. Only computations that live natively on a binary
field are viable here, and we would rather say so now than discover it in public
later.

### 6.4 The three transaction forms

Three new transaction forms are required, and they must be frozen **together** in
a single consensus change. Every new form means new matrices and therefore a new
proof-system identity; arriving piecemeal, they would partition the network three
times instead of once.

| form | what it does |
|---|---|
| **register** | enrol a declaration and receive an on-chain identifier. The registry licence is paid here. |
| **order** | post a bounty conditioned on a valid proof of a named statement |
| **claim** | present the proof; consensus verifies it; the payment is released |

The chain can already condition a payment on the revelation of a committed
value, and a completed proof can be made to reveal one — so paying a prover
without a timelock is implementable as-is. What is
missing is the refund path if nobody proves, which needs a timelock the small
proof class cannot currently afford. The two available answers are payment on
delivery, and the case where the client is their own prover — which we expect to
dominate.

### 6.5 Who pays whom

Three costs, one payer: the client.

| who bears it | what | how it is covered |
|---|---|---|
| the prover | GPU time | paid by the client — or the client *is* the prover |
| the miner | the block space and proving time a client slot occupies | a **per-proof fee**, which must be non-zero or miners will simply ignore the slot, exactly as a fee-less transaction is ignored |
| every node, forever | permanent RAM per registered relation, and a scan proportional to its size on every block | a **registration licence**, paid in JTM and **burned** |

The registry must be **bounded from day one**. A registered relation costs every
node on the network memory for as long as it is registered — on the order of
tens to hundreds of megabytes each, **measured on the relation parameters the
chain already carries** **[measured]** — so
a few dozen registrations is a real cost on every machine that follows the
chain. The scarcity of places is
not a marketing device; it is a physical constraint, and it is what justifies
pricing the licence as a real cost and burning it rather than paying it to
anyone. The 1 000 JTM in force today is a provisional placeholder, described in
the source as symbolic (section 6.7).

Note what is *not* on this list: the verifier. The universal verifier is code,
not a role. Miners run it, and every node re-runs it exactly as it re-runs every
other consensus check. There is nobody to pay for verification and no trusted
party to appoint.

### 6.6 Why we prove first, and why that costs nothing in security

**When the first catalogue entry ships — no date is set — we intend to be the
first provers.** This is stated plainly because it looks like centralisation and
is worth explaining rather than hiding.

A dedicated GPU pointed at an empty queue loses to any alternative use of that
GPU. Two-sided markets generally start this way, and we expect the established
proving networks started the same way. The point that matters is this: **a centralised prover costs nothing in
security.** Its output is verified by consensus; it cannot produce an accepted
proof of a false statement any more than a miner can produce an accepted invalid
block. The only risk it carries is **availability** — a business risk, not a
protocol risk.

The condition we will not trade away: **a client must always be able to prove for
themselves.** Open format, and nothing in the protocol that makes us a required
intermediary; a public SDK is the intended means, and it is not shipped. That is
the difference between a service and a toll, and it is the first thing a serious
counterparty will check. The intention is that proving opens to others on a
threshold — when the queue exceeds our capacity, or when a client requires a
second source — not on a date; that threshold is not defined yet.

### 6.7 What v1.5 actually ships, and what it does not

This subsection is new in v3.0 and it is the one to read if you want to know
what is in the binary rather than in the plan. Everything here is governed by
the v1.5 clock: below block 38 400 none of it is interpreted, and since that
block all of it is in force on the public network **[code]**.

| element | v1.5 on the public network (identical in v1.5.0 and v1.5.1) | tag |
|---|---|---|
| client slot | a block may carry **at most one** client proof, next to its ordinary transactions | **[code]** |
| capacity | therefore **480 client proofs per target-time day** at most, and more only through a later change to the carrier | **[derived]** |
| client registry | append-only, **16 entries in total** (a single bank in v1.5); an entry is a digest `D` of a client computation's matrices, not a client identity | **[code]** |
| catalogue | **closed**: only a digest listed in the release's consensus constant can be registered. On the public network the list is **empty** | **[code]** |
| registration | pays a **licence of 1 000 JTM, burned** to an address with no known spend secret | **[code, marked PROVISIONAL in the source]** |
| submission | pays a **1 JTM fee** to the miner who carries the proof | **[code, PROVISIONAL]** |
| activation delay | an entry becomes carriable **120 blocks** after its registration — the time every node has to fetch the matrices | **[code]** |
| binding | each object is paid by an ordinary transaction whose zero-value marker output commits to it; a relay cannot add, drop or re-attach an object | **[code]** |
| privacy | **none.** A client proof attests a computation; its public inputs and commitments are public. It is **not** a zero-knowledge proof | **[code]** |

Three consequences, stated so that nobody has to infer them.

- **The three transaction forms of section 6.4 did not all ship.** What shipped
  is *register* and a *submission* that pays the carrying miner. The bounty pair
  — *order* and *claim*, a payment released by consensus on a valid proof — is
  not in v1.5, and the refund path it needs still lacks the timelock the small
  class cannot afford. The expected dominant case, the client proving for
  themselves and paying the carrying fee, is the one v1.5 serves.
- **"Closed catalogue" is a decision, not a limitation we hide.** v2.0 said the
  licence is burned because registry places are a physical cost on every node.
  The further step taken since is that the project, not the market, decides
  which computations enter the list, release by release; a release may add a
  digest and never remove one the chain may have registered **[code]**. There
  are no purchasable places. The licence amount and its destination are
  **provisional**: the source says so in capitals, and so do we.
- **The client proof is not private, and v1.5 does not pretend it is.** The
  witness-hiding construction of section 3.1 covers the *spending witness* of an
  ordinary transaction and has been in place since genesis. A v1.5 client proof
  is not designed to hide its witness: the witness is not published, but nothing
  protects it from leaking through the proof. Making the external proof of a
  client batch genuinely zero-knowledge, and batching several client proofs
  recursively into the one slot, is the intended v1.6 line of work — for the
  test network, with no date and no public activation height (section 8).

The whitepaper figure of "ten to fifteen" catalogue entries in section 6.3
stands as an intention, not a count: the count today is zero.

---

## 7. Token economics

### 7.1 Supply

| parameter | value |
|---|---:|
| ticker | **JTM** |
| precision | 1 JTM = 1 000 000 μJTM **[code]** |
| **maximum supply** | **21 000 000 JTM** — hard cap **[code]** |
| initial block subsidy | 50 JTM **[code]** |
| subsidy at the time of writing | 25 JTM — the first halving fell at height 28 800 on the public chain **[chain]** |
| genesis coinbase | **none** — emission starts at height 1 **[code]** |
| coins allocated before mining | **none** **[code]** |
| development allocation | **10 % of the subsidy** inside a window that ends at height 369 600 (section 7.3) **[code]** |
| emission reaches zero | height **3 467 664** **[code]** — at 180 s per block from height 38 400, about **19.6 years after that activation** **[derived]**; v2.0's "9.89 years" assumed 90 s blocks throughout |

On the word *premine*. The genesis block carries no coinbase and nobody held a
coin before the first block was mined; in that strict sense there is none. But
a reader who hears "zero premine" and assumes "no allocation to the developers"
would be misled: a consensus-enforced development allocation exists, it is
described in full in section 7.3, and we would rather you read that section than
the slogan.

The cap is not an aspiration and it is not enforced by a counter. It is enforced
by the height schedule itself: the sum of `block_reward(h)` over every
coinbase-carrying height equals 21 000 000 JTM to the micro-unit, and a consensus
test checks that sum height by height **[code]**.

### 7.2 Emission schedule

The schedule is written in **heights** and did not move with the interval:

| height range | subsidy |
|---|---:|
| 1 → 28 800 | 50 JTM |
| 28 800 → 172 800 | 25 JTM |
| 172 800 → 831 800 | 12.5 JTM |
| 831 800 → 1 490 800 | 6.25 JTM |
| 1 490 800 → 2 149 800 | 3.125 JTM |
| 2 149 800 → 2 808 800 | 1.5625 JTM |
| 2 808 800 → 3 467 664 | 0.78125 JTM |
| 3 467 664 → ∞ | **0** |

Ranges are half-open: the height on the right is the **first** block of the next
tier, not the last of this one.

Three details that a reader recomputing this will trip over, so they are stated
rather than left to be discovered:

- **The schedule is not regular.** The first two halving heights are fixed
  constants — 28 800 and 172 800 — and the remaining five are spaced by 659 000
  blocks each **[code]**. Anyone deriving the table from a single interval will
  get the wrong answer.
- **The code counts seven halvings; six of them actually halve.** The subsidy
  falls from 50 to 0.78125 JTM in six steps. The seventh does not halve it, it
  sets it to **zero**. Left alone, that seventh event would fall at height
  3 467 800 and the total would overshoot the cap by 106.25 JTM — exactly **136
  blocks** of the final tier. Those 136 blocks are trimmed, which is why emission
  ends at **3 467 664** and the sum lands *on* 21 000 000 rather than near it
  **[code]**.
- **Halvings are defined in blocks, never in dates — and the interval did
  change.** From height 38 400 a block targets 180 s instead of 90 s, so every
  calendar figure stretches by two from that point while the cap does not move.
  Concretely **[derived]**: daily emission goes from 960 × 25 = **24 000 JTM** to
  480 × 25 = **12 000 JTM**; the second halving (height 172 800) falls about
  **9 months** after activation instead of about 4½; emission ends about **19.6
  years** after activation instead of about 9.8. Anyone who computed dates from
  v2.0 should recompute them.

There is **no tail emission and no perpetual floor**. After height 3 467 664
miners are paid by fees alone.

### 7.3 Development allocation

This is what the project's community calls the "dev fee", and this is exactly
what it is in the code **[code]**:

| recipient | share of each block subsidy inside the window |
|---|---:|
| **miner** | **90 %** |
| network fund | 5 % |
| lab fund | 5 % |

- **Duration: two target-time years, counted in payouts, not in blocks.** The
  rule is **730 mandatory payouts, one per target-time day**. At 90 s a day is
  960 blocks; at 180 s it is 480. Payouts fall every 960th block up to and
  including the v1.5 activation height, then every 480th block after it
  **[code]**.
- **With the activation at 38 400, the window closes at height 369 600** — the
  730th payout — instead of the 700 800 that v2.0 published for a chain that
  stayed at 90 s **[code: `J + (730 − J/960) × 480`]**. After that block,
  **miners receive 100 % of the subsidy**, permanently.
- Each payout is `5 % × the subsidy tier in force at that payout block × the
  number of blocks it covers`, to each fund, visible on-chain like any other
  transaction. It cannot be omitted or redirected by a miner **[code]**.

The number this produces has changed since v2.0, and in the direction that is
less favourable to the funds, so we state it with its derivation:

| | JTM | share of cap |
|---|---:|---:|
| v2.0 figure — 730 payouts of 960 blocks, window to 700 800 | 1 160 400 | 5.53 % |
| **v3.0 figure — 40 payouts of 960 blocks, then 690 of 480, window to 369 600** | **747 000** | **3.56 %** |

**[derived]** from the constants above: 29 payouts at 50 JTM × 960 blocks, 11 at
25 JTM × 960 (the 30th falls exactly on the first halving), 279 at 25 JTM × 480,
411 at 12.5 JTM × 480 (the second halving, 172 800, is again exactly a payout
height), each taken at 10 %. The consensus test suite pins the same closed form
at neighbouring activation heights (742 200 JTM at 30 720, 748 200 at 40 320:
600 JTM per 960-block step) **[code]**, and 38 400 sits between them. The
reason the total falls is simple: the same 730 payouts now cover half as many
blocks for most of the window, while the halving heights did not move.

Two properties of the rule are unchanged from v2.0. Both halvings inside the
window land *exactly* on payout heights, so each of those days is paid at the
**post**-halving tier: the rounding can leave part of the maximum share
unissued and can never create issuance. And the amount that would be entitled
block by block (749 996 JTM for this window **[derived]**) is slightly larger
than what is minted; the difference is not minted and does not go anywhere else
**[code]**.

**What the funds are for, and who holds them.** The network fund pays for
infrastructure that the network needs and that nobody is individually paid to
run — seed nodes, the explorer, release builds and distribution. The lab fund
pays for the work in sections 5 and 6; the maintainers intend it to fund
community programmes such as creator rewards as well, case by case and with no
standing commitment. Both are controlled by
keys held by the maintainers; there is no on-chain governance of them today, and
section 11 says so rather than implying a structure that does not exist. No
spending policy is published. What *is* true and checkable is that both are paid
by consensus on a published schedule that ends, and every payout is a visible
on-chain transaction.

### 7.4 Fees and the state-growth burn

The fee schedule sets a deterministic minimum rather than running an auction,
and it has four terms **[code]**; a transaction may pay more than the minimum,
and the excess goes to the miner **[code]**:

| term | amount (μJTM) |
|---|---:|
| base | 5 000 |
| per input | 100 |
| per output | 700 |
| **state growth** | **2 500 × multiplier × net new slots** |

`net new slots` is outputs minus inputs, floored at zero, so a transaction that
consolidates — that leaves the state smaller than it found it — pays nothing on
this term at all. The multiplier is a deterministic staircase on how full the
state accumulator is: ×1 normally, **×2 above 50 % occupancy, ×4 above 75 %, ×8
above 90 %** **[code]**.

Every term except the last is claimable by the miner. **The state-growth term is
burned**, and the coinbase cap counts only what is left after it, so a miner
cannot recover it by any route **[code]**.

The reason is the accumulator of section 4.3. A transaction that consumes as many
slots as it creates costs the network nothing permanent. One that leaves more
live slots behind has made every node carry more state, forever. The burn prices
that externality at the moment it is created, and burning it rather than paying
it to the miner removes any incentive to inflate the state in order to collect
it.

---

## 8. Roadmap

Four stages. Each is deliverable on its own and does not block the next. Dates
are deliberately absent: this project has crossed five consensus upgrades by
announcing a **height** and a notice period, never a clock time, and the roadmap
uses the same discipline. Everything below that is not marked done is an
intention, not a dated commitment.

**Stage 1 — the epoch-anchor upgrade. Done.** Crossed on the public network at
height 17 750 on 22 September 2026 **[chain]**, after being crossed twice on the
test chain and deployed as a dormant build first — the only one of the five
upgrades published that way. It was the last of three upgrades the chain needed
before anything new could be built on it.

**Stage 2 — the CPU proof of work. Done.** TowerWalk (section 5) crossed the
public network at height 24 846 on 29 September 2026 **[chain]**. Its release
was published already armed at that height, with a notice period, not as a
dormant build. Of the two preconditions v2.0 listed, one was met first — the
anchor was fixed against a settled difficulty (2^235 **[code]**). The other was
not: the open-source CPU miner, towerminer, was published on 30 September 2026,
the day **after** the fork (release date on GitHub); until then the
optimised miner ran only on the maintainers' own machines, which mined a large
share of the first post-fork blocks.

**Stage 3 — the proof slot, the registry and the SDK. Carrier live since
block 38 400 (13 October 2026); catalogue empty.** v1.5.0 is the release that
teaches the circuit to carry a client proof whose verification data is read from
the chain's registry rather than compiled in, with the closed catalogue, the
licence and the carrying fee of section 6.7 **[code]**. What is **not** shipped
and has no date: the first catalogue entries (the agent-trace predicate of
section 6.2 is the intended first one), the SDK, the bounty pair of transaction
forms, and the opening of the prover set.

Two things about this crossing should be named plainly. First, a weakness that
v1.5.0 accepted knowingly: from block 38 400 a proof covers only the blocks from
38 400 onward (section 4.2), and the v1.5.0 binary pinned no checkpoint above
24 846, so a node joining from a snapshot taken after 38 400 relied on the
headers' proof of work alone for the 24 846–38 399 range. v1.5.1 closed it by
pinning blocks 38 399 and 38 400 **[code]**. Second, a fault nobody had accepted:
with about 70 % of the hashrate still on the old rules, the taller old branch
captured the synchronisation of v1.5.0 nodes (section 2). v1.5.1 fixed it
without changing a consensus rule: a peer whose chain a node has refused is
remembered for an hour and never used as a synchronisation source; a
synchronisation candidate is judged by work, not height, and must agree with the
pinned blocks; and the next valid block on top of the tip is never held behind a
pending synchronisation **[code]**. The lesson is recorded here so that the next
hard fork is rehearsed against it: a branch that does not update and keeps
mining, a fresh node that joins after the height, and a measure of how much of
the network has actually updated before the height is reached.

**Stage 3b — zero-knowledge client proofs and recursive batching. Test network
only, intended.** The intended v1.6 line of work would make the external proof
of a client batch genuinely zero-knowledge and let the one client slot carry a
recursive batch of several client proofs instead of one. It is itself a hard
fork with a new relation generation and a new root (section 4.2). Only early
preparatory work exists; the new relation is not written, and it **has no date
and no public activation height**. Nothing in v1.5 should be read as "private
proofs", and nothing here is a date.

**Stage 4 — distribution and liquidity.** Once stage 3 is demonstrable rather
than described. We name no venue, no date and no price, and nothing in this
document is a commitment that JTM will trade anywhere.

### Parameters deliberately left open

- **The licence** — its amount (1 000 JTM) and destination (burned) are marked
  provisional in the source **[code]**; the alternatives considered (a dividend
  to the following blocks' miners, a split, a treasury) are enumerated there and
  one of them would need a change inside the relation.
- **The catalogue** — which computations enter, and when. Empty today.
- **The number of client slots per block** — one, by construction of the
  carrier; raising it is a change of the carrier, not of a constant.

The two parameters v2.0 left open — the interval and the anchor — are closed
(180 s from 38 400; 2^235 at 24 846).

---

## 9. What we do not claim

This section exists because the two claims a reader most expects from a CPU
proof-of-work chain are the two we will not make, and because the surest way to
be taken apart in public is to present the conjectured as the proven. Each
subsection ends with what we do anyway.

### 9.1 Jetsam is not ASIC-resistant, and the gap is not small

512 KiB of SRAM is roughly **0.18–0.2 mm²** on a 7 nm process **[estimate, from
published SRAM cell density]**. The per-step logic — four lanes of add, multiply
by a constant, XOR, shift — is a few thousand gates, negligible beside the
memory. A hundred-square-millimetre die could hold on the order of 400 engines
and produce several hundred thousand hashes per second at a few tens of watts
**[estimate]**.

Against a CPU that is roughly **×30 to ×100 per unit of capital and around ×50
per watt** **[estimate]**. That is not a reduced advantage. That is a complete
takeover of the network by whoever builds it, and there is a direct precedent:
Scrypt had a comparable profile — small pad, simple per-step logic — and its
first ASICs made CPU and GPU mining unprofitable within months. Algorithms that
have genuinely resisted ASICs did so by requiring tens of square millimetres of
*logic* per engine, not by having a pad.

An earlier draft of this document said the ASIC advantage would be "divided, not
eliminated". That was wrong, and it is corrected here rather than quietly
dropped.

**What is true**: the economics decide the date. An ASIC is built when the annual
reward times the expected captured share exceeds the several million dollars of
non-recurring engineering, plus nine to fifteen months of design time
**[estimate]**. Below a certain network value nobody funds that. The risk is
therefore correlated with the project's success, which is exactly when it hurts
most.

**What we do anyway**: the proof of work is outside the circuit (section 3.3), so
replacing it costs a consensus parameter and a digest function rather than a
relation migration. That is a real defence and it is one very few proof-carrying
chains have. We intend to publish, in advance, the network-value threshold at
which a revision is triggered, rather than decide it under pressure on the day
the first ASIC appears; that threshold is not published yet and has no date.

### 9.2 FPGAs are viable and we have not disproven them

A large FPGA can hold a few dozen pads in on-chip memory. Our reading of the
achievable clock and step latency puts a top-end card somewhere between
**8 000 and 70 000 H/s** **[estimate]** — a range that wide because **no
synthesis has been run**. This is our own unpublished analysis, not a citation.

At those numbers an FPGA loses badly per unit of capital, because the cards are
expensive, and wins modestly per watt. It is not an economic threat. Its real
significance is that it is the prototyping step for section 9.1.

### 9.3 A CPU proof of work attracts botnet mining

Botnet participation is documented on CPU-mined chains of any size. It is a
structural cost of making mining cheap on hardware that is already everywhere,
and it is the same property that makes mining accessible to honest participants.

**What we do about it: nothing, and we say so.** We have no mitigation to offer.
Anybody who tells you they have solved this has not.

### 9.4 The mining population is mercenary, and rented compute is cheap

CPU miners sell their production. They do not hold, and they do not support a
price.

The sharper version of the risk is about attacks, not sell pressure. In 2025 a
third-party project redirected a large share of a major CPU-mined chain's
hashrate into its own pool by paying miners in its own token, and reorganised
blocks with it. No hashrate was rented and no marketplace was involved; the
miners followed the payment, not the chain. That is the mechanism to worry about,
and an unlisted algorithm does not prevent it.

Nor does an unlisted algorithm make hashrate unbuyable. **General-purpose compute
rents by the hour from dozens of providers.** Not being listed on a hashrate
marketplace raises the friction — an attacker has to build and deploy a miner
rather than click a button — but it does not change the price of compute. Any
claim that "0 % of this network's hashrate is rentable" is wrong, and an earlier
draft of this document made it.

**And the honest floor**: the cost of attacking a proof-of-work chain is
proportional to its total hashrate. This one is young and small. At this size the
majority-attack cost is low, and it would be low under any mining algorithm. The
defence is the network's size, not its choice of function, and network size is
not something we can claim to have.

### 9.5 Implementation risk in the proof system

Jetsam runs a recursive proof system in production. Proof systems are young and
soundness bugs in them are subtle. The soundness bound we quote is inherited
research, not our own result, and we say so where we quote it (section 3.1).

**What we do anyway**: the code is public, relations are pinned by digest, each
generation is selected by the block's own height, and a binary that cannot honour
an armed generation refuses to start rather than forking the network quietly
(section 3.4). None of that is the same as a formal proof of soundness, and we do
not present it as one.

### 9.6 The proof market has no customer

Section 6 describes a product with a real design, a real cost model, since block
38 400 an on-chain carrier, and **no identified paying client and an empty
catalogue**. That is the single largest open risk in this document, larger than
any of the hardware questions above. It is stated here rather than in a
footnote.

### 9.7 A client proof is not a private proof, and "post-quantum" is a narrower claim than it sounds

Two phrases that will be attached to this project by others, and that we want
attached only in their exact meaning.

**"Zero-knowledge."** The chain does not claim zero-knowledge anywhere. The
spending proof of an ordinary transaction is witness-hiding by construction,
with no complete hiding proof written (section 3.1). The v1.5 client proof is
**not zero-knowledge** — it attests that a committed computation was carried
out, with public inputs and public commitments, and it is not designed to hide
its witness. Private client proofs are intended v1.6 work for the test network,
with no date (section 8). Amounts, addresses and fees of ordinary transactions
are public, as they always were.

**"Post-quantum."** What is true: no elliptic curve and no signature scheme is
reachable from consensus, the proof system or the proof of work, so there is
nothing here for Shor's algorithm to break. What is *not* a theorem: that the
chain is therefore secure against a quantum adversary. Security rests on
Poseidon2b and on the soundness of the proof system, whose lowest documented
provable figure is about 64 bits, whose certified figure against a quantum
adversary is about 64.7 bits in an idealised model, and whose 128-bit figures
rely on a conjecture (section 3.1). "Hash-based, no curves" is the claim we make.
"Quantum-resistant" as a one-word badge is not, and we would prefer it were not
made on our behalf.

### 9.8 The mining network is concentrated

A young CPU-mined chain has few miners, and the halflife decision of section
4.5 was taken explicitly because the departure of one large miner is the
scenario to absorb. We publish no hashrate share and no miner count here;
the explorer shows the coinbase of every block, and a reader can measure the
concentration for themselves rather than take our word for it. Section 2 shows
what concentration meant in practice at block 38 400: with most of the
hashrate on the old rules, the chain went about 49 minutes without a block.

---

## 10. Where Jetsam sits

A factual placement, on design axes rather than adjectives. The right-hand
columns describe categories, not specific products, and the rows are
architectural choices rather than claims of superiority.

| | **Jetsam** | general-purpose proving network | succinct chain, proof-of-stake | conventional PoW L1 |
|---|---|---|---|---|
| sybil resistance | proof of work | none of its own | stake | proof of work |
| what the proof covers | the chain's own history since the root of the current relation generation; checkpoints below it | a client's program | the chain's own history | nothing |
| who verifies a client proof | consensus, on-chain | the client, or a contract elsewhere | not applicable | not applicable |
| node cost vs. chain age | constant for proofs; headers grow linearly, lightly | not applicable | constant | grows |
| field | binary tower, GF(2^128) | usually prime | prime | not applicable |
| what a client submits | a parameterised entry from a catalogue | an arbitrary program | not applicable | a transaction |
| mining and proving hardware | deliberately separated (section 5.2) | one market | not applicable | one market |

The trade this table shows is real in both directions. A general-purpose proving
network accepts an arbitrary program; Jetsam accepts a catalogue entry. In
exchange, a Jetsam client's proof is verified by the chain's own consensus and
settled in the same place, with no verifier contract to deploy and trust
elsewhere.

---

## 11. Governance, licensing and provenance

**Who decides an upgrade.** There is no on-chain governance and this document
will not pretend otherwise. Consensus changes are proposed and implemented by the
maintainers and published with an activation height announced in advance and a
notice period. So far only one of them (v1.3) was published first as a
**dormant build** — one that contains the new rules but does not activate them —
and armed later; the others, v1.5.0 included, were published already armed.
v1.5.1 (14 October 2026) changed no rule and set no height: it pinned two block
ids and fixed a synchronisation fault. Node operators adopt an upgrade by
choosing to run it. An operator who does not upgrade does not get a different
chain until the activation height is reached; after it, the nodes that did not
upgrade are on a different chain — and at block 38 400 about 70 % of the
network's hashrate was (section 2).

**The funds.** The two development funds (section 7.3) are controlled by keys
held by the maintainers. Their payouts are made by consensus on a schedule that
ends at height 369 600, and each one is an on-chain transaction anyone can
see. No legal entity is claimed at the time of writing; the project publishes
under "the Jetsam developers", and the maintainers are pseudonymous.

**Provenance.** Jetsam is a derivative work of **Parano1d**, an Apache-2.0
codebase by another author, used under that licence. The unmodified upstream tree
is kept as a visible baseline commit in the public repository, every change is a
reviewable diff against it, and the full list lives in `CHANGES-FROM-UPSTREAM.md`
with attribution in `NOTICE`. Jetsam is an **independent network**: it is not
endorsed by, affiliated with or supported by the upstream project or its author,
the two networks reject each other's peers and each other's work, and JTM has no
relationship to the upstream's coin.

What Jetsam changed, without any comment on the upstream's own choices: the
monetary policy was rewritten to halve by height under a hard 21 000 000 cap that
reaches zero; the block interval is 90 s, then 180 s from height 38 400;
difficulty is anchored on the parent; the proof of work is a different
instantiation with different constants, tags and schedule — and, since block
24 846, a different function altogether — so the two networks' hashes are
mutually invalid; the development allocation lasts two target-time years instead
of three and is paid to Jetsam's own funds; and the client proof slot of v1.5 is
Jetsam's own work.
The recursive HistoryStep architecture and the proving stack are kept
structurally as they are, because they are the good part. They are credited by
name in the repository's `NOTICE`, together with the other upstream components
the stack builds on and their own licences.

**Licences.** Code: Apache-2.0. This document: CC BY 4.0.

---

## 12. Glossary, references and links

### Glossary

**ASERT** — the difficulty algorithm. Each block's target is derived from an
anchor and the time elapsed since it, with an exponential response; here it is
anchored on the parent's timestamp. *Halflife* is the time offset that doubles or
halves the target.

**bech32m** — the address encoding. Jetsam addresses begin `j1`.

**FRI** — Fast Reed-Solomon Interactive Oracle Proof of Proximity: the low-degree
test the proof system uses to make commitments checkable with a small number of
queries.

**GKR / sumcheck** — an interactive proof technique for structured computations,
used for sub-relations where it is cheaper than a generic circuit.

**H/s** — hashes per second: complete proof-of-work digests computed per second.

**HistoryStep** — the relation each block proves: that its state transition
follows the rules and that the previous block's terminal commitment was valid.

**IVC** — incrementally verifiable computation: the general name for the
technique in section 4.2, proving a long computation one step at a time with each
step attesting to the last. Jetsam's HistoryStep is an instance of it.

**Pack / relation generation** — the compiled parameters (the "matrices") for one
version of the constraint system. A block is proved and verified against the
generation its own height selects, and each generation's recursion starts from
its own root: genesis for the first generation, the block just below the
activation height for each later one.

**Midstate** — the part of a hash computation that does not change between
mining attempts and can therefore be computed once. Section 5.1 explains how
TowerHash removes it.

**Row** — one constraint in the proof system, and the unit everything in the
circuit is budgeted in. A block's proof has a fixed number of them available.

**Slot** — a seat in the state accumulator holding one live output. Spending
erases the slot and returns it to the pool.

**Terminal** — the small commitment a block's proof exposes, meaning "everything
from the root of this relation generation up to here checked out"; the next
block's proof verifies it.

**Finality depth** — the number of blocks (8) below which a node refuses to
reorganise its own chain. A local rule, not a network guarantee (section 4.5).

**Checkpoint** — a block hash compiled into a release; no node of that release
accepts a history that does not contain it. Today: blocks 24 846, 38 399 and
38 400 **[code]**. Below the root of the current
relation generation, checkpoints and header proof of work hold the history that
the recursive proof no longer covers (section 4.2).

**TowerHash** — the proof-of-work function of the public network's first 24 845
blocks: a sponge over Poseidon2b. Still computed as the seed of TowerWalk.

**TowerWalk** — the cache-resident proof-of-work function of section 5, live on
the public network since block 24 846.

**Catalogue** — the closed list of client computations (identified by the digest
of their matrices) that a release allows to be registered. Empty on the public
network at v1.5.1.

**Client slot** — the place in a v1.5 block for at most one client proof.

**Activation height** — the block height at which a set of consensus rules takes
effect. Always announced in blocks; a calendar date for it is an estimate.

### References

1. Grassi, Khovratovich, Rechberger et al. — *Poseidon2b: Binary Field Versions
   of Poseidon and Poseidon2*. IACR ePrint 2025/1893.
2. Diamond, Posen — *Succinct Arguments over Towers of Binary Fields* (Binius).
   IACR ePrint 2023/1784.
3. Ben-Sasson, Bentov, Horesh, Riabzev — *Fast Reed-Solomon Interactive Oracle
   Proofs of Proximity* (FRI). ECCC TR17-134.
4. Goldwasser, Kalai, Rothblum — *Delegating Computation: Interactive Proofs for
   Muggles* (GKR).
5. *Ligerito* — a polynomial-commitment construction the proving stack draws on.
   Its authors and the implementation Jetsam's port derives from are credited in
   the repository's `NOTICE`; we do not restate an author list here that we have
   not checked against the paper.
6. Block, Tiwari — *On the Concrete Security of Non-interactive FRI*.
   IACR ePrint 2024/1161. The method behind the figures in section 3.1; the
   result is theirs, the profiles put through it are ours.
7. Bitcoin Cash — *aserti3-2d* difficulty algorithm specification.

### Links

| | |
|---|---|
| site | <https://jetsamchain.com> |
| source code, releases and issues | <https://github.com/jetsam-chain/jetsam> — this document's code references are to tag `v1.5.1` (commit `dce63e6`) |
| CPU miner (open source) | <https://github.com/jetsam-chain/towerminer> |
| block explorer | <https://explorer.jetsamchain.com> |
| announcements | <https://x.com/jetsamchain> |
| Discord | <https://discord.gg/yjNgkyEw2W> |
| Telegram | <https://t.me/jetsamchain> |
| articles | <https://medium.com/@jetsamchain> |
| provenance | `NOTICE` and `CHANGES-FROM-UPSTREAM.md` in the repository |

### Version history

| version | date | what changed |
|---|---|---|
| **v3.1** | **15 Oct 2026** | Updated for the v1.5 crossing and its aftermath. v1.5 is stated as in force since block 38 400 (13 Oct 2026, 20:48:25 UTC) everywhere v3.0 said "armed" (sections 0, 2, 3.4, 4.2, 4.5, 5.7, 6, 6.7, 8, 9.6, 11, glossary). The crossing is described as it happened: about 70 % of the network stayed on the old rules, the chain produced no block for about 49 minutes between heights 38 410 and 38 411, and the taller old branch stalled the synchronisation of v1.5.0 nodes (sections 0, 2, 5.7, 8, 9.8, 11). v1.5.1 (14 Oct 2026, commit `dce63e6`) is now the code reference: its hard checkpoints at 38 399 and 38 400 replace the "planned follow-up" of v3.0 (sections 4.2, 8, glossary), and its synchronisation fix is described (sections 2, 8). Live figures re-read on 15 Oct 2026: height, supply, slot count, state size, and the mean interval of the 180 s era (sections 2, 4.3, 4.5). Minor: the licence is called a provisional placeholder rather than "priced high" (section 6.5); the fee schedule is described as a deterministic minimum with any excess going to the miner (section 7.4). |
| **v3.0** | **12 Oct 2026** | Updated for v1.4 (crossed) and v1.5.0 (armed). TowerWalk moves from "planned" to "live since block 24 846" with the anchor and checkpoint as crossed (sections 2, 5.3, 5.7, 8). Block interval 90 s → 180 s from height 38 400, halflife unchanged in seconds, finality depth 8 blocks ≈ 24 min, defined as a local rule and not a guarantee (section 4.5). New section 6.7 states exactly what the v1.5 client slot ships: one proof per block, closed and empty catalogue, provisional licence, no privacy. **Development allocation recomputed for the 480-block cadence: 747 000 JTM, 3.56 % of the cap, window ending at height 369 600** instead of 1 160 400 / 5.53 % / 700 800 (section 7.3), and the word "premine" is qualified wherever it appears (sections 0, 7.1). Emission calendar restated at 180 s (section 7.2). Third relation generation, circuit sizes 2^23 / 2^25 (section 3.4). New sections 9.7 (not-ZK, post-quantum stated narrowly) and 9.8 (concentration). Live figures re-read on 12 Oct 2026. Corrected before publication after a technical and a security review: the recursive proof is stated to cover the history from the root of its relation generation, with checkpoints below it, rather than as reaching the first block (sections 0, 1.1, 4.2, 10); only v1.3 was published as a dormant build (sections 2, 8, 11); towerminer was published the day after the 24 846 fork (section 8); the 7 September 2026 halt is stated (sections 0, 2); spend authorisation is described as witness-hiding by construction rather than zero-knowledge (sections 3.1, 4.4, 9.7); soundness figures are attributed to their components with the lowest kept as the floor (sections 3.1, 9.7); finality depth defined (section 4.5); client-form size, state size, CPU threading and undated intentions restated (sections 2, 4.3, 5.4, 6, 8, 9.1). |
| v1.0-draft | 23 Sep 2026 | internal draft, not published |
| **v2.0** | **24 Sep 2026** | First public version. Adds the abstract, the founding argument, the live-network status, the design principles including the no-curves property and its exact soundness caveats, the proof-market uses, the comparison, governance and provenance. Corrects the ASIC claim of the draft (section 9.1), withdraws its "0 % rentable" claim (section 9.4), states the source of every figure, and adds the CPU measurement the draft promised and did not contain (section 5.4). It also publishes the **development allocation the payout rule actually mints — 1 160 400 JTM — rather than the larger per-block entitlement of 1 163 996** (section 7.3), and separates the two distinct limits that were previously conflated into a single "22 %" (section 5.5). |

---

*Every consensus parameter in this document can be read in the public source and
is enforced by the network. Performance figures are our own measurements and
carry their method. Projections are labelled as projections. Where a result is
somebody else's, it is attributed.*
