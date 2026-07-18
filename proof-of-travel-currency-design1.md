# Proof-of-Travel: A Passport-Rank-Weighted, Travel-Evidence-Backed Currency

*A system design in the spirit of Nakamoto's proof-of-work paper — substituting
computational work with verified, evidence-based travel as the "work" that anchors value.*

## Abstract

Bitcoin replaced a trusted third party with computational proof-of-work: whoever
expends the most verifiable energy extends the chain. This design substitutes travel
as the costly, hard-to-fake signal: whoever accumulates the most *verifiable,
evidence-backed* border crossings — weighted by how hard their passport makes that
travel — earns issuance. Reputation/rank of the traveler's passport (visa-free access
strength) acts as a **difficulty multiplier**, since a low-rank passport holder who
travels widely has overcome more real-world friction (visas, denials, cost) than a
high-rank passport holder doing the same trip. Family-linked evidence increases
confidence that travel is genuine (staged solo photos are cheaper to fake than a
consistent multi-person, multi-location history) but introduces serious privacy and
consent obligations that are addressed explicitly, not as an afterthought.

---

## 1. Why Travel as "Work"

Proof-of-work is valuable because it's *expensive to produce, cheap to verify*. Travel
has similar shape:
- **Expensive to produce:** cost, time, visas, employer/family constraints, and — for
  low-rank passport holders — bureaucratic friction (visa denials, proof-of-funds
  requirements, interview scrutiny) that high-rank passport holders don't face for the
  same route.
- **Cheap to verify (if evidence is structured right):** passport stamps/e-gate logs,
  boarding passes, geotagged media, and government entry/exit records are independently
  checkable against airline, immigration, and mapping data.

This gives the **Passport Rank Multiplier**: the same verified trip is worth more
"proof" for a holder of a low-rank passport than a high-rank one, because it represents
more overcome friction — the opposite of just rewarding whoever already has the most
powerful passport.

---

## 2. Core Formula

```
ProofScore(trip) = TravelEvidenceScore(trip) × PassportFrictionMultiplier(holder, route)

TokenIssuance(user, period) = BaseRate × Σ ProofScore(trip) over verified trips in period
```

- **PassportFrictionMultiplier** — derived from a Passport Strength Oracle (as in the
  earlier design) but *inverted* relative to a pure "trust score": a passport that grants
  visa-free access to the destination gets a low multiplier (≈1.0); a passport that
  required a visa, interview, or has a history of denial for that destination gets a
  higher multiplier (e.g., up to 3–5x), reflecting the extra friction genuinely overcome.
  This must be calibrated by real visa-requirement data (e.g., bilateral visa policy
  databases), not guesswork.
- **TravelEvidenceScore** — 0 to 1, from the verification pipeline in §3, reflecting
  confidence the trip actually happened as claimed.

---

## 3. Evidence & Verification Pipeline

This is the part that needs the most rigor, because "photo/video evidence" is exactly
the kind of input that's easy to fake and easy to overreach on privacy.

**Tier 1 — Authoritative records (highest weight, least privacy exposure)**
- E-passport chip entry/exit stamps (ICAO PKD-verifiable), airline PNR/boarding pass
  data, immigration API confirmations where a bilateral data-sharing agreement exists.
- These alone can produce a usable `TravelEvidenceScore` without any photo/video at all.

**Tier 2 — Device-attested media (medium weight)**
- Photo/video with cryptographically signed metadata at time of capture (device-level
  attestation, e.g., a signed EXIF/C2PA content-credential), geolocation, and timestamp,
  cross-checked against known landmark coordinates and against the Tier-1 record for the
  same window.
- Verification checks: reverse-geocoding consistency, weather-at-time-and-place
  consistency, anti-deepfake/anti-splice detection on the media itself.

**Tier 3 — Family/group consistency (highest-friction to fake, highest privacy sensitivity)**
- A group of pre-enrolled, consenting adult family members whose *on-device* biometric
  templates (never raw images) can confirm "these same people appear together across
  multiple independently-verified locations over time" — which is much harder to fake
  than one person's photos alone.
- **This tier is opt-in only, per person, and adults-only for biometric matching.**

**Privacy and child-safety guardrails (non-negotiable in this design):**
- No minor's image, biometric template, or location history is ever used as proof
  material. Family evidence is built only from consenting adults; if children appear
  incidentally in a photo, they are not identified, matched, or tracked, and the
  system should actively redact/blur minors before any processing.
- Biometric matching happens **on-device**; only a signed hash/attestation is sent to
  verifiers — raw photos/videos and biometric templates are never centrally stored.
  This limits both the exploitation surface and the "honeypot" risk of a central photo
  database of families and their travel movements.
- Every family member's participation is independently revocable; a person can remove
  themselves from another's evidence chain at any time, and future proofs stop
  referencing them.
- Because this system, if built carelessly, would essentially be a location-and-family
  tracking database, the architecture below treats "collect the minimum, verify
  on-device, store only attestations" as a hard constraint, not an optimization.

---

## 4. Chain Structure (Nakamoto-style)

Instead of one global chain of blocks, each identity maintains a **personal proof chain**,
anchored periodically into a shared public ledger (similar to how many identity/DID
systems checkpoint into a base chain):

```
Genesis(identity) → Trip₁ proof → Trip₂ proof → Trip₃ proof → ... 
        each proof references: prior proof hash, Tier-1/2/3 evidence hashes,
        PassportFrictionMultiplier at time of trip, resulting ProofScore
```

- **Validator network:** independent nodes re-run the verification pipeline (checking
  signatures, geolocation math, oracle data) and reach consensus (e.g., BFT-style
  voting among validators) before a proof is accepted and tokens are issued —
  analogous to miners validating a block, except the "work" being checked is evidence
  authenticity rather than a hash puzzle.
- **Difficulty adjustment:** as with Bitcoin's retargeting, the friction multipliers
  and evidence thresholds are periodically recalibrated (e.g., if a route's visa
  requirement changes, or if a spoofing technique is discovered and evidence
  requirements need to tighten).
- **Sybil resistance:** the identity layer from the earlier passport-backed-currency
  design (one passport → one root identity, via ZK commitment + nullifier) prevents one
  person from farming multiple chains.

---

## 5. Incentive & Value Model

- **Issuance:** new tokens minted per validated ProofScore, per period, with a
  decreasing `BaseRate` over time (a halving-style schedule) so early, harder-to-fake
  travel history is worth more than later, potentially gamed patterns — mirroring
  Bitcoin's diminishing block reward.
- **Value grounding:** the token's market value still depends on adoption/exchange
  markets like any cryptocurrency — proof-of-travel determines *issuance*, not
  automatically *price*. This design doesn't assume the token is pegged to anything;
  that's a separate policy decision.
- **Anti-gaming economics:** because low-rank-passport travel is worth more per trip,
  there's an incentive to route travel through genuinely difficult itineraries rather
  than reuse of the same easy, high-rank-passport hops — but this also means the system
  must be very carefully audited so it doesn't inadvertently reward risky, exploitative,
  or unsafe travel patterns (e.g., encouraging someone to take on debt or physical risk
  purely to farm tokens). A per-period cap on issuance per identity is recommended.

---

## 6. Threat Model Highlights

| Threat | Mitigation |
|---|---|
| Deepfaked or staged travel photos | Device-attestation (C2PA-style signing) + Tier-1 authoritative record cross-check; photos alone are never sufficient evidence |
| Reused old photos for new "trips" | Timestamp + freshness window + liveness signals; validators check for duplicate media hashes across the network |
| Family collusion to fake a group trip | Requires Tier-1 records for each participant independently, not just consistent photos |
| Central honeypot of biometric/location data | On-device matching only; ledger stores attestations/hashes, never raw media or templates |
| Passport-rank gaming (e.g., acquiring a second, weaker passport to farm multiplier) | Root identity tied to one passport at a time via nullifier; multiplier keyed to the *actually used* travel document per trip, publicly auditable |
| Discriminatory outcomes based on nationality | Multiplier is about *route friction*, published and auditable, not a blanket judgment on individuals; appeals process for miscalibrated routes |

---

## 7. Decentralization-Linked Value & Supply Mechanics

The core idea here: **the protocol should be able to prove, on an ongoing basis, that
no single actor or small cluster controls it — and supply/value mechanics respond
automatically when that stops being true.** This is the Nakamoto-consensus parallel to
Bitcoin's fixed-supply/difficulty-adjustment logic, but the "difficulty" being adjusted
is *concentration*, not just hash rate.

### 7.1 Decentralization Index (DI)

A publicly computable, on-chain metric, recomputed every epoch, combining:

- **Validator diversity:** Nakamoto coefficient of the validator set (minimum number of
  validators needed to control >33%/50% of consensus weight). Higher is better.
- **Geographic/jurisdictional spread:** entropy of validator locations and, notably,
  entropy of *passport-issuing countries* backing validators — a system where all
  validators are rooted in one or two nations' passports is not meaningfully
  decentralized even if it has many nodes.
- **Holdings concentration:** Gini coefficient / Nakamoto coefficient of token balances
  (how many holders control >X% of supply).
- **Client diversity:** share of nodes running independently-authored, independently
  audited client software (a single reference client run by everyone is a centralization
  risk even with many operators).
- **Governance participation spread:** entropy of voting power in protocol-parameter
  votes over recent epochs.

```
DI(epoch) = weighted_combination(
    ValidatorNakamotoCoeff,
    PassportCountryEntropy,
    HoldingsGiniInverse,
    ClientDiversityShare,
    GovernanceParticipationEntropy
)
```

DI is published every epoch alongside the raw inputs, so anyone can independently
recompute and challenge it — the same "don't trust, verify" property Nakamoto consensus
relies on for block validity.

### 7.2 Anti-Control Mechanisms

- **Diversity-weighted governance quorum:** protocol-parameter changes require
  supermajority approval *distributed across* a minimum number of independent
  passport-country clusters and validator operators — not just a raw token-weighted
  majority. This blocks a single wealthy actor or aligned bloc from unilaterally
  changing issuance rules, even if they accumulate tokens or validators.
- **Per-entity issuance and stake caps:** no single identity (root passport commitment)
  or validator operator can accumulate more than a capped share of new issuance or
  consensus weight per epoch, regardless of how much travel-proof or capital they have.
- **Mandatory client plurality:** consensus rules reject blocks/proofs if client
  diversity drops below a threshold, discouraging silent convergence on one
  vendor-controlled implementation.
- **No privileged genesis or backdoor authority:** no admin key, upgrade multisig, or
  "founder" allocation with outsized power; protocol upgrades happen only through the
  diversity-weighted governance process above.
- **Circuit breaker:** if DI drops below a critical threshold (e.g., Nakamoto
  coefficient falls to a small number of entities), the protocol automatically
  restricts itself — e.g., freezing further supply growth and/or large transfers —
  until DI recovers, rather than continuing to run "normally" under de facto capture.

### 7.3 Inflation / Deflation Tied to Proof of Decentralization

Issuance (from §5) is scaled by DI, not fixed purely by ProofScore:

```
TokenIssuance(user, period) = BaseRate × DI_factor(epoch) × Σ ProofScore(trip)

DI_factor(epoch) = f(DI(epoch))   // e.g., near 1.0 when DI is healthy,
                                   // tapering toward 0 as DI falls toward the
                                   // circuit-breaker threshold
```

- **Deflationary pressure from concentration:** a small transaction-fee burn is
  redirected disproportionately from over-concentrated holdings (e.g., balances above
  the per-entity cap accrue a higher burn rate on movement), gently pushing supply back
  toward wider distribution rather than just relying on voluntary redistribution.
- **Halving-style base decay independent of DI:** `BaseRate` still halves on a fixed
  schedule (as in §5) so early participation isn't infinitely rewarded — DI_factor is a
  *multiplier on top of*, not a replacement for, that schedule.
- **Published proof, not a claimed policy:** every epoch's DI, DI_factor, total minted,
  and total burned are all independently recomputable from public data — "proof of
  decentralization" is meant literally: it's a number anyone can audit, not a promise
  from an operating entity.

### 7.4 Trade-offs and Honest Limitations

- Any concentration metric can be gamed at the margins (e.g., splitting holdings across
  many identities) — the identity layer's one-passport-per-account nullifier (§4 of the
  earlier design) limits but doesn't eliminate this; sophisticated Sybil networks using
  multiple real people's passports ("passport farms") remain a real threat that would
  need dedicated detection (e.g., statistical anomaly detection on travel patterns).
- A circuit breaker that halts activity when DI drops is a strong anti-capture tool but
  also a denial-of-service vector: an attacker who can *push* DI below threshold (rather
  than capture it) could freeze the system. The threshold and the freeze's exact
  behavior would need careful game-theoretic modeling before deployment.
- Tying issuance to a computed index adds real protocol complexity relative to Bitcoin's
  simple fixed schedule — more moving parts generally means more attack surface, so this
  section should be read as a target design worth stress-testing, not a finished spec.

---

## 9. Government-Independent Evidence Layer (Self-Sovereign Travel Proof)

The design so far leaned on Tier-1 "authoritative records" (passport chip stamps,
immigration APIs) as the strongest evidence tier. This section reworks the evidence
layer so that **ongoing travel verification never depends on any single government's
cooperation, data-sharing agreement, or infrastructure** — only self-captured
selfie/video evidence, corroborated by a decentralized network, is used. The passport
itself is still a government-issued document (that can't be avoided — it's what makes
this a *passport*-backed system), but it is used **only once**, at account creation, to
establish a root identity and country-of-issuance. After that, no government API,
registry, or cooperation is required for any individual trip to be verified.

### 9.1 What changes from §3

| | Original design | This variant |
|---|---|---|
| Root identity | Passport chip + ICAO PKD, one-time | Same — unavoidable, but one-time only |
| Ongoing trip verification | Immigration API pings, e-gate logs (Tier 1) | **Removed.** No per-trip government dependency at all |
| Primary evidence | Device-attested photo/video (Tier 2), family consistency (Tier 3) | Self-captured selfie/video is now the *primary* and *only* evidence type, verified by a decentralized corroboration network |

### 9.2 Decentralized Corroboration Network (DCN)

Since there's no government record to check a selfie against, trust comes from
**redundant, independent, hard-to-collude verification** — the same trick Nakamoto
consensus uses (many independent parties, majority-honest assumption) applied to
evidence-checking instead of block validity:

- **Capture-time attestation:** the traveler's device signs the selfie/video at
  capture (content-credential style signature — e.g., C2PA), embedding timestamp and
  raw GPS/GNSS + cell-tower/Wi-Fi-assisted location, so the *device* vouches for
  "this media was captured here, now" independent of any app-layer claim.
- **Landmark/scene verification (automated):** an open, replicable computer-vision
  model checks the selfie's background against public imagery of the claimed location
  (landmarks, terrain, signage, sun angle for time-of-day/season plausibility).
  Because the model and its reference imagery are public, any validator node can
  re-run this check — no single party's judgment is trusted.
- **Peer corroboration:** independent validator nodes (a decentralized set, not one
  operator) cross-check whether *other, unrelated* travelers' public geotagged
  submissions place them at overlapping times/locations (e.g., two strangers'
  selfies both plausibly taken at the same viewpoint the same week) — convergent,
  unrelated evidence is much harder to fake at scale than one person's photo stream.
  This is conceptually similar to decentralized proof-of-location protocols, where
  many independent witnesses jointly attest to a claim rather than one authority.
- **Liveness & anti-deepfake checks:** standard active/passive liveness detection on
  the video, plus splice/deepfake-detection models, run redundantly by multiple
  validators so no single validator's model failure lets a fake through.
- **Consensus threshold:** a trip proof is accepted only once a quorum of
  independent validators (diverse operators, per the anti-control rules in §7)
  agree the evidence is consistent — mirroring how a Nakamoto-style chain accepts a
  block only after majority-honest validation, not a single node's say-so.

### 9.3 Why this is harder, not easier, to fake at scale

Removing the government check doesn't lower the bar — it shifts the security
assumption from "trust one issuer" to "trust that a majority of independent,
economically-incentivized validators aren't colluding," which is exactly Nakamoto's
core assumption applied to evidence review. Practically, this means:

- A single faked selfie is easy; a faked selfie that also matches independently
  reconstructed lighting/landmark/peer-corroboration data across many unrelated
  validators is materially harder.
- The system should expect a **higher false-accept and false-reject rate at launch**
  than a government-API-backed system, and should size economic penalties (slashing
  stake, reputation loss for repeat fraud attempts) accordingly rather than assume
  perfect accuracy.

### 9.4 Passport-country ranking still applies unchanged

The **PassportFrictionMultiplier** from §2 is untouched by this change — it still
comes from the Passport Strength Oracle (visa-free access data, §4 of the earlier
design) and still scales ProofScore by how much friction the holder's passport implies
for a given route. Only *how a trip is verified* changed; *how much a verified trip is
worth* still depends on passport-country ranking exactly as before.

### 9.5 Privacy note (unchanged principle, restated for this variant)

Because this variant leans even more heavily on personal selfies/videos, the §3
guardrails apply with equal or greater force: on-device biometric processing only,
no centralized raw-media storage, adults-only for any biometric matching, and no
minors' images used as proof material under any circumstance. A government-independent
system that centralizes raw selfie/video streams would be a worse privacy outcome than
the government-anchored version, not a better one — decentralization has to apply to
*where the data lives*, not just who can transact.

---

## 10. Open Questions / Next Design Decisions

- Which authoritative data sources (immigration APIs, airline systems) are realistically
  available via partnership vs. which will have to rely on Tier 2/3 evidence alone?
- What issuance cap per identity per period prevents both farming abuse and perverse
  incentives to over-travel?
- Governance: who recalibrates friction multipliers when visa policy changes, and how
  fast?
- Legal basis for any biometric processing, even on-device and opt-in — this varies a
  lot by jurisdiction (e.g., GDPR biometric data rules, BIPA in Illinois) and would need
  real legal review before implementation, especially anywhere family members might be
  processed.

This is a conceptual architecture, not an implementation-ready spec — the evidence
pipeline and privacy safeguards in §3 in particular would need real cryptographic and
legal engineering before touching real people's travel or family data.
