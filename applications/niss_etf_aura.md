# Cryptex: Non-Interactive Secret Sharing in a PoA EtF Network

- **Team Name:** Driemworks
- **Payment Address:** TODO
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 2

## Project Overview :page_facing_up:

Cryptex is an EtF (encryption to the future) network based on Aura consensus. Our proposal builds the EtF network and a non-interactive secret sharing scheme on top of it.

### Overview

Cryptex is blockchain that uses a modified Aura which uses a treshold identity based encryption scheme to seal blocks. We then implement an encryption-to-the-future (EtF) scheme, where messages can be encrypted for arbitrary slots in the future. Our runtime will contain a pallet which will manage slot scheduling and verification. Finally, we propose a modification which enables a non-interactive cryptographically-verifiable-rule-based secret sharing mechanism on the EtF network and a light client with extra verifications to ensure proper syncing with the chain.

Cryptex introduces several new cryptographic primitives to the ecosystem which would be beneficial. By developing EtF capabilities, other networks can likewise benefit from EtF-based consensus. This proposal lays the foundation for a more robust system later on, using a proof of stake consensus (Sassafrass) and more sophisticated cryptographic primitives for EtF, such as [McFly](http://fc23.ifca.ai/preproceedings/189.pdf) or based on [commitment witness encryption ](https://eprint.iacr.org/2021/1423.pdf). An EtF network can enable randomness beacons, sealed-bid auctions, and as we will see shortly, non-interactive secret sharing. 

- An indication of why your team is interested in creating this project.

We want to build more extensive and secure decentralized data tools which allow general decentralized secret sharing. We believe that the internet is a better place when it's more fair for all.

### Project Details

The major components

1. [IBE Block Seal ](#ibe-block-seal-in-aura)
2. [Encryption-to-Future](#encryption-to-future-slots)
3. [Non-interactive secret sharing](#non-interactive-rule-based-secret-sharing)
4. [Light Client](#light-client)
5. [SDK](#sdk)

#### What this is not
- this does not use a proof of stake consensus. For the scope of the proposal, we are assuming a static, well defined validator set using PoA consensus based on Aura. 
- the proposal does not highlight any specific privacy preserving tools
- this is not using threshold signatures 
- most of that is scoped for the future though

#### Notation and Terminology

For the following, assume that we are using curve BLS12-381. As such, we will refer to its scalar field using $\mathbb{Z}_p$ where $p$ is the modulus for BLS12-381. Similarly, we have the pairing friendly elliptic curve groups $\mathbb{G}_1$ and $\mathbb{G}_2$.

#### IBE Block Seal in Aura

##### Overview

https://docs.rs/sc-consensus-aura/latest/sc_consensus_aura/

For simplicity, our network will use Aura for consensus. In the future, we intend to migrate to [Sassafrass](https://eprint.iacr.org/2023/031.pdf). For now, we assume that there is a static set of validators defined on network genesis. In Aura, each validator defined in the validator set authors a block in sequential (round robin) order. We will create a fork of Aura, wherein blocks are sealed using IBE signatures using nugget BLS, as per [this comment](https://github.com/w3f/Grants-Program/pull/1660#issuecomment-1563616111). Though we will not be taking advantage of many properties of nugget BLS that make it beneficial, our future plans include functionality which will, so we opt to just use it from the start [TODO double check if I want that...]. 
  
Let $A = \{A_1, ..., A_n\}$ be the well defined set of authorites in the chain spec. For now, we'll assume that this set is static. In Aura slots are divided into discrete slots of $t$ seconds each. For any slot $s$, the winner of the slot is determined by $A[s\;\%\;|A|]$, where $A$ is the set of authorities defined on genesis. For now, we assume that each slot has a unique winner among the set of validators.
  
- **Identity Based Encryption**
The IBE scheme we use is the [Waters IBE](https://eprint.iacr.org/2004/180.pdf). It may not be incredibly optimized, but it is straight forward to use and implement, and has provable security.

---
- **Genesis/Setup**

  1.  (standard stuff) Each validator generates a private key and public key for the underlying signature scheme of the blockchain. Theoretically this could be implemented on any scheme, but we use BLS12-381. Each $A_i \in A$ generates some $\left<sk_i, pk_i\right>$, storing the secret key $sk_i$ securely (in their [keystore](https://paritytech.github.io/substrate/master/sp_keystore/trait.Keystore.html)), with the public keys used to define the initial validators. 
   
  2. We define system parameters on genesis:
    - a randomly chosen $\alpha \in \mathbb{Z}_p$ (output of VRF?). For now, we will assume that this value is static and only defined on genesis.
    - We choose some random generator $g \in \mathbb{G}$ and calculate $g_1 = g^\alpha$ and randomly choose some $g_2 \in \mathbb{G}$.
    - We choose a random 128-bit [am i sure about that??] vector $U = (U_i)$ randomly sampled from $\mathbb{G}$ by and encode the tuple $(g, g_1, g_2, u', U)$ within the genesis block.
  
  3. On genesis: For each $i \in [n]$, the root user produces a ciphertext $ct_i$ of the secret $\alpha$, which is encrypted under $ID_i$, and encodes it on chain. Prior to the first block, each $A_i$ decrypts $ct_i$ to recover $\alpha$ and places it in its keystore. Later on we will replace this with a more decentralized MPC protocol.

- **KeyGen and Identity**
  
  Each slot winner $A_i$ broadcasts a new ephemeral public key $epk_i$ corresponding to its slot $i$. We consider $epk_i$ its epheremal identity,  For now, we will omit the scenario where a slot winner fails to supply a new ephemeral public key [Q: Am I omitting too many imporant pieces?].

- **Block Sealing**
  The winner of a slot $s$ calculates the secret key corresponding to $epk_i$ and uses it to sign the block. The secret key is calculated as $sk_i = (g_2^\alpha, (u' \sum_{i \in \mathcal{V}_i} u_i)^r, g^r)$, where $r \in \mathbb{Z}_p$ is randomly chosen and $\mathcal{V}_i := \{j \in [n] | epk_{i}[j] = 1\}$. The resuling $sk_i$ is used to sign the block.

- **Validation**
  When other nodes import the block, they validate it by generating an ID for the IBE scheme using the author of the block and then verifying the signature. 

- **Encryption**
  For a message $M \in \mathbb{G}_1$ and an ephemeral public key $epk_i$, choose a random $t \in \mathbb{Z}_p$ and calculate the ciphertext: $C = (e(g_1, g_2)^t M, g^t, (u' \prod_{j \in \mathcal{V}_i} u_i)^t)$

- **Decryption**
  Given a ciphertext $C = (C_1, C_2, C_3)$ as defined above, and a secret $d_i = (d_1, d_2)$, then calculate $M' = C_1 \frac{e(d_2, C_3)}{e(d_1, C_2)}$. If the ciphertext is encrypted for $epk_i$, then $M = M'$

##### Implementation
using the [SHAKE128 XOF](https://oag.ca.gov/sites/all/files/agweb/pdfs/bciis/06-nist-fips-202.pdf)

in general, this should say which parts of existing code I am going to modify, and any new things that I think I need to add. 

- Genesis Config
  - Setup authorities and params in chain spec
  - The Aura pallet genesis should be modified so that it can accept and verify ciphertexts from the root node.
- epoch randomness will be provided by pallet-randomness-whatever-its-called
- ephemeral public key derivation
- secret key calculation
- need an extrinsic to submit new pubkey, only callable by authorities?
  - mark them as inactive/active etc

#### Encryption-to-Future Slots

##### Overview

We propose an Encryption-to-Future (EtF) scheme on top of the modified Aura consensus proposed above.

At a high level, our approach is as follows: Once `RandomnessFromTwoEpochsAgo` is known for an upcoming epoch, we can compute the inputs for specific slots in that epoch. E.g. suppose that the current epoch is $e_{k-2}$, then we can calculate the slot winners for epoch $e_k$ and begin scheduling slots as soon as we know the randomness of the current epoch. Additionally, since we already know what the randomness was in previous epochs, we can schedule any availabe slots in the current or next epochs, $e_{k-2}$, $e_{k-1}$. 

In Aura, blocks are finalized in a discrete interval of time, say $t$ seconds. Assume that there are $m$ slots in each epoch. Then there are roughly $mt$ seconds per epoch, which we can discretely timestamp as slot $sl_{k+1}$ starting at time $kt$ and ending at time $2kt -1$. Suppose the current epoch and current slot is given by $(e_k, sl_m)$. Then we can encrypt to any future slots in the next two epochs by calculating the slot winners based on the order of the validator set. We will assume that the validator set can not grow larger than the number of slots. We'll detail the calculation below.

As can be seen, it will be paramount that all participants agree on the same 'time'.

##### Implementation

- slot scheduling
- encryption
- decryption

1. Given some amount of time $t$, calculate a future target slot, $sl_{fut}$, that you want to encrypt to. Since we have a static block time (say of $t_b$ sec/block), we can calculate the range $sl_{fut, 0} = sl_{curr} + floor(t/t_b)$ and $sl_{fut, 1} = sl_{curr} + ceil(t/t_b)$, Then the user can calculate $t_i = t - (sl_{fut, i} - sl_{curr}) * t_b$ for $i \in \{0, 1\}$.  e.g. we can provide best case lower and upper bounds to meet the target time, in number of blocks. Based on the target slot, we can identity a slot winner (we need to make sure all parties agree on this, even light clients...). We'll call the identified winner $A_{fut} = A[sl_{fut, i}]$ for $i = 0$ or $i = 1$.
2. Encryption:
   1. Use the public key of $w_{fut}$ to get an ID for the IBE scheme, (i.e. output of `create_nugget_bls`).
3. Decryption:

#### Non-Interactive Rule-Based Secret Sharing

Once we have achieved EtF and slot scheduling, we will implement a scheme on top of it in order to enable non-interactive rule-based secret sharing. The general idea is that some party $U$ has a secret $m \in \{0, 1\}^*$, to which they want to grant decryption rights if any given participant meets a condition $C$. In our case, this condition is some specific transaction in the blockchain, for example, a transaction that deposits one token to $U$'s wallet. First, $U$ will choose some future slot $sl_{fut}$ and encrypt their message for the expected authority's ephemeral public key (since we have a static validator set, this should work for now), resulting in some ciphertext $ct_{fut}$, which can then be added to some external storage system and identified via some cryptographic hash (e.g. Sha256 as in IFPS) which is then published on-chain along with a transaction type(i.e. extrinsic) that should grant decryption rights to the signer. 

Before the future slot occurs, another participant, $V$, can prepare a transaction that meets the conditions that $U$ published and sign it. Now, $V$ will encrypt the signed transaction for the future slot $sl_{fut}$ as well and publish it. 

When $sl_{fut}$ occurs, the validator decrypts the transaction published by $V$ as well as the 

##### Transaction Types and Commitments

#### Light Client

We present a light client using smoldot. The light client will connect to specific nodes, as we do not need a mempool. It should also ensure that users clocks are set correctly (i.e. block number), else give an error. We need to make sure requests are properly synchronized to ensure accurate and predictible time of decryption.

#### SDK

Do we really need an SDK at this point...? most funcs will be done in the node/client, and the rest can be implemented pretty simply with polkadotjs, idk. it'd be great, but we would need to rescope things

---
TODOS
- thresholds
- zpk
- PoS


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

- Name of team leader
- Names of team members

### Contact

- **Contact Name:** Full name of the contact person in your team
- **Contact Email:** Contact email (e.g. john@duo.com)
- **Website:** Your website

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

This section should break the development roadmap down into milestones and deliverables. To assist you in defining it, we have created a document with examples for some grant categories [here](../docs/Support%20Docs/grant_guidelines_per_category.md). Since these will be part of the agreement, it helps to describe _the functionality we should expect in as much detail as possible_, plus how we can verify and test that functionality. Whenever milestones are delivered, we refer to this document to ensure that everything has been delivered as expected.

Below we provide an **example roadmap**. In the descriptions, it should be clear how your project is related to Substrate, Kusama or Polkadot. We _recommend_ that teams structure their roadmap as 1 milestone ≈ 1 month.

> :exclamation: If any of your deliverables is based on somebody else's work, make sure you work and publish _under the terms of the license_ of the respective project and that you **highlight this fact in your milestone documentation** and in the source code if applicable! **Teams that submit others' work without attributing it will be immediately terminated.**

### Overview

- **Total Estimated Duration:** Duration of the whole project (e.g. 2 months)
- **Full-Time Equivalent (FTE):**  Average number of full-time employees working on the project throughout its duration (see [Wikipedia](https://en.wikipedia.org/wiki/Full-time_equivalent), e.g. 2 FTE)
- **Total Costs:** Requested amount in USD for the whole project (e.g. 12,000 USD). Note that the acceptance criteria and additional benefits vary depending on the [level](../README.md#level_slider-levels) of funding requested. This and the costs for each milestone need to be provided in USD; if the grant is paid out in Bitcoin, the amount will be calculated according to the exchange rate at the time of payment.

### Milestone 1 Example — Basic functionality

- **Estimated duration:** 1 month
- **FTE:**  1,5
- **Costs:** 8,000 USD

> :exclamation: **The default deliverables 0a-0d below are mandatory for all milestones**, and deliverable 0e at least for the last one. 

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | Apache 2.0 / GPLv3 / MIT / Unlicense |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |
| 1. | Substrate module: X | We will create a Substrate module that will... (Please list the functionality that will be implemented for the first milestone. You can refer to details provided in previous sections.) |
| 2. | Substrate module: Y | The Y Substrate module will... |
| 3. | Substrate module: Z | The Z Substrate module will... |
| 4. | Substrate chain | Modules X, Y & Z of our custom chain will interact in such a way... (Please describe the deliverable here as detailed as possible) |
| 5. | Library: ABC | We will deliver a JS library that will implement the functionality described under "ABC Library" |
| 6. | Smart contracts: ... | We will deliver a set of ink! smart contracts that will...


### Milestone 2 Example — Additional features

- **Estimated Duration:** 1 month
- **FTE:**  1,5
- **Costs:** 8,000 USD

...


## Future Plans

Please include here

- how you intend to use, enhance, promote and support your project in the short term, and
- the team's long-term plans and intentions in relation to it.

## Referral Program (optional) :moneybag: 

You can find more information about the program [here](../README.md#moneybag-referral-program).
- **Referrer:** Name of the Polkadot Ambassador or GitHub account of the Web3 Foundation grantee
- **Payment Address:** BTC, Ethereum (USDC/DAI) or Polkadot/Kusama (USDT) payment address. Please also specify the currency. (e.g. 0x8920... (DAI))

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** Web3 Foundation Website / Medium / Twitter / Element / Announcement by another team / personal recommendation / etc.

Here you can also add any additional information that you think is relevant to this application but isn't part of it already, such as:

- Work you have already done.
- If there are any other teams who have already contributed (financially) to the project.
- Previous grants you may have applied for.
