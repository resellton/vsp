VSP — Voloweb Secure Protocol

A Post-Quantum, CA-Less Trust Architecture for the Modern Web

Version: 1.0

Date: November 28, 2025

Author: Soham Kumawat

Organization: Voloweb (Resellton)

Status: Experimental / Proposal

Abstract

The current trust model of the internet, reliant on centralized Certificate Authorities (CAs) and pre-quantum cryptography (RSA/ECC), is approaching obsolescence. The Voloweb Secure Protocol (VSP) proposes a decentralized, quantum-resistant alternative to SSL/TLS. By leveraging Kyber512 for key encapsulation, Dilithium2 for identity signatures, and a Multi-Node Verification (MNV) network, VSP eliminates single points of failure and ensures perfect forward secrecy. This paper outlines the VSP architecture, handshake logic, and the transition path from legacy PKI to a user-centric Web of Trust.

1. Introduction

For over two decades, the security of the web has relied on the X.509 Certificate Authority infrastructure. While successful in scaling the early internet, this model suffers from critical structural weaknesses:

Centralized Trust: A compromise of a single CA (e.g., DigiNotar, Comodo) threatens the integrity of the entire web.

Quantum Vulnerability: Standard key exchange algorithms (ECDH, RSA) will be broken by Shor’s algorithm within the next decade.

Privacy Leaks: Certificate Transparency logs and OCSP requests leak browsing habits to third parties.

VSP introduces a paradigm shift: trust is not "issued" by a corporation but "verified" by a decentralized consensus of independent nodes, secured by mathematics that quantum computers cannot easily break.

2. System Architecture

VSP replaces the traditional Client-Server-CA triangle with a decentralized mesh. The architecture consists of four primary components:

DIM (Decentralized Identity Mapping): An immutable ledger that maps Domain Names to Dilithium Public Keys.

VSP Daemon: A lightweight background service on the client and server that handles cryptographic offloading.

MNV (Multi-Node Verification): A network of geographically distributed nodes that witness and vouch for server identities.

Browser Integration: A client-side extension (Phase 1) or native implementation (Phase 2) that intercepts HTTP traffic and tunnels it through VSP.

2.1 Architectural Diagram

graph TD
    subgraph Client_Side [Client Environment]
        User[User / Browser] -->|Requests voloweb.io| Ext[VSP Extension]
        Ext <-->|Localhost API| Daemon[Client VSP Daemon]
    end

    subgraph The_Internet [Public Network]
        Daemon -->|1. Handshake| Server[Target Server]
        Daemon -->|2. Verify Identity| NodeA[Verification Node A]
        Daemon -->|2. Verify Identity| NodeB[Verification Node B]
        Daemon -->|2. Verify Identity| NodeC[Verification Node C]
    end

    subgraph Trust_Layer [Decentralized Infrastructure]
        NodeA -.->|Query| DIM[(DIM Ledger)]
        NodeB -.->|Query| DIM
        NodeC -.->|Query| DIM
    end

    classDef secure fill:#f9f,stroke:#333,stroke-width:2px;
    class Daemon,Server,DIM secure;



3. The VSP Handshake Protocol (PQ-KEH)

Unlike TLS 1.3, which relies on a pre-installed list of root certificates, VSP builds trust dynamically. The handshake uses Post-Quantum Keyless Ephemeral Handshake (PQ-KEH) to ensure that no long-term encryption keys ever exist on the server.

3.1 Handshake Logic

Hello: Client requests a secure channel and sends a quantum-safe nonce.

Identity: Server responds with its Dilithium Public Key and a signature of the nonce.

Challenge (MNV): The Client pauses and asks 3 random Verification Nodes: "Is this public key actually associated with voloweb.io?"

Consensus: If 2/3 nodes confirm the key matches the DIM record, the client proceeds.

Encapsulation: Client generates a shared secret, encapsulates it using Kyber512, and sends it to the server.

Tunnel: Both sides derive session keys (ChaCha20-Poly1305) and begin encrypted data transfer.

3.2 Sequence Diagram

sequenceDiagram
    participant C as Client (Browser)
    participant S as Server (VSP Host)
    participant MNV as Verification Nodes
    participant D as DIM Ledger

    Note over C,S: Phase 1: Identity Assertion
    C->>S: ClientHello (Nonce, PQ-Algo-List)
    S-->>C: ServerHello (Dilithium PubKey + Signed Nonce)

    Note over C,MNV: Phase 2: Decentralized Verification
    C->>MNV: Verify(Domain, PubKey)
    par Parallel Queries
        MNV->>D: Lookup Record
        D-->>MNV: Return Valid PubKey
    end
    MNV-->>C: Signed Proof (Valid/Invalid)

    alt Verification Failed
        C->>C: Abort Connection (Phishing Detected)
    else Verification Passed
        Note over C,S: Phase 3: Quantum Key Exchange
        C->>C: Generate Shared Secret
        C->>C: Encapsulate with Kyber512
        C->>S: ClientFinish (Kyber Ciphertext)
        S->>S: Decapsulate with Kyber Private Key
        
        Note over C,S: Phase 4: Encrypted Tunnel
        C<->S: AEAD Encrypted Stream (ChaCha20-Poly1305)
    end



4. Cryptographic Primitives

VSP strictly adheres to the NIST Post-Quantum Cryptography Standardization outcomes (2024).

|

| Component | Algorithm | Purpose |
| Key Encapsulation | Kyber512 (ML-KEM) | Generating shared secrets. Chosen for high speed and small packet size compared to McEliece. |
| Digital Signatures | Dilithium2 (ML-DSA) | Verifying server identity (authentication). Replaces RSA/ECDSA. |
| Symmetric Encryption | ChaCha20-Poly1305 | Encrypting the actual data stream. High performance on mobile/IoT devices without AES hardware acceleration. |
| Hashing | BLAKE3 | Integrity checks and Merkle Tree generation for the DIM ledger. |

5. Performance Benchmarks

Preliminary tests indicate that VSP provides robust security with minimal latency overhead, utilizing lightweight primitives optimized for modern hardware.

Kyber512 Encapsulation: < 0.5ms

Dilithium Signature Verification: < 1ms

VSP Handshake Total: < 40ms + MNV RTT (Round Trip Time)

Note: Benchmarks are estimated based on single-core performance on standard x86_64 server hardware.

6. Comparative Analysis: VSP vs. TLS 1.3

VSP offers significant structural advantages over the legacy TLS 1.3 protocol, particularly regarding long-term trust and quantum resistance.

| Feature | TLS 1.3 | VSP |
| PQ-safe | ❌ No | ✔ Yes |
| CA dependency | ✔ Yes | ❌ No |
| Identity storage | Certificate | DIM Ledger |
| Trust model | Hierarchical | Decentralized |
| Forward secrecy | Partial | Complete (ephemeral-only) |

7. Deployment & Compatibility

7.1 The "Dual-Stack" Strategy

To ensure adoption does not break the user experience, VSP is designed to run alongside TLS.

Server Side: The VSP Daemon listens on UDP/4433 (QUIC-based transport), while Nginx/Apache continues to serve standard HTTPS on TCP/443.

Client Side: If the VSP handshake fails (e.g., firewall blocks UDP), the browser extension transparently falls back to standard TLS 1.3.

7.2 Server Requirements

OS: Linux (Ubuntu/Debian/CentOS)

Resources: Minimal (~50MB RAM for Daemon)

Keys: Ephemeral keys are generated in RAM. The only persistent file is the identity.key (Dilithium) which is never used for encryption, only signing.

8. Challenges & Mitigations

| Challenge | Risk | Mitigation Strategy |
| Latency | MNV queries add RTT (Round Trip Time). | Optimistic Loading: Start loading non-sensitive assets immediately while MNV verifies in the background. |
| Sybil Attacks | Attacker floods MNV network with fake nodes. | Reputation Staking: Verification nodes must stake tokens or prove long-term uptime to vote on identities. |
| Middleboxes | Corporate firewalls blocking unknown protocols. | Port Masquerading: VSP packets mimic standard DTLS or QUIC traffic to pass through Deep Packet Inspection (DPI). |

9. Conclusion

VSP is not just an upgrade; it is a reconstruction of internet trust. By removing the dependency on Certificate Authorities and proactively adopting post-quantum cryptography, Voloweb aims to secure the next generation of the internet against threats that do not yet exist, but are inevitable.

VSP: Trust Code, Not Corporations.

© 2025 Voloweb. All Rights Reserved.
