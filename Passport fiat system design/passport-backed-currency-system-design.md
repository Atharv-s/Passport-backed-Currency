# Passport-Backed Currency (PBC) — System Design

## 1. Concept Summary

A digital currency system where a person's **passport is the root identity anchor** for
their account, and a country's **passport strength** (global mobility / trust index) feeds
into how the currency is priced, risk-weighted, and moved across borders. Three ideas are
fused into one design:

1. **Identity-anchored ledger** — every account is cryptographically bound to a verified
   passport, giving the currency Sybil-resistance, per-capita issuance/UBI capability, and
   built-in KYC/AML.
2. **Passport-strength-weighted value engine** — a "Passport Strength Oracle" (built from
   visa-free access rankings + country risk/sanctions data) modulates trust scores,
   collateral requirements, transfer limits, and FX spreads *per issuing country*, not
   per individual.
3. **Passport-gated remittance rail** — cross-border transfers require live passport
   verification at send and receive time, with fees/limits dynamically priced off the
   sender's and receiver's country risk scores.

This is presented as a neutral architecture. The choice of who runs it (a single central
bank, a multilateral consortium, or a private fintech) and how "value" is defined
(pegged to fiat, floating, algorithmic) is a policy decision layered on top — flagged
in §9.

---

## 2. High-Level Architecture

See the accompanying diagram (`architecture.mermaid`). Six layers:

| Layer | Responsibility |
|---|---|
| User Layer | Citizens, merchants, correspondent banks/foreign CBDCs |
| Identity & Verification | Passport chip read, biometric liveness, government registry check, ZK identity commitment |
| Passport Strength Oracle | Ingests mobility index + country risk/sanctions data, publishes per-country scores |
| Currency Core | Identity-bound account registry, ledger, risk/value modulation engine, exchange & fee engine, compliance engine |
| Payment Rails | Domestic transfers, cross-border remittance gateway, ISO 20022/SWIFT bridge, other CBDC/stablecoin bridge |
| Governance | Central bank/consortium nodes, auditors, policy parameters |

---

## 3. Identity & Verification Layer

**Goal:** bind one wallet to one real, currently-valid passport, without storing raw
passport data on the ledger.

- **Chip read (ICAO 9303 e-passport):** NFC read of the passport's chip via phone or
  kiosk; retrieves the Document Security Object (SOD) and biometric data page.
- **PKD signature check:** validate the chip's data against the issuing country's
  certificate via the **ICAO Public Key Directory (PKD)** — confirms the passport is
  genuine and unexpired.
- **Liveness + biometric match:** selfie/liveness check matched against the chip photo
  to prevent stolen-document use.
- **Government registry ping (optional, bilateral):** where a data-sharing agreement
  exists, confirm the passport hasn't been reported lost/stolen/revoked.
- **Identity commitment (ZK):** instead of storing the passport number, generate a
  **zero-knowledge commitment** — e.g. `commit = H(passport_number || country_code || salt)`
  — proving "one unique passport per account" and "issuing country = X" without revealing
  the underlying document to the ledger or to other participants. Nullifier schemes
  (as used in privacy-preserving identity protocols) prevent the same passport from
  opening a second account.

**Output:** an `identity_token` containing `{account_id, issuing_country, verification_level, expiry}` — this is what the Currency Core actually sees.

---

## 4. Passport Strength Oracle

This is the piece that makes it "passport-backed" rather than just "passport-verified."

- **Inputs:**
  - A global mobility/visa-free-access index (e.g., an aggregation similar to the
    Henley or Arton indices — visa-free/visa-on-arrival destination counts per passport).
  - Country risk data: sovereign credit rating, FATF grey/black list status, sanctions
    exposure, currency volatility history, corruption/rule-of-law indices.
- **Processing:** a scheduled job (e.g., weekly) aggregates these into a normalized
  **Country Trust Score (CTS)**, 0–1, per issuing country.
- **Publishing:** the CTS is signed and published on-chain/on-ledger so all nodes can
  verify it wasn't tampered with; historical scores are retained for auditability.
- **Important boundary:** the CTS applies to the *issuing country as a jurisdiction*,
  not a judgment on the individual holder. This distinction should be enforced in code
  and disclosed in policy — conflating the two risks discriminatory outcomes against
  individuals based solely on nationality, which is a real design and ethical hazard
  (see §9).

---

## 5. Currency Core

- **Account Registry:** maps `identity_token.account_id → account record`. One passport
  → one account (enforced via the ZK nullifier).
- **Ledger:** account balances and transaction history. Can be implemented as a
  permissioned blockchain (for multi-issuer/consortium models) or a conventional
  double-entry database (for a single central-bank model) — the rest of the design is
  agnostic to this choice.
- **Value & Risk Modulation Engine:** consumes the CTS to set, per issuing country:
  - collateral/reserve ratio required to mint or hold large balances,
  - daily/monthly transfer limits,
  - eligibility for credit-like features (if any).
- **Exchange Rate / Fee Engine:** if the currency floats against other currencies, this
  engine sets bid/ask spreads; cross-border transfer fees are risk-priced using the CTS
  of both the sending and receiving country (higher perceived risk → wider spread/fee,
  similar to how correspondent banks price country risk today).
- **Compliance Engine:** real-time sanctions list screening, AML transaction monitoring,
  travel-rule data attachment for cross-border transfers, using the CTS as one signal
  among several (never the sole basis for blocking a legitimate transaction).

---

## 6. Payment Rails

- **Domestic transfers:** standard account-to-account within the system.
- **Cross-border remittance gateway:** requires a fresh passport verification handshake
  (not just a login) for transfers above a policy-set threshold, reusing the Identity
  layer.
- **ISO 20022 / SWIFT bridge:** translates PBC transactions into standard messaging
  formats for interoperability with legacy banking rails.
- **Other CBDC/stablecoin bridge:** atomic-swap or HTLC-based bridge for exchanging PBC
  against other digital currencies.

---

## 7. Key Workflows

**Onboarding**
1. User scans passport chip + selfie liveness check.
2. Client verifies PKD signature locally or via verification service.
3. ZK identity commitment generated; nullifier checked against registry (reject duplicates).
4. Account created; `issuing_country` and `verification_level` recorded; CTS looked up.

**Domestic transfer**
1. Sender authenticates (session token, not full passport re-scan).
2. Compliance engine screens transaction.
3. Ledger updates atomically.

**Cross-border remittance**
1. Sender authenticates; if above threshold, re-verify passport liveness.
2. Compliance engine screens sender + receiver against sanctions lists.
3. Exchange/Fee engine prices the transfer using sender CTS + receiver CTS.
4. Funds move via remittance gateway → SWIFT/CBDC bridge → receiver's institution.
5. Receiver's passport-bound account is credited after receiver-side compliance check.

---

## 8. Security & Privacy

- Passport numbers and biometric templates **never touch the shared ledger** — only ZK
  commitments and nullifiers do.
- Selective disclosure: a user can prove "I hold a passport from a country with CTS >
  0.7" without revealing which country, where policy allows range proofs instead of
  exact values.
- Key management: loss of a passport requires a re-verification and account recovery
  flow (biometric + old-chip revocation check) rather than a simple password reset.
- Auditor nodes get view-only access to aggregate flows, not individual identities,
  unless under legal order.

---

## 9. Governance, Policy & Risk Flags

- **Who sets the CTS methodology and the fee/collateral curves derived from it is a
  policy decision, not a technical one** — it should sit with a transparent governance
  body (central bank board, multilateral consortium with published charter), not be
  silently embedded in code.
- **Discrimination risk:** pricing or limiting transactions based on issuing-country
  score can shade into discrimination against individuals for their nationality alone.
  Mitigations: CTS affects *pricing/limits*, never outright transaction denial for
  individuals; appeals process; regular fairness audits; published methodology.
- **Concentration/systemic risk:** a single-issuer model concentrates monetary policy
  power; a consortium model needs clear rules for how member central banks agree on
  parameter changes.
- **Regulatory reality:** real-world deployment would need to satisfy each jurisdiction's
  data protection law (e.g., biometric data handling), AML/CFT law, and — for any CBDC
  component — central bank legal authority. This design is a technical blueprint, not a
  compliance guarantee.

---

## 10. Suggested Tech Stack (illustrative)

- **Passport verification:** ICAO PKD client libraries, NFC read SDKs (mobile), a
  liveness-detection vendor or open model.
- **Ledger:** Hyperledger Fabric or a permissioned Cosmos SDK chain for
  consortium/multi-issuer; PostgreSQL-backed double-entry ledger for single-issuer.
- **ZK layer:** circom/snarkjs or Noir for the identity commitment and nullifier circuits.
- **Oracle:** a scheduled service (e.g., a small ETL job) pulling public mobility-index
  and country-risk datasets, signing outputs with a governance key before publishing.
- **Interoperability:** ISO 20022 message library, HTLC bridge contracts for CBDC/stablecoin interop.

---

## 11. Open Questions Before Building

- Single-issuer (one central bank/country) vs. multi-issuer consortium?
- Is the currency's value pegged (e.g., 1:1 to a fiat basket) or floating?
- What's the actual source for the "passport strength" data, and is it licensed for
  commercial/governmental use?
- What's the appeals/override process when the CTS produces an unfair outcome for an
  individual?
