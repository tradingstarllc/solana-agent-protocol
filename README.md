# Solana Agent Protocol (SAP)

**Application-layer standards for AI agent identity, verification, and trust on Solana.**

---

## What Is This?

SAP defines how AI agents establish identity, prove capabilities, and build reputation on Solana. Think ERC-8004 (Trustless Agents) but Solana-native — leveraging DePIN hardware attestations, STARK zero-knowledge proofs, and on-chain AI scoring.

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

- **API:** https://web-production-419d9.up.railway.app (90+ endpoints)
- **SDK:** `npm install @moltlaunch/sdk@2.3.0`
- **On-chain AI:** POA-Scorer on Solana devnet via Cauldron/Frostbite
- **Registry:** 174 Colosseum hackathon projects evaluated

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
