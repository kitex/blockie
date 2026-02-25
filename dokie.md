# Cosmos SDK v0.53: 3-Node Enterprise Cluster Deployment on RHEL 10

**Author:** Sugandha Amatya  
**Status:** Phases 1–3 Complete (Infrastructure, Binary Distribution, and Genesis Finalization)  
**Environment:** 3x Red Hat Enterprise Linux (RHEL) 10 VMs via KVM, managed via Ansible from a host ThinkPad.

---

## 🏗️ Architecture & Node Responsibilities

This lab simulates an enterprise-grade blockchain topology consisting of three distinct node types. 



* **Validator Node (`192.168.122.197`)**: The highly secured core of the network. It holds the cryptographic signing keys (`priv_validator_key.json`), proposes new blocks, and votes on consensus. It is isolated from the public internet.
* **Sentry Node**: The perimeter defense. It acts as a reverse proxy for the Validator, absorbing public peer-to-peer (P2P) traffic and forwarding valid transactions to the Validator via a private, hidden connection.
* **Full Node**: The public data replica. It synchronizes the entire blockchain ledger, serves RPC/REST queries to external applications, but does *not* participate in block creation or hold signing authority.

---

## 🚀 Deployment Phases & Ansible Mapping

### Phase 1: Infrastructure Initialization (`setup-nodes.yml`)
* **Target:** `hosts: cosmos` (All nodes)
* **High-Level Goal:** Provision VMs, register OS, and install core dependencies including hardware security modules.
* **Ansible Implementation:**
    * **OS Preparation:** Executed an automated "Nuclear Reset" via `ansible.builtin.shell` to unlock and register RHEL 10 subscription managers.
    * **Dependency Management:** Used `ansible.builtin.dnf` to install `gcc`, `make`, and `jq`.
    * **Environment Setup:** Configured Go v1.22/v1.23 and Rust repositories via `ansible.builtin.yum_repository`.
    * **HSM Integration:** Applied `when: "'validator' in group_names"` to isolate the installation of `SoftHSMv2` and Rust specifically to the Validator node, laying the groundwork for bank-grade secure key management.

### Phase 2: Application Layer Orchestration (`distribute-binary.yml`)
* **Target:** `hosts: cosmos` (All nodes)
* **High-Level Goal:** Compile the Cosmos SDK v0.53 binary (`fxd`) statically and distribute it to all nodes to initialize their data directories.
* **Ansible Implementation:**
    * **Compilation (Host):** Compiled from source on the ThinkPad with `CGO_ENABLED=0` to ensure GLIBC independence on RHEL 10.
    * **Binary Deployment:** Used `ansible.builtin.copy` to move the compiled `fxd` binary to `/usr/local/bin/` on the VMs.
    * **Node Initialization:** Executed `fxd init` via `ansible.builtin.command` to bootstrap the `~/.simapp` directory structures. This automatically generated the unique `priv_validator_key.json` and `node_key.json` files for each node.

### Phase 3: Security & Genesis - The "Air-Gapped" Workflow
Due to modern RHEL 10 kernel entropy management causing cryptographic generation hangs in KVM, key generation was strategically shifted to an "air-gapped" model using the host ThinkPad.

#### Part A: Key Injection (`inject-keys.yml`)
* **Target:** `hosts: validator`
* **Ansible Implementation:** * Iterated through locally generated test keys on the ThinkPad using `with_fileglob` (`~/cosmos_keys_clean/*`).
    * Securely pushed the unencrypted test keys (`admin.info`, `admin.address`) into the Validator's directory (`/home/sugandha/.simapp/keyring-test/`) via `ansible.builtin.copy`.

#### Part B: Genesis Finalization (`finalize-genesis.yml`)
* **Target:** `hosts: validator`
* **Ansible Implementation:**
    * **Gatekeeper Check:** Used `ansible.builtin.stat` to verify `admin.info` was present before proceeding.
    * **State Initialization:** Used `ansible.builtin.shell` to execute sequential `fxd` commands:
        1. `add-genesis-account`: Allocated 1,000,000,000 `stake` tokens to the `admin`.
        2. `gentx`: Cryptographically bound the `admin` funds to the Validator's staking power using the `priv_validator_key.json`.
        3. `collect-gentxs`: Merged the transactions to finalize the genesis file.
    * **Artifact Retrieval:** Used `ansible.builtin.fetch` to pull the finalized "Golden Source" `genesis.json` back to the ThinkPad for Phase 4 distribution.



---

## 🛠️ Ansible Configuration

### Inventory Configuration (`hosts.ini`)
```ini
[validator]
192.168.122.197 ansible_user=sugandha

[sentry]
192.168.122.X   ansible_user=sugandha  # IP of your Sentry VM

[fullnode]
192.168.122.Y   ansible_user=sugandha  # IP of your Full Node VM

[cosmos:children]
validator
sentry
fullnode


### Variable Matrix (`group_vars/all.yml`)
| Variable Name | Value | Purpose |
| :--- | :--- | :--- |
| `cosmos_user` | `sugandha` | Service account for all operations. |
| `cosmos_home` | `/home/sugandha/.simapp` | Base path for config, data, and keys. |
| `binary_path` | `/usr/local/bin/fxd` | Global path for the compiled SDK binary. |
| `chain_id` | `fx-local-1` | Unique cryptographic identifier for the chain. |

---

## 📂 Key Files Tracking

| File Name | Location | Confidentiality | Purpose |
| :--- | :--- | :--- | :--- |
| `fxd` | `/usr/local/bin/` (All Nodes) | Public | The compiled Cosmos SDK blockchain application binary. |
| `admin.info` | `~/.simapp/keyring-test/` (Validator) | Public* | Contains the public wallet address for the admin account. *(Note: Test backend used for lab).* |
| `priv_validator_key.json`| `~/.simapp/config/` (All Nodes) | **Strictly Private** | The cryptographic key that signs blocks. Never share the Validator's copy. |
| `genesis.json` | `ThinkPad: ./final_genesis.json` | Public | The "Golden Source" birth certificate of the network. Must be identical on all nodes to establish consensus. |

---

## 📈 Next Steps: Phase 4 Distribution (`sync-network.yml`)
* **Goal:** Distribute the `final_genesis.json` to Sentry and Full Node.
* **Logic:** Update `persistent_peers` in `config.toml` to link the nodes cryptographically and enable the `systemd` service to start the blockchain.
