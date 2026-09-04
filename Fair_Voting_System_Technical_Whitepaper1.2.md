

**Subtitle**: A Coercion-Resistant Anonymous Voting System Based on Dual-Password Mechanism + Verifiable Public Randomness + Trust-Minimized Settlement
**Version**: v2.1 (integrating the dual-password mechanism, verifiable public randomness, and trust-minimized settlement layer)
**Author**: ethanycq
**Status**: Technical solution specification (for review / project initiation / compliance review)

---

## Executive Summary

This whitepaper proposes a **coercion-resistant anonymous voting system** built on two pillars:

1. **Dual-password mechanism** — Each voter holds one **true password** and any number of **anti-coercion passwords**. The two types of passwords are **fully indistinguishable** in length, input method, and page feedback, yet the system records different content: the true password casts the voter's real choice, while an anti-coercion password casts an **abstention**.
2. **Verifiable Public Randomness (VPR)** — All sources of randomness in the system (random ballot tokens, threshold key shares, time-bucket offsets, audit-chain genesis salt) are driven by **unpredictable, post-verifiable** true random numbers, eliminating both "insider reconstruction of the identity mapping" and "post-hoc forgery of the audit chain."
3. **Trust-minimized settlement layer** — Through homomorphic encryption (no single ballot is ever decrypted), distributed key generation (a complete private key never exists), and a time-lock fallback (mathematically guaranteed output), the assumption of "trusting that the authority will not misbehave" is transformed into "misbehavior carries extremely high cost, extremely low gain, and is certain to be discovered."

The security foundation rests on four principles:

1. **True-password priority**: Once a real ballot is cast, it is permanently固化 (finalized); no subsequent coercion ballot can override it.
2. **Indistinguishability**: Even if the coercer monitors the entire voting process, they cannot tell which type of password the voter used.
3. **Verifiable randomness**: All random numbers are anchored to public entropy sources and can be independently replayed and verified by third parties; the system cannot forge them.
4. **Making coercion unprofitable**: All the coercer can obtain from the victim is an "abstention," not the option they desired.
5. **Trust-minimized settlement**: Trust no single authority; use mathematics to make "misbehavior" meaningless in gain, harmless in consequence, and verifiable in action.

This solution is intended for **internal organizational decision voting** (union elections, board/council votes, homeowners' association committees, industry associations, academic reviews, etc.), preventing management or those in power from interfering with individual voters.

---

## 1. Design Goals and Threat Model

### 1.1 Design Goals

| Goal | Description |
| --- | --- |
| Coercion resistance | Even when monitored or pressured, the voter can ensure the final tally does not reflect the coercer's will |
| Anonymity | No one at any stage can link a ballot to a voter |
| Verifiability | Results and randomness can be verified by third parties as unaltered, without revealing individual choices |
| Unreceiptability | A voter cannot prove to others what they voted, eliminating vote selling |
| Usability | Ordinary voters can use it correctly without cryptographic knowledge |

### 1.2 Attacker Capability Boundary (Important Prerequisite)

This solution **explicitly assumes** the attacker (the powerful party within the organization) possesses the following capabilities:

- ✅ Physically monitoring the voter's actions (standing behind them, screen recording, demanding screenshots)
- ✅ Forcing the voter to vote in person or to vote repeatedly
- ✅ Verbal pressure, threats, inducements
- ✅ Controlling the organization's internal IT policies

At the same time, it is **explicitly assumed the attacker does NOT possess** the following capabilities:

- ❌ Access to application servers, databases, or code repositories
- ❌ Bribing or coercing a **threshold number** of escrow agents
- ❌ Unlawfully detaining or committing violence against voters
- ❌ Predicting or manipulating public entropy sources (external markets / blockchain / lotteries, etc.)

> **All security of this solution rests on the premise that "the server and database are independent of the attacker."** If this premise is broken, the threat model must be re-evaluated. During review, one should first confirm whether this premise can be guaranteed by the organization's charter or a third-party escrow agreement.

### 1.3 Explicitly Non-Defended Threats

Honest disclosure: the following scenarios **cannot** be defended by this solution and require institutional and legal remedies.

1. **Forced silence**: The coercer completely isolates the voter throughout the voting period, preventing them from casting a real ballot at any safe moment. In this case the ballot result is abstention.
2. **Election cancelled or results not recognized**: The coercer declares the vote invalid or ignores the results.
3. **Voluntary bribery**: The voter voluntarily uses the true password to cast the option the coercer desires. The system does not prevent this, nor should it.
4. **Violent physical harm**: Physical injury inflicted by the coercer. Technology cannot prevent violence, but violence also **cannot force out the true password** — when interrogated, the voter gives any anti-coercion password, which the other party likewise cannot verify as genuine (see 5.3). Thus "forcing out the true password" is technically covered; the only undefendable aspect is the violent harm itself, which requires legal remedy.

> **Note: Strategic voting is NOT a "non-defended threat" but should be respected.** A voter casting a non-first-preference option based on their own preferences and judgment (e.g., voting for the second-best candidate to prevent the most disliked one from winning) is an **expression of free will and is itself the will of the people**. This solution only defends against "being forced," not "acting autonomously" — the system does not interfere, nor should it, with strategic voting, nor treat it as manipulation or anomaly.

---

## 2. Core Mechanism: The Dual-Password System

### 2.1 The Two Passwords

| | True Password | Anti-Coercion Password |
| --- | --- | --- |
| Usage scenario | Voting in a trusted environment on a trusted device | When coerced, monitored, or using an untrusted device |
| System records | The voter's actual choice | Abstention |
| Page feedback | "Your ballot has been successfully recorded." | **Exactly the same** |
| Quantity | 1 | **Unlimited; can be added at any time** |

**Indistinguishability requirements** (hard constraints):

- All passwords have **exactly the same length** (preventing the coercer from distinguishing by observing the number of masking dots)
- Input method, error messages, login latency, and post-vote redirection flow are **completely identical**
- Pages, screenshots, and screen recordings **contain no element** indicating the current password type

### 2.2 The True-Password-Priority Principle

**Once a real ballot is cast, it is permanently finalized; an anti-coercion ballot cannot override it.**

```
If this voter has a true-password voting record → use the [last true-password vote]
Otherwise, if an anti-coercion voting record exists    → record as [abstention]
Otherwise                                             → record as [did not vote]
```

This principle is the cornerstone of coercion resistance for the entire solution. It means:

- If the coercer forces the voter to "vote again," the result is still abstention; the real ballot is unaffected
- The voter need not wait until the deadline; **the decisive point is "when the real ballot is safely cast," not "the last operation"**

### 2.3 Can Add, Cannot Delete

| Operation | Permission | Design rationale |
| --- | --- | --- |
| Add an anti-coercion password | User self-service at any time | A password forced into being added is still an anti-coercion password — harmless |
| Delete/modify an anti-coercion password | Must be done in person at a designated third-party institution, fully recorded | Prevent the coercer from forcing the voter to remove protection |
| Modify the true password | Must be done in person at a designated third-party institution | Equivalent protection |

**The asymmetric design is the key**: the coercer can force you to "add," but adding any number is meaningless (all are abstentions); the coercer cannot force you to "delete," because you yourself cannot delete it either.

### 2.4 Indistinguishable Feedback

Regardless of the voter's state, the voting page **always and only** displays:

> **"Your ballot has been successfully recorded."**

The page **does NOT display**: voting history, current ballot content, days remaining, whether the deadline has passed, or number of modifications.

Therefore, **screenshots and screen recordings have no verification value whatsoever** — this is the key to dismantling "vote buying": the buyer cannot verify whether the voter fulfilled their side, and a rational vote-buyer will abandon such an investment.

### 2.5 Advance Setup Window

The setup of anti-coercion passwords must be completed **six months before the voting system goes live**.

Rationale: At this point the voting topics have not yet been announced, so no one has a motive to monitor the setup process. The highest form of security awareness is to complete defensive deployment before the threat even exists.

**Setup-period requirements**:

- Guided completion through personal trusted channels (personal phone / personal email, not company channels)
- Immediately conduct a simulated drill after setup
- Drill results are **fed back only to the individual through personal trusted channels**, not aggregated, published, or retained

### 2.6 Unified Public Voting Window

**Cancel random deadlines; adopt a unified public voting window** (e.g., 30 days, identical for everyone).

Rationale:

- The coercer knowing the deadline is **harmless** — they can neither override the real ballot nor force out the true password
- Random deadlines create two-way information asymmetry; voters may unknowingly miss the vote (self-harm)
- A unified window is simple and transparent, facilitating operational reminders and user planning

**Post-deadline behavior**: Set a 24-hour grace period (votes during the grace period are genuinely recorded); after the grace period, return "success" but do not record, preventing the coercer from detecting system state changes through probing.

---

## 3. Core Mechanism: Verifiable Public Randomness (VPR)

### 3.1 Why True Random Numbers Are Needed

The "random ballot token," "random time-bucket offset," "threshold key shares," and "audit-chain salt" in the original design all depend on random numbers. If software pseudo-random numbers (PRNG) are used, two fatal risks arise:

1. **Predictability → anonymity collapses**: If the server is breached or an insider learns the PRNG seed, they can replay the generation sequence and reconstruct the "token → user" mapping, instantly destroying anonymity.
2. **Unauditability → verifiability suffers**: A pseudo-random number is "pulled out of thin air" by the system; there is no way to prove to a third party that "this random sequence was not manipulated," shaking the trust foundation of the audit chain.

Therefore we introduce **Verifiable Public Randomness (VPR)**, which must simultaneously satisfy two hard requirements:

- **Unpredictable**: During the voting period, no one (including system administrators or attackers) can predict the random number's value;
- **Post-verifiable**: After generation, any third party can independently replay the computation, proving it genuinely came from the declared public entropy source and was not tampered with.

### 3.2 Method: Commit–Reveal + Multi-Source Public Entropy Combination

The core idea of VPR is: **entrust randomness to external, uncontrollable public events beyond the system's control**. Completed in four steps:

```
Select entropy sources → Pre-commit → Reveal at maturity → Combine & verify
```

#### Step 1: Select Entropy Sources (announced before voting begins)

Entropy sources must be "future, public, independently uncontrollable" events. It is recommended to select **3–5 independent entropy sources** to prevent a single source from being manipulated:

| Example entropy source | Value | Source of uncontrollability |
| --- | --- | --- |
| Benchmark indices of 12 countries | Precise to 0.01 | Open market, multi-party game |
| Daily precipitation in 12 national capitals | Precise to 0.1 | Natural phenomenon, completely uncontrollable |

| Country | Locally recognized benchmark index (most influential) |
| --- | --- |
| China | CSI 300 |
| United States | S&P 500 |
| Russia | MOEX Russia Index |
| United Kingdom | FTSE 100 |
| Germany | DAX 40 |
| South Korea | KOSPI |
| Saudi Arabia | TASI (Tadawul All Share) |
| France | CAC 40 |
| Japan | Nikkei 225 |
| United Arab Emirates | ADX General (Abu Dhabi) |
| Israel | TA-35 |
| India | BSE Sensex |

#### Step 2: Pre-commit (Commit, before voting begins)

Before voting begins, the system publishes an "event identifier + commitment hash" for each entropy source:

```
Commit_i = SHA-256( event_desc_i  ‖  source_i  ‖  announced_at )
```

- `event_desc_i`: Description of the "specific future event" for this entropy source (e.g., "Bitcoin block #800,000")
- Once published, the commitment hash is locked and cannot be changed

#### Step 3: Reveal at Maturity (Reveal, after the event occurs)

After the entropy-source event occurs at the scheduled time, its value is **publicly queryable**; anyone can independently obtain the raw data (via blockchain explorers, market-data APIs, official lottery announcements, etc.).

#### Step 4: Combine & Verify

The entropy values are combined according to the published rules into a true-random seed:

```
R = SHA-256( entropy_1  ‖  entropy_2  ‖  …  ‖  entropy_k  ‖  domain_sep )
```

- `domain_sep`: A domain separator (e.g., `"ballot-token"` / `"threshold-key"` / `"audit-chain"`); one set of entropy sources can derive random numbers for multiple purposes without interference
- **Verifiable**: Any third party can replay this hash computation with the public entropy values to verify R's authenticity
- **Ungforgable**: The entropy sources are external public events that the system cannot alter; the commitment hash comes first, so a favorable value cannot be "filled in" after the fact

### 3.3 Demonstrating the Two Properties

| Property | How it is satisfied |
| --- | --- |
| **Unpredictable** | Entropy sources are "future public events" (next block, tomorrow's close, next lottery draw); their values do not yet exist during voting, so no one can predict them. Multi-source combination further amplifies unpredictability |
| **Post-verifiable** | Entropy values become publicly queryable afterward; the combination algorithm is a public hash; anyone can replay to verify. The commitment hash locks the "entropy source list," preventing mid-course replacement |

### 3.4 Four Application Points in the Voting System

| Application point | Original (pseudo-random) | After VPR retrofit |
| --- | --- | --- |
| **Random token generation** | `random()` + salt | Use the true-random number derived from `R(ballot-token)` as the token and salt, eliminating "predicting PRNG to reconstruct identity mapping" |
| **Threshold key-share generation** | Key vault generation | Each custodian mixes in `R(threshold-key)` when generating shares, ensuring unpredictability |
| **Timestamp bucket offset** | Fixed offset | Use `R(time-offset)` for a random offset, breaking the time–identity linkage |
| **Audit-chain genesis salt** | System-defined salt | Genesis salt = `R(audit-chain)`, anchored to a public entropy source; third parties can verify the chain was not "recalculated after the fact" |

### 3.5 Timeline

| Time | Action |
| --- | --- |
| T-7 days (before voting) | Publish entropy-source list + commitment hashes for each source |
| T-1 day (before voting) | All entropy events have occurred; reveal entropy values, generate the true-random seed R, lock system randomness |
| T-0 onward (voting) | Derive random tokens, time offsets, and audit salt from R; generate threshold key shares |
| After voting ends | Third parties replay-verify R using public entropy sources → then verify the audit chain and randomness were not manipulated |

### 3.6 Optional Enhancements

- **Posterior anchoring**: The **last block** of the audit chain additionally anchors a "public event after the voting deadline" (e.g., an index close on the day after the deadline), making the entire chain **impossible to forge in advance** during the voting period — it can only be closed after that event occurs.
- **Threshold joint generation**: For higher strength, each custodian can contribute an entropy share + commit–reveal; finally `R = XOR(all shares)`. Any two parties together can generate it, but neither can predict it alone — isomorphic to the threshold-key idea.

---

## 4. System Architecture

### 4.1 Layered Architecture

```
┌─────────────────────────────────────────────┐
│  Auth Layer   Independent password auth     │
│              (SSO single sign-on forbidden) │
├─────────────────────────────────────────────┤
│  Voting Layer Client-side encryption →      │
│              server stores ciphertext only  │
├─────────────────────────────────────────────┤
│  Storage Layer True-random tokens +         │
│              encrypted ballots + hash chain │
├─────────────────────────────────────────────┤
│  Random Layer  VPR verifiable public        │
│              randomness (cuts across all)   │
├─────────────────────────────────────────────┤
│  Settlement Layer Homomorphic tally +       │
│              threshold decryption +         │
│              time-lock fallback             │
└─────────────────────────────────────────────┘
```

> The random layer is a new horizontal infrastructure: every randomness touchpoint in auth, voting, storage, and settlement is uniformly supplied by VPR, avoiding any "stray pseudo-random number" from becoming an anonymity breach.

### 4.2 Authentication Layer

**Independent password authentication is mandatory; SSO / QR scan / fingerprint / passwordless login are prohibited.**

In reality, the coercer often also controls the organization's IT policies. If the vote is required to "unify on an enterprise WeChat login," the dual-password mechanism has nowhere to reside and the entire solution fails.

Therefore the authentication method must be **locked by an external contract** (escrow agreement / election charter), explicitly stating:

- Independent passwords must be used
- Each ballot submission requires re-authentication
- Sessions time out after 5 minutes
- **No party may unilaterally change the authentication method**

This is an **institutional constraint**, not a code implementation.

### 4.3 Voting Layer: Client-Side Encryption

**Key design: Ballots are encrypted locally on the client (browser / app) before upload; the server stores only ciphertext.**

```
key      = KDF(password entered by user)   # true & anti-coercion passwords derive different keys
plaintext = actual choice (true) or abstention (anti-coercion)
upload   = encrypt(plaintext, key)
```

The server **cannot itself determine** whether a given ballot came from a true password or an anti-coercion password. This design eliminates three categories of risk at once:

- Insiders querying the database cannot learn who voted what
- No "I was coerced" flag field exists in the database
- If the server is breached, only ciphertext is leaked

### 4.4 Storage Layer

- The ballot table **does not store user IDs**, only the **true-random token** derived from VPR
- The routing mapping (user → token) is generated temporarily during the session, **destroyed when the session ends**, and never persisted
- Eligibility verification uses an independent list, recording only "someone is eligible," not "who corresponds to which token"
- Token generation concatenates a VPR-derived salt to prevent reverse inference

### 4.5 Settlement Layer: Trust-Minimized Key Escrow

**Core problem**: The decryption key is held in shares by "three institutions" — the only remaining **human-trust环节** in the whole solution. Once an institution is bribed, colludes, or is coerced, both anonymity and result integrity may collapse. Can mathematics replace this human element?

**Honest conclusion: It cannot be completely eliminated, but it can be reshaped to be nearly harmless.** A private key is a secret, and a secret must reside on some physical medium. Cryptography can split a secret into N pieces, require K to reconstruct, and guarantee the complete form never existed — but the question of "who holds which piece" mathematics cannot answer; it is forever a human arrangement.

Therefore the goal of trust-minimization is not to "eliminate institutions," but three things: **① reduce the trust requirement from "believe they won't misbehave" to "their misbehavior is certain to be discovered"; ② reduce the failure consequence from "key leak = everyone exposed" to "key leak = only totals known early"; ③ change "when to decrypt" from a human agreement to a mathematical decision.**

#### 4.5.1 Five Layers of Mathematical Hardening

| Layer | Technology | What it solves | What it still cannot solve |
| --- | --- | --- | --- |
| L1 Distributed key generation | Pedersen DKG / FROST | The complete private key never exists at any moment or on any device, eliminating the single point of failure at "cut time" | Does not reduce the number of share holders requiring trust |
| L2 Fully homomorphic encryption | Paillier / BFV / CKKS | No single ballot is ever decrypted; only the sum is decrypted | Key still held by people |
| L3 Zero-knowledge proof + bulletin board | zk-SNARK / Bulletproofs | Anyone can independently verify tally correctness; cheating detectable & attributable | Does not prevent decryption, only makes cheating detectable |
| L4 Verifiable delay function | Wesolowski / Pietrzak VDF | Decryption moment decided by mathematics, not human agreement | Cannot prevent unilateral decryption |
| L5 Accountable threshold | Accountable threshold | After cheating, mathematically identify which shares participated | Post-hoc accountability, not pre-emptive prevention |

#### 4.5.2 L2 Fully Homomorphic Encryption (the Best Cost-Effectiveness Move)

Taking Paillier as an example, it has additive homomorphism: `E(m₁) × E(m₂) = E(m₁ + m₂)`.

Therefore tallying **fundamentally does not need to decrypt a single ballot** — multiply all ballot ciphertexts under modular arithmetic to directly obtain `E(total A)`, `E(total B)` … and only decrypt those totals. **From encryption to destruction, no single ballot is ever decrypted.**

| Consequence of key leak | Traditional threshold (decrypt per ballot) | Homomorphic tally |
| --- | --- | --- |
| Leak scope | Everyone's choices exposed — catastrophic | Only final totals leaked early (and that number was going to be published anyway) |

This layer does not change the trust structure, but downgrades the consequence of human-link failure from **catastrophe to mild** — even if all institutions collude, they can only learn early a number that was going to be public anyway.

> **Connection to the dual-password mechanism (key)**: Homomorphic tally cannot merge multiple ballots by user, but this solution's dual-password mechanism fits naturally — **anti-coercion passwords always cast "abstention," and abstention ballots are excluded during tallying (see 7.1); the real-option ballot cast with the true password is retained**. Thus "true-password priority" holds automatically under homomorphic tally: no matter how many abstentions are coerced, all abstentions are excluded; only the real ballot cast with the true password counts.
>
> Trade-off: By default the true password is **cast-once-and-finalized immediately** (consistent with the "real ballot lands and is finalized" principle); if changing a vote is needed, the client generates a homomorphic "cancellation ballot" on re-vote to cancel the old one (optional enhancement, not required).

#### 4.5.3 L1 Distributed Key Generation (DKG)

Traditional threshold schemes first generate a complete private key and then split it — at "the moment of cutting," the complete key exists, a single point of failure. **DKG lets N parties each generate a random piece and jointly produce a public key through an interactive protocol; the private key never exists in complete form on any device.**

Share generation must mix in VPR true randomness (see 3.4), ensuring shares are unpredictable and non-reproducible.

#### 4.5.4 L4 Verifiable Delay Function (Time Lock) — Key Trade-off

A VDF requires the decryptor to perform T consecutive operations that cannot be parallelized, making the key "automatically solvable at the appointed time."

But one must be clear about a **counter-intuitive trade-off** — the time lock and the human threshold defend against **opposite** things:

| | Human threshold | Time-lock VDF |
| --- | --- | --- |
| Prevents "unilateral偷偷 decryption" | ✅ Requires collusion to decrypt | ❌ Once time is up, anyone can decrypt |
| Prevents "institutional collective absence" (liveness) | ❌ If they don't show up, it deadlocks | ✅ Mathematics guarantees it can be solved |

The time lock removes the human element, but **also removes the ability to "prevent unilateral action."** Therefore it cannot replace the threshold; it can only combine and complement it:

- **Normal path**: Threshold decryption — fast, controllable, prevents unilateral action
- **Fallback path**: If T days after the deadline the threshold parties are collectively unreachable or coerced into not showing up, the time lock auto-engages — preventing denial of service

T must be set far longer than the normal tallying time (e.g., normal = 1 day, T = 30 days), ensuring the fallback never triggers before the normal path.

#### 4.5.5 L3 Zero-Knowledge Proof + Public Bulletin Board & L5 Accountable Threshold

Each ballot carries a ZKP proving "I encrypted a valid option (one of A/B/C/abstention)" without revealing which; the tallying process also carries a ZKP proving "I correctly summed all ciphertexts without tampering." Anyone (candidates, external auditors) can independently verify the entire process using public data.

After cheating occurs, the accountable threshold can mathematically identify the set of shares that participated, enabling accountability.

#### 4.5.6 Non-Cryptographic Supplement: Let Opposing Stakeholders Each Hold a Share

Rather than seeking "neutral institutions," it is better to let **parties with opposing interests each hold a share**: one to the union, one to management, one to external audit, one to employee representatives …

For any party to cheat, it must persuade several other parties with fundamentally opposed stances to cooperate simultaneously. This relies on game theory rather than cryptography, but is often more reliable in practice — opposing-interest parties naturally lack incentive to collude.

It is recommended to raise the threshold scale from 2-of-3 to **5-of-9**, including at least two opposing-interest stakeholders.

#### 4.5.7 Recommended Architecture and Trust Model

```
Ballot upload: FHE encryption + ZKP validity proof + write to public bulletin board
   ↓
Tally phase: Homomorphic sum (no single ballot ever decrypted) → only [encrypted totals]
   ↓
Decrypt phase: K-of-N threshold (DKG, complete key never existed) → decrypt only totals
   ↓
Fallback: If threshold not completed T days after deadline → VDF time lock auto-solvable
   ↓
Verify: Anyone can independently verify the entire process with public data
```

Under this architecture, human trust is compressed to: **K of N parties must collude to decrypt early; even full collusion only leaks totals (which were going to be public anyway); any cheating is caught on the spot by zero-knowledge verification on the public bulletin board; if institutions are collectively absent, the time lock guarantees a result.**

| Trust model | Before (three-institution threshold) | After (trust-minimized) |
| --- | --- | --- |
| Trust basis | Trust institutions won't misbehave | Misbehavior is certain to be discovered |
| Consequence of key leak | All voting content exposed | Only totals leaked early |
| Did a complete key ever exist | Yes (at the cutting moment) | No (guaranteed by DKG) |
| Is any single ballot decrypted | Yes | No (homomorphic tally) |
| Who decides decryption moment | Human agreement | Mathematics (VDF fallback) |
| What if institutions absent | Tally deadlocks | Time lock produces result |
| Can it be attributed | No | Participating shares can be identified |

**Conclusion: This is not "trusting institutions," but placing institutions in a structure where "misbehavior carries extremely high cost, extremely low gain, and is certain to be discovered."**

---

## 5. Security Analysis

### 5.1 Coercion Scenarios

| Coercer's method | System response | Result |
| --- | --- | --- |
| Standing behind to monitor voting | Page shows only "success," not the option | Cannot confirm what was voted |
| Demanding screenshot/screen recording | Screenshot contains no useful information | No verification value |
| Demanding "vote A with your true password" | Masked input; cannot verify password type; voter uses anti-coercion password | Recorded as abstention/ignored |
| Forcing repeated voting | Real ballot already finalized; subsequent ballots cannot override | Coercion ineffective |
| Forcing disclosure of password | Voter gives anti-coercion password; no verification interface, coercer cannot confirm authenticity | True password not leaked |
| Using that password to vote | System recognizes it as anti-coercion | Abstention/ignored |
| Forcing deletion of anti-coercion password | User has no self-service deletion; must go to third-party institution | Protection cannot be removed |
| Forcing voting after deadline | Page still returns "success" | Actually not recorded |
| Monitoring until deadline | If real ballot was cast early, unaffected; otherwise abstention | See 1.3 |
| Keylogger | True password entered only on trusted devices; untrusted devices see only anti-coercion passwords | Everything logged is invalid |
| Cracking randomness to reconstruct identity | Randomness from VPR, anchored to public entropy, unpredictable | Cannot reverse token→user mapping |

### 5.2 Vote-Buying Scenarios

When outside actors spread money to buy votes en masse, because the system **provides no verifiable receipt** (no history, no options, no useful screenshots), the buyer cannot verify whether the voter fulfilled their side.

**A transaction that cannot be verified cannot scale** — a rational vote-buyer will abandon such an investment.

### 5.3 Violence Cannot Extract the True Password (Not an Undefended Scenario)

One key conclusion needs clarification: **violent interrogation does NOT constitute a security vulnerability in this solution.** Violence and ordinary coercion are fully equivalent on the dimension of "can the true password be forced out" — neither has a means of verification.

If a voter is violently interrogated for the true password, they can同样 give **any anti-coercion password**. Because:

- The system provides no interface to "verify whether a password is the true password"
- The two password types behave identically

The coercer **has no oracle**; no matter how much pressure is applied, they cannot verify the authenticity of the obtained password. The true password remains with the voter.

Therefore, **"violently forcing out the true password" is NOT listed in the "non-defended threats" of section 1.3** — it is within the technical coverage of this solution. The only thing beyond the technical scope is the physical injury caused by violence itself (a matter for criminal-law remedy, not cryptography).

**Usage discipline**: The true password is entered only on trusted devices (personal phone, home computer); on company or any untrusted device, always enter only the anti-coercion password.

---

## 6. Technical Specifications

### 6.1 Data Structures

**Ballot table (encrypted storage)**

| Field | Description |
| --- | --- |
| `ballot_token` | True-random token (VPR-derived, salted) |
| `encrypted_ballot` | Client-encrypted ballot ciphertext (FHE) |
| `zk_proof` | Zero-knowledge proof of ballot validity |
| `submitted_at_bucket` | Timestamp (bucketed, e.g., precise to hour + VPR random offset) |

**Note**: The database **does NOT have** an `is_coerced` field. The server does not know, nor need to know, whether a ballot came from coercion.

### 6.2 Voting and Settlement Logic

**Voting interface (server-side)**

```python
def submit_vote(encrypted_ballot, zk_proof, session):
    # Verify ballot validity but do NOT decrypt (zero-knowledge proof)
    assert verify_zk(zk_proof, encrypted_ballot)
    store(session.ballot_token, encrypted_ballot, now_bucket())
    publish_to_bulletin_board(hash(encrypted_ballot))   # write to public bulletin board
    log(session.ballot_token)          # record only the token, not the user ID
    return {"message": "Your ballot has been successfully recorded."}   # same message in all cases
```

**Settlement logic (homomorphic tally; no single ballot ever decrypted)**

```python
# 1. Homomorphic sum: accumulate directly over ciphertext, no single ballot decrypted
encrypted_tally = homomorphic_sum(all_encrypted_ballots)

# 2. Threshold decryption: decrypt only [the totals]
tally = threshold_decrypt(encrypted_tally, shares)   # requires K-of-N shares

# 3. Output and destroy
emit(tally)          # output only the totals per option
destroy(shares, intermediate_data)
```

> Fallback path: If threshold decryption is not completed T days after the deadline, activate the VDF time lock; anyone can continuously compute to obtain the decryption key.

### 6.3 Logging Rules

- Operation logs **do not record user IDs**, only true-random session tokens
- Do not record which password type was used or the input content
- Timestamps are **bucketed + VPR random offset**, breaking temporal linkage to the ballot table

> If logs saying "user X voted at time T" coexist with records of "(token, T, ballot)," a join on timestamp reconstructs the identity mapping and anonymity collapses. This is the most easily overlooked implementation trap.

### 6.4 Hash Audit Chain

Each ballot and its true-random token are jointly hashed into a chain. The genesis salt is VPR-derived (optional posterior anchoring, see 3.6):

```
H_0 = SHA-256( R(audit-chain) )
H_n = Hash( H_{n-1} ‖ ballot_token ‖ encrypted_ballot )
```

- Routine backends **do not display details**
- Third-party institutions can verify hash-chain integrity before settlement, confirming data was not tampered with
- Because the genesis salt is anchored to a public entropy source, third parties can further verify the entire chain **was not forged by post-hoc recalculation**

### 6.5 Tallying Process

1. Voting window closes; enter a 24-hour grace period
2. **Homomorphic tally**: accumulate directly over ciphertext to obtain encrypted totals per option (no single ballot decrypted)
3. **Threshold decryption**: K-of-N (recommend 5-of-9) share holders inject shares; decrypt only totals
4. Generate and publish a **zero-knowledge proof** of the tallying process for anyone to independently verify
5. Immediately destroy key shares and all intermediate data after output
6. Issue a tally-process compliance certificate (recording no individual information)
7. Third parties replay-verify VPR randomness using public entropy sources → verify audit-chain integrity
8. **Fallback**: If step 3 is not completed within T days (recommend 30 days) after the deadline, the VDF time lock auto-engages

**The system shall never provide a detail-export function.** No form of `SELECT * FROM ballots` query interface shall be implemented.

### 6.6 Anomaly Monitoring and Participation-Rate Threshold

**(1) Minimum participation-rate threshold — fallback for "forced silence"**

- Set a minimum valid participation-rate threshold (e.g., 50%): if actual votes ÷ eligible population is below the threshold, the result is invalid and a re-vote is scheduled.
- Value: If the coercer forces mass abstentions, or completely isolates voters preventing them from casting real ballots, the participation rate will be driven below the threshold, making the suppression detectable and the election voidable/re-votable. This is the only technical fallback for "forced silence."
- The threshold and re-vote rules must be written into the election charter and locked before voting.

**(2) Anomaly alerts — turning "unprofitability" into an auditable metric**

- Continuously monitor participation rate, abstention rate, and other metrics; establish a historical baseline.
- When any metric deviates from the historical baseline by more than **2σ** (two standard deviations), automatically trigger a third-party institution review.
- The baseline and threshold methodology must be published and locked before voting to avoid post-hoc disputes.

---

## 7. Prerequisites for Implementation

The following three are **hard prerequisites**; if any one is missing, the solution's security does not hold:

### 7.1 Election Rules Locked (Critical)

**It must be explicitly stipulated that abstention ballots are excluded from both numerator and denominator; counting framings such as "majority of all members" that equate abstention with opposition are prohibited.**

Rationale: Under a "majority of all members approve" rule, abstention is effectively equivalent to opposition. If the coercer's desired outcome is precisely "not passed," then abstentions passively help the coercer achieve their goal.

This rule must be written into the election charter, backed by an external institution, and **no party may interpret or change it after the fact**. At the same time, the minimum participation-rate threshold and anomaly-alert thresholds from section 6.6 should also be written into the charter and locked.

### 7.2 Authentication Method Locked

Independent-password authentication (SSO / QR scan / biometrics disabled) must be fixed by an external contract; no party may unilaterally change it.

### 7.3 Server Independence Guarantee

Application servers, databases, and code repositories must be deployed in an environment inaccessible to the coercer and supervised by a third-party institution. The escrow agreement must specify permission boundaries and audit clauses.

### 7.4 Entropy-Source List Locked (New)

The VPR entropy-source list, commitment hashes, and combination algorithm must be **published and locked before voting**; no party may change the entropy sources or combination rules after the fact — this is the prerequisite for "verifiable public randomness" to be credible.

---

## 8. Implementation Timeline

| Phase | Time | Task |
| --- | --- | --- |
| T-6 months | Setup window | All members complete true- and anti-coercion-password setup; simulated drill |
| T-5 months | Institution signing & DKG | Determine N escrow institutions (recommend 9, including opposing stakeholders); perform distributed key generation (complete private key never exists) |
| T-7 days | Entropy commitment | Publish VPR entropy-source list and commitment hashes |
| T-1 month | Rules publication | Publish election charter, counting framing, voting window |
| T-1 day | Entropy reveal | Entropy events occur; reveal and generate the true-random seed R |
| T-0 | Voting opens | Strong reminder via personal trusted channels: vote with the true password as early as possible |
| T+30 days | Window closes | 24-hour grace period |
| T+31 days | Tallying | Homomorphic tally → threshold-decrypt totals → publish ZKP → destroy intermediate data |
| T+31~T+61 days | Fallback window | If threshold incomplete, VDF time lock auto-engages |
| T+32 days onward | Public audit | Third parties replay-verify VPR randomness, audit chain, and tally ZKP |
| Ongoing | Operations | Push security reminders and operational discipline quarterly |

---

## 9. Operations and User Discipline

### 9.1 Vote Early (Core Operational Action)

Once voting opens, push a reminder through **personal trusted channels** (personal SMS / personal email, **not company channels**):

> "Voting is now open. Please use your true password to vote as early as possible on a trusted device — once your real ballot is cast, it is locked, and no subsequent action can override it."

Rationale: The earlier the real ballot lands, the smaller the coercer's available window.

### 9.2 Personal Trusted-Channel Verification

After voting, feed back through personal trusted channels:

> "You have successfully cast a ballot. Current real-ballot status: recorded / not recorded."

This solves the usability problem of "the user cannot confirm which password they used":

- The coercer is not on the voter's personal device, so it **will not leak**
- The voter can confirm in a safe environment whether the real ballot took effect; if not, they can still vote again

### 9.3 User Discipline (Must Be Repeatedly Communicated)

1. **Enter the true password only on trusted devices**; on company or public devices, always use the anti-coercion password
2. When asked to vote, **always use the anti-coercion password** — the real ballot is already locked anyway, or the result is harmless
3. When forced to disclose a password, give any anti-coercion password
4. Never enter the true password on untrusted devices (defense against keyloggers)

### 9.4 Regular Reminders

Push a security reminder once per quarter, containing the above discipline and the current password status (not the password content itself).

---

## 10. Limitations

Honest disclosure of the solution's boundaries:

| Limitation | Description | Mitigation |
| --- | --- | --- |
| **Forced silence** | The coercer isolates the voter throughout the voting period, preventing a real ballot from being cast; result is abstention | Minimum participation-rate threshold (6.6) + institutional & legal remedies; transparent participation rate makes suppression detectable and triggers re-vote |
| **Election cancelled / results not recognized** | The coercer declares it invalid or ignores the results | External & charter constraints |
| **Voluntary bribery** | Voter voluntarily uses true password to cast a specific option | System does not prevent it, nor should it |
| **Violent physical harm** | Violence itself lies beyond the scope of technical defense (criminal-law domain) | Technology cannot prevent violence; but violence also cannot force out the true password, see 5.3 |
| **Forgotten password** | Long disuse may cause confusion | Drills after setup + quarterly reminders + institutional remote recovery channel |
| **Unilateral rule changes** | Coercer modifies authentication method or counting framing | Locked by external contract, see section 7 |
| **Entropy source manipulated** | A single public entropy source manipulated by extreme force | Multi-source combination dilutes single-source risk; entropy list published and locked in advance |
| **Key escrow still requires human arrangement** | Mathematics cannot answer who holds which share | Five-layer hardening + opposing-interest checks and balances, see 4.5 |
| **VDF hardware race** | Attacker's computing power far exceeds expectations and may unlock the time lock early | Set T conservatively with a safety margin; use only as fallback, not the main path |

**This solution protects "that one ballot," not "this entire election."** It can defend against "being forced to support," but not against "being forced into silence" or "the election being cancelled." Crossing that boundary requires institutional design (external contracts, participation-rate transparency, appeal channels), not more code.

---

## 11. Core Principles

> **Do not defend against coercion by "hiding information"; defend by "giving false information."**

With hidden information, the coercer cannot see it, but they can watch you operate. With false information, even while watching you operate, they cannot tell real from fake.

> **Drive the net gain the coercer extracts to zero.**

The output of an anti-coercion password is abstention, not opposition — the ballot the coercer forced you to cast is voided; they got nothing. A coercion that yields no profit is one a rational person will not repeat.

> **The decisive point is "the one time the real ballot is safely cast," not "the last operation."**

The real ballot is finalized upon landing. Therefore the top operational priority is: **guide every voter to vote as early as possible in a trusted environment.**

> **Do not hand randomness to the system itself; hand it to the uncontrollable public world.**

A pseudo-random number is "the system pulling it out of thin air" — predictable and unauditable. A true-random number anchored to public entropy is unpredictable and post-verifiable — this is the common foundation of both anonymity and verifiability.

> **The goal of trust-minimization is not to eliminate people, but to make misbehavior unprofitable.**

Cryptography cannot answer "who holds the shares," but it can achieve: the complete key never existed, no single ballot is ever decrypted, cheating is certain to be discovered, and there is a mathematical fallback for absence. The final form is not "trusting institutions," but placing institutions in a structure where "misbehavior carries extremely high cost, extremely low gain, and is certain to be discovered."

---

## Appendix: Development Checklist

**Must implement**

- [ ] Dual-password authentication; both password types have identical permissions and feedback
- [ ] Client-side local ballot encryption (FHE); server stores only ciphertext
- [ ] Ballot-validity zero-knowledge proof + public bulletin board
- [ ] True-password-priority settlement logic
- [ ] Anti-coercion passwords can be self-added but not self-deleted
- [ ] Page feedback always returns exactly one phrase
- [ ] Logs stripped of user IDs; timestamps bucketed + VPR random offset
- [ ] VPR verifiable public randomness (entropy commitment + reveal + combine & verify)
- [ ] Random tokens, threshold key shares, and audit salt all VPR-derived
- [ ] Hash audit chain (genesis salt anchored to public entropy, optional posterior anchoring)
- [ ] Homomorphic-tally counting (no single ballot ever decrypted)
- [ ] DKG distributed key generation (complete private key never existed)
- [ ] Threshold decryption (recommend 5-of-9); decrypt only totals
- [ ] VDF time-lock fallback (auto-engages T days after deadline)
- [ ] Personal trusted-channel verification push
- [ ] 24-hour grace period
- [ ] Entropy-source list and commitment hashes published and locked in advance
- [ ] Minimum valid participation-rate threshold judgment (below threshold → result invalid, re-vote scheduled)
- [ ] Participation-/abstention-rate anomaly monitoring (deviation >2σ from baseline → auto-trigger third-party review)

**Must NOT implement**

- [ ] Countdown / days-remaining display
- [ ] Voting history query
- [ ] Any form of voting-status提示
- [ ] `SELECT * FROM ballots` or any detail export
- [ ] Coercion-flag fields such as `is_coerced`
- [ ] Password-type verification interface (no feature to "verify whether a password is the true password")
- [ ] SSO / QR-scan / biometric login
- [ ] Software PRNG as a security-critical randomness source (must be replaced with VPR)
- [ ] Random deadlines (use a unified public voting window)
- [ ] Any single-ballot decryption function (tallying decrypts only the sum)

---

*This whitepaper is a technical specification. Before deployment it must undergo legal and compliance review and be adjusted according to applicable laws and the organization's charter. Modules such as homomorphic encryption, zero-knowledge proofs, DKG, and VDF should be implemented by a team with applied-cryptography experience and subjected to third-party security audit.*
