# mpc-research

Research notes for multi-chain MPC custody, threshold signing, and the surrounding systems architecture.

## Recommended Reading Order

1. [001_ecc_cryptography_and_mpc_research_note.md](./001_ecc_cryptography_and_mpc_research_note.md)  
   Cryptographic foundations: ECC, signature schemes, and why threshold ECDSA is harder than Schnorr-style systems.

2. [002_unified_threshold_signing.md](./002_unified_threshold_signing.md)  
   Multi-chain custody architecture: what should be unified, what should remain chain-specific, and where the signing substrate sits.

3. [003_bip32_hd_wallets_and_hardened_derivation_in_mpc.md](./003_bip32_hd_wallets_and_hardened_derivation_in_mpc.md)  
   BIP32, HD-wallet semantics, hardened vs non-hardened derivation, and why hardened derivation is a real MPC boundary.

4. [004_keyshare_refresh_and_reshare.md](./004_keyshare_refresh_and_reshare.md)  
   Key-share lifecycle semantics: threshold storage shares, active signing shares, refresh, and reshare.

5. [005_cggmp24_bridging_lockness_and_synedrion.md](./005_cggmp24_bridging_lockness_and_synedrion.md)  
   Case study: bridging Lockness-compatible / CGGMP24 state into Synedrion and back.

6. [006_relay_and_p2p.md](./006_relay_and_p2p.md)  
   Communication-layer architecture: relay, direct P2P, hybrid topologies, and operational tradeoffs.

7. [007_policy_and_control_plane.md](./007_policy_and_control_plane.md)  
   [NOT READY] Policy, approval, and control-plane design. Pending further research.

8. [008_hardware_and_deployment_boundary.md](./008_hardware_and_deployment_boundary.md)  
   [NOT READY] Hardware and deployment boundary. Pending further research.

9. [009_backup_recovery_and_disaster_recovery.md](./009_backup_recovery_and_disaster_recovery.md)  
   [NOT READY] Backup, recovery, and disaster recovery. Pending further research.

10. [010_presigning_capacity_and_performance.md](./010_presigning_capacity_and_performance.md)  
   [NOT READY] Presigning, capacity planning, and performance envelopes. Pending further research.

11. [011_node_infra.md](./011_node_infra.md)  
   Chain-side infrastructure experiments: Ethereum, Base, Bitcoin, and Solana node operations.