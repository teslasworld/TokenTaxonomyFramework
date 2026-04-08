# Getting Started - VRE-PARFE-HydroRE Extension Set

**IWA dMRV v3.0 Extension Set | Version 1.0**
**Issuing Authority:** Three T's (Mauritius) Limited | Registration No: C19166743
**Verification Layer:** Hedera Consensus Service (HCS)

---

## 1. Introduction

This Extension Set defines how to measure, report, and verify renewable energy generation using **VRE PARFE** — the Verified Renewable Energy Provenance And Regulatory Framework Engine.

VRE PARFE was built by an engineer who owns a run-of-river hydropower station in Mauritius. The core problem it solves is simple: **how do you prove that the energy reading from your meter is real, unaltered, and happened when you say it did?**

The answer VRE PARFE gives is:

1. A physical hardware meter (ZEM63) takes the reading.
2. An IoT gateway (ZGW10) cryptographically signs it with a key that cannot be extracted.
3. The signed reading is written to the **Hedera Consensus Service** — a public ledger where records are ordered by consensus timestamp and cannot be deleted, modified, or reordered.
4. When enough verified energy accumulates, an NFT Digital Passport is minted on the **Hedera Token Service** — a permanent, publicly auditable certificate.

Nobody has to trust the issuing authority. Anyone can replay the ledger from record 1 and verify every signature independently.

**This is not governed by Verra, Gold Standard, or any third-party registry.** The Hedera ledger is the verification layer. The issuing authority is Three T's (Mauritius) Limited, operating as a sovereign, self-issued methodology.

---

## 2. The Physical System

```
[ZEM63 Energy Meter]
  |  RS-485 / Modbus (interval kWh readings)
  v
[ZGW10 IoT Gateway]
  |  - Authorises the sensor against the SENSOR_LINK HCS record
  |  - Checks the reading against the physics gate (kwh <= physicsLimitKwh)
  |  - Signs the reading: Ed25519 signature over SHA-256(stable JSON)
  |  - Embeds the full public key PEM in every message
  |  MQTT => edge-notary software
  v
[Hedera Consensus Service (HCS)]
  |  - TELEMETRY records accumulate on the licensee's topic
  |  - Sequence numbers provide an unambiguous ordering
  |  - Records are immutable — no deletion, no amendment
  |  Mirror node polling (every 5 seconds)
  v
[Mission Control - Accumulator + Minter]
  |  - Reads TELEMETRY records from HCS mirror
  |  - Accumulates kWh per sensor/token class
  |  - When threshold is reached, mints an NFT on HTS
  |  - Writes a MINT_EVENT to HCS (the audit trail of the mint)
  v
[Hedera Token Service (HTS) - NFT Digital Passport]
     - Immutable on-chain certificate
     - Links back to the HCS topic + sequence range that justifies it
     - Contains: measurement value, GEF, carbon offset, ISO standards
```

---

## 3. Why This Is Different

### 3.1 Physics Validation Gate

Before any reading reaches the Hedera ledger, the edge notary checks it against a physical limit:

```
physicsLimitKwh = capacityMW x 1000 x readingIntervalHours
```

For the reference implementation (1 MW station, ~12-minute intervals):

```
physicsLimitKwh = 1.0 x 1000 x 0.2 = 200 kWh
```

**Any reading above 200 kWh is rejected before HCS submission.** A Physics Gate Exception record is written instead. The `capacityMW` limit is stored in the **GATEWAY_DEPLOY HCS record** — set once by the issuing authority at gateway commissioning and immutable after that.

### 3.2 Schema Immutability

The data format used to produce every TELEMETRY record is a **frozen object** in the protocol code (`SCHEMAS_V2.js`). It cannot be mutated at runtime. When you verify a signature on a TELEMETRY record, you know with certainty what fields were present and what they mean. The schema is the contract. It is locked.

---

## 4. The Verification Chain

Anyone — auditor, regulator, buyer, or curious member of the public — can verify a VRE PARFE certificate with no special tools, no account, and no trust in the issuing authority.

**Step 1** — Find the MINT_EVENT. Every minted NFT contains a `proof` block with `topic_id`, `seq_from`, and `seq_to`.

**Step 2** — Go to [https://hashscan.io](https://hashscan.io) and enter the topic ID. Read every TELEMETRY record between `seq_from` and `seq_to`.

**Step 3** — Each TELEMETRY record contains `sig.payloadHash`, `sig.signature`, and `sig.pub` (full Ed25519 public key PEM). Recompute the hash, verify the signature.

**Step 4** — Confirm every `kwh` value is <= `physicsLimitKwh` for this gateway.

**Step 5** — Sum the `kwh` values. Confirm they match `measurementValue` in the MINT_EVENT. Multiply by `gef` (0.9908 for Mauritius) to confirm `co2Amount`.

If all five steps pass, the certificate is valid. No trust required.

---

## 5. The Three Token Types

| Token | Full Name | Unit | One token equals | ISO Standard |
|---|---|---|---|---|
| **REC** | Renewable Energy Certificate | kWh | 100 kWh of verified renewable generation | ISO 14064-1:2018 |
| **H2O** | Water Stewardship Certificate | m3 | 5,000 litres of verified water flow | ISO 14046:2014 |
| **CO2** | Carbon Offset Certificate | kgCO2e | kWh x GEF (0.9908 for Mauritius) | ISO 14064-1:2018 |

---

## 6. References

| Resource | Link |
|---|---|
| IWA dMRV Framework v3.0 | https://interworkalliance.github.io/TokenTaxonomyFramework/dmrv/spec/index.html |
| IWA Token Taxonomy Framework | https://github.com/InterWorkAlliance/TokenTaxonomyFramework |
| Reference REC topic on Hashscan | https://hashscan.io/testnet/topic/0.0.7498906 |
| Global governance topic | https://hashscan.io/testnet/topic/0.0.7546907 |
| VRE PARFE source repository | https://github.com/teslasworld/VP-preprod |

---
## Live Reference Data — Hedera Testnet

The following Hedera Consensus Service topics contain live reference data for this Extension Set. All topics are on Hedera **testnet** and publicly readable without authentication at [hashscan.io](https://hashscan.io/testnet).

> **Note on testnet:** The protocol operates on Hedera testnet for pre-production validation. Tokens carry no monetary value. The testnet is used to prove the complete end-to-end process — from hardware measurement through physics validation, cryptographic signing, ledger anchoring, and token issuance — before mainnet deployment. The integrity properties demonstrated here are identical on mainnet.

---

### Topic 0.0.7498906 — Asset Telemetry & Token Issuance
**Purpose:** Primary REC/H2O/CO2 token minting stream.

Every message is a complete asset record containing:
- Issuer identity (Three T's (Mauritius) Limited, reg. C19166743)
- Asset class (REC / H2O / CO2) and unique serial number
- Measurement value (kWh) and Grid Emission Factor (0.9908 kgCO₂e/kWh)
- Regulatory classifications: ISO 14064-1:2018, EU-CBAM-COMPLIANT, US-GAAP Scope 2, CORSIA-Eligible
- Fiscal policy split: PRIMARY 70/20/7/3% | SECONDARY 4/4/1% PERPETUITY
- Verification method: Hedera Consensus Service (HCS)

**Hashscan:** https://hashscan.io/testnet/topic/0.0.7498906

---

### Topic 0.0.7546907 — Sovereign Governance Spine
**Purpose:** Immutable governance and commissioning record.

Contains the complete lifecycle of the licensee, gateway, and sensor registration:
- `LICENSE_GRANT` — licensee identity, jurisdiction, asset type, agreement reference
- `GATEWAY_DEPLOY` — gateway hardware MAC, location, capacityMW, Ed25519 public key, commissioning date
- `SENSOR_LINK` — sensor serial number, model (ZEM-63), manufacturer (EpiSensor), calibration date, Ed25519 public key, ACTIVE status

Every governance record is:
- Signed by the Sovereign DID (`did:three-ts:grandfather:0.0.7479746`)
- Immutable once written — cannot be altered or deleted
- Publicly verifiable without any trust relationship with the issuing authority

**Hashscan:** https://hashscan.io/testnet/topic/0.0.7546907

---

### Topic 0.0.8480236 — Edge-Notary Signed Telemetry (V2.2)
**Purpose:** Cryptographically signed hardware telemetry — the primary evidence layer for ISSA 5000 assurance.

This is the most technically significant topic for independent verification. Every message contains:
- Raw hardware measurement (`kwh`) from the ZEM-63 revenue-grade meter
- Hardware sequence number (`seq`) for gap detection
- **Full Ed25519 public key PEM embedded in every message** — no external key registry required
- **SHA-256 payload hash** of the stable-stringified telemetry
- **Ed25519 signature** over the payload hash, produced by the ZGW10 gateway's hardware signing key
- Algorithm declaration: `Ed25519+SHA256(stable-json)`

**Independent verification procedure:**
Any party can verify any record by:
1. Retrieving the message from the Hedera mirror node
2. Recomputing the SHA-256 hash of the stable-stringified payload
3. Verifying the Ed25519 signature against the embedded `sig.pub` public key
4. Confirming the hardware sequence number is monotonically increasing (gap detection)

No account, API key, or trust relationship with Three T's (Mauritius) Limited is required at any step.

**Hashscan:** https://hashscan.io/testnet/topic/0.0.8480236

---

### The Verification Chain — Summary

```
ZEM-63 meter (hardware measurement)
    ↓
ZGW10 gateway (physics gate → Ed25519 sign → SHA-256 hash)
    ↓
Topic 0.0.8480236 (signed telemetry — primary evidence)
    ↓
Topic 0.0.7546907 (governance spine — LICENSE_GRANT, GATEWAY_DEPLOY, SENSOR_LINK)
    ↓
Topic 0.0.7498906 (token issuance — REC/H2O/CO2 mint events)
    ↓
HTS NFT (proof-carrying token with seq_from/seq_to reference back to 0.0.8480236)
```

The chain is complete, public, immutable, and self-verifying at every layer.


*VRE-PARFE-HydroRE Extension Set v1.0 | Three T's (Mauritius) Limited | 2026*
