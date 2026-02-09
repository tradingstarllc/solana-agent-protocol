---
sap: '0002'
title: Hardware-Anchored Identity
authors:
  - MoltLaunch (Colosseum Hackathon 2026)
category: Standard
status: Draft
created: 2026-02-08
requires: SAP-0001
---

# SAP-0002: Hardware-Anchored Agent Identity

## Abstract

This proposal defines a standard for tying AI agent identity to physical hardware characteristics. By hashing hardware, runtime, code, and network fingerprints into a deterministic identity, agents can prove they are unique physical entities — making Sybil attacks require real infrastructure investment.

## Motivation

Current agent identity on Solana is wallet-based. Creating a new wallet is free, instant, and unlimited. This means:

- One operator can create 1,000 "unique" agents for $0
- Sybil attacks on voting, staking, and gaming systems are trivial
- No way to distinguish "10 different agents" from "1 agent with 10 wallets"

Hardware-anchored identity changes the cost model: each unique identity requires a unique physical machine.

## Specification

### Identity Components

An agent's hardware-anchored identity is constructed from four component hashes:

```
identity_hash = SHA-256(hw_hash | rt_hash | code_hash | net_hash)
```

#### Component 1: Hardware Hash (`hw`)

```typescript
hw_hash = SHA-256(JSON.stringify({
    platform: os.platform(),      // "linux", "darwin", "win32"
    arch: os.arch(),              // "x64", "arm64"
    cpus: os.cpus().length,       // CPU count
    cpuModel: os.cpus()[0].model, // CPU model string
    totalMemory: os.totalmem(),   // Total RAM in bytes
    hostname: os.hostname()       // Machine hostname
}))
```

**Privacy:** The hash is one-way. Verifiers see `hw_hash` but cannot derive the actual CPU model, hostname, or memory size.

**Collision resistance:** Two machines with identical hardware specs AND hostname would collide. In practice, hostnames differ even for identical hardware.

#### Component 2: Runtime Hash (`rt`)

```typescript
rt_hash = SHA-256(JSON.stringify({
    nodeVersion: process.version,  // "v20.11.0"
    pid: process.pid,              // Process ID
    execPath: process.execPath,    // "/usr/bin/node"
    cwd: process.cwd(),            // Working directory
    user: process.env.USER,        // OS username
    home: process.env.HOME,        // Home directory
    shell: process.env.SHELL       // Default shell
}))
```

**Note:** `process.pid` changes on restart. For persistent identity, exclude PID and use a stored nonce instead.

#### Component 3: Code Hash (`code`)

```typescript
code_hash = SHA-256(readFileSync(codeEntryPoint, 'utf-8'))
```

Hashes the agent's main source file. Any code change produces a new identity. This prevents:
- Clone armies using identical code
- Trivial code modification to create "different" agents (changes identity)

**Trade-off:** Legitimate code updates change the identity. Agents should re-register after updates.

#### Component 4: Network Hash (`net`)

```typescript
const interfaces = os.networkInterfaces();
const fingerprint = Object.keys(interfaces)
    .sort()
    .map(name => {
        const iface = interfaces[name].find(n => !n.internal && n.family === 'IPv4');
        return iface ? `${name}:${iface.mac}` : null;
    })
    .filter(Boolean)
    .join('|');

net_hash = SHA-256(fingerprint)
```

MAC addresses provide device-level uniqueness. Spoofable but adds cost.

### Trust Levels

| Level | Components Used | Identity Strength | Sybil Cost |
|-------|----------------|-------------------|------------|
| 1 | None (wallet only) | Weak | $0 |
| 2 | `code_hash` only | Low | $0 (change a line) |
| 3 | `hw + rt + code + net` | Medium | $100/mo (new server) |
| 4 | Level 3 + TPM | Strong | $200/mo (physical machine) |
| 5 | Level 4 + DePIN | Strongest | $500+/mo (registered device) |

### TPM Integration (Level 4)

Trusted Platform Module 2.0 provides hardware-rooted identity:

```typescript
// Linux: Read from sysfs
const tpmData = [
    readFileSync('/sys/class/dmi/id/board_serial'),
    readFileSync('/sys/class/dmi/id/product_uuid'),
    readFileSync('/sys/class/dmi/id/chassis_serial')
].filter(Boolean);

tpm_hash = SHA-256(tpmData.join('|'))

// macOS: IOPlatformUUID
const uuid = execSync('ioreg -rd1 -c IOPlatformExpertDevice | grep IOPlatformUUID');
tpm_hash = SHA-256(uuid)
```

TPM endorsement keys are burned at manufacture — physically impossible to duplicate.

### On-Chain Representation

Identity SHOULD be stored on Solana as:

**Option A: Memo Program** (current, simple)
```
MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr
Data: "sap:identity:{agentId}:{identityHash}"
```

**Option B: PDA** (future, composable)
```
seeds = ["sap-identity", agent_pubkey]
data = {
    identity_hash: [u8; 32],
    trust_level: u8,
    components: u8,        // Bitmask of included components
    registered_at: i64,
    expires_at: i64,
    depin_provider: Option<Pubkey>,  // DePIN device PDA reference
}
```

### Sybil Detection

#### Pairwise Check

```
Given: identity(agent_A), identity(agent_B)
If: identity(agent_A).hash == identity(agent_B).hash
Then: SYBIL_DETECTED (same hardware, same code)
```

#### Cluster Check (Multi-Agent)

```
Given: identities = [agent_1, agent_2, ..., agent_n]
Group by: identity_hash
Clusters: groups with > 1 member
Flagged: all agents in clusters
```

### Registration Flow

```
1. Agent installs SAP SDK
2. SDK collects fingerprint components (with user consent)
3. Components hashed → deterministic identity_hash
4. identity_hash registered on Solana (Memo or PDA)
5. Any protocol can query: "Is this identity registered?"
6. Table/group checks: "Do any of these share identity?"
```

## Security Considerations

### Fingerprint Evasion

**Risk:** Attackers may spoof `os.cpus()`, `os.hostname()`, or MAC addresses.

**Mitigation:** TPM (Level 4) and DePIN (Level 5) provide hardware-rooted attestation that cannot be spoofed in software.

### Containerization

**Risk:** Docker/Kubernetes containers share host hardware. Multiple containers on one host produce similar (not identical) fingerprints.

**Mitigation:** Container-specific fingerprinting using cgroup IDs, container IDs, and mounted volume paths. DePIN attestation (Level 5) bypasses this by verifying the physical device.

### Privacy

**Property:** Hardware fingerprints are one-way hashed. Verifiers learn:
- ✅ "These two agents run on the same machine" (hash match)
- ❌ NOT the actual CPU model, hostname, IP, or memory

**Consent:** Agents MUST obtain user consent before collecting fingerprint data. The registration file SHOULD include a privacy disclosure.

### Replay Attacks

**Risk:** Attacker captures identity_hash and replays it.

**Mitigation:** Registration includes a timestamp and expiry. Verifiers check:
1. Identity is registered (on-chain)
2. Registration is not expired
3. Agent can re-derive the same hash on demand (live challenge)

## References

- [SAP-0001: Validation Protocol](SAP-0001-validation-protocol.md)
- [SAP-0003: DePIN Device Attestation](SAP-0003-depin-attestation.md)
- [ERC-8004: Trustless Agents](https://eips.ethereum.org/EIPS/eip-8004)
- TPM 2.0 Specification, Trusted Computing Group
- [MoltLaunch SDK](https://www.npmjs.com/package/@moltlaunch/sdk) — Reference implementation
