# Policy, Approval, and Control Plane

## 0. Scope

This note is intended to cover:

- the separation between signing substrate and control plane
- policy engines, approval workflows, and authorization boundaries
- how transaction intent, policy checks, and MPC signing are stitched together
- operator override, break-glass paths, and audit requirements
- multi-tenant custody concerns such as vaults, workspaces, and policy namespaces

This note has not been written yet.


## 1. Status

Pending further research.

The main questions to settle are:

- which parts of institutional custody systems are product policy and which parts are cryptographic infrastructure
- how vendors such as Fireblocks and ChainUp appear to separate policy, orchestration, and signing
- what a clean control-plane architecture would look like
- how approval and policy decisions should be represented so that they remain portable across chains and signing backends


## 2. Why This Topic Sits Here

At that point, the remaining question is not how a signature is produced, but how a system decides whether a signature should be produced at all.
