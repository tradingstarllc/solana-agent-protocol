---
sap: "0001"
title: Validation Protocol
authors:
  - MoltLaunch (Colosseum Hackathon 2026)
category: Standard
status: Draft
created: 2026-02-08
---

# MoltLaunch Validation Protocol Specification

**MIP-1: Unified Agent Validation Protocol**
**Version:** 0.1.0
**Status:** Draft
**Created:** 2026-02-08
**Authors:** MoltLaunch Team
**Chain:** Solana (primary), EVM-compatible (via bridge)

---

## Abstract

The MoltLaunch Validation Protocol enables trustless verification of autonomous AI agents across blockchain networks using hardware-anchored identity, DePIN (Decentralized Physical Infrastructure Network) device attestations, STARK zero-knowledge proofs, and on-chain AI scoring on Solana. The protocol defines a unified validation request/response interface that combines identity verification, behavioral analysis, Sybil resistance scoring, and cryptographic proof generation into a single composable primitive. Validation responses are compatible with the ERC-8004 Agent Verification standard, enabling cross-chain interoperability between Solana-native and EVM-based agent ecosystems.

---

## 1. Motivation

### 1.1 The Identity Crisis in Agent Verification

Current agent verification systems rely on wallet-based identity: an agent registers a public key, and all trust signals are anchored to that key. This model is fundamentally broken for three reasons:

1. **Wallets are free.** Creating a new Solana keypair costs zero. An adversary can generate millions of identities per second. Any verification system that anchors trust solely to a wallet is trivially Sybil-attackable — the cost to create a "verified" agent is merely the verification fee.

2. **Public scores create gaming incentives.** When verification scores are fully disclosed (e.g., "this agent scored 78/100"), agents optimize for the scoring function rather than actual quality. Score inflation becomes rational behavior. Threshold proofs (pass/fail) reduce this incentive, but without identity anchoring, a failing agent simply re-registers under a new wallet.

3. **No hardware binding.** Software-only identity means an agent can be copied, forked, or impersonated trivially. There is no way to distinguish between "the original agent running on its creator's infrastructure" and "a copy running on a malicious operator's machine."

### 1.2 The Sybil Cost Problem

The fundamental metric for any identity system is the **cost to create one additional verified identity**. In current systems:

| System | Sybil Cost | Time to Create |
|--------|-----------|----------------|
| New Solana wallet | $0 | <1ms |
| Wallet + SOL stake | ~$1 | ~10s |
| Wallet + social link | ~$5 | ~10min |
| Wallet + KYC (human) | ~$50 | ~24h |

None of these are sufficient for high-stakes agent verification. The MoltLaunch Validation Protocol introduces hardware-anchored identity that raises the Sybil cost to the price of physical hardware — $200-$2,000+ depending on the trust level required.

### 1.3 Why Hardware Identity Matters

Physical devices have properties that software identities do not:

- **Uniqueness:** Each TPM, GPU, and DePIN device has a unique hardware fingerprint that cannot be duplicated without physical access.
- **Cost:** Hardware costs real money. A fleet of 100 DePIN devices costs $20,000+, making large-scale Sybil attacks economically infeasible.
- **Attestation:** DePIN networks (io.net, Akash, Render, Helium) already verify device existence through proof-of-work, proof-of-coverage, and computational benchmarks. MoltLaunch composes these existing attestations into agent identity.
- **Revocation:** A compromised device can be physically decommissioned, providing a revocation path that software-only identity lacks.

---

## 2. Specification

### 2.1 Validation Request

A validation request is a JSON object submitted to the unified validation endpoint. All fields except `requestor`, `agentId`, and `validationType` are optional and default to sensible values.

```typescript
interface ValidationRequest {
  // Required fields
  requestor: string;         // Requesting entity (wallet address or API key)
  chain: string;             // Target chain: "solana" | "evm:8453" | "evm:1" | "evm:42161"
  agentId: string;           // Unique agent identifier

  // Validation configuration
  validationType: (
    | "identity"             // Hardware-anchored identity verification
    | "scoring"              // PoA score computation
    | "behavioral"           // Execution trace analysis
    | "sybil"                // Sybil resistance assessment
    | "proof"                // STARK proof generation
  )[];

  trustRequired: 1 | 2 | 3 | 4 | 5;  // Minimum trust level (see §2.3)

  // Optional parameters
  threshold?: number;        // Score threshold for STARK proofs (default: 60)
  callback?: string;         // Webhook URL for async result delivery
  nonce?: string;            // Client-generated nonce for replay protection
  timestamp?: number;        // Unix timestamp (must be within ±60s of server time)
  signature?: string;        // Ed25519 signature of request body by requestor wallet
  includeRaw?: boolean;      // Include raw feature data in response (default: false)
  erc8004?: boolean;         // Include ERC-8004 compatibility fields (default: true)
}
```

**Example:**

```json
{
  "requestor": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
  "chain": "solana",
  "agentId": "agent-moltbot-v2",
  "validationType": ["identity", "scoring", "proof"],
  "trustRequired": 3,
  "threshold": 70,
  "nonce": "a1b2c3d4e5f6...",
  "timestamp": 1707436800,
  "signature": "3Yt8x..."
}
```

### 2.2 Validation Response

```typescript
interface ValidationResponse {
  // Metadata
  requestId: string;              // Unique request identifier (UUIDv4)
  agentId: string;                // Echoed agent identifier
  chain: string;                  // Echoed chain
  timestamp: number;              // Server timestamp
  expiry: number;                 // Response validity expiry (Unix timestamp)
  status: "complete" | "pending" | "failed";

  // Identity section (when "identity" requested)
  identity?: {
    verified: boolean;            // Whether identity could be verified
    anchorType: "none" | "software" | "tpm" | "depin";
    fingerprint?: string;         // SHA-256 hash of hardware fingerprint (if available)
    depinProvider?: string;       // DePIN network name (if depin-anchored)
    depinDeviceId?: string;       // Hashed device ID on DePIN network
    depinAttestation?: {
      provider: string;           // e.g., "io.net", "akash", "render"
      deviceType: string;         // e.g., "gpu", "cpu", "sensor", "hotspot"
      lastSeen: number;           // Last attestation timestamp
      uptimeHours: number;        // Cumulative uptime
      proofHash: string;          // Hash of provider's attestation proof
    };
    trustLevel: 0 | 1 | 2 | 3 | 4 | 5;
    sybilCost: string;            // Estimated USD cost to replicate this identity
  };

  // Scoring section (when "scoring" requested)
  scoring?: {
    tier: "excellent" | "verified" | "needs_work" | "unverified";
    passed: boolean;              // Whether score >= threshold
    threshold: number;            // Threshold used
    score?: number;               // Only included if includeRaw: true
    features?: {                  // Only included if includeRaw: true
      hasGithub: boolean;
      hasApiEndpoint: boolean;
      capabilityCount: number;
      codeLines: number;
      hasDocumentation: boolean;
      testCoverage: number;
    };
    computedOnChain: boolean;     // Whether scoring ran on Frostbite VM
    scorerProgram?: string;       // Solana program address of scorer
    weightAccount?: string;       // Solana account holding model weights
  };

  // Behavioral section (when "behavioral" requested)
  behavioral?: {
    executionTraces: number;      // Number of traced executions
    avgResponseTime: number;      // Average response time (ms)
    errorRate: number;            // Error rate (0.0-1.0)
    consistencyScore: number;     // Behavioral consistency (0-100)
    anomalies: string[];          // Detected behavioral anomalies
    slotscribeIntegration: boolean; // Whether traces came from SlotScribe
    lastTraceTimestamp?: number;
  };

  // Sybil section (when "sybil" requested)
  sybil?: {
    score: number;                // Sybil resistance score (0-100)
    riskLevel: "low" | "medium" | "high" | "critical";
    factors: {
      walletAge: number;          // Days since first transaction
      transactionCount: number;   // Total on-chain transactions
      uniqueInteractions: number; // Unique programs/contracts interacted with
      hardwareAnchored: boolean;  // Whether identity is hardware-bound
      depinVerified: boolean;     // Whether DePIN attestation exists
      socialLinked: boolean;      // Whether social accounts are linked
      stakeAmount: number;        // SOL staked as identity bond
    };
    estimatedSybilCost: string;   // USD cost to replicate this identity
  };

  // Proof section (when "proof" requested)
  proof?: {
    type: "stark";                // Proof system used
    proofBytes: string;           // Hex-encoded STARK proof (~6KB)
    publicInputs: {
      agentCommitment: string;    // SHA-256(agentId || identity fingerprint)
      threshold: number;          // Score threshold proven against
      timestamp: number;          // Proof generation timestamp
      expiry: number;             // Proof validity expiry
    };
    verifierProgram?: string;     // Solana program for on-chain verification
    txHash?: string;              // Solana transaction hash (if anchored on-chain)
    proofType: "threshold" | "consistency" | "streak" | "stability";
    verified: boolean;            // Whether proof was verified server-side
  };

  // ERC-8004 Compatibility (when erc8004: true)
  erc8004Compat?: {
    validationResponse: {
      agentId: string;
      isValid: boolean;
      score: number;              // Normalized 0-100
      attestationHash: string;
      timestamp: number;
      expiresAt: number;
      validatorAddress: string;   // MoltLaunch validator identifier
      chain: string;
      metadata: {
        provider: string;         // "moltlaunch"
        version: string;          // Protocol version
        proofSystem: string;      // "circle-stark"
        trustLevel: number;
        sybilCost: string;
        depinAnchored: boolean;
      };
    };
  };

  // Pricing
  cost: {
    amount: string;               // Cost in USDC
    currency: "USDC";
    paymentTx?: string;           // Payment transaction hash
    x402?: {                      // x402 payment details (if applicable)
      payTo: string;
      network: string;
      maxAmountRequired: string;
    };
  };
}
```

### 2.3 Trust Ladder

The Trust Ladder defines five levels of identity assurance, each with increasing Sybil resistance cost and verification rigor.

| Level | Name | Description | Sybil Cost | What It Proves | Verification Method |
|-------|------|-------------|-----------|----------------|---------------------|
| 0 | **Unverified** | No verification performed | $0 | Nothing | None |
| 1 | **Wallet-Bound** | Agent tied to a funded wallet | ~$1 | Wallet exists and has SOL | Wallet signature + balance check |
| 2 | **Score-Verified** | PoA scoring completed, threshold met | ~$5 | Agent has code, docs, tests | On-chain AI scoring via Cauldron/Frostbite |
| 3 | **Software-Anchored** | Software fingerprint of runtime environment | ~$50 | Agent runs on a specific machine | Browser/runtime fingerprinting + behavioral analysis |
| 4 | **TPM-Anchored** | Hardware TPM attestation of device identity | ~$500 | Agent runs on specific physical hardware | TPM 2.0 attestation + remote verification |
| 5 | **DePIN-Anchored** | Identity anchored to verified DePIN device | ~$2,000+ | Agent runs on a known, staked physical device | DePIN provider attestation + on-chain stake verification |

**Trust Level Selection Guidance:**

- **Level 1-2:** Suitable for low-stakes interactions (forum posting, information queries)
- **Level 3:** Suitable for financial operations under $1,000
- **Level 4:** Suitable for high-value operations, pool access, governance voting
- **Level 5:** Required for protocol-level operations, validator roles, and custody operations

### 2.4 Identity Anchoring

#### 2.4.1 Fingerprint Construction

Identity fingerprints are constructed as a layered hash, incorporating available hardware signals at each trust level:

```
Level 1 (Wallet):
  fingerprint = SHA-256(walletPubkey)

Level 2 (Score):
  fingerprint = SHA-256(walletPubkey || poaScore || scorerProgramId)

Level 3 (Software):
  fingerprint = SHA-256(
    walletPubkey ||
    userAgent ||
    screenResolution ||
    gpuRenderer ||
    timezone ||
    installedFonts ||
    canvasHash ||
    webglHash
  )

Level 4 (TPM):
  fingerprint = SHA-256(
    walletPubkey ||
    tpmEndorsementKey ||
    tpmAttestationCert ||
    platformPCRs[0..7]
  )

Level 5 (DePIN):
  fingerprint = SHA-256(
    walletPubkey ||
    depinProvider ||
    depinDeviceId ||
    depinAttestationProof ||
    depinStakeAmount ||
    depinUptimeProof
  )
```

#### 2.4.2 Privacy Properties

- **Level 1-2:** No hardware data collected. Fingerprint is deterministic from public data.
- **Level 3:** Software fingerprint components are hashed locally before transmission. The server never sees raw browser/runtime data — only the composite hash.
- **Level 4:** TPM attestation keys are anonymized via Direct Anonymous Attestation (DAA). The TPM proves it is genuine without revealing which specific TPM it is.
- **Level 5:** DePIN device IDs are hashed with a per-agent salt. The same physical device produces different fingerprints for different agents, preventing cross-agent tracking.

#### 2.4.3 Fingerprint Stability

Fingerprints MUST remain stable across sessions for the same agent on the same hardware. Unstable components (e.g., dynamic IP, battery level) are excluded. If a fingerprint changes, the agent's trust level MAY be downgraded pending re-verification.

### 2.5 STARK Proof Types

The protocol supports four types of STARK proofs, each designed for different verification scenarios. All proofs use Circle STARKs over the Mersenne-31 (M31) field, implemented via the STWO prover.

#### 2.5.1 Threshold Proof

**Proves:** Agent's PoA score is greater than or equal to a specified threshold.

**Public Inputs:** `(agentCommitment, threshold, timestamp, expiry)`
**Private Witness:** `(score, features[6])`

**Circuit Constraints:**
1. `computedScore(features) == score` — Score is correctly computed from features
2. `score >= threshold` — Score meets or exceeds threshold
3. `timestamp <= expiry` — Proof is within validity window

**Privacy Guarantee:** Verifier learns only that `score >= threshold`. The exact score and individual feature values remain private.

**Use Case:** General verification — "Is this agent good enough?"

#### 2.5.2 Consistency Proof

**Proves:** Agent's score has not deviated by more than `δ` points across `n` consecutive evaluations.

**Public Inputs:** `(agentCommitment, maxDeviation, evaluationCount, timestamp)`
**Private Witness:** `(scores[n], timestamps[n])`

**Circuit Constraints:**
1. `max(scores) - min(scores) <= maxDeviation`
2. `timestamps` are monotonically increasing
3. All `scores` are correctly committed

**Privacy Guarantee:** Verifier learns only that the agent is consistent. Individual scores, trends, and evaluation timestamps remain private.

**Use Case:** Stability assessment — "Is this agent reliably good, or did it get lucky once?"

#### 2.5.3 Streak Proof

**Proves:** Agent has passed verification (score >= threshold) for `k` consecutive evaluations without failure.

**Public Inputs:** `(agentCommitment, threshold, streakLength, timestamp)`
**Private Witness:** `(scores[k], timestamps[k])`

**Circuit Constraints:**
1. For all `i`: `scores[i] >= threshold`
2. `timestamps` are monotonically increasing with minimum interval
3. `len(scores) >= streakLength`

**Privacy Guarantee:** Verifier learns only the streak length and threshold. No individual scores are revealed.

**Use Case:** Reliability track record — "Has this agent consistently performed?"

#### 2.5.4 Stability Proof

**Proves:** Agent's score standard deviation is below a specified bound over a time window.

**Public Inputs:** `(agentCommitment, maxStdDev, windowStart, windowEnd)`
**Private Witness:** `(scores[n], timestamps[n])`

**Circuit Constraints:**
1. `stddev(scores) <= maxStdDev` (computed in M31 arithmetic)
2. All `timestamps` fall within `[windowStart, windowEnd]`
3. Minimum `n` evaluations within window

**Privacy Guarantee:** Verifier learns only that variance is bounded. Distribution shape, mean, and individual values remain private.

**Use Case:** Risk assessment for capital allocation — "Is this agent predictable?"

#### 2.5.5 Proof Parameters

| Parameter | Value |
|-----------|-------|
| Field | Mersenne-31 (M31 = 2³¹ - 1) |
| Proof System | Circle STARK (STWO) |
| Proof Size | ~6 KB |
| Verification Cost | ~50,000 CU (Solana) |
| Security Level | 128-bit (post-quantum) |
| Trusted Setup | None required |
| Prover Time | ~2-5 seconds |

### 2.6 DePIN Integration

#### 2.6.1 Supported Providers

| Provider | Device Type | Attestation Method | Trust Signal |
|----------|-------------|-------------------|--------------|
| **io.net** | GPU clusters | Computational benchmarks + uptime proofs | GPU exists, performs work |
| **Akash Network** | Compute nodes | Provider on-chain registration + lease history | Node is active, has stake |
| **Render Network** | GPU rendering | Job completion proofs | GPU renders correctly |
| **Helium** | IoT hotspots | Proof of Coverage (PoC) | Physical device at location |
| **Hivemapper** | Dashcams | Map data contribution proofs | Device captures real-world data |
| **Nosana** | CI/CD nodes | Job execution proofs | Node executes pipelines |

#### 2.6.2 Attestation Flow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Agent     │────▶│  MoltLaunch  │────▶│  DePIN Provider │
│  Claims     │     │  Validator   │     │  (e.g., io.net) │
│  Device ID  │     │              │◀────│  Attestation    │
└─────────────┘     └──────────────┘     └─────────────────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Verify:     │
                    │  - Device    │
                    │    exists    │
                    │  - Uptime    │
                    │    > 24h     │
                    │  - Stake     │
                    │    active    │
                    └──────────────┘
```

#### 2.6.3 Device-to-Identity Mapping

Each DePIN device attestation is mapped to an agent identity via:

```
depinIdentity = SHA-256(
  depinProvider ||
  depinDeviceId ||
  agentPubkey ||
  attestationTimestamp
)
```

**Constraints:**
- One device can anchor at most 3 agent identities (prevents abuse while allowing multi-agent setups)
- Device must have minimum 24 hours of verified uptime
- Device stake must be active (not unbonding)
- Attestation must be less than 7 days old

### 2.7 On-Chain Anchoring

#### 2.7.1 Solana Memo Program (Current)

Validation attestations are currently anchored on Solana using the Memo program (`MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`):

```
Memo format:
  MOLT:V1:<agentId>:<attestationHash>:<trustLevel>:<expiry>

Example:
  MOLT:V1:agent-moltbot-v2:a3b4c5d6...1f2e:3:1710028800
```

**Properties:**
- Immutable once written
- Indexed by Solana explorers
- Queryable via RPC `getSignaturesForAddress`
- Low cost (~0.000005 SOL per attestation)

#### 2.7.2 Program Derived Addresses (Future — Phase 3)

A dedicated Solana program will store attestations in PDAs for richer querying:

```rust
#[account]
pub struct AgentAttestation {
    pub agent_id: [u8; 32],           // SHA-256 of agentId
    pub trust_level: u8,              // 0-5
    pub attestation_hash: [u8; 32],   // Full attestation hash
    pub fingerprint: [u8; 32],        // Hardware fingerprint hash
    pub scorer_program: Pubkey,       // Program that computed score
    pub proof_hash: [u8; 32],         // STARK proof hash
    pub created_at: i64,              // Unix timestamp
    pub expires_at: i64,              // Expiry timestamp
    pub revoked: bool,                // Revocation flag
    pub bump: u8,                     // PDA bump seed
}

// PDA derivation:
// seeds = [b"attestation", agent_id, &trust_level.to_le_bytes()]
```

#### 2.7.3 Cross-Chain via Wormhole (Future — Phase 4)

Attestations will be bridgeable to EVM chains via Wormhole:

```
Solana Attestation → Wormhole VAA → EVM Verifier Contract
```

This enables:
- Base (EVM:8453) agents to verify Solana-anchored attestations
- Ethereum mainnet (EVM:1) smart contracts to gate access based on MoltLaunch trust levels
- Unified identity across chains without re-verification

### 2.8 ERC-8004 Compatibility

#### 2.8.1 Field Mapping

The MoltLaunch Validation Response maps to the ERC-8004 `validationResponse` format as follows:

| ERC-8004 Field | MoltLaunch Source | Notes |
|----------------|-------------------|-------|
| `agentId` | `response.agentId` | Direct mapping |
| `isValid` | `response.scoring.passed` | Threshold check result |
| `score` | `response.scoring.score` | Normalized 0-100 (only if `includeRaw: true`) |
| `attestationHash` | `SHA-256(response)` | Hash of full response body |
| `timestamp` | `response.timestamp` | Unix timestamp |
| `expiresAt` | `response.expiry` | Unix timestamp |
| `validatorAddress` | MoltLaunch validator pubkey | Solana address of validator |
| `chain` | `response.chain` | Target chain identifier |
| `metadata` | See below | Extended metadata |

#### 2.8.2 Extended Metadata

ERC-8004's `metadata` field is used to carry MoltLaunch-specific extensions:

```json
{
  "provider": "moltlaunch",
  "version": "0.1.0",
  "proofSystem": "circle-stark",
  "trustLevel": 3,
  "sybilCost": "$500",
  "depinAnchored": true,
  "depinProvider": "io.net",
  "proofAvailable": true,
  "proofType": "threshold",
  "behavioralTraced": true
}
```

#### 2.8.3 Cross-Standard Queries

Agents verified by ERC-8004-compliant validators on EVM chains can have their attestations referenced in MoltLaunch responses:

```typescript
interface CrossChainReference {
  standard: "erc-8004";
  chain: string;                // e.g., "evm:8453"
  contractAddress: string;      // ERC-8004 registry contract
  attestationId: string;        // On-chain attestation ID
  verifiedAt: number;
}
```

---

## 3. API Endpoints

### 3.1 Unified Validation Endpoint

```
POST /api/validate
Content-Type: application/json
Authorization: Bearer <api_key> | x402 payment header

Request Body: ValidationRequest (see §2.1)
Response Body: ValidationResponse (see §2.2)
```

**Behavior:**
- Synchronous for trust levels 1-2 (returns in <5s)
- Asynchronous for trust levels 3-5 (returns `status: "pending"` with `requestId`)
- When `callback` is provided, results are POSTed to the callback URL on completion

**HTTP Status Codes:**
| Code | Meaning |
|------|---------|
| 200 | Validation complete |
| 202 | Validation accepted, processing asynchronously |
| 400 | Invalid request format |
| 402 | Payment required (x402 payment instructions in response) |
| 404 | Agent not found |
| 429 | Rate limited |
| 500 | Internal validation error |

### 3.2 Async Status Check

```
GET /api/validate/status/:requestId
Authorization: Bearer <api_key>

Response: {
  requestId: string;
  status: "pending" | "processing" | "complete" | "failed";
  progress?: number;            // 0-100 percentage
  estimatedCompletion?: number;  // Unix timestamp
  result?: ValidationResponse;   // Present when status is "complete"
  error?: string;               // Present when status is "failed"
}
```

### 3.3 Protocol Spec

```
GET /api/validate/spec

Response: This specification document in JSON format
Content-Type: application/json
```

No authentication required. Returns the machine-readable version of this spec including all type definitions and the trust ladder.

### 3.4 Existing Endpoints (Feeding Into Protocol)

These existing MoltLaunch API endpoints provide data that feeds into the unified validation protocol:

| Endpoint | Method | Purpose | Protocol Integration |
|----------|--------|---------|---------------------|
| `/api/verify/deep` | POST | Deep verification with PoA scoring | Feeds `scoring` section |
| `/api/verify/status/:agentId` | GET | Current verification status | Cached validation state |
| `/api/agents` | GET | List registered agents | Agent registry |
| `/api/agents/:agentId` | GET | Agent details | Identity data |
| `/api/agents/:agentId/behavioral` | GET | Behavioral metrics | Feeds `behavioral` section |
| `/api/identity/link` | POST | Link social/SNS identity | Feeds `sybil.socialLinked` |
| `/api/identity/:agentId` | GET | Identity details | Feeds `identity` section |
| `/api/blink/verify/:agentId` | GET | Solana Blink for verification | Alternative verification entry point |
| `/api/priority-fee` | GET | Current priority fee estimate | Transaction optimization |

### 3.5 SDK Methods

```typescript
import { MoltLaunch } from '@moltlaunch/sdk';

const ml = new MoltLaunch({ apiKey: 'your-key' });

// Unified validation
const result = await ml.validate({
  agentId: 'agent-moltbot-v2',
  validationType: ['identity', 'scoring', 'proof'],
  trustRequired: 3,
});

// Check async status
const status = await ml.validateStatus(result.requestId);

// Convenience methods
const identity = await ml.verifyIdentity('agent-id', { trustLevel: 4 });
const proof = await ml.generateProof('agent-id', { type: 'threshold', threshold: 70 });
const sybil = await ml.checkSybil('agent-id');
```

---

## 4. Pricing

### 4.1 Trust Level Pricing

| Trust Level | Cost (USDC) | Validity | Includes |
|-------------|-------------|----------|----------|
| Level 1 (Wallet) | Free | 7 days | Wallet verification only |
| Level 2 (Score) | $0.25 | 30 days | PoA scoring + attestation |
| Level 3 (Software) | $1.00 | 30 days | + Software fingerprint + STARK proof |
| Level 4 (TPM) | $5.00 | 90 days | + TPM attestation + all proof types |
| Level 5 (DePIN) | $10.00 | 90 days | + DePIN attestation + full protocol |

### 4.2 x402 Micropayments

All paid validations support the x402 HTTP payment protocol:

1. Client sends `POST /api/validate` without payment
2. Server returns `HTTP 402` with payment instructions:
   ```json
   {
     "x402": {
       "version": 1,
       "payTo": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
       "network": "solana",
       "maxAmountRequired": "1000000",
       "asset": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
       "description": "MoltLaunch Validation — Trust Level 3"
     }
   }
   ```
3. Client completes USDC payment on Solana
4. Client retries request with payment transaction signature in `X-Payment-Tx` header
5. Server verifies payment and processes validation

### 4.3 Bulk Pricing

For validators and protocols performing >100 validations/month:

| Volume | Discount |
|--------|----------|
| 100-500/mo | 20% |
| 500-2,000/mo | 40% |
| 2,000+/mo | Custom |

---

## 5. Security Considerations

### 5.1 Fingerprint Evasion

**Threat:** An adversary modifies browser/runtime properties to generate a different software fingerprint, effectively creating a new identity.

**Mitigation:**
- Software fingerprints (Level 3) are treated as a probabilistic signal, not a hard identity proof. Multiple fingerprint changes trigger anomaly flags.
- Trust Levels 4-5 use hardware attestations that cannot be spoofed without physical hardware.
- Behavioral consistency scoring detects agents that change fingerprints while maintaining the same behavioral patterns.

**Residual Risk:** Level 3 identity can be evaded by a sophisticated adversary willing to invest in VM/container diversity. This is why hardware anchoring (Levels 4-5) exists for high-stakes scenarios.

### 5.2 Containerization and Virtualization

**Threat:** Agents running in containers (Docker, Kubernetes) or VMs may not have access to TPM or stable hardware identifiers, reducing fingerprint quality.

**Mitigation:**
- Container-based agents are limited to Trust Level 3 (software fingerprint)
- DePIN-anchored identity (Level 5) bypasses this by anchoring to the DePIN device rather than the agent's own runtime
- Agents can attest to their DePIN device via out-of-band verification (device signs a challenge from MoltLaunch)

**Design Choice:** We deliberately support containerized agents at lower trust levels rather than excluding them, since much of the agent ecosystem runs in containers.

### 5.3 Privacy of Hardware Data

**Threat:** Collection of hardware fingerprints and DePIN device IDs could enable tracking or deanonymization.

**Mitigation:**
- All fingerprints are one-way hashed before transmission (see §2.4.2)
- DePIN device IDs are salted per-agent, preventing cross-agent correlation
- TPM attestations use Direct Anonymous Attestation (DAA)
- STARK proofs reveal no information about the underlying identity — only that it meets the stated criteria
- MoltLaunch servers store only hashes, never raw hardware data
- Users may request data deletion per applicable privacy regulations

### 5.4 Validator Centralization

**Current Limitation:** MoltLaunch currently operates as a centralized validator. All validation requests are processed by MoltLaunch infrastructure, creating a single point of trust.

**Mitigation Path:**
- Phase 1-2 (current): Centralized but transparent — all attestations are anchored on-chain and independently verifiable
- Phase 3: Attestation program on-chain, enabling third-party verification of stored attestations
- Phase 4-5: Decentralized validator network with stake-weighted consensus on validation results

**Transparency Measures:**
- Open-source scoring model (weights published on-chain)
- Deterministic scoring (same inputs → same outputs)
- All STARK proofs are independently verifiable
- Attestation history is immutable on Solana

### 5.5 Replay and Man-in-the-Middle

**Threat:** An adversary captures a valid validation response and replays it, or intercepts and modifies responses.

**Mitigation:**
- Every request requires a nonce (single-use, 24h TTL cache)
- Timestamps must be within ±60 seconds of server time
- Requests are signed by the requestor's wallet
- Responses include server signature for tamper detection
- On-chain attestations provide a ground truth that cannot be modified

---

## 6. Roadmap

### Phase 1: Protocol Specification (Current)

- [x] Publish Validation Protocol Spec (this document)
- [x] Define trust ladder and pricing
- [x] Define STARK proof types
- [x] Document ERC-8004 compatibility
- [ ] Community feedback and revision

### Phase 2: Implementation

- [ ] Implement `POST /api/validate` unified endpoint
- [ ] Implement async validation with callbacks
- [ ] Integrate software fingerprinting (Level 3)
- [ ] Ship `@moltlaunch/sdk` v3 with validation methods
- [ ] Deploy STARK proof generation (threshold proofs first)

### Phase 3: On-Chain Program

- [ ] Deploy `AgentAttestation` PDA program to Solana devnet
- [ ] Implement on-chain STARK verifier (Murkl collaboration)
- [ ] Enable third-party attestation queries via RPC
- [ ] Launch on mainnet-beta

### Phase 4: DePIN & Cross-Chain

- [ ] io.net integration (GPU attestations)
- [ ] Akash integration (compute attestations)
- [ ] Wormhole bridge for EVM attestation portability
- [ ] TPM attestation support (Level 4)

### Phase 5: Decentralized Validator Network

- [ ] Validator staking and registration
- [ ] Multi-validator consensus on validation results
- [ ] Slashing for dishonest validators
- [ ] Governance over scoring weights and trust level definitions
- [ ] Permissionless validator onboarding

---

## 7. References

1. **ERC-8004:** Agent Verification Standard. Ethereum Improvement Proposals. 2025. https://eips.ethereum.org/EIPS/eip-8004

2. **Ben-Sasson, E., Bentov, I., Horesh, Y., Riabzev, M.** "Scalable, transparent, and post-quantum secure computational integrity." IACR Cryptology ePrint Archive, 2018/046.

3. **Habock, U.** "Circle STARKs." IACR Cryptology ePrint Archive, 2024.

4. **StarkWare.** "STWO: Open-Source STARK Prover." 2024. https://github.com/starkware-libs/stwo

5. **Cauldron Labs.** "Frostbite: On-Chain AI Inference for Solana." 2025.

6. **Coinbase.** "x402: HTTP Payment Protocol for Machine-to-Machine Payments." 2025. https://github.com/coinbase/x402

7. **io.net.** "Decentralized GPU Computing Network." https://io.net

8. **Akash Network.** "The Decentralized Cloud." https://akash.network

9. **Render Network.** "Distributed GPU Rendering." https://rendernetwork.com

10. **Helium Network.** "The People's Network — Decentralized Wireless." https://helium.com

11. **Hivemapper.** "Decentralized Mapping Network." https://hivemapper.com

12. **Nosana.** "Decentralized CI/CD on Solana." https://nosana.io

---

## Appendix A: Full Example — Validation Request & Response

### Request

```bash
curl -X POST https://web-production-419d9.up.railway.app/api/validate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <api_key>" \
  -d '{
    "requestor": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
    "chain": "solana",
    "agentId": "agent-moltbot-v2",
    "validationType": ["identity", "scoring", "sybil", "proof"],
    "trustRequired": 3,
    "threshold": 70,
    "erc8004": true
  }'
```

### Response

```json
{
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "agentId": "agent-moltbot-v2",
  "chain": "solana",
  "timestamp": 1707436800,
  "expiry": 1710028800,
  "status": "complete",

  "identity": {
    "verified": true,
    "anchorType": "software",
    "fingerprint": "a3b4c5d6e7f8...1a2b3c4d",
    "trustLevel": 3,
    "sybilCost": "$50"
  },

  "scoring": {
    "tier": "excellent",
    "passed": true,
    "threshold": 70,
    "computedOnChain": true,
    "scorerProgram": "FRsToriMLgDc1Ud53ngzHUZvCRoazCaGeGUuzkwoha7m",
    "weightAccount": "GnSxMWbZEa538vJ9Pf3veDrKP1LkzPiaaVmC4mRnM91N"
  },

  "sybil": {
    "score": 82,
    "riskLevel": "low",
    "factors": {
      "walletAge": 45,
      "transactionCount": 312,
      "uniqueInteractions": 28,
      "hardwareAnchored": false,
      "depinVerified": false,
      "socialLinked": true,
      "stakeAmount": 2.5
    },
    "estimatedSybilCost": "$50"
  },

  "proof": {
    "type": "stark",
    "proofBytes": "0x0a1b2c3d...",
    "publicInputs": {
      "agentCommitment": "e5f6a7b8...",
      "threshold": 70,
      "timestamp": 1707436800,
      "expiry": 1710028800
    },
    "proofType": "threshold",
    "verified": true
  },

  "erc8004Compat": {
    "validationResponse": {
      "agentId": "agent-moltbot-v2",
      "isValid": true,
      "score": 85,
      "attestationHash": "c4d5e6f7...",
      "timestamp": 1707436800,
      "expiresAt": 1710028800,
      "validatorAddress": "MoLTv1...validator",
      "chain": "solana",
      "metadata": {
        "provider": "moltlaunch",
        "version": "0.1.0",
        "proofSystem": "circle-stark",
        "trustLevel": 3,
        "sybilCost": "$50",
        "depinAnchored": false
      }
    }
  },

  "cost": {
    "amount": "1.00",
    "currency": "USDC",
    "paymentTx": "5Ht7x..."
  }
}
```

---

## Appendix B: Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-02-08 | Initial draft |

---

*MoltLaunch Validation Protocol — Trust Before Capital*

*https://moltlaunch.app | https://web-production-419d9.up.railway.app*
