BTCv33 — Post-Quantum Bitcoin Sidechain

Post-quantum, sharded, privacy-preserving blockchain with two-way Bitcoin peg.
BTCv33 is the latest revision of the BTCv32 codebase, integrating seven targeted improvements (S1–S7) that harden cryptographic security to NIST FIPS 203/204/205 standards, eliminate pre-quantum primitives from all value-bearing paths, and lay the groundwork for autonomous governance-driven cryptographic agility.

Table of Contents

Overview

What's New in BTCv33

Post-Quantum Security Architecture

Novelty & Differentiating Features

Cryptographic Primitives Reference

Project Structure

Installation & Setup

Configuration Reference

Migration Guide (BTCv32 → BTCv33)

Governance & Upgrade Lifecycle

HNDL Key Rotation

API Endpoints

Testing & Verification

Roadmap & Future Upgrades

Security Considerations

Contributing

License

Overview

BTCv33 is a production-grade Python/Rust hybrid blockchain node that extends Bitcoin's security model with post-quantum cryptographic primitives standardised under NIST's 2024 PQC standards (FIPS 203, 204, and 205). It functions as a two-way pegged Bitcoin sidechain capable of:

Stealth-only transaction outputs — all value outputs use DKSAP (Dual-Key Stealth Address Protocol) with ML-KEM-768 or hybrid EC+PQ encryption

Sharded block production — horizontal scalability via per-shard block headers

Zero-knowledge balance proofs — spend-without-reveal using ZKP circuits

On-chain governance — soft/hard cryptographic suite transitions driven by real-time telemetry

BTCv33 raises the minimum post-quantum security floor to ML-KEM-768 across all value outputs, introduces a transcript-bound hybrid KEM combiner, and delivers a high-performance Rust-backed ML-KEM implementation with optional GPU batch acceleration.

What's New in BTCv33

The following seven improvements were integrated over the BTCv32 base:

IDAreaChangeS1CryptographyTranscript-bound hybrid KEM combiner (X-Wing style, NIST SP 800-56C)S2CryptographyML-KEM-512 deprecated for all value outputs; ML-KEM-768 raised to hard default and floorS3Key ManagementDeterministic ML-KEM keygen via FIPS 203 §7.1 derandomized API + KAT vectorsS4GovernanceAutomatic SOFT→HARD telemetry trigger; composite dual-signatures; HNDL mitigationS5Performancepyoqs replaced with ml-kem (pure Rust/pyo3); GPU batch KEM hooks; tunable Bloom filterS6GovernanceMiner incentive hooks for PQ migration; versioned P2P capability announcements; CBOM generationS7MiscellaneousStreamlined hybrid codegen; suite metadata API; MIGRATION.md 2026 guidance 

Post-Quantum Security Architecture

Threat Model

BTCv33 is designed to be secure against both classical and quantum adversaries, with particular focus on the Harvest Now, Decrypt Later (HNDL) attack vector — where a quantum-capable adversary records encrypted transactions today and decrypts them once a sufficiently powerful quantum computer becomes available.

Layered PQ Defence

┌──────────────────────────────────────────────────────────────────┐ │ BTCv33 Security Stack │ ├──────────────────────────────────────────────────────────────────┤ │ Key Encapsulation │ ML-KEM-768 (FIPS 203) [floor] │ │ │ ML-KEM-1024 (optional) [high-value] │ ├──────────────────────────────────────────────────────────────────┤ │ Digital Signatures │ ML-DSA-65 (FIPS 204) [primary] │ │ │ Composite ECDSA+ML-DSA-65 [transition] │ │ │ Falcon-512 / Dilithium3 [legacy] │ ├──────────────────────────────────────────────────────────────────┤ │ Hybrid KEM Combiner │ X-Wing / SP 800-56C transcript-bound │ │ (S1) │ HKDF with domain separation │ ├──────────────────────────────────────────────────────────────────┤ │ Key Derivation │ FIPS 203 §7.1 derandomized keygen │ │ (S3) │ Deterministic seed → ML-KEM keypair │ ├──────────────────────────────────────────────────────────────────┤ │ Transaction Hashing │ BLAKE3 (primary) / SHA-256 (fallback) │ ├──────────────────────────────────────────────────────────────────┤ │ Balance Privacy │ Zero-knowledge proofs (ZKP circuits) │ ├──────────────────────────────────────────────────────────────────┤ │ Stealth Addressing │ DKSAP with ephemeral EC + ML-KEM │ └──────────────────────────────────────────────────────────────────┘ 

Improvement S1 — Transcript-Bound Hybrid KEM Combiner

The core key exchange in stealth addressing now uses a transcript-bound combiner (hybrid_combiner.py) inspired by the X-Wing construction and conformant with NIST SP 800-56C. Rather than naively concatenating KEM shared secrets, the combiner feeds both the classical EC shared secret and the ML-KEM-768 shared secret, along with a protocol transcript (ciphertexts, public keys, and domain label), into a single HKDF invocation:

combined_secret = HKDF( IKM = ec_shared_secret || mlkem_shared_secret, info = "BTCv33-hybrid-v1" || ec_eph_pk || mlkem_ciphertext || mlkem_pk, ... ) 

This construction guarantees that the combined secret is secure if either the classical EC component or the ML-KEM component is unbroken — providing a smooth security migration with no reduction in pre-quantum security.

Improvement S2 — ML-KEM-768 Security Floor

ML-KEM-512 (targeting NIST Security Category 1, roughly AES-128 equivalent) is deprecated for all value-bearing outputs as of BTCv33. The new policy is:

Value outputs (value > DUST_VALUE_THRESHOLD): minimum pq_level=768 enforced at __post_init__; violation raises PQSecurityFloorViolationError

Dust outputs (value <= 0.0001 BTC): ML-KEM-512 permitted for performance

Suite 3 (ML-KEM-512 general): marked as dust-only; any suite-3 value output is rejected by the mempool

ML-KEM-768 targets NIST Security Category 3 (~AES-192 equivalent post-quantum), providing comfortable margins against near-term and medium-term quantum hardware.

Improvement S3 — Deterministic Key Generation

BTCv33 implements FIPS 203 §7.1's derandomized ML-KEM.KeyGen_internal(d, z) interface, replacing the prior call to os.urandom() within key generation. This means:

ML-KEM keypairs are now fully deterministic given a seed (d, z) — identical to how HD wallets derive keys from BIP-39 seeds

Known Answer Test (KAT) vectors (scripts/kat_vectors.py) can validate correct key generation against NIST reference implementations

Wallet backups only require storing the seed — ML-KEM keys are re-derived, not stored separately

Improvement S4 — Composite Dual Signatures & HNDL Mitigation

Transactions now support a composite signature type (SIG_TYPE_COMPOSITE_ECDSA_MLDSA65 = 5) in which both a classical ECDSA signature and an ML-DSA-65 signature are attached to each input:

class TransactionInput: signature: Optional[str] # Primary: ML-DSA-65 public_key: Optional[str] composite_sig: Optional[str] # S4: Classical ECDSA composite_pub_key: Optional[str] 

This dual-signature scheme is crucial for the transition period: legacy nodes can verify the classical component while upgraded nodes verify both, providing backward compatibility without sacrificing PQ security for upgraded participants.

HNDL mitigation is enforced at the wallet and chain level: high-value UTXOs (>= 1.0 BTC equivalent) must be rotated within 52,560 blocks (~1 year). Stale UTXOs raise HNDLRotationRequiredError and are blocked from spending until re-encrypted under a fresh ML-KEM keypair.

Novelty & Differentiating Features

1. Governance-Driven Cryptographic Agility

Unlike hardcoded cryptographic systems, BTCv33 implements on-chain cryptographic suite governance. Any node operator or miner can propose a new suite via a governance transaction. The chain autonomously tracks the fraction of PQ-compliant outputs in recent blocks (pq_migration_ratio) and automatically escalates from SOFT to HARD enforcement once a threshold is reached — without requiring a scheduled hard fork date:

pq_migration_ratio >= 0.95 for 1,000 consecutive blocks → HARD enforcement fires automatically → Suite 0 (EC-only) and Suite 3 (ML-KEM-512) deprecated 

This telemetry-triggered upgrade model is novel in the blockchain space and enables smooth, organic migration without coordination failures.

2. Cryptographic Bill of Materials (CBOM)

Inspired by Software Bills of Materials (SBOMs), BTCv33 introduces a Cryptographic Bill of Materials (cbom/generate_cbom.py) exposed via GET /cbom. The CBOM documents every cryptographic primitive in use on a running node: algorithm names, parameter sets, NIST standard references, deprecation status, and suite identifiers. This is critical for:

Regulatory compliance audits

Identifying deprecated primitives before they become a vulnerability

Coordinating network-wide upgrades across heterogeneous node deployments

3. Transcript-Bound Hybrid KEM (X-Wing Style)

The hybrid combiner (S1) is not merely a concatenation — it is a formally analysed construction that binds both KEM outputs to the full protocol transcript, preventing mix-and-match attacks where an adversary substitutes one ciphertext component for another. This places BTCv33 ahead of most blockchain PQ implementations which either replace EC entirely (losing classical security during transition) or concatenate naively (lacking composable security proofs).

4. Rust-Backed ML-KEM with GPU Batch Acceleration

The pyoqs dependency (a C FFI wrapper with non-trivial supply-chain risk) is replaced with ml-kem, a pure-Rust crate bound to Python via pyo3. This delivers:

Faster key generation and encapsulation (Rust vs C-via-Python)

A smaller, auditable dependency tree

Optional GPU batch processing hooks for high-throughput nodes processing many KEM operations per block

5. Versioned P2P Capability Announcements

The P2P layer (S6) introduces VersionedCapabilityAnnouncement messages. Nodes now broadcast their supported suite IDs and software version at handshake time, and the protocol throttles legacy peers rather than disconnecting them abruptly. This allows the network to observe the actual distribution of PQ-ready nodes in real time and informs governance telemetry.

6. Tunable Bloom Filter for Stealth Scanning

Block headers contain a Bloom filter over stealth output view tags, enabling light wallets to rapidly rule out blocks containing no relevant outputs without downloading full transaction data. BTCv33 makes the filter parameters m (bit length) and k (hash count) tunable and stored in the block header, so nodes can adapt the filter to the density of stealth outputs in the network:

At m=512, k=4, n=100 stealth outputs: ~0.4% false-positive rate vs old m=256, k=3: ~3% false-positive rate 

Cryptographic Primitives Reference

PrimitiveStandardSecurity LevelStatusML-KEM-768FIPS 203NIST Cat. 3 (~AES-192)✅ DefaultML-KEM-1024FIPS 203NIST Cat. 5 (~AES-256)✅ OptionalML-KEM-512FIPS 203NIST Cat. 1 (~AES-128)⚠️ Dust-onlyML-DSA-65FIPS 204NIST Cat. 3✅ Primary SignatureFalcon-512FIPS 206 (draft)NIST Cat. 1⚠️ LegacyDilithium3(pre-FIPS 204)NIST Cat. 3⚠️ LegacyECDSA (secp256k1)—~128-bit classical⚠️ Composite onlyBLAKE3—256-bit✅ Primary HashSHA-256FIPS 180-4128-bit collision✅ Fallback HashHKDF-SHA256RFC 5869—✅ KDF in combiner 

Project Structure

btcv32/ ├── pyproject.toml ├── setup.py ← ml-kem extras_require (replaces pyoqs) ├── requirements.txt ├── requirements-dev.txt ├── docker-compose.yml ├── Dockerfile ← Multi-stage; ARG CRYPTO_SUITE; CBOM hook ├── MIGRATION.md ← 2026 FIPS 203/204/205 guidance ├── cbom/ │ └── generate_cbom.py ← NEW: Cryptographic Bill of Materials ├── scripts/ │ ├── setup_circuits.sh │ ├── benchmark_scan.py │ ├── wallet_upgrade.py │ └── kat_vectors.py ← NEW: Known Answer Test vectors └── btcv32/ ├── core/ │ ├── transaction.py ← composite dual-sig support (S4) │ ├── block.py ← tunable Bloom filter m/k (S5) │ ├── chain.py ← auto SOFT→HARD trigger; HNDL rotation (S4) │ ├── mempool.py │ └── exceptions.py ├── crypto/ │ ├── suite_registry.py ← ML-KEM-768 hard floor; metadata API │ ├── pqc.py ← ml-kem (Rust) backend; GPU batch │ ├── hybrid_combiner.py ← NEW: transcript-bound HKDF combiner │ ├── stealth_hybrid.py ← uses new combiner │ ├── stealth_pq.py ← deterministic keygen; KAT hooks │ ├── keystore.py ← deterministic ML-KEM seed API │ ├── stealth.py │ ├── stealth_ec.py │ ├── merkle.py │ └── zkp.py ├── network/ │ ├── p2p.py ← versioned capability announcements │ ├── messages.py ← VersionedCapabilityAnnouncement │ ├── dht.py │ ├── ratelimit.py │ └── sync.py ├── governance/ │ ├── proposals.py ← auto SOFT→HARD telemetry; miner incentives │ ├── voting.py │ └── executor.py ├── wallet/ │ ├── stealth_wallet.py ← HNDL key-rotation windows │ └── seed_wallet.py └── api/ ├── routes.py ← suite metadata + CBOM endpoints ├── server.py ├── auth.py └── schemas.py 

Installation & Setup

Prerequisites

Python 3.11+

Rust toolchain (for ml-kem pyo3 backend)

Docker (recommended for production deployments)

Standard Install

git clone https://github.com/your-org/btcv33.git cd btcv33 # Install with ML-KEM Rust backend (recommended) pip install -e ".[ml-kem]" # Or install base dependencies only (falls back to pyoqs) pip install -r requirements.txt # Install dev dependencies pip install -r requirements-dev.txt # Set up ZKP circuits bash scripts/setup_circuits.sh 

Docker (Recommended)

# Build with PQ ML-KEM-768 suite docker build \ --build-arg PQ=true \ --build-arg CRYPTO_SUITE=mlkem768 \ -t btcv33 . # Start node docker-compose up -d 

Configuration Reference

Environment VariableDefaultDescriptionBTCV32_PQ_MIN_KEM_LEVEL768Minimum ML-KEM security level for value outputsBTCV32_PREFERRED_SUITE_ID2Active crypto suite IDBTCV32_SUPPORTED_SUITE_IDS1,2,4,5Suites accepted from peersBTCV32_STEALTH_PQ_MANDATORYfalseForce PQ stealth outputs onlyBTCV32_AUTO_HARD_PQ_RATIO0.95PQ migration ratio to trigger HARD enforcementBTCV32_AUTO_HARD_WINDOW1000Blocks window for auto HARD triggerBTCV32_DUST_THRESHOLD0.0001BTC value below which ML-KEM-512 is still allowedBTCV32_BLOOM_M512Bloom filter bit length per block headerBTCV32_BLOOM_K4Bloom filter hash function countBTCV32_CHAIN_IDbtcv32-mainnet-1Chain identifier for replay protection 

Migration Guide (BTCv32 → BTCv33)

Phase 1 — Node Upgrade

pip install "btcv33[ml-kem]" export BTCV32_PQ_MIN_KEM_LEVEL=768 

Verify the Rust backend is active:

pip show ml-kem 

Phase 2 — SOFT Enforcement

export BTCV32_PREFERRED_SUITE_ID=2 export BTCV32_AUTO_HARD_PQ_RATIO=0.95 export BTCV32_AUTO_HARD_WINDOW=1000 

Submit a governance proposal to activate Suite 2:

POST /governance/suite-proposal { "proposal_id": "activate-suite-2-v33", "new_suite_id": 2, "old_suite_id": 0, "soft_activation_height": <current_height + 1000>, "hard_cutoff_height": <current_height + 12000>, "overlap_blocks": 10000, "auto_hard_pq_ratio": 0.95, "auto_hard_window_blocks": 1000, "miner_incentive_bps": 10 } 

Phase 3 — HARD Enforcement (Auto or Manual)

HARD enforcement fires automatically once pq_migration_ratio >= 0.95 for 1,000 consecutive blocks. To trigger manually:

export BTCV32_STEALTH_PQ_MANDATORY=true export BTCV32_PREFERRED_SUITE_ID=2 export BTCV32_SUPPORTED_SUITE_IDS=1,2,4,5 

Suite 0 (EC-only) and Suite 3 (ML-KEM-512 for value outputs) are deprecated after HARD enforcement.

Governance & Upgrade Lifecycle

Suite Proposal Submitted │ ▼ SOFT Activation (new_suite preferred, old_suite still accepted) │ ├─── Monitor: GET /metrics/migration (pq_migration_ratio) │ pq_migration_ratio >= 0.95 for 1,000 blocks? │ YES ▼ NO Auto HARD Trigger Continue SOFT period │ ▼ HARD Enforcement (old suite rejected by mempool) │ ▼ Suite deprecated, CBOM updated 

Miner incentive hooks (miner_incentive_bps) provide a basis-point fee bonus to miners who include PQ-compliant transactions, accelerating network migration organically.

HNDL Key Rotation

High-value UTXOs (>= 1.0 BTC equivalent) must be re-encrypted under a fresh ML-KEM keypair within 52,560 blocks (~1 year) to defend against Harvest Now, Decrypt Later attacks.

python scripts/wallet_upgrade.py \ --pq \ --pq-level 768 \ --rotate-hndl \ --utxo-tx-id <TXID> \ --utxo-index <IDX> \ --scan-priv $BTCV32_STEALTH_SCAN_PRIV \ --spend-priv $BTCV32_STEALTH_SPEND_PRIV 

Monitor UTXO ages at:

GET /wallet/hndl-status 

UTXOs approaching or exceeding the rotation window will appear as rotation_required: true in the response.

API Endpoints

MethodPathDescriptionGET/crypto/suitesList all registered crypto suites, activation heights, and deprecation statusGET/cbomCryptographic Bill of Materials for this nodeGET/metrics/migrationReal-time pq_migration_ratio and auto-HARD trigger statusGET/network/capabilitiesPeer capability matrix (suite IDs, software versions)GET/wallet/hndl-statusHNDL rotation status for all wallet UTXOsPOST/governance/suite-proposalSubmit a new crypto suite activation proposalPOST/transactionsBroadcast a new transactionGET/blocks/:heightFetch block by height 

Testing & Verification

Quick Verification Checklist

[ ] BTCV32_PQ_MIN_KEM_LEVEL=768 set on all nodes

[ ] ml-kem package installed (pip show ml-kem)

[ ] Docker image built with --build-arg PQ=true --build-arg CRYPTO_SUITE=mlkem768

[ ] GET /crypto/suites shows suite_id=2 active and suite_id=3 marked as dust-only

[ ] GET /cbom returns a valid CBOM document

[ ] GET /metrics/migration shows pq_migration_ratio >= 0.95

[ ] GET /network/capabilities shows all peers at software_version >= 33.0.0

[ ] HNDL rotation scheduled for all high-value UTXOs

[ ] KAT vectors generated and passing (pytest tests/kat/)

[ ] BTCV32_AUTO_HARD_PQ_RATIO=0.95 configured

Running Tests

# Full test suite pytest # KAT (Known Answer Test) vectors only pytest tests/kat/ # Generate fresh KAT vectors python scripts/kat_vectors.py # Benchmark PQ operations python scripts/benchmark_scan.py 

Roadmap & Future Upgrades

Near-Term (v34 — 2026 Q3)

SLH-DSA (FIPS 205) Signature Support Integrate SPHINCS+-based stateless hash-based signatures as an alternative to ML-DSA-65. SLH-DSA provides security guarantees based purely on hash function security (minimal structural assumptions), making it a valuable complement for extremely long-lived UTXOs where lattice assumptions may require re-evaluation.

ML-KEM-1024 Mandatory for High-Value UTXOs Raise the floor for UTXOs above a configurable threshold (e.g., 10 BTC) to ML-KEM-1024 (NIST Category 5, ~AES-256 post-quantum). The groundwork is already in place via pq_level field support for 1024.

Falcon-512 → ML-DSA Migration Tooling Automated wallet tooling to re-sign all legacy Falcon-512 or Dilithium3 inputs with ML-DSA-65, retiring the remaining pre-standard signature types.

Medium-Term (v35 — 2027)

ZKP Circuit Upgrades for PQ Commitments Current ZKP balance proofs use classical Pedersen commitments. Future work will integrate Lattice-based or hash-based commitment schemes compatible with the PQ signer, ensuring the entire privacy layer is quantum-resistant end-to-end.

Threshold ML-KEM with FROST-style DKG Enable multi-signature stealth addresses using a threshold variant of ML-KEM, allowing m-of-n key holders to collaboratively decrypt incoming stealth payments. This removes single-key custodial risk for institutional participants.

Layer 2 PQ Payment Channels Adapt the HTLC-based payment channel design to use ML-DSA-65 for channel state signatures and ML-KEM-768 for ephemeral per-channel key exchange, enabling quantum-resistant Lightning-style payment routing on top of BTCv33.

Cross-Chain PQ Atomic Swaps Develop HTLC templates compatible with both BTCv33's ML-KEM stealth outputs and standard Bitcoin timelocks, enabling trustless atomic swaps between BTCv33 and mainchain Bitcoin without exposing either side to quantum threats.

Long-Term (v36+ — 2028 and beyond)

Recursive ZK-SNARKs for Block Compression Integrate recursive proof systems (e.g., Nova, Plonky3) to compress shard state transitions into a single verifiable proof, enabling ultra-light client verification and cross-shard atomicity without full state download.

Hybrid Governance DAO with PQ Voting Keys Replace ECDSA-signed governance votes with ML-DSA-65-signed votes, and introduce time-locked voting commitments to prevent last-minute vote manipulation while maintaining governance throughput.

Post-Quantum BLS Aggregate Signatures Research and prototype lattice-based signature aggregation to reduce block validation overhead at high transaction volumes, analogous to BLS aggregation in Ethereum but on a PQ-secure basis.

Formal Verification of Hybrid Combiner Commission machine-checked security proofs (e.g., in EasyCrypt or ProVerif) for the X-Wing-style hybrid combiner to provide cryptographic assurance beyond implementation-level testing.

CBOM Standardisation & Interoperability Contribute BTCv33's CBOM schema to emerging IETF/NIST cryptographic inventory standards, enabling cross-project tooling for PQ readiness audits across the broader blockchain ecosystem.

Security Considerations

Known Limitations

ZKP layer: Balance proofs currently rely on classical elliptic curve commitments. The ZKP subsystem is not yet fully post-quantum and should not be relied upon for quantum-adversarial privacy until v35+ work is complete.

Dust outputs with ML-KEM-512: The dust threshold exemption (value <= 0.0001 BTC) means a small fraction of outputs remain at Category 1 security. Operators who require uniform Category 3+ security should set BTCV32_DUST_THRESHOLD=0.0 to eliminate this exemption.

Classical ECDSA in composite signatures: During the transition period, the composite dual-signature scheme retains ECDSA signatures. These are quantum-vulnerable; once HARD enforcement completes, operators should disable composite mode and use ML-DSA-65 exclusively.

Responsible Disclosure

Security vulnerabilities should be reported to the security contact listed in SECURITY.md. Please do not file public GitHub issues for suspected vulnerabilities.

Contributing

Contributions are welcome. Please read CONTRIBUTING.md before submitting pull requests. All cryptographic changes must be accompanied by:

Updated KAT vectors (scripts/kat_vectors.py)

Updated CBOM schema if new primitives are introduced

A rationale referencing the relevant NIST standard or IETF RFC

Passing pytest tests/ on both the CPython and Rust-backend configurations

License

See LICENSE for details.

BTCv33 — securing Bitcoin's future against the quantum horizon.

