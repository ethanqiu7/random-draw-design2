# Neutron-Chain RNG Protocol Specification

**Version**: v1.0
**Status**: Draft
**Author**: ethanycq
**Last Updated**: 2025-09-04
**Scope**: A publicly verifiable true-random number generation mechanism based on the mixture of cosmic-ray neutron monitor data and future blockchain block hashes (for lotteries, voting, draws, on-chain seeds, etc.).

---

## 1. Protocol Goals and Design Principles

### 1.1 Goals

This protocol defines a mechanism that **outputs one set of 256-bit true-random numbers every 10 minutes**, satisfying the following three core properties:

| Property | Meaning |
|----------|---------|
| **Unpredictability** | Before the output moment, no participant (including the operator) can predict the result with non-negligible probability |
| **Tamper-resistance** | Once published, the output is publicly recorded; no one can modify it after the fact without detection |
| **Public Verifiability** | Anyone, at any time, can reproduce the complete computation from public information alone and compare the result |

### 1.2 Design Principles

1. **Fixed rules, dynamic parameters**: The entropy source list, detrending algorithm, mixing function, output format, and delay window are **permanently fixed** after deployment; the specific future block numbers and raw neutron data for each round are determined dynamically at runtime.
2. **Physical entropy + cryptographic anchor, dual insurance**: Cosmic-ray neutron counts provide physical unpredictability; blockchain future block hashes provide a public, unpredictable cryptographic anchor.
3. **Future-data commitment**: All raw data participating in the mixture **has not yet been produced** at the registration/voting deadline (T0), fundamentally eliminating "see the result first, then regret" timing attacks.
4. **Minimal trust assumption**: As long as **any one** of the three parties—the physical entropy source, BTC, or ETH—remains honest and unpredictable, the final output is unpredictable.
5. **Open source and reproducible**: All rules, code, raw data, and computation processes are public, so anyone can verify independently.

---

## 2. Terms and Notation

| Symbol | Meaning |
|--------|---------|
| T0 | Registration/voting deadline (start of a round) |
| Δ | Delay window; this protocol uses Δ = 10 minutes |
| N | **Neutron entropy bit string** (256-bit), obtained after detrending and bit extraction |
| `B_btc` | Locked future Bitcoin block hash (32 bytes) |
| `B_eth` | Locked future Ethereum block hash (32 bytes) |
| `⊕` | **XOR** operation: same → 0, different → 1 |
| `‖` | Byte string concatenation |
| SHA-256 | Standard SHA-256 hash function |
| NMDB | Neutron Monitor Database, the public repository of cosmic-ray neutron monitor data |

---

## 3. Entropy Sources

### 3.1 Physical Entropy Source: Four-Station Neutron Monitor Data

This protocol fixes the following **4** NMDB stations, each with count rates updated **every 2 minutes**:

| Code | Station | Location (Country/Region) | Geomagnetic Cutoff Rigidity |
|------|---------|--------------------------|----------------------------|
| KIEL2 | Kiel | Germany | Mid-latitude |
| APTY | Apatity | Russia | Polar (low cutoff) |
| SOPO | South Pole | Antarctica (USA) | Polar |
| TERA | Terre Adélie | Antarctica (France) | Polar |

> **Selection rationale**: The 4 stations belong to 4 independent legal/operational jurisdictions (Germany, Russia, USA, France), and geographically cover both mid-latitude and polar regions. Statistical independence has been empirically verified (average |r| ≈ 0.16 after detrending; independent degrees of freedom ≈ 3.5, close to full rank).

### 3.2 Cryptographic Anchor Source: Dual-Chain Future Block Hashes

| Anchor | Chain | Block Time | Locking Strategy |
|--------|-------|-----------|------------------|
| `B_btc` | Bitcoin | ~10 min | Lock **current height + 2** (margin to prevent timeout) |
| `B_eth` | Ethereum | ~12 sec | Lock **current height + 50** (≈10 min) |

> **Critical constraint**: The locked block **must not yet have been produced at T0**. This is the lifeline of the protocol's security—using a block already in existence before T0 would expose it to a selective-disclosure attack by miners.

---

## 4. Data Processing: Differential Detrending

### 4.1 Problem

Raw neutron count rates exhibit **slow drift** (solar modulation, diurnal variation, instrument trend). Measured examples:

| Station | Slope (/sample) | Drift per Hour |
|---------|----------------|----------------|
| KIEL2 | +0.0606 | +1.82 ⚠ |
| SOPO | −0.0374 | −1.12 ⚠ |
| TERA | +0.0148 | +0.44 |
| APTY | +0.0060 | +0.18 (most stable) |

Drift over 10 minutes is 0.3–0.5%. Taking the least-significant bit directly would introduce **systematic bias**, making the result predictable.

### 4.2 Algorithm

For each station `s ∈ {KIEL2, APTY, SOPO, TERA}`, take `m = 5` two-minute samples `x_1, x_2, ..., x_m` within the 10-minute window:

```
1. Difference detrending:  d_i = x_i − x_{i−1},   i = 2..m   (produces m−1 = 4 first differences)
2. Detrended residual:     r_i = d_i − mean(d)            (remove mean, eliminate DC bias)
3. Bit extraction:         b_i = LSB( round(r_i × 100) )  (×100 amplifies decimals, take LSB)
4. Concatenation:          station entropy s_bits = b_2 ‖ b_3 ‖ ... ‖ b_m  (4 bits)
```

After concatenating the four stations, compress via SHA-256 into the 256-bit `N`:

```
N = SHA-256( KIEL2_bits ‖ APTY_bits ‖ SOPO_bits ‖ TERA_bits ‖ round_number )
```

### 4.3 Missing-Value Handling

If a station has a missing value at a given 2-minute sample (API timeout, `null`, etc.):

- **Interpolation is forbidden** (it would artificially introduce correlation);
- Fill with the **value from the same position in the previous period**;
- If a single station's missing rate exceeds 20%, that round is automatically **degraded** (see Section 9, Exception Handling).

---

## 5. Core Mixing Function: XOR Dual-Anchor

### 5.1 Definition

```
Seed = SHA-256( N ⊕ B_btc ⊕ B_eth )

Output = SHA-256( Seed ‖ "NeutronChain-RNG-v1" )
```

Here `N`, `B_btc`, and `B_eth` are all 32 bytes (256-bit); **byte-aligned XOR is applied first, then hashing**.

### 5.2 XOR Semantics

`⊕` is byte-wise XOR (bit-wise: same → 0, different → 1). The key security property of XOR mixing:

> **As long as at least one of the values participating in the XOR is completely random and unknowable at T0, every bit of the final Seed is unpredictable.**

| Attacker's Control Scope | Can Predict Seed? | Reason |
|--------------------------|:-----------------:|--------|
| Only neutron stations (knows N) | ❌ No | `B_btc`, `B_eth` are future secrets at T0 |
| Only BTC 51% (knows `B_btc`) | ❌ No | N, `B_eth` are unknowable |
| Only ETH 51% (knows `B_eth`) | ❌ No | N, `B_btc` are unknowable |
| Controls neutron + BTC 51% (not `B_eth`) | ❌ No | `B_eth` poisons the result |
| Controls all three parties | ⚠️ Theoretically | Cost is exponential; practically infeasible |

### 5.3 Why XOR Instead of Concatenation

- XOR guarantees that **if any component remains secret, the whole is unpredictable** (a clean information-theoretic security reduction);
- Concatenation would also work, but if one component is predicted, an attacker can narrow the search space given the "known prefix";
- XOR naturally **aligns output length** (= the longest input length), requiring no extra truncation.

---

## 6. Complete Protocol Flow

```
T0: Registration/voting deadline
    │
    ├─ [Fixed rules] Lock:
    │     • Entropy = {KIEL2, APTY, SOPO, TERA}
    │     • Algorithm = differential detrending + LSB
    │     • Anchor = BTC #(current+2), ETH #(current+50)
    │     • Mixing = SHA-256( N ⊕ BTC ⊕ ETH )
    │     • Delay Δ = 10 minutes
    │
    └─ [Dynamic computation] Record T0 timestamp and current BTC/ETH heights

T0+10 min: Data collection
    │
    ├─ Fetch 4-station neutron counts (NMDB public API, fresh at T0+10)
    ├─ Process per §4 algorithm → obtain N (256-bit)
    ├─ Read BTC #(current+2) block hash
    ├─ Read ETH #(current+50) block hash
    │
    ├─ Compute Seed = SHA-256( N ⊕ BTC ⊕ ETH )
    └─ Compute Output = SHA-256( Seed ‖ "NeutronChain-RNG-v1" )

Publication:
    ├─ Output (final random number, main display)
    ├─ Seed (for audit)
    ├─ Raw neutron data (4 stations × 5 samples)
    ├─ Block hashes BTC / ETH
    └─ Complete computation log (for reproduction)

Verification (anyone, anytime):
    Download raw data → reproduce detrending + LSB → N
    → read block hashes → recompute Seed → compare Output
```

### 6.1 On the Trade-off Regarding Commit-Reveal

Because **both the neutron data and the block hashes are "future secrets" at T0**, there is naturally no premature-leakage problem. Therefore this protocol **does not require** a traditional two-phase Commit-Reveal to hide already-existing data.

However, to prevent the operator from **selectively delaying or recomputing** after obtaining the result at T0+10, it is recommended to write the Output into a **smart contract** (see Section 8), leveraging on-chain immutability and automatic publication to eliminate any human-operated window.

---

## 7. Trust Model and Security Boundaries

### 7.1 Trust Assumptions

1. **Honest NMDB data source**: The 4 stations' data is submitted with signatures from independent operators; a single station's failure or misbehavior does not affect the whole (XOR + multi-source).
2. **Immutable blockchain**: The BTC/ETH networks are free of 51% attacks, or at least an attacker cannot control both BTC and ETH simultaneously.
3. **SHA-256 is collision-resistant and preimage-resistant**.

### 7.2 Required Engineering Safeguards

| Risk | Required Measure |
|------|------------------|
| Single operator tampers with local counts | **Multi-party independent signatures** for submission, or **t-of-4 threshold signatures** (≥3 parties to confirm) |
| Selective block disclosure | Lock **future** blocks + dual-chain anchoring |
| Long-term slow drift | Differential detrending + NIST SP 800-90B health tests |
| Low output rate (≈0.1 bps) | Use Seed → ChaCha20/AES-CTR **output expansion**; for high-frequency use, do not draw raw bits directly |

### 7.3 Explicit Non-Goals

- This protocol does **not claim** that neutron data itself is "immutable"—neutrons only solve the unpredictability of entropy;
- **Immutability** is jointly guaranteed by "on-chain records + multi-party signatures + open-source replay";
- **Public verifiability** is guaranteed by "fixed rules + public data + open-source code".

> **In one line**: Neutrons are the fuel, blocks are the anchor, cryptography is the engine.

---

## 8. Recommended Implementation: Smart-Contract Auto-Anchoring

To eliminate the operator's human-operated window, it is recommended to deploy the "T0+10 computation and publication" as an **open-source smart contract**:

```
Contract state (per round):
  - roundId            // round number
  - t0Timestamp        // T0 moment
  - btcTargetBlock     // locked BTC block number (current+2)
  - ethTargetBlock     // locked ETH block number (current+50)
  - neutronEntropy     // neutron entropy N submitted by operator (requires multi-sig)
  - finalizedOutput    // final Output (written permanently after computation)

Core functions:
  1. finalizeRound():
        require(block.number >= ethTargetBlock)
        B_btc = blockhash(btcTargetBlock)   // BTC side via oracle/submission
        B_eth = blockhash(ethTargetBlock)   // native ETH block hash
        Seed = SHA-256( N ⊕ B_btc ⊕ B_eth )
        Output = SHA-256( Seed ‖ "NeutronChain-RNG-v1" )
        finalizedOutput = Output            // permanently on-chain, immutable

  2. submitNeutron(bytes32 N, bytes[] signatures):
        require(verifySignatures(N, signatures))  // ≥3/4 signatures
        neutronEntropy = N
```

Once deployed with fixed rules, anyone can audit the contract; once Output is on-chain, it is visible to all and cannot be rolled back.

---

## 9. Exception Handling and Degradation

| Situation | Handling |
|-----------|----------|
| A station's data is missing | Fill with the value from the same position in the previous period; missing rate > 20% triggers degradation |
| Locked BTC block not produced after 10 min | Auto-advance to the next block (timeout mechanism) |
| Locked ETH block not produced | Likewise advance (+1) |
| A station significantly deviates from its historical distribution | Health test raises an alert; temporarily exclude that station (≥3 stations still usable) |
| Operator fails to submit N within the window | Contract rejects finalization; this round marked invalid, auto-resuming next round |

---

## 10. Lottery Capacity Estimate

Entropy per 10-minute window ≈ **55 bits**; daily entropy ≈ **9,925 bits (0.115 bps)**:

| Participants K | Minimum Entropy Required | Available in 10 min |
|----------------|:------------------------:|:------------------:|
| 10 | 3.3 bits | 55 bits ✅ |
| 1,000 | 10 bits | 55 bits ✅ |
| 10,000 | 13.3 bits | 55 bits ✅ |
| 100,000 | 16.6 bits | 55 bits ✅ |

**The capacity margin for lotteries at the ten-thousand-person scale exceeds 4000×, so capacity is sufficient.** High-frequency scenarios (thousands of calls per second) require a ChaCha20 stream cipher for expansion.

---

## 11. Four-Step Public Verification (Runnable on a Phone)

1. **Download**: Fetch this round's raw count rates for the 4 stations from the NMDB public API;
2. **Recompute**: Locally apply the §4 algorithm (detrending + LSB) → obtain N;
3. **Read**: From BTC/ETH block explorers, read the two block hashes locked for this round;
4. **Compare**: Offline, recompute `Output = SHA-256( SHA-256(N⊕BTC⊕ETH) ‖ "NeutronChain-RNG-v1" )` and compare it with the value published by the website/contract.

If they match → this round's result is trustworthy; if not → submit evidence and raise a challenge.

---

## 12. Appendix: Worked Example (Illustrative)

```
Assumptions:
  N      = 10110011... (256-bit, from 4-station detrended LSB)
  B_btc  = 11001010... (BTC locked block hash)
  B_eth  = 01101100... (ETH locked block hash)

Step 1 XOR:
  N      = 10110011
  B_btc  = 11001010
  ───────────── XOR
          = 01111001
          ⊕ B_eth (01101100)
          = 00010101...

Step 2 Hash:
  Seed   = SHA-256(00010101...) = a3f5c8d2...

Step 3 Output:
  Output = SHA-256(a3f5c8d2... ‖ "NeutronChain-RNG-v1")
         = 7f2a9b1c...  ← the final published 256-bit random number
```

---

## 13. Version and Revision History

| Version | Date | Changes |
|---------|------|---------|
| v1.0 | 2026-09-04 | Initial specification: four-station entropy source + dual-chain XOR anchoring + differential detrending |

---

## Disclaimer

This protocol is a technical specification draft and constitutes no security guarantee whatsoever. Before actual deployment, the following should be performed:

- Third-party security audit (cryptography + smart contract)
- Long-term entropy-source health testing (NIST SP 800-90B)
- Operational drills of the multi-signature collection architecture

Any use based on this protocol is undertaken at the user's own risk.
