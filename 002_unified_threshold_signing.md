# Unified Threshold Signing for Multi-Chain Custody

## 0. Scope

This note asks a practical architecture question:

> In real custody systems, how is multi-chain support actually implemented, and which layer should be generalized in a research prototype?

The note covers:

- what public product material from Fireblocks and ChainUp suggests about multi-chain custody architecture
- why "multi-chain support" should be decomposed into custody surface, chain adapter, and signing substrate rather than treated as one monolithic feature
- which library is the right fit for the current prototype
- what that library naturally supports
- what it does not support
- how custom curve support is introduced in the current codebase
- what the current prototype demonstrates at the architecture layer

The note is not a full survey of all MPC custody vendors.


## 1. Problem Statement

In production custody systems, the engineering problem is not merely "how to threshold-sign one chain." The harder problem is how to support many chains without building one independent signing stack per asset family.

That pressure appears most clearly when comparing chains across signature families:

- EVM chains typically use `secp256k1 + ECDSA`
- Bitcoin uses `secp256k1`, but the wallet and transaction model is UTXO-based rather than account-based
- Solana uses `Ed25519`

If every new chain forces a full rewrite of:

- key generation
- secret sharing
- threshold signing
- policy enforcement
- transaction serialization
- address derivation

then the custody stack becomes operationally fragmented very quickly.

The design question is therefore:

> Which layer should be reused across chains, and which layer should be chain- or curve-specific?


## 2. What Public Product Material Suggests About Industry Architecture

### Case Study 2.1 Fireblocks

Fireblocks' public documentation presents the platform around a common object model rather than around chain-specific signing silos. Its developer docs describe:

- a `Workspace`
- `Vault accounts`
- asset wallets under those vault structures
- asset objects identified through a common API surface [R1][R2]

This is architecturally important. The custody surface is presented first as:

- workspace and tenant boundary
- account boundary
- wallet / asset inventory
- approval and policy boundary

and only then as specific assets.

Fireblocks' public materials also separate asset support from wallet structure. Asset onboarding is not described as "build a new custody system," but as integrating additional assets or tokens into the same platform surface [R2][R3].

The public documentation does **not** expose Fireblocks' internal signer implementation in enough detail to claim a precise code-level architecture. But the product surface strongly suggests that multi-chain support is delivered through:

1. a shared custody control plane
2. a shared signing substrate
3. asset- and chain-specific handling layered above that substrate

That is the key point relevant to this prototype.


### Case Study 2.2 ChainUp

ChainUp's public custody materials present a similar pattern. Its product and documentation pages describe:

- MPC-based custody across `200+` blockchain networks [R4]
- an MPC system composed of `Workspace` and `Wallet` objects [R5]
- wallet creation and address allocation under workspaces [R5]

Again, the architecture is not introduced as "one chain, one stack." It is introduced as a unified custody product where multiple chains live under one workspace and wallet hierarchy.

That object model matters because it means the chain-specific parts are downstream of a common custody abstraction. The system first decides:

- who owns the workspace
- which wallet belongs to which business unit
- how assets are grouped
- how approvals and key-share handling work

and only after that does it care whether a given wallet uses:

- an account-based chain
- a UTXO chain
- a chain with memo/tag semantics

This is exactly the kind of separation a multi-chain MPC architecture needs.


### 2.3 Industry Pattern

Taken together, the public material from Fireblocks and ChainUp suggests a common pattern:

1. **custody control plane**
   workspaces, vaults, wallets, approvals, policy, segregation
2. **signing substrate**
   MPC / TSS / threshold protocols and key-share lifecycle
3. **curve adapter**
   `secp256k1`, `Ed25519`, and other signature-family specifics
4. **chain adapter**
   address encoding, payload hashing, transaction building, broadcast semantics

The key insight is that "multi-chain support" does **not** mean one MPC implementation per chain. It means:

- reuse the control plane broadly
- reuse as much signing math as possible
- isolate true curve differences in a narrow interface
- isolate chain-specific transaction rules above that

This is exactly the design hypothesis your prototype is testing.


## 3. Library Choice

For this prototype, the most suitable library is `Taurus multi-party-sig` [C1][C2].

The reason is not that Taurus already gives end-to-end Solana support. It does not. The reason is that Taurus exposes the right abstraction boundary for the problem this prototype is actually trying to solve.

The current repository uses Taurus as a reusable MPC math and protocol substrate, then adds a custom `Ed25519` adapter on top of it. This makes Taurus a better fit than a chain-specific wallet library and also a better fit than a library whose public surface is tightly optimized around only one signature family.

Your own repository README already states the right design goal:

- reuse protocol logic
- isolate curve behavior behind a dedicated adapter
- keep chain-specific logic above that boundary [C1]

That is the correct framing for a multi-chain custody research prototype.


## 4. What Taurus Naturally Supports

The strongest reason to pick Taurus is its explicit curve abstraction.

At the library level, Taurus exposes:

- `curve.Curve`
- `curve.Scalar`
- `curve.Point` [C3]

This is not cosmetic API design. It is the exact boundary needed for multi-curve MPC experiments.

Once a curve implements those interfaces, Taurus can already reuse generic helpers such as:

- random scalar sampling
- polynomial generation
- polynomial evaluation
- Lagrange interpolation [C4]

The protocol tree in Taurus also shows that it is not a one-protocol codebase. The module already contains:

- `protocols/cmp`
- `protocols/doerner`
- `protocols/frost`

So the natural capability of Taurus is not "one chain." Its natural capability is:

- reusable threshold-signing math
- reusable protocol families
- reusable curve-parameterized algebra

That is exactly the substrate a multi-chain custody architecture wants.


## 5. What Taurus Does Not Naturally Support

Taurus is the right protocol base for this prototype, but it is not a full multi-chain custody stack by itself.

Out of the box, Taurus does **not** give you:

- chain-specific address derivation
- chain-specific transaction building
- chain-specific payload hashing rules
- wallet product semantics such as workspaces, vaults, policy engines, or approval flows
- broadcast orchestration
- recovery UX
- production key custody or networked party orchestration

In other words, Taurus is a good answer to:

> how do I reuse threshold-signing math across curves?

It is **not** by itself a complete answer to:

> how do I ship a Fireblocks- or ChainUp-like multi-chain custody product?

That difference is important. It prevents the prototype from overclaiming what has already been solved.


## 6. How Custom Curve Support Works in the Current Prototype

Your prototype adds a custom `Ed25519` implementation on top of Taurus by wrapping `filippo.io/edwards25519` into Taurus' curve interfaces [C5].

The adapter defines:

- the curve order as `Ed25519Order`
- a nominal curve type `CurveEd25519`
- a `ScalarWrapper`
- a `PointWrapper` [C5]

At the curve level, the adapter implements the Taurus entry points:

- `Name()`
- `NewPoint()`
- `NewBasePoint()`
- `NewScalar()`
- `Order()`
- `SafeScalarBytes()`
- `ScalarBits()` [C5]

At the scalar and point level, it implements the mutable arithmetic Taurus expects:

- addition
- subtraction
- multiplication
- inversion
- negation
- equality
- marshaling / unmarshaling
- scalar action on points [C5]

This is the architectural heart of the repository. Once `CurveEd25519` satisfies Taurus' interfaces, the code can immediately reuse Taurus helpers that were not written specifically for Ed25519.

That reuse is visible in `pkg/mpc/frost_math.go`:

- `sample.Scalar(rand.Reader, group)` is used for secret generation
- `polynomial.NewPolynomial(group, 1, secret)` is used for share construction
- `polynomial.Lagrange(group, signers)` is used for interpolation weights [C6]

This is the strongest empirical result in the repository:

> the reusable boundary is not the chain, but the curve-aware MPC math layer.


## 7. What the Current Prototype Demonstrates

At the application layer, `main.go` models a simplified Solana-oriented flow [C7]:

- run DKG
- persist local shares
- sign a message with a `2-of-3` subset
- verify the resulting `Ed25519` signature locally

This is the right experimental shape for the research question. It tries to demonstrate that:

1. the protocol substrate can remain reusable
2. the curve can be swapped by implementing an adapter
3. the chain-specific layer can then sit above that

The important architectural point is that Solana here is being used as a forcing function. Because Solana uses `Ed25519`, it is a good test of whether the reusable MPC layer is truly curve-generic or only accidentally reusable for `secp256k1`.


## 8. What the Current Prototype Leaves Above the Library Layer

The current repository is intentionally focused on the reusable signing substrate rather than on a full custody product.

Two scope boundaries matter most.

### 8.1 It is not a full chain adapter

The current `main.go` signs a dummy message:

```go
msg := []byte("Solana Transaction Data 12345")
```

That means the prototype is testing threshold-signature flow, not full Solana transaction assembly [C7].

Likewise, the displayed "address" is currently just the public key in hex:

```go
address := hex.EncodeToString(pubBytes)
```

For a real Solana wallet product, this layer would need to become Base58 and be integrated into a real transaction builder [C7].

### 8.2 It is not a custody control plane

The prototype does not implement:

- user / tenant isolation
- workspace or vault hierarchy
- policy approval
- recovery flows
- party networking

So it is not yet comparable to Fireblocks or ChainUp at the product layer. It only targets the reusable signing substrate.


## 9. The Right Abstraction Boundary

The main conclusion of this note is that multi-chain custody should be designed in layers.

The correct split is:

1. **custody / product layer**
   workspace, vault, policy, approval, asset inventory
2. **protocol layer**
   threshold DKG, share lifecycle, signing rounds
3. **curve adapter layer**
   point/scalar arithmetic and serialization for a signature family
4. **chain adapter layer**
   address format, payload encoding, transaction building, broadcast

This prototype is correctly aimed at layer 3.

That is also why Taurus is the right library choice for the current repository. It already gives a reusable layer-2 substrate with a strong layer-3 abstraction boundary. The prototype then asks whether `Ed25519` can be plugged into that boundary cleanly enough to support a Solana-style path.

The answer from the current code is yes for the architectural direction: the reusable asset is the curve-parameterized MPC layer, while chain semantics remain a higher layer.


## 10. Recommendation

For this repository, the right next step is **not** to build one chain after another by hardcoding separate signing stacks. The right next step is:

1. keep Taurus as the protocol base
2. finish the `Ed25519` curve adapter properly
3. define a clean chain-adapter interface above signing
4. treat EVM / Bitcoin / Solana support as chain-adapter problems whenever the underlying signing substrate can be shared

In short:

> the reusable asset is the threshold-signing substrate plus the curve adapter boundary, not the chain-specific transaction code.

That is the same architecture direction that public product material from Fireblocks and ChainUp makes plausible at the industry level, and it is the direction your Taurus-based prototype is already testing in code.


## References

- [R1] Fireblocks developer docs, object model.  
  <https://developers.fireblocks.com/docs/object-model>

- [R2] Fireblocks API reference, assets API.  
  <https://developers.fireblocks.com/reference/getasset>

- [R3] Fireblocks documentation, adding support for tokens and assets.  
  <https://developers.fireblocks.com/docs/add-support-for-tokens>

- [R4] ChainUp Custody product page.  
  <https://www.chainup.com/product/custody>

- [R5] ChainUp Custody docs, create MPC system.  
  <https://custodydocs-en.chainup.com/quick-start/mpc-wallet/create-wallet>

- [C1] Prototype README.  
  <https://github.com/mpc-infra-labs/unified-threshold-signing-prototype/blob/main/README.md>

- [C2] Prototype module selection (`go.mod`).  
  <https://github.com/mpc-infra-labs/unified-threshold-signing-prototype/blob/main/go.mod>

- [C3] Taurus curve interfaces.  
  <https://github.com/taurushq-io/multi-party-sig/blob/v0.7.0-alpha-2025-01-28/pkg/math/curve/curve.go>

- [C4] Taurus polynomial helpers.  
  <https://github.com/taurushq-io/multi-party-sig/blob/v0.7.0-alpha-2025-01-28/pkg/math/polynomial/polynomial.go>  
  <https://github.com/taurushq-io/multi-party-sig/blob/v0.7.0-alpha-2025-01-28/pkg/math/polynomial/lagrange.go>

- [C5] Custom `Ed25519` curve adapter.  
  <https://github.com/mpc-infra-labs/unified-threshold-signing-prototype/blob/main/pkg/mpc/ed25519_adapter.go>

- [C6] Prototype DKG and signing math.  
  <https://github.com/mpc-infra-labs/unified-threshold-signing-prototype/blob/main/pkg/mpc/frost_math.go>

- [C7] Prototype entrypoint and Solana-oriented flow.  
  <https://github.com/mpc-infra-labs/unified-threshold-signing-prototype/blob/main/main.go>
