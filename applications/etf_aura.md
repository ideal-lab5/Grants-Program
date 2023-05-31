# EtF Network with Aura 

- **Team Name:** Ideal Labs
- **Payment Address:** 0xTODO
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 2

## Project Overview :page_facing_up:

This is an EtF (encryption-to-the-future) network based on Aura. This proposal adds identity based signatures (IBS) block seals to Aura and enables an EtF network, wherein messages can be encrypted 'to a slot' in the future.

### Overview

Cryptex is blockchain that uses a modified Aura that seals blocks using identity based signatures. We then implement an encryption-to-the-future (EtF) scheme, where messages can be encrypted for arbitrary slots in the future. Our proposal consists of a runtime, which modifies Aura and introduces a new pallet to enable the identity based cryptosystem (IBC), a light client and an SDK, which handles synchronization with the blockchain, slot scheduling for encryption, and offchain encryption and decryption functionality.

Being the first EtF network in the ecosystem, Cryptex introduces several new cryptographic primitives which would be useful to others as well. By developing EtF capabilities, other networks can likewise benefit from EtF-based consensus. This proposal lays the foundation for a more robust system later on, using a proof of stake consensus (Sassafrass) and more sophisticated cryptographic primitives for EtF, such as [McFly](http://fc23.ifca.ai/preproceedings/189.pdf) or based on [commitment witness encryption ](https://eprint.iacr.org/2021/1423.pdf). An EtF network can enable randomness beacons, sealed-bid auctions, and non-interactive secret sharing.

We want to build more extensive and secure decentralized data tools which allow general decentralized secret sharing. We believe that the internet is a better place when it's more fair for all.

### Project Details

The major pieces:
1. [IBS Block Seal](#ibe-block-seal-in-aura)
2. [Encryption-to-Future](#encryption-to-future-slots)
3. [User Agent](#light-client)

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
  
More concretely, let $A = \{A_1, ..., A_n\}$ be the well defined set of authorites in the chain spec. For now, we'll assume that this set is static. In Aura slots are divided into discrete slots of $t$ seconds each. For any slot $s$, the winner of the slot is determined by $A[s\;\%\;|A|]$, where $A$ is the set of authorities defined on genesis. Note that this implies, in most cases, that a validator will author many blocks during an epoch.
  
- **Identity Based Encryption**
The IBE scheme we use is the [Waters IBE](https://eprint.iacr.org/2004/180.pdf). We use this to assign a unique role to each slot, which is derived from the slot winner's identity, slot number, and epoch. 

---
We provide an overview of how the IBC can be used in the context of a PoA blockchain. 

- **Genesis/Setup**

  1.  (standard stuff) Each validator generates a private key and public key for the underlying signature scheme of the blockchain. Theoretically this could be implemented on any scheme, but we use BLS12-381. Each $A_i \in A$ generates keys $(sk_i, pk_i)$, storing the secret key $sk_i$ securely (in their [keystore](https://paritytech.github.io/substrate/master/sp_keystore/trait.Keystore.html)), with the public keys used to define the initial validators. 
   
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

- Develop a new pallet to enable the identity based cryptosystem to store, submit system parameters

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

##### Implementation Details

Since all of this functionality should happen outside the context of a runtime, we implement this as a specialized light client based on [smoldot](https://github.com/smol-dot/smoldot). 

###### Slot Scheduling
 
  As mentioned above, the idea is that we can take some arbitrary time in the future, $t_{fut}$, and identify a slot and epoch when that future time is expected to occur (assuming persistent liveness of the network). Very roughly, our approach will be similar to the following:
  
  If we assume that the current slot index is $sl_{prev}$ and the epoch is $e_k$, then we allow slots to be schedule starting from the next slot in the queue, $sl_{curr}$. Given that each slot lasts a static amount of time, say $t_{slot} \; sec/slot$, we can calculate the slot number $t$ seconds in the future with $sl_{fut} = ((t / t_{slot}) \% s) +1$ where $s$ is the number of slots per epoch. We can then identify the slot winner by calculating $A[{fut}\; \% \; |A|]$.

  The slot schedule calculation will happen in the user-agent, in conjun. The users have already calculated a future slot number and epoch. Then, the light client fetches the authorities for some specified epoch and calculated the inputs for the slot.

###### Encryption-to-future-slots (EtF)

  Assuming that we have identified a future slot winner $A_{fut}$ for slot $sl_{fut}$, we can derive their ephemeral public key (aka the role) for that slot and derive a shared key using [this approach](https://github.com/w3f/Grants-Program/pull/1660#issuecomment-1563616111). We can then use a symmetric encryption scheme, such as El Gamal, to encrypt a message using the resulting public key. The ciphertext can be stored offchain with a reference to it published on-chain (in a pallet or contract) by calculating its sha256 hash, for example.

###### Decryption

When a slot winner's slot is active, they derive a secret key (as above) which they then use to seal the block. Additionally, the derived secret key can also be used to decrypt any messages that were encrypted for this slot. In our first iteration of the network, we will publish this secret key along with the block, so anybody can decrypt the message after the slot completes. We intend to modify this approach in the future, for example, when using threshold signatures later on.


#### User Agent - SDK + Light Client

We present an SDK and light client using smoldot. The light client will connect directly to specific validators. It should also ensure that users' clocks are set correctly (i.e. block number), since we need to make sure requests are properly synchronized to ensure accurate and predictible slot scheduling/selection and decryption.

The SDK handles encryption to roles and decryption capabilities later on. It will also be able to handle communication with smart contracts hosted on the blockchain and external storage, realized as IPFS. 

#### High Level Architecture

We propose the architecture of the system at a high level. It consist of three pieces:
- **the blockchain**: The PoA blockchain with IBS block seals. It is a substrate based runtime with a new pallet that enables the identity based cryptosystem along with our modifications to Aura.
- **sdk/client**: A user-agent which handles slot scheduling, encryption, and decryption, as well as synchronization with the blockchain.
- **the storage layer**: Could be anything, we will use IPFS in conjunction with a smart contract or a pallet to store ciphertexts off-chain and their identifiers on-chain.

![high-level-architecture](./etf.drawio.png)

### Ecosystem Fit

Help us locate your project in the Polkadot/Substrate/Kusama landscape and what problems it tries to solve by answering each of these questions:

- Where and how does your project fit into the ecosystem?
  - to date, there is no EtF network in the ecosystem/
- Who is your target audience (parachain/dapp/wallet/UI developers, designers, your own user base, some dapp's userbase, yourself)?
  - At this stage, the target audience includes both parachain developers who may want to take advantage of the primitives we plan to introduce, as well as our own user base, 
- What need(s) does your project meet?
- Are there any other projects similar to yours in the Substrate / Polkadot / Kusama ecosystem?
  - If so, how is your project different?
  - If not, are there similar projects in related ecosystems?

## Team :busts_in_silhouette:

### Team members

- Tony Riemer
- Carlos Montoya
- Juan Girini

### Contact

- **Contact Name:** Tony Riemer
- **Contact Email:** driemworks@idealabs.network
- **Website:** https://idealabs.network

### Legal Structure

- **Registered Address:** Address of your registered legal entity, if available. Please keep it in a single line. (e.g. High Street 1, London LK1 234, UK)
- **Registered Legal Entity:** 


### Team's experience

Tony has worked on two, [here as "iridium"](https://github.com/w3f/Grants-Program/blob/master/applications/iris.md) and [here as "Ideal Labs"](https://github.com/w3f/Grants-Program/blob/master/applications/iris_followup.md).

### Tony Riemer

I am an engineer and math-lover with a passion for exploring new ideas. I studied mathematics at the University of Wisconsin and subsequently went to work at [Fannie Mae](https://en.wikipedia.org/wiki/Fannie_Mae) and then [Capital One](https://en.wikipedia.org/wiki/Capital_One), where I mainly worked on fintech products, like systems for loan servicing and efficient pricing algorithms. For the previous year and a half, I've been working exclusively in the web3 space, including having worked on two web3 foundation grants [here](https://github.com/w3f/Grants-Program/blob/master/applications/iris.md) and [here](https://github.com/w3f/Grants-Program/blob/master/applications/iris_followup.md) and as an independent consultant. Beyond the web3-sphere, I have dabbled in many open source projects as well as have built several of my own, ranging from computer vision, machine learning, to IoT and video games.  Most recently, I attended the Polkadot Blockchain Academy in Buenos Aires, and this new proposal is an application of ideas I learned there applied to my previous grant.
### Carlos Montoya
I have been doing software for more than 20 years, most recently in the startup world. 
- **Blockchain Experience**
Through 2022 I was mainly focused on building smart contracts with solidity, and on taking part in some encode-club bootcamps and ETH Global hackathons. I built several apps, one of them a decentralized job-board protocol called [web3Jobs](https://ethglobal.com/showcase/web3jobsfevm-inz64) ([Repo](https://github.com/encode-g2-project)). In Early 2023 I had the fortune to participate in the Polkadot blockchain academy in Buenos Aires. Cryptex's idea emerged during my time in the academy.
- **Software Engineering Experience**
  Since early 2021 I have been building [TeamClass](https://www.teamclass.com) as CTO and partner. TeamClass is a b2b marketplace for helping companies with their team-building initiatives through virtual events. We bootstrapped TeamClass ourselves and made sales by 3.8M in our first year. Previously, between 2016 and 2020 I was completely focused on building [StellarEmploy](https://www.stellaremploy.com) with my co-founders, where we had the opportunity to participate in NY ERA (Accelerator), and got institutional capital. StellarEmploy's technology was recently acquired by Learning Collider. Finally, between 2004 and 2015, I was CTO and Chief Architect at [MVM Software Engineering](https://www.mvm.com.co/?lang=en), a technology firm with a deep focus on energy solutions. During my time there I had the responsibility of defining the way of doing software for the entire company, leading very skilled people, building complex software products, and managing hundreds of initiatives for helping the company to expand its operations in Colombia, the Dominican Republic, and Mexico. 
- **Carnegie Mellon University** Master of Science Information Technology, 2011 - 2013
- **Tecnológico de Monterrey** Master in Information Technology Management, 2011 - 2013
- **Universidad Pontificia Bolivariana** Innovation and Technology Management, 2009 - 2010
- **Universidad Autónoma de Manizales** Systems Engineer, 1997 - 2002

### Juan Girini

 I am a software engineer with nearly 20 years of experience. Over the years, I have worked in various industries and have gained valuable experience in backend projects for the web2 space. However, my passion for decentralization has led me to focus on web3.

I studied Information Systems Engineering at [Universidad Tecnológica Nacional](https://utn.edu.ar/) in Argentina. In early 2023, I graduated from the Polkadot Blockchain Academy in Buenos Aires. This life-changing experience opened doors for me into the world of Substrate. During my time at the academy, I had the opportunity to meet with Carlos and Tony. Together, we started to conceive this project.

Following my graduation from the academy, I joined Parity as a Core Rust Engineer in the Pallet Contracts team. Some of my previous working experiences are Backend Developer at [Scayle](https://www.scayle.com/); Lead Engineer at [Cohabs](https://www.cohabs.com/); and Head of Development at [Barracuda Digital](https://barracuda.digital/).

### Team Code Repos

- https://github.com/ideal-lab5/cryptex-node
- https://github.com/ideal-lab5/cryptex-sdk
- https://github.com/ideal-lab5/contracts
- https://github.com/ideal-lab5/dkg

Please also provide the GitHub accounts of all team members. If they contain no activity, references to projects hosted elsewhere or live are also fine.

- https://github.com/driemworks
- https://github.com/carloskiron
- https://github.com/juangirini

### Team LinkedIn Profiles

- https://www.linkedin.com/in/tony-riemer/
- https://www.linkedin.com/in/cmonvel/
- https://www.linkedin.com/in/juan-girini/



## Development Status :open_book:

If you've already started implementing your project or it is part of a larger repository, please provide a link and a description of the code here. In any case, please provide some documentation on the research and other work you have conducted before applying. This could be:

- This proposal is a result of the discussion here: https://github.com/w3f/Grants-Program/pull/1660
- There are many protocols that use some form of witness encryption to accopmlish something similar, for example [time lock encryption](https://eprint.iacr.org/2015/482.pdf) or [commitment witness encryption](https://eprint.iacr.org/2021/1423). Our protocol is inspired by these ideas but uses a simpler design. 

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
| **0e.** | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |

### Milestone 1 — IBS Block Seal

- **Estimated duration:** 1.5 months
- **FTE:**  1.5
- **Costs:** 15,000 USD

Goal: Implement the IBS block seal in Aura. We do this by creating a new pallet to facilitate the identity based cryptosystem, as well as modifying the Aura pallet and client code.

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1. | Substrate module: IBE Pallet/IBC Setup | We create a new pallet responsible for storing parameters needed for the identity based cryptosystem. This take includes param generation and distribution of the msk to authorities. The outcome of the deliverable is the pallet capable of storing system params for the IBC and exposing a trait which will be used by the modified aura pallet to derive roles.  |
| 2. | Substrate module: Aura Pallet | We modify the Aura pallet to be able to calculate epk's for each known session validator. Pubkeys will be calculated *on session planning* and encoded in runtime storage. |
| 3. | Substrate module: Aura Client | We modify the Aura client to sign blocks with its secret key generated with the identity based cryptosystem as detailed above. We also modify the signature validation phase of consensus to verify the IBS signatures properly. For the sake of ease, the block author will publish its secret along with the block. |

### Milestone 2 — Etf Implementation

- **Estimated Duration:** 1.5 months
- **FTE:**  1.5
- **Costs:** 15,000 USD

Goal: We want to enable encryption to future slots, including slot scheduling, encryption, and decryption. Encryption and decryption activities should occur in an offchain context. The capabilities delivered here are all related to offchain calculations.

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1. | Client: Slot Scheduler | We implement slot scheduling logic to identify a future slot and derive its role/inputs. We will implement this within a smoldot client. |
| 2. | SDK: Encryption + Storage | Using the output of the slot scheduler, the user agent will be able to create a shared key with the role for the future slot and encrypt data (el gamal) for the role. We also integrate with IPFS and use it to store the ciphertext. |
| 3. | SDK: Decryption | After a block is authored for the specified future slot, we can decrypt the secret by fetching the secret published with the block and using it to decrypt the ciphertext created previously. |
 
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