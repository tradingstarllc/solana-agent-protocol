---
sap: '0003'
title: DePIN Device Attestation for Agent Identity
authors:
  - MoltLaunch (Colosseum Hackathon 2026)
category: Standard
status: Draft
created: 2026-02-08
requires: SAP-0001, SAP-0002
---

# SAP-0003: DePIN Device Attestation for Agent Identity

## Abstract

This proposal defines how AI agents can bind their identity to verified DePIN (Decentralized Physical Infrastructure Network) devices on Solana. By referencing existing on-chain device attestations from networks like io.net, Helium, and Nosana, agents achieve Trust Level 5 — the strongest Sybil resistance available, rooted in physical, decentralized hardware verification.

## Motivation

SAP-0002 defines hardware-anchored identity using software-level fingerprinting and TPM. These are effective but have limitations:

- **Software fingerprints** (Level 3) can be spoofed by modifying OS-level calls
- **TPM** (Level 4) requires direct hardware access, not available on cloud/VPS
- **Neither** leverages Solana's unique DePIN ecosystem

Solana hosts the largest DePIN ecosystem in crypto. These networks already solve device verification:

| Network | What They Verify | On-Chain Proof |
|---------|-----------------|----------------|
| **io.net** | GPU compute capacity | Device PDA with specs |
| **Helium** | IoT/5G hotspot location + uptime | Hotspot account |
| **Hivemapper** | Dashcam location + footage | Device NFT |
| **Nosana** | AI compute nodes | Node account |
| **Akash** | Cloud compute providers | Provider record |
| **Render** | GPU rendering capacity | Node account |

These device attestations are **already on Solana**. SAP-0003 defines how to reference them for agent identity.

## Specification

### DePIN Identity Binding

An agent binds to a DePIN device by creating an on-chain link:

```
┌─────────────────────────────────────────────────────────┐
│                  Identity Binding                        │
│                                                         │
│  Agent Identity PDA (SAP-0002)                          │
│  ├── identity_hash: [u8; 32]                            │
│  ├── trust_level: 5                                     │
│  └── depin_binding: {                                   │
│        provider: "io.net" | "helium" | "nosana" | ...   │
│        device_pda: Pubkey     // Reference to device    │
│        binding_hash: [u8; 32] // SHA-256(identity + device) │
│        bound_at: i64                                    │
│      }                                                  │
│                                                         │
│  DePIN Device PDA (provider-specific)                   │
│  ├── device_id: String                                  │
│  ├── specs: { gpu, cpu, memory, ... }                   │
│  ├── verified: bool                                     │
│  └── owner: Pubkey                                      │
│                                                         │
│  Binding proves: This agent runs on THIS verified device│
└─────────────────────────────────────────────────────────┘
```

### Provider Integration

#### io.net (GPU Compute)

```typescript
interface IoNetBinding {
    provider: "io.net";
    deviceId: string;           // io.net device identifier
    devicePDA: PublicKey;        // On-chain device account
    gpuModel: string;           // Verified GPU model
    computeCapacity: number;    // Verified TFLOPS
}
```

**Verification:** The io.net network verifies GPU specs via benchmark. The device PDA stores verified specs. SAP references this PDA to prove the agent runs on verified compute.

#### Helium (IoT/5G)

```typescript
interface HeliumBinding {
    provider: "helium";
    hotspotAddress: PublicKey;   // Helium hotspot account
    location: string;           // H3 hex (privacy: reduced resolution)
    networkType: "iot" | "5g";
}
```

**Verification:** Helium verifies hotspot location via proof-of-coverage. Agents bound to Helium hotspots have verified physical location.

#### Nosana (AI Compute)

```typescript
interface NosanaBinding {
    provider: "nosana";
    nodeAddress: PublicKey;      // Nosana node account
    gpuType: string;
    jobsCompleted: number;      // Track record
}
```

**Verification:** Nosana verifies AI compute nodes via job execution. Agents bound to Nosana nodes have verified AI processing capability.

### Binding Protocol

#### Step 1: Agent Proves Device Access

The agent must prove it runs on the claimed device:

```typescript
// Agent generates challenge response
const challenge = randomBytes(32);
const response = SHA-256(challenge + devicePrivateData);

// This can only be computed ON the device
```

#### Step 2: Create Binding Transaction

```typescript
// Solana transaction creating the binding
const bindingIx = createBindingInstruction({
    agentIdentityPDA,       // SAP-0002 identity
    depinDevicePDA,         // Provider's device account
    bindingHash: SHA-256(identityHash + devicePDA.toBase58()),
    provider: "io.net",
    timestamp: Date.now()
});
```

#### Step 3: Verification by Protocols

```typescript
// Any protocol can verify the binding
const binding = await getBinding(agentIdentityPDA);

if (binding.depinDevice) {
    // Verify device PDA exists and is active
    const device = await connection.getAccountInfo(binding.depinDevice);
    if (device && isActive(device)) {
        // Trust Level 5 confirmed
        // Agent runs on verified DePIN hardware
    }
}
```

### Sybil Cost Analysis

| Setup | Monthly Cost | Agents Created |
|-------|-------------|----------------|
| Wallets only | $0 | Unlimited |
| Cloud VMs (Level 3) | $100/agent | Limited by budget |
| Dedicated servers + TPM (Level 4) | $200/agent | Need physical machines |
| **DePIN devices (Level 5)** | **$500+/agent** | **Need registered devices** |

**Why DePIN is strongest:** DePIN networks require:
1. Physical hardware (can't be virtualized)
2. Network registration (verified by the DePIN protocol)
3. Ongoing operation (maintaining uptime/coverage)
4. Stake/collateral in some networks

Creating 10 Sybil identities at Level 5 costs $5,000+/month minimum — economically irrational for most attack scenarios.

### Multi-Provider Binding

Agents MAY bind to multiple DePIN providers for stronger attestation:

```typescript
const identity = {
    hash: "7f04b937...",
    trustLevel: 5,
    depinBindings: [
        { provider: "io.net", devicePDA: "...", verified: true },
        { provider: "nosana", devicePDA: "...", verified: true }
    ],
    multiProviderBonus: true  // Extra trust for multiple attestations
};
```

## Security Considerations

### Device Ownership vs Access

**Risk:** An attacker could temporarily access a DePIN device to create a binding, then operate from different hardware.

**Mitigation:** Bindings include periodic re-verification. The agent must prove ongoing device access, not just one-time binding.

### Device Sharing

**Risk:** Multiple agents could bind to the same DePIN device.

**Mitigation:** One-to-one binding enforced on-chain. A device PDA can only be bound to one agent identity at a time. Re-binding requires unbinding first.

### Provider Centralization

**Risk:** DePIN provider could fabricate device attestations.

**Mitigation:** Use multiple providers (multi-provider binding). No single provider compromise breaks identity. The DePIN networks themselves have economic security (staking, slashing).

### Privacy

**Property:** The binding reveals:
- ✅ "This agent runs on a verified io.net GPU"
- ❌ NOT the specific GPU serial, location, or owner identity

Device specs (GPU model, compute capacity) may be visible in the provider's PDA but cannot be linked to the agent operator's real identity.

## Rationale

### Why Not Just Use DePIN Directly?

DePIN networks verify devices, not agents. An io.net device doesn't know what software runs on it. SAP-0003 bridges the gap: "This agent (SAP-0002 identity) runs on this device (DePIN attestation)."

### Why Solana?

This proposal is only possible on Solana because:
1. All major DePIN networks are Solana-native
2. Device attestations are already on-chain as PDAs
3. CPI (Cross-Program Invocation) enables composable verification
4. No other chain has this ecosystem

## References

- [SAP-0001: Validation Protocol](SAP-0001-validation-protocol.md)
- [SAP-0002: Hardware-Anchored Identity](SAP-0002-hardware-identity.md)
- [io.net Documentation](https://docs.io.net)
- [Helium Network](https://docs.helium.com)
- [Nosana](https://docs.nosana.io)
- [ERC-8004: Trustless Agents](https://eips.ethereum.org/EIPS/eip-8004)
- [MoltLaunch SDK](https://www.npmjs.com/package/@moltlaunch/sdk) — Reference implementation
