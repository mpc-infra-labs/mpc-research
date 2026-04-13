# KeyShare, Key Refresh, and Key Resharing

## 0. Scope

This note covers:

- what `KeyShare` means in different libraries
- the mathematical distinction between refresh and reshare
- what the CGGMP paper says
- what `LFDT-cggmp24` currently implements
- what `synedrion` implements for `KeyRefresh` and `KeyResharing`
- what the `threshold-key-bridge-prototype` codebase makes explicit about these boundaries

This note does not cover:

- threshold ECDSA signing math
- zero-knowledge proof internals
- transport or networking architecture


## 1. Problem Statement

The words `key share`, `refresh`, and `reshare` are overloaded.

These terms are often used loosely, but the code paths are different. Different libraries use `KeyShare` for different objects:

- threshold storage state
- active signing state
- or a wrapper that can be converted between the two

That distinction matters immediately, because refresh and reshare are defined over different objects.


## 2. The Three Objects

### 2.1 Threshold Storage Share

The threshold-storage view is Shamir-style.

Let the secret key scalar be `x`. Let:

$$f(z) = x + a_1 z + a_2 z^2 + \dots + a_{t-1} z^{t-1}$$

Then party `i` holds:

$$x_i = f(\alpha_i)$$

where `alpha_i` is the evaluation point for that holder.

This is a `t-of-n` representation:

- any `t` valid shares reconstruct the secret
- fewer than `t` do not

Public share form is:

$$X_i = x_i \cdot G$$

and the global public key is:

$$X = x \cdot G$$

This is the right mental model for:

- `cggmp24` storage-side threshold state
- `synedrion::ThresholdKeyShare`

### 2.2 Active Signing Share

The active-signing view is additive over the selected signer set.

For a chosen subset `S` of size `t`, define the Lagrange coefficient at zero:

$$\lambda_{i,S}(0) = \prod_{j \in S, j \ne i} \frac{0-\alpha_j}{\alpha_i-\alpha_j}$$

Then the active share for signer `i` is:

$$w_i = \lambda_{i,S}(0)\,x_i$$

so that:

$$\sum_{i \in S} w_i = x$$

This is not a storage format. It is the signing-time view attached to one chosen subset.

This is the right mental model for:

- `synedrion::KeyShare`
- the active signing moment in the prototype

This is also why the prototype keeps checking:

$$w_i = \lambda_i x_i$$

across the two sides.

### 2.3 Why the Naming Clash Matters

The same system may therefore contain two different "shares":

- a threshold-storage share `x_i`
- an active additive share `w_i`

That is exactly what happens in `synedrion`:

- `ThresholdKeyShare` is the threshold-storage object
- `KeyShare` is the active additive signing object for one concrete participant set


## 3. Refresh and Reshare: Mathematical Definitions

### 3.1 Refresh

Refresh changes the shares without changing the secret key.

The invariant is:

$$x' = x$$

and therefore:

$$X' = X$$

In threshold-storage form, refresh means adding a zero-constant polynomial:

$$r(z) = b_1 z + b_2 z^2 + \dots + b_{t-1} z^{t-1}$$

with:

$$r(0) = 0$$

Then:

$$f'(z) = f(z) + r(z)$$

and the new shares are:

$$x_i' = f'(\alpha_i) = x_i + r(\alpha_i)$$

Since `r(0)=0`, the secret remains:

$$f'(0) = f(0) = x$$

In additive form, the same idea becomes:

$$w_i' = w_i + \delta_i$$

with:

$$\sum_i \delta_i = 0$$

So refresh preserves:

- the secret key
- the public key
- the owner set
- the threshold policy

It only re-randomizes the shares and usually refreshes associated auxiliary material.

### 3.2 Reshare

Reshare keeps the secret key, but changes the distribution under which it is held.

The invariant is again:

$$x' = x$$

and:

$$X' = X$$

But now the threshold structure itself may change.

The new shares are defined by a different polynomial:

$$g(z) = x + c_1 z + c_2 z^2 + \dots + c_{t'-1} z^{t'-1}$$

with new holders and possibly a new threshold `t'`.

The new holders receive:

$$x_j' = g(\beta_j)$$

This means reshare may change:

- the holder set
- the share identifiers
- the threshold
- the concrete shares themselves

while preserving:

- the secret key
- the resulting public key

Refresh is re-randomization under the same structure.  
Reshare is redistribution under a new structure.


## 4. What the CGGMP Paper Says

The ePrint abstract states that the protocol "includes a periodic refresh mechanism and guarantees proactive security" [R1]. Synedrion's source code also labels its `KeyRefresh` implementation as the paper's "Auxiliary Info. & Key Refresh in Three Rounds (Fig. 7)" [R2].

The distinction is:

- `refresh` is part of the CGGMP paper
- `threshold resharing` is not part of CGGMP proper

Synedrion states this directly in its README:

- `Auxiliary Info. & Key Refresh` is listed as one of the protocols implemented from the paper
- `Threshold Key Resharing` is listed as "technically not a part of the CGGMP'24 proper" [R3]

The relevant boundary is:

- paper-level key refresh
- non-paper threshold resharing needed to make threshold-storage lifecycle practical


## 5. What LFDT-cggmp24 Currently Implements

The `LFDT-cggmp24` crate README describes CGGMP24 as supporting "a key refresh protocol", but immediately states that the crate itself **does not currently support key refresh** [R4].

Its own unsupported list is explicit:

- `Key refresh for both threshold (i.e., t-out-of-n) and non-threshold (i.e., n-out-of-n) keys`
- `Identifiable abort` [R4]

So the current implementation boundary is:

- the paper includes refresh
- the crate currently does not expose a usable refresh flow

The implementation boundary is easy to blur:

- what CGGMP as a paper family contains
- what a particular crate currently implements

Those are not the same statement.


## 6. What Synedrion Implements

### 6.1 `KeyShare` and `ThresholdKeyShare`

Synedrion separates the two objects cleanly.

`ThresholdKeyShare` is the threshold-storage object:

- it stores `threshold`
- it stores `share_ids`
- it stores `secret_share`
- it stores `public_shares`

`KeyShare` is the active signing object:

- it stores one additive secret share
- it stores the public shares for the active participant set

This is visible in the conversion functions in `src/entities/threshold.rs` [R5].

Two methods matter most.

`ThresholdKeyShare::to_key_share(&ids)`:

- takes a threshold-storage share
- chooses a concrete subset `ids` of size `t`
- multiplies by interpolation coefficients
- returns a `t-of-t` active `KeyShare` for that subset [R5]

`ThresholdKeyShare::from_key_share(key_share)`:

- goes the other way
- creates a `t-of-t` threshold keyshare from an active `KeyShare`
- the source comment says this is "a t-of-t threshold keyshare that can be used in KeyResharing protocol" [R5]

`KeyShare` and `ThresholdKeyShare` are therefore not synonyms in Synedrion.

### 6.2 Synedrion Refresh

Synedrion refresh is implemented over `KeyShare`, not over arbitrary threshold-storage state.

The protocol output is `KeyShareChange`, defined as:

- `secret_share_change`
- `public_share_changes` [R6]

Updating a share is literally:

$$w_i' = w_i + \Delta_i$$

because `KeyShare::update()` adds the secret delta to the local secret share and adds the public deltas to the stored public-share map [R6].

The strongest code-level evidence appears in the Synedrion test for `KeyRefresh`:

- it checks that each secret change corresponds to the published public-share change
- it then sums all `secret_share_change` values
- it asserts that the result is zero [R2]

That is exactly the additive refresh invariant:

$$\sum_i \delta_i = 0$$

So Synedrion refresh is best understood as:

- same active participant set
- same effective threshold for that active set
- additive zero-sum rerandomization
- refreshed auxiliary information

It is not a threshold-resharing protocol.

### 6.3 Synedrion Resharing

Synedrion `KeyResharing` is the opposite: it is about threshold-storage redistribution.

Its entry point takes:

- `old_holder`
- `new_holder`
- `new_holders`
- `new_threshold` [R7]

The type definitions make the purpose explicit:

- `OldHolder` contains an old `ThresholdKeyShare`
- `NewHolder` contains the old verifying key, old threshold, and old holder set [R7]

The final output is a new `ThresholdKeyShare` with:

- the new holder set
- the new share IDs
- the new threshold [R7]

The tests in `src/tests/threshold.rs` and `src/protocols/key_resharing.rs` show exactly this usage:

- start with one threshold-holder set
- run `KeyResharing`
- end with a different holder set and the chosen new threshold
- keep the same verifying key [R7][R8]

So in Synedrion:

- `refresh` updates active additive shares
- `reshare` rebuilds threshold-storage shares


## 7. What `threshold-key-bridge-prototype` Makes Explicit

The `threshold-key-bridge-prototype` codebase makes these boundaries unusually visible.

### 7.1 Refresh Path

In `src/simulation/synedrion.rs`, `run_synedrion_refresh_simulation()` takes:

- `Vec<synedrion::KeyShare<...>>`

not `ThresholdKeyShare` [L1]

The function signature also includes a threshold parameter `_t`, but the parameter is unused in the refresh body [L1]. That matches the underlying API boundary: the refresh flow is driven by the current active key-share set, not by threshold-storage metadata.

It instantiates:

- `KeyRefresh::<P, SimpleVerifier>::new(ids_conv)` [L1]

and later applies the result with:

- `share.update(change)` [L1]

That confirms the object of refresh is the active `KeyShare`.

The same file then persists refreshed threshold storage by calling:

- `ThresholdKeyShare::from_key_share(share)` [L1]

After refresh, threshold-storage state is reconstructed from refreshed active shares. That is consistent with Synedrion's API boundary.

### 7.2 Reshare Path

The bridge repo's own resharing helpers in `src/bridge/core.rs` are written in exactly the right mathematical language:

- additive-to-threshold conversion for `t < n` requires interaction
- each participant treats its additive share as the constant term of a random polynomial
- receivers aggregate subshares into new Shamir shares [L2]

That is reshare, not refresh.

The repo therefore exposes the split directly:

- signing-time additive shares support zero-sum refresh
- threshold-storage migration requires resharing


## 8. Refresh vs Reshare

If the holder set and threshold do not change, and the goal is only to rerandomize shares while preserving the same key, that is refresh.

If the holder set or threshold may change, or if one representation of threshold-storage state must be transformed into another, that is reshare.

In equations:

Refresh:

- same secret `x`
- same public key `X`
- same holders
- same threshold
- new shares via zero-sum or zero-constant perturbation

Reshare:

- same secret `x`
- same public key `X`
- possibly different holders
- possibly different threshold
- completely new share polynomial


## 9. Conclusion

Three implementation statements matter here.

First, the CGGMP paper includes refresh and presents a proactive protocol family [R1].

Second, the `LFDT-cggmp24` crate currently does not implement usable key refresh, even though the paper family includes it [R4].

Third, `synedrion` draws a clear software boundary:

- `KeyShare` is the active additive signing object
- `ThresholdKeyShare` is the threshold-storage object
- `KeyRefresh` operates on `KeyShare`
- `KeyResharing` operates on `ThresholdKeyShare`

That is also the boundary made visible by the prototype.

The one-line summary is:

> Refresh preserves the threshold structure and rerandomizes shares; reshare preserves the key but rebuilds the share structure itself.


## References

- [L1] `threshold-key-bridge-prototype/src/simulation/synedrion.rs`: [mpc-infra-labs/threshold-key-bridge-prototype](https://github.com/mpc-infra-labs/threshold-key-bridge-prototype/blob/main/src/simulation/synedrion.rs)
- [L2] `threshold-key-bridge-prototype/src/bridge/core.rs`: [mpc-infra-labs/threshold-key-bridge-prototype](https://github.com/mpc-infra-labs/threshold-key-bridge-prototype/blob/main/src/bridge/core.rs)
- [L3] `threshold-key-bridge-prototype/README.md`: [mpc-infra-labs/threshold-key-bridge-prototype](https://github.com/mpc-infra-labs/threshold-key-bridge-prototype/blob/main/README.md)
- [R1] CGGMP paper: [UC Non-Interactive, Proactive, Threshold ECDSA with Identifiable Aborts](https://eprint.iacr.org/2021/060)
- [R2] Synedrion `key_refresh.rs`: `Auxiliary Info. & Key Refresh in Three Rounds (Fig. 7)` in [entropyxyz/synedrion](https://github.com/entropyxyz/synedrion)
- [R3] Synedrion README protocol list: [entropyxyz/synedrion](https://github.com/entropyxyz/synedrion)
- [R4] `LFDT-cggmp24` README and crate docs: [LFDT-Lockness/cggmp21](https://github.com/LFDT-Lockness/cggmp21)
- [R5] Synedrion threshold share conversions: [src/entities/threshold.rs](https://github.com/entropyxyz/synedrion/blob/master/src/entities/threshold.rs)
- [R6] Synedrion `KeyShare` / `KeyShareChange`: [src/entities/full.rs](https://github.com/entropyxyz/synedrion/blob/master/src/entities/full.rs)
- [R7] Synedrion `KeyResharing`: [src/protocols/key_resharing.rs](https://github.com/entropyxyz/synedrion/blob/master/src/protocols/key_resharing.rs)
- [R8] Synedrion threshold tests: [src/tests/threshold.rs](https://github.com/entropyxyz/synedrion/blob/master/src/tests/threshold.rs)
