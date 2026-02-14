# Solana Agent Protocol (SAP)

**Application-layer standards for AI agent identity, verification, and trust on Solana.**

---

## What Is This?

SAP defines how AI agents establish identity, prove capabilities, and build reputation on Solana. Think ERC-8004 (Trustless Agents) but Solana-native — leveraging DePIN hardware attestations, STARK-inspired zero-knowledge proofs, and on-chain AI scoring.

> **Note on STARK proofs:** Current implementation provides ~32-bit security with real FRI protocol and Merkle commitments. This is a simplified demonstration prover, not production-grade. See `stark-prover/README.md` for honest parameter comparison.

This is **not** a SIMD (Solana Improvement Document). SIMDs are for protocol-level changes. SAP is for application-layer standards that any Solana program or agent can adopt.

## Proposals

| SAP | Title | Status | Summary |
|-----|-------|--------|---------|
| [SAP-0001](proposals/SAP-0001-validation-protocol.md) | Validation Protocol | Draft | Unified validation request/response format with pluggable trust levels |
| [SAP-0002](proposals/SAP-0002-hardware-identity.md) | Hardware-Anchored Identity | Draft | Tie agent identity to physical hardware via fingerprinting, TPM, and DePIN |
| [SAP-0003](proposals/SAP-0003-depin-attestation.md) | DePIN Device Attestation | Draft | Link agent identity to verified DePIN devices (io.net, Helium, etc.) |

## Why Solana?

1. **DePIN is Solana-native** — io.net, Helium, Hivemapper, Nosana all store device attestations as Solana PDAs
2. **Speed** — 400ms blocks enable real-time verification during agent interactions
3. **Cost** — $0.00025/tx makes per-interaction verification economical
4. **Composability** — Agent identity PDAs can be referenced via CPI from any Solana program
5. **Ecosystem** — Solana Agent Kit, Actions/Blinks, Dialect already provide agent infrastructure

## The Trust Ladder

SAP defines 6 trust levels with increasing Sybil resistance:

```
Level 0: No verification              → $0     Sybil cost
Level 1: API key / wallet             → $0     Sybil cost
Level 2: Code hash verification       → $0     Sybil cost
Level 3: Hardware fingerprint          → $100/mo per identity
Level 4: TPM attestation              → $200/mo per identity
Level 5: DePIN device verification    → $500+/mo per identity
```

Higher levels require more physical infrastructure to forge, making Sybil attacks economically irrational.

## Cross-Chain Compatibility

SAP validation responses include fields compatible with [ERC-8004 (Trustless Agents)](https://eips.ethereum.org/EIPS/eip-8004), enabling:
- Agent verified on Solana → attestation readable on Ethereum/Base
- Pluggable trust model: SAP validators can serve as ERC-8004 validator contracts
- Portable agent identity across chains

## Reference Implementation

[MoltLaunch](https://github.com/tradingstarllc/moltlaunch) provides a working implementation:

- **Anchor Program (V3):** 12 instructions, 4 PDAs on Solana devnet (`6AZSAhq4iJTwCfGEVssoa1p3GnBqGkbcQ1iDdP1U1pSb`)
  - Admin: `initialize`, `add_authority`, `remove_authority`, `set_paused`, `transfer_admin`
  - Agent: `register_agent`, `flag_agent`, `unflag_agent`, `refresh_identity_signals`
  - Attestation: `submit_attestation`, `revoke_attestation`, `close_attestation`
- **4 PDAs:**
  - `ProtocolConfig` `["moltlaunch"]` — singleton, admin, revocation nonce, pause state
  - `Authority` `["authority", pubkey]` — per verifier, type (Single/Multisig/Oracle/NCN), active status
  - `AgentIdentity` `["agent", wallet]` — composable signals, derived trust score, infra type, flags
  - `Attestation` `["attestation", agent_wallet, authority_pubkey]` — per (agent, authority) pair, signal type, expiry, revocation
- **API:** https://youragent.id (90+ endpoints)
- **SDK:** `npm install @moltlaunch/sdk@2.4.0`
- **On-chain AI:** POA-Scorer on Solana devnet via Cauldron/Frostbite
- **Registry:** 174 Colosseum hackathon projects evaluated

## Governance

Attestation authority is being decentralized via a 4-phase plan. See [GOVERNANCE.md](https://github.com/tradingstarllc/moltlaunch/blob/main/GOVERNANCE.md) for full details.

| Phase | Model | Status |
|-------|-------|--------|
| 1 | Single authority (hackathon) | ✅ Complete |
| 2 | Squads multisig (2-of-3) | ✅ Deployed on devnet |
| 3 | Validator network (3-of-5 consensus) | 📋 Planned |
| 4 | DAO governance via Realms | 📋 Future |

**Squads Multisig:** [`3gCjhVMKazL2VKQgqQ8vP93vLzPTos1e7XLm1jr7X9t5`](https://explorer.solana.com/address/3gCjhVMKazL2VKQgqQ8vP93vLzPTos1e7XLm1jr7X9t5?cluster=devnet)

Partner seats are open — any SAP-aligned project can join the multisig to co-govern attestations. See [governance doc](https://github.com/tradingstarllc/moltlaunch/blob/main/GOVERNANCE.md) for details on claiming a seat.

## Contributing

This is an open standard. Contributions welcome:

1. **Review proposals** — Open issues with feedback
2. **Propose new SAPs** — Fork, write your proposal, open a PR
3. **Build implementations** — Reference implementations in any language
4. **Integrate** — Use SAP standards in your Solana agent project

### Proposal Format

```markdown
---
sap: '0001'
title: Proposal Title
authors:
  - Name (Organization)
category: Standard | Meta
status: Draft | Review | Accepted | Living
created: YYYY-MM-DD
---

## Abstract
## Motivation
## Specification
## Rationale
## Security Considerations
## References
```

## Related Work

| Standard | Chain | Focus | Relationship |
|----------|-------|-------|-------------|
| [ERC-8004](https://eips.ethereum.org/EIPS/eip-8004) | Ethereum | Trustless Agents | SAP extends with hardware identity + DePIN |
| [SIMD](https://github.com/solana-foundation/solana-improvement-documents) | Solana | Protocol changes | SAP is application-layer, not protocol |
| [AgentSkills](https://agentskills.io) | Cross-chain | Agent capability discovery | Complementary — skills vs identity |

## License

MIT

---

<p align="center">
  <strong>Trust infrastructure for the agent economy on Solana</strong><br>
  Born at the <a href="https://www.colosseum.org/">Colosseum Agent Hackathon 2026</a>
</p>
