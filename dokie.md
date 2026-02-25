# Cosmos SDK v0.53: 3-Node Enterprise Cluster Deployment on RHEL 10

**Author:** Sugandha Amatya  
**Status:** Phases 1–3 Complete (Infrastructure, Binary Distribution, and Genesis Finalization)  
**Environment:** 3x Red Hat Enterprise Linux (RHEL) 10 VMs via KVM, managed via Ansible from a host ThinkPad.

## 🏗️ Architecture & Node Responsibilities

This lab simulates an enterprise-grade blockchain topology consisting of three distinct node types. 

* **Validator Node (`192.168.122.197`)**: The highly secured core of the network. It holds the cryptographic signing keys (`priv_validator_key.json`), proposes new blocks, and votes on consensus. It is shielded from the public internet.
* **Sentry Node**: The perimeter defense. It acts as a reverse proxy for the Validator, absorbing public peer-to-peer (P2P) traffic and forwarding valid transactions to the Validator via a private, hidden connection.
* **Full Node**: The public data replica. It synchronizes the entire blockchain ledger, serves RPC/REST queries to external applications, but does *not* participate in block creation or hold signing authority.

---

## 🚀 Deployment History

### Phase 1: Infrastructure & Base OS
* **Virtualization:** Provisioned three RHEL 10 VMs using KVM.
* **Subscription Management:** Automated a "Nuclear Reset" via Ansible to unlock and register RHEL 10 repositories across all nodes.
* **Dependency Installation (`setup-nodes.yml`):**
    * Installed Go v1.22/v1.23, `gcc`, `make`, and `jq`.
    * **HSM Integration:** Installed `SoftHSMv2` and Rust specifically on the **Validator** node. This lays the groundwork for hardware-level cryptographic isolation, simulating a bank's secure key management environment.
* **Access Control:** Established passwordless SSH keys between the host ThinkPad and the VMs.

### Phase 2: Application Layer (The Binary)
* **Compilation:** Compiled the Cosmos SDK v0.53 binary from source on the ThinkPad host. The default `simd` binary was renamed to `fxd`.
* **Static Linking:** Enforced `CGO_ENABLED=0` during compilation to ensure the binary is portable and runs on RHEL 10 without GLIBC dependency conflicts.
* **Distribution (`distribute-binary.yml`):**
    * Pushed the compiled binary to `/usr/local/bin/fxd` on all three VMs.
    * Executed `fxd init` on all nodes to generate their default directory structures (`~/.simapp/config/`).
    * **Crucial File Created:** `priv_validator_key.json` was automatically generated inside the Validator's config folder. This file holds the node's block-signing identity.

### Phase 3: Security & Genesis (The Network "Birth")
Due to modern RHEL 10 kernel entropy management causing cryptographic generation hangs in KVM, key generation was strategically shifted to an "air-gapped" model using the host ThinkPad.

* **Offline Key Generation:** Generated the `admin` account keys locally on the ThinkPad, placing them in `~/cosmos_keys_clean/keyring-test`.
* **Key Injection (`inject-keys.yml`):**
    * Securely pushed the unencrypted test keys (`admin.info`, `admin.address`, etc.) from the ThinkPad to the Validator VM at `/home/sugandha/.simapp/keyring-test/`.
* **Genesis Finalization (`finalize-genesis.yml`):**
    * **Add Account:** Executed `fxd genesis add-genesis-account` to allocate 1,000,000,000 `stake` tokens to the `admin` address inside the network's draft `genesis.json`.
    * **Gentx Creation:** Executed `fxd genesis gentx` to create the genesis transaction. This action used the Validator's `priv_validator_key.json` to cryptographically bind the `admin` funds to the Validator's staking power.
    * **Collect Gentxs:** Executed `fxd genesis collect-gentxs` to merge the transactions and finalize the genesis file.
    * **Golden Source Extraction:** Fetched the finalized `/home/sugandha/.simapp/config/genesis.json` back to the host ThinkPad as `final_genesis.json`.

---

## 📂 Key Files Tracking

| File Name | Location | Confidentiality | Purpose |
| :--- | :--- | :--- | :--- |
| `fxd` | `/usr/local/bin/` (All Nodes) | Public | The compiled Cosmos SDK blockchain application binary. |
| `admin.info` | `~/.simapp/keyring-test/` (Validator) | Public* | Contains the public wallet address for the admin account. *(Note: Test backend used for lab).* |
| `priv_validator_key.json`| `~/.simapp/config/` (All Nodes) | **Strictly Private** | The cryptographic key that signs blocks. Never share the Validator's copy. |
| `genesis.json` | `ThinkPad: ./final_genesis.json` | Public | The "Golden Source" birth certificate of the network. Must be identical on all nodes to establish consensus. |
