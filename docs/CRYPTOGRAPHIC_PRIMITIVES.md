# Cryptographic Primitives and zk-Proof Architecture

## AgDR-FullSystem v2.1 — Technical Specification

## Overview

AgDR-FullSystem v2.1 employs three cryptographic layers: BLAKE2b/BLAKE3 hashing for record keying and Merkle chain construction, Ed25519 digital signatures for authentication, and zero-knowledge proof stubs for verifiable computation.

## BLAKE2b / BLAKE3 Hashing

BLAKE2b is the primary hash function. Record keys use `hashlib.blake2b(f"{agent_id}:{time.time_ns()}".encode()).hexdigest()`. Merkle roots use `BLAKE2b(prev_root + JSON(record))` creating forward-secret chains. Output: 512 bits, collision resistance: 256 bits, speed ~1 GB/s, RFC 7693. BLAKE3 available for production with ~3x throughput.

## Ed25519 Digital Signatures

Ed25519 authenticates AKI capture records at Step 6. Key size: 256 bits, signature: 512 bits, ~76,000 verifications/sec, RFC 8032. Deterministic generation ensures reproducible verification.

## Zero-Knowledge Proof Architecture

Reference stub: `generate_zk_proof()` produces BLAKE2b digest, `verify_zk_proof()` returns True (interface contract). Production: 256-bit SMT depth aligns with gnark, circom, halo2, and zk-STARKs. Proofs must attest to coherence score correctness, PPP construction, Merkle root derivation, and reasoning trace existence without revealing the trace.

## Forward Secrecy

`BLAKE2b(prev_root || record_json)` ensures root N reveals nothing about root N-1. Combined with zk-proofs, enables verification without full chain access.

## For Implementors

Replace zk stubs before production. Protect Ed25519 keys in HSM/encrypted storage. Prefer BLAKE3 for parallel chain verification. Ensure deterministic zk verification for BFT consensus compatibility.

Built on AgDR v0.2 + AKI from accountability.ai
