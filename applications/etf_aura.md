# EtF Network with Aura 

- **Team Name:** Ideal Labs
- **Payment Address:** 0xTODO
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 2

## Project Overview :page_facing_up:

This is an EtF (encryption-to-the-future) network based on Aura. This proposal adds identity based signatures (IBS) block seals to Aura and enables an EtF network, wherein messages can be encrypted 'to a slot' in the future.

### Overview

Cryptex is blockchain that uses a modified Aura that seals blocks using identity based signatures. We then implement an encryption-to-the-future (EtF) scheme, where messages can be encrypted for arbitrary slots in the future. Our runtime will contain a pallet which will manage slot scheduling and verification. Finally, we propose a modification which enables a non-interactive cryptographically-verifiable-rule-based secret sharing mechanism on the EtF network and a light client with extra verifications to ensure proper syncing with the chain.

Being the first EtF network in the ecosystem, Cryptex introduces several new cryptographic primitives which would be useful to others as well. By developing EtF capabilities, other networks can likewise benefit from EtF-based consensus. This proposal lays the foundation for a more robust system later on, using a proof of stake consensus (Sassafrass) and more sophisticated cryptographic primitives for EtF, such as [McFly](http://fc23.ifca.ai/preproceedings/189.pdf) or based on [commitment witness encryption ](https://eprint.iacr.org/2021/1423.pdf). An EtF network can enable randomness beacons, sealed-bid auctions, and non-interactive secret sharing.

- An indication of why your team is interested in creating this project.

We want to build more extensive and secure decentralized data tools which allow general decentralized secret sharing. We believe that the internet is a better place when it's more fair for all.

### Project Details

The major pieces:
1. [IBS Block Seal](#ibe-block-seal-in-aura)
2. [Encryption-to-Future](#encryption-to-future-slots)
3. [Light Client](#light-client)

We also describe plans to implement the following, but it is not scoped within this proposal:
4. [Non-interactive secret sharing](#non-interactive-rule-based-secret-sharing)
5. [SDK](#sdk)

#### What this is not
- this does not use a proof of stake consensus. For the scope of the proposal, we are assuming a static, well defined validator set using PoA consensus based on Aura. 
- the proposal does not highlight any specific privacy preserving tools
- this is not using threshold signatures 
- most of that is scoped for the future though

#### Notation and Terminology

For the following, assume that we are using curve BLS12-381. As such, we will refer to its scalar field using $\mathbb{Z}_p$ where $p$ is the modulus for BLS12-381. Similarly, we have the pairing friendly elliptic curve groups $\mathbb{G}_1$ and $\mathbb{G}_2$.

#### IBS Block Seal in Aura

##### Overview

https://docs.rs/sc-consensus-aura/latest/sc_consensus_aura/

For simplicity, our network will use Aura for consensus. In the future, we intend to migrate to [Sassafrass](https://eprint.iacr.org/2023/031.pdf). For now, we assume that there is a static set of validators defined on network genesis. In Aura, each validator defined in the validator set authors a block in sequential (round robin) order. We will create a fork of Aura, wherein blocks are sealed using identity based signatures using nugget BLS a Waters IBE, as per [this comment](https://github.com/w3f/Grants-Program/pull/1660#issuecomment-1563616111). Though we will not be taking advantage of many properties of nugget BLS that make it beneficial, our future plans include functionality which will, so we opt to just use it from the start. 
  
Let $A = \{A_1, ..., A_n\}$ be the well defined set of authorites in the chain spec. For now, we'll assume that this set is static. In Aura slots are divided into discrete slots of $t$ seconds each. For any slot $s$, the winner of the slot is determined by $A[s\;\%\;|A|]$, where $A$ is the set of authorities defined on genesis. For now, we assume that each slot has a unique winner among the set of validators.
  
- **Identity Based Encryption**
The IBE scheme we use is the [Waters IBE](https://eprint.iacr.org/2004/180.pdf). We use this to assign a unique role to each slot, which is derived from the slot winner's identity, slot number, and epoch. 

---
- **Genesis/Setup**

  1.  (standard stuff) Each validator generates a private key and public key for the underlying signature scheme of the blockchain. Theoretically this could be implemented on any scheme, but we use BLS12-381. Each $A_i \in A$ generates some $<sk_i, pk_i>$, storing the secret key $sk_i$ securely (in their [keystore](https://paritytech.github.io/substrate/master/sp_keystore/trait.Keystore.html)), with the public keys used to define the initial validators. 
   
  2. We define system parameters on genesis:
    - a randomly chosen $\alpha \in \mathbb{Z}_p$ (output of VRF?). For now, we will assume that this value is static and only defined on genesis.
    - We choose some random generator $g \in \mathbb{G}$ and calculate $g_1 = g^\alpha$ and randomly choose some $g_2 \in \mathbb{G}$.
    - We choose a random 128-bit [am i sure about that??] vector $U = (U_i)$ randomly sampled from $\mathbb{G}$ by and encode the tuple $(g, g_1, g_2, u', U)$ within the genesis block.
  
  3. On genesis: For each $i \in [n]$, the root user produces a ciphertext $ct_i$ of the secret $\alpha$, which is encrypted under $ID_i$, and encodes it on chain. Prior to the first block, each $A_i$ decrypts $ct_i$ to recover $\alpha$ and places it in its keystore. Later on we will replace this with a more decentralized MPC protocol.

- **KeyGen and Identity**
  In the Waters IBE scheme, public keys are derived from an identity $v = (v_i)_{i \in [n]}$ by calculating $\mathcal{V} = \{ i \in [n] | v_i = 1 \}$. We will use the validator addresses as the public identifier and calculate an ephemeral public key by hashing the validator address with the epoch number and the slot number $s$. That is, ephemeral public keys can be calculated from publicly available information,, with $epk_i$ being a hash of $A_i$'s epheremal public key for the slot $i$ in the epoch with randomness $r$. Since we are using nugget BLS, we will need the public key in both $\mathbb{G}_1$ and $\mathbb{G}_2$.

- **Block Sealing**
  The winner of a slot $s$ calculates the secret key corresponding to $epk_i$ and uses it to sign the block. The secret key is calculated as $sk_i = (g_2^\alpha, (u' \sum_{i \in \mathcal{V}_i} u_i)^r, g^r)$, where $r \in \mathbb{Z}_p$ is randomly chosen and $\mathcal{V}_i := \{j \in [n] | epk_{i}[j] = 1\}$. The resulting $sk_i$ is used to sign the block.

- **Validation**
  When other nodes import the block, they validate it by generating an ID for the IBE scheme using the author of the block and then verifying the signature. 

##### Implementation Details

In order to implement an IBE block seal in Aura, we need to:
- setup the system parameters, define authorities, and distribute $\alpha$ to each authority

- randomness will be provided by [insecure-randomness-collective-flip](https://github.com/paritytech/substrate/blob/master/frame/insecure-randomness-collective-flip/README.md)

- Ephemeral public key derivation
  - Ephemeral public keys are derived from publicly available information. When the epoch randomness is known (and slot winners can be calculated), the ephemeral public keys of slot winners are calculated and encoded on-chain when the session starts. 

- Secret Key Derivation + Sealing blocks
  - slot winners can derive their secret key prior to sealing a block:
  - https://github.com/paritytech/substrate/blob/252156d9006ee45eda09ab80f687c066f2f4eaac/client/consensus/aura/src/standalone.rs#LL131C2-L131C2
  - in particular, using nugget bls we will us `sign_nugglet_bls` to sign the block, and `verify_nugget_bls` to verify it later on, where the transcript contains the epoch number and slot number.


#### Encryption-to-Future Slots

##### Overview

We propose an Encryption-to-Future (EtF) scheme on top of the modified Aura consensus proposed above.

The high level idea is that given a duration of time, $t$, identify a role to which we can encrypt a message such that it will be decryptable after time $t$ passes. We accomplish this by:
  - Given a duration $t$ from the current slot, calculate a future epoch and slot which will be active when that time occurs. Then, based on the order of the (static) authority set, determine the slot winner and calculate the inputs for that slot.
  - Encryption: create a shared key, IbeSS, using the input calculated previously. Then encrypt a secret (e.g. using El Gamal)
  - Decryption: when the slot winner calcuates and reveals the secret, the ciphertext is decryptable.

As can be seen, it will be paramount that all participants agree on the same 'time'.

##### Implementation

Since all of this functionality should happen outside the context of a runtime, we implement this as a specialized light client based on [smoldot](https://github.com/smol-dot/smoldot). 

###### Slot Scheduling
 
  As mentioned above, the idea is that we can take some arbitrary time in the future, $t_{fut}$, and identify a slot and epoch when that future time is expected to occur (assuming persistent liveness of the network). Assume that the current slot index, $sl_{curr}$, is publicly known and agreed on by everyone (see [light client](#light-client) for more info). 

###### Encryption-to-future-slots (EtF)

  Assuming that we have identified a future slot winner $A_{fut}$ for slot $sl_{fut}$, we can derive their ephemeral public key (aka the role) for that slot and create a shared key using [this approach](https://github.com/w3f/Grants-Program/pull/1660#issuecomment-1563616111). We can then use a symmetric encryption scheme, such as El Gamal, to encrypt a message using the resulting public key. The ciphertext can be stored offchain with a reference to it published on-chain (in a pallet or contract) by calculating its sha256 hash, for example.

###### Decryption

When a slot winner's slot is active, they derive a secret key (as above) which they then use to seal the block. Additionally, the derived secret key can also be used to decrypt any messages that were encrypted for this slot. In our first iteration of the network, we will publish this secret key along with the block, so anybody can decrypt the message after the slot completes. We intend to modify this approach in the future, for example, when using threshold signatures later on.


#### Light Client

We present a light client using smoldot. The light client will connections to specific nodes. It should also ensure that users clocks are set correctly (i.e. block number), else give an error. We need to make sure requests are properly synchronized to ensure accurate and predictible slot scheduling/selection.

#### High Level Architecture

We propose the architecture of the system at a high level. It consist of three pieces:
- **the blockchain**: The PoA blockchain with IBS block seals. It is a substrate based runtime with a new pallet that enables the identity based cryptosystem along with our modifications to Aura.
- **the offchain layer**: A user-agent (client) which handles slot scheduling, encryption, and decryption.
- **the storage layer**: Could be anything, we will use IPFS in conjunction with a smart contract or a pallet to store ciphertexts off-chain and their identifiers on-chain.

![high-level-architecture](./etf.drawio.png)

### Ecosystem Fit

Help us locate your project in the Polkadot/Substrate/Kusama landscape and what problems it tries to solve by answering each of these questions:

- Where and how does your project fit into the ecosystem?
- Who is your target audience (parachain/dapp/wallet/UI developers, designers, your own user base, some dapp's userbase, yourself)?
- What need(s) does your project meet?
- Are there any other projects similar to yours in the Substrate / Polkadot / Kusama ecosystem?
  - If so, how is your project different?
  - If not, are there similar projects in related ecosystems?

## Team :busts_in_silhouette:

### Team members

- Tony Riemer
- Names of team members

### Contact

- **Contact Name:** Tony Riemer
- **Contact Email:** driemworks@idealabs.network
- **Website:** idealabs.network

### Legal Structure

- **Registered Address:** Address of your registered legal entity, if available. Please keep it in a single line. (e.g. High Street 1, London LK1 234, UK)
- **Registered Legal Entity:** Name of your registered legal entity, if available. (e.g. Duo Ltd.)

### Team's experience

Please describe the team's relevant experience. If your project involves development work, we would appreciate it if you singled out a few interesting projects or contributions made by team members in the past. 

If anyone on your team has applied for a grant at the Web3 Foundation previously, please list the name of the project and legal entity here.

### Team Code Repos

- https://github.com/<your_organisation>/<project_1>
- https://github.com/<your_organisation>/<project_2>

Please also provide the GitHub accounts of all team members. If they contain no activity, references to projects hosted elsewhere or live are also fine.

- https://github.com/<team_member_1>
- https://github.com/<team_member_2>

### Team LinkedIn Profiles (if available)

- https://www.linkedin.com/<person_1>
- https://www.linkedin.com/<person_2>


## Development Status :open_book:

If you've already started implementing your project or it is part of a larger repository, please provide a link and a description of the code here. In any case, please provide some documentation on the research and other work you have conducted before applying. This could be:

- links to improvement proposals or [RFPs](https://github.com/w3f/Grants-Program/tree/master/docs/RFPs) (requests for proposal),
- academic publications relevant to the problem,
- links to your research diary, blog posts, articles, forum discussions or open GitHub issues,
- references to conversations you might have had related to this project with anyone from the Web3 Foundation,
- previous interface iterations, such as mock-ups and wireframes.

## Development Roadmap :nut_and_bolt:

### Overview

- **Total Estimated Duration:** 3 months
- **Full-Time Equivalent (FTE):**  1.5 FTE
- **Total Costs:** 30,000 USD


### Common Deliverables

It should be assumed that, unless specified otherwise, each deliverable also includes proper testing (e.g. deliverable (0c)) and documentation of the item, including manual testing, unit testing, and benchmarking.

The following items are mandatory for each milestone:

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | GPLv3 |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |

### Milestone 1 — IBS Block Seal

- **Estimated duration:** 1.5 months
- **FTE:**  1,5
- **Costs:** 15,000 USD

Goal: Implement the IBS block seal in Aura. We do this by creating a new pallet to facilitate the identity based cryptosystem, as well as modifying the Aura pallet and client code.

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1. | Substrate module: IBE Pallet/IBC Setup | We create a new pallet responsible for storing parameters needed for the identity based cryptosystem. This take includes param generation and distribution of the msk to authorities. The outcome of the deliverable is the pallet capable of storing system params for the IBC and exposing a trait which will be used by the modified aura pallet to derive roles.  |
| 2. | Substrate module: Aura Pallet | We modify the Aura pallet to be able to calculate epk's for each known session validator. Pubkeys will be calculated *on session planning* and encoded in runtime storage. |
| 3. | Substrate module: Aura Client | We modify the Aura client to sign blocks with its secret key generated with the identity based cryptosystem as detailed above. We also modify the signature validation phase of consensus to verify the IBS signatures properly. For the sake of ease, the block author will publish its secret along with the block. |

### Milestone 2 — Etf Implementation

- **Estimated Duration:** 1.5 months
- **FTE:**  1,5
- **Costs:** 15,000 USD

Goal: We want to enable encryption to future slots, including slot scheduling, encryption, and decryption. Encryption and decryption activities should occur in an offchain context.

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1. | Slot Scheduler | We implement slot scheduling logic to identify a future slot and derive its role/inputs. We will implement this within a smoldot client. |
| 2. | Offchain Encryption | Using the output of the slot scheduler, the user agent will be able to create a shared key with the role for the future slot and encrypt data (el gamal) for the role.  |
| 3. | Offchain Decryption | We modify Aura to sign blocks with its secret key generated with the identity based cryptosystem as detailed above. We also modify the signature validation phase of consensus to verify the IBS signatures properly.|
 
...


## Future Plans
  
- enhance the design to use threshold encryption and privacy preserving mechanisms
- implement non-interactive rule-based secret sharing on the EtF network
- when ready, migrate to the new sassafrass consensus instead of Aura 
- we aim to become a parachain

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** Web3 Foundation Website / Medium / Twitter / Element / Announcement by another team / personal recommendation / etc.
- Tony initially heard about this a year ago via the substrate website. Collectively, we all learned about the grants program at the polkadot blockchain academy.

Here you can also add any additional information that you think is relevant to this application but isn't part of it already, such as:

- As stated previously, Tony has worked on two grants previous to this one. The items in this grant are very much inspired by the Iris grant, however, it is intended to fix all of the vulnerabilities and issues (i.e. lack of scalability) that Iris failed to do.

## Appendix

In the Waters IBE, encryption and decryption are defined as follows:

- **Encryption**
  For a message $M \in \mathbb{G}_1$ and an ephemeral public key $epk_i$, choose a random $t \in \mathbb{Z}_p$ and calculate the ciphertext: $C = (e(g_1, g_2)^t M, g^t, (u' \prod_{j \in \mathcal{V}_i} u_i)^t)$ where $\mathcal{V}_j$ is as before.

- **Decryption**
  Given a ciphertext $C = (C_1, C_2, C_3)$ as defined above, and a secret $d_i = (d_1, d_2)$, then calculate $M' = C_1 \frac{e(d_2, C_3)}{e(d_1, C_2)}$. If the ciphertext is encrypted for $epk_i$, then $M = M'$