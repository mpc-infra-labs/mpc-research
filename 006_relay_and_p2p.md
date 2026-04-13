# Relay, P2P, and Hybrid Communication Architectures for MPC Systems

## 0. Scope

This note covers:

- relay, direct P2P, and hybrid communication topologies for MPC systems
- the main architectural tradeoffs of each topology
- which topology is more common in institutional systems
- what `LFDT-cggmp24`, `synedrion`, and `manul` do and do not provide at the communication layer
- communication-layer concerns that remain outside the MPC protocol itself, including identity, PKI, certificate rotation, replay control, observability, and failure isolation

This note does not cover:

- the cryptographic details of MPC protocols themselves
- threshold ECDSA or Schnorr math
- protocol proofs

The focus here is architectural rather than cryptographic.


## 1. Framing

The communication layer is where most production pain lives.

The protocol determines whether a signature can be produced. The communication architecture determines whether the system can be operated, upgraded, audited, and recovered when something goes wrong.

In practice, the communication architecture has to answer at least six separate questions:

- how parties discover each other
- how party identity is authenticated
- how messages are routed
- how retries, duplicates, and out-of-order delivery are handled
- how certificates or transport credentials are rotated
- how operators observe, audit, and intervene in a live session

These are architecture questions, not MPC questions.


## 2. Three Topology Families

### 2.1 Relay

In a relay topology, parties do not connect directly to one another. Each party opens an outbound connection to a relay, coordinator, or message hub. Messages are routed through that service.

This is usually the easiest topology to operate.

Advantages:

- no inbound connectivity is required on participant nodes
- NAT traversal becomes someone else's problem
- the network policy is simpler because every participant only needs egress to a small set of endpoints
- audit logging, rate limiting, and session tracing become centralized
- certificate and endpoint rotation are much easier because the trust graph is hub-and-spoke rather than full mesh
- relay services can absorb transient offline participants through buffering, queueing, or durable storage

Disadvantages:

- the relay becomes an operational choke point
- latency now includes an extra hop
- throughput and fan-out are limited by relay capacity
- a compromised relay cannot usually forge protocol messages if message authentication is done correctly, but it can still delay, drop, reorder, or correlate traffic
- the system becomes more platform-shaped and less self-sovereign

This is why relay is common in custody systems.

### 2.2 Direct P2P

In a direct P2P topology, each party communicates with the other parties directly. In the extreme case this becomes a full mesh.

Advantages:

- no central relay bottleneck
- no single routing hub with complete traffic visibility
- low steady-state latency once connectivity is established
- clearer separation between cryptographic quorum and infrastructure provider

Disadvantages:

- discovery, NAT traversal, firewall policy, and endpoint reachability become much harder
- certificate management becomes `O(n^2)` operationally because every peer relationship matters
- observability becomes fragmented across nodes
- retry logic, duplicate suppression, and round synchronization become harder to debug
- rolling upgrades are trickier because version skew appears across many pairwise links
- mobile approvers, laptops, SGX enclaves, HSM wrappers, and cross-region parties are all more painful to integrate

P2P is attractive in research environments and in designs that strongly prefer coordinator independence. It is much less attractive once the system also needs auditability and low-friction operations.

### 2.3 Hybrid

Hybrid systems use relay for some functions and direct links for others.

Common hybrid patterns include:

- relay for discovery and session admission, then direct channels for round traffic
- relay for all control-plane traffic, direct channels only for large artifacts
- direct delivery between always-on backend parties, relay fallback for intermittently connected participants
- regional relays with direct intra-region traffic and relayed inter-region traffic

Hybrid is often the practical answer. The control plane stays centralized enough to operate, while the data plane can still use direct paths where they are worth the trouble.


## 3. Operational Tradeoffs

| Topic | Relay | Direct P2P | Hybrid |
|---|---|---|---|
| Inbound exposure | Minimal | High | Medium |
| NAT/firewall pain | Low | High | Medium |
| Auditability | Strong | Weakest | Strong |
| Cert rotation complexity | Low | High | Medium |
| Dependency on central infra | High | Low | Medium |
| Failure isolation | Hub-centric | Peer-centric | Split |
| Observability | Centralized | Fragmented | Mixed |
| Upgrade ergonomics | Easier | Harder | Medium |
| Sovereignty from coordinator | Lowest | Highest | Medium |
| Enterprise operability | Strongest | Weakest | Strong |

Topology is not just a networking choice. It is an operating-model choice.


## 4. Which Mode Is More Common in Industry

For institutional systems, the dominant pattern is not unmanaged full-mesh P2P. It is some form of relay-assisted or coordinator-assisted architecture, often with hybrid extensions.

The reason is straightforward:

- institutions want approval workflows and policy engines
- they want central audit logs
- they want outbound-only network posture where possible
- they want easier certificate and endpoint rotation
- they want support teams to be able to diagnose sessions without SSHing into every party host

Public product positioning points in that direction. Fireblocks presents MPC as part of a broader platform with policy controls, governance, and managed operations rather than as an unmanaged peer mesh [R1]. Coinbase's open-source `cb-mpc` library explicitly says it is only a cryptographic library and does not manage peer authentication, transport security, storage, or policy, leaving those responsibilities to the integrating application [R2]. In practice, that usually means a relay-assisted control plane.

The practical conclusion is:

- pure P2P is viable
- relay-assisted is usually easier to operate
- hybrid is usually the best fit once the system has to survive audits, outages, cert rotation, and cross-environment deployments


## 5. Library-Level Support

### 5.1 LFDT-cggmp24

The `cggmp21` / `cggmp24` family is transport-agnostic rather than transport-providing. The official `cggmp21` repository states that the library is agnostic to how messages are delivered and explicitly says applications may use `libp2p`, a centralized delivery server, a database, or a blockchain to deliver messages [R3]. It also states that all messages must be authenticated and encrypted by the application [R3].

Architecturally, that means:

- relay is supported
- direct P2P is supported
- hybrid is supported
- none of them are implemented for you

This is a bare-metal position. The library gives protocol messages and round flow. It does not give:

- service discovery
- peer authentication
- mTLS or QUIC termination
- certificate issuance or rotation
- retry policy
- durable queueing
- observability
- operator control plane

`LFDT-cggmp24` fits well when the communication plane is already decided elsewhere.

### 5.2 Synedrion

Synedrion is more structured at the session layer, but it is still not a transport product. Its repository states that it uses `manul` as its framework for round-based protocols [R4].

The transport abstraction in Synedrion is therefore shaped by the underlying `manul` model rather than by a concrete network stack.

### 5.3 manul

`manul` is the clearest statement of the architecture boundary. Its documentation describes the framework as `Sans-I/O`, explicitly meaning "bring your own async libraries, or don't" [R5]. It is also generic over signer, verifier, and signature types [R5].

More importantly, `manul` models message patterns explicitly:

- direct messages
- broadcast
- echo-broadcast
- caching of next-round messages when peers advance at different speeds [R5]

This makes `manul` stronger than a raw message loop. It understands rounds and message classes. It still stops before the infrastructure layer.

`manul` does not give:

- PKI
- certificate rotation
- transport encryption
- service discovery
- NAT traversal
- queue durability
- production telemetry

What it gives is a protocol engine for whichever transport model the application chooses.


## 6. What Each Stack Means Architecturally

The practical difference between the stacks is not "which one supports relay" and "which one supports P2P." All three effectively support both, because none of them hard-code a transport.

The real difference is where the abstraction boundary sits.

`LFDT-cggmp24`:

- lower-level and more transport-neutral
- easiest to embed into a custom relay or custom P2P stack
- leaves almost all communication semantics to the integrator

`synedrion + manul`:

- more opinionated about sessions, rounds, message destinations, and protocol-driver structure
- still transport-agnostic
- better fit when the operator wants a reusable communication state machine rather than only cryptographic round logic

The architectural summary is:

- `LFDT-cggmp24` is closer to a cryptographic protocol core
- `manul` is closer to a protocol-runtime framework
- `synedrion` inherits the strengths and limits of `manul` at the interaction layer


## 7. Certificate Rotation and Identity Architecture

This is where relay and P2P start to diverge sharply.

In a production MPC system, "certificate rotation" is not one thing. It is at least four distinct rotation problems:

- transport certificates used for mTLS or QUIC
- long-term party identity keys
- protocol signing keys used to authenticate MPC messages
- enrollment metadata mapping a real machine or enclave to a protocol party index

Those should not all be the same key.

### 7.1 Relay-Friendly Rotation

Relay makes rotation materially easier.

A typical pattern is:

1. each party maintains outbound mTLS only to the relay tier
2. the relay tier trusts both old and new certificates during a bounded overlap window
3. session admission freezes a party roster for the lifetime of the MPC session
4. newly rotated identities are used only for new sessions

This is materially easier because rotation affects `n` party-to-relay trust links rather than `n(n-1)` peer links.

It also supports clean separation between:

- transport identity
- session identity
- cryptographic party identity

That separation is much harder to keep clean in a peer mesh.

### 7.2 P2P Rotation Pain

In direct P2P systems, certificate rotation is not just a PKI task. It is a distributed coordination event.

Problems appear immediately:

- different peers may trust different CA bundles during rollout
- stale DNS or peer caches can pin old endpoints
- one lagging peer can strand a whole round
- mixed old/new cert state can look like byzantine failure
- emergency revocation is harder because every node must learn the new trust state quickly

If direct P2P is chosen, the safer pattern is usually:

- short-lived transport certs
- separate long-lived party identity keys
- explicit roster freezing per session
- dual-trust overlap windows
- aggressive telemetry for handshake failures and peer-version skew

Without those controls, P2P rotation incidents quickly look like protocol faults.


## 8. Communication-Layer Concerns Outside MPC

Skipping MPC itself, the communication architecture still has to solve the following:

### 8.1 Roster and Session Control

Every session needs:

- a stable session ID
- a frozen participant roster
- a mapping from transport endpoints to party identities
- an admission policy for new sessions

Without that, retries and reconnects can cross-contaminate sessions.

### 8.2 Replay and Duplicate Handling

Round-based protocols live or die by transcript hygiene.

The communication plane needs:

- per-session message namespaces
- per-round sequence handling
- duplicate suppression
- retention rules for forensic replay

Relay makes this easier because it can become the canonical transcript observer.

### 8.3 Backpressure and Durability

Pure P2P tends to underinvest in backpressure. A relay can make:

- queue depth visible
- retry storms visible
- slow consumers visible

For operations, this is the difference between diagnosis and guessing.

### 8.4 Observability

Operators need:

- message counts by round
- participant lag
- handshake failures
- certificate-expiry visibility
- per-session audit trails

Relay centralizes this naturally. P2P requires deliberate observability engineering.


## 9. Recommended Default

For a research prototype, direct P2P is often acceptable because it minimizes infrastructure and makes protocol behavior easier to see.

For a production institutional system, the default recommendation is different:

- use relay-assisted topology for the control plane
- prefer outbound-only participant connectivity
- separate transport identity from party identity
- freeze the roster per session
- treat certificate rotation as an infrastructure workflow, not a protocol event
- add direct peer channels only where there is a measured reason to do so

The production default should usually be hybrid with a strong relay bias, not pure full-mesh P2P.


## 10. Conclusion

MPC libraries usually do not choose your network architecture for you. `LFDT-cggmp24` is explicitly transport-agnostic. `synedrion` is more structured at the session layer, but relies on `manul`, which is also `Sans-I/O` and transport-agnostic. That means relay, P2P, and hybrid are all possible.

The question is not what the cryptographic library permits. The question is what the operating model can survive.

In institutional systems, relay-assisted or hybrid architectures are usually the stronger default because they make identity, audit, cert rotation, and operational intervention tractable. Pure P2P remains useful where coordinator independence matters most, but it shifts too much operational burden onto the communication layer to be the default choice for most custody-grade deployments.


## References

- [R1] Fireblocks platform positioning: [Treasury Management](https://www.fireblocks.com/platforms/treasury-management/)
- [R2] Coinbase `cb-mpc` README: [coinbase/cb-mpc](https://github.com/coinbase/cb-mpc)
- [R3] `cggmp21` README networking section: [LFDT-Lockness/cggmp21](https://github.com/LFDT-Lockness/cggmp21)
- [R4] Synedrion repository: [entropyxyz/synedrion](https://github.com/entropyxyz/synedrion)
- [R5] `manul` documentation: [docs.rs/manul](https://docs.rs/manul/latest/manul/)
