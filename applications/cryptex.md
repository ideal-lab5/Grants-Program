# Cryptex

- **Team Name:** Ideal Labs
- **Payment Address:** 0x2CDA3C7D6e21Cc4f43C170c0dFf2e9F3B3B5E889 (USDC)
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 3

## Project Overview :page_facing_up:

**Cryptex** is an [Aura](https://docs.substrate.io/learn/consensus/)-based blockchain that uses threshold identity encryption (TIBE) to seal blocks. 

### Overview

The goal of our proposal is to implement a scalable distributed key generation mechanism in a substrate based blockchain. The resulting distributed key pairs can then be used for a myriad of applications, such as securing digital assets, enabling decentralized data marketplaces, permissioned file systems, and more. The team believes that the very closed nature of data storage and ownership in traditional data storage models both hinders innovation, as it adds a large barrier to finding, using, or monetizing that data, as well as is a net-negative to data owners, as they have to trust a centralized authority, [who doesn't always behave in your best interest](https://www.cbsnews.com/news/facebook-settlement-privacy-lawsuit-claim-amount-2023/).

The proposal consists of three major components. Firstly, a standalone library that enables the aggregatable distributed key generator and verifiable secret sharing scheme. For this piece, we will begin by leveraging the work done by [Kobi Gurkan](https://github.com/kobigurk/aggregatable-dkg). Secondly, we'll integrate this into the SASLE within our runtime, based on the latest work done in the implementation of [sassafrass](https://github.com/paritytech/substrate/tree/davxy-sassafras-protocol). The DKG will be initiated via an extrinsic, where the caller defines parameters such as the number of slots and the threshold used by the DKG. With this enabled, the network will be able to generate owned custodial wallets. Our initial use case will be to enable wallet-to-wallet communication and rule-based file sharing capabilities. Along with this, we will build a smart contract rules system to gate access to data, in order to enable non-interactive rule-based secret sharing within the network. We will also develop a JS/TS SDK using polkadotjs that facilitates easy integrations of Cryptex into existing applications (e.g. for secure data exchange) or in new decentralized apps build on Cryptex.

Motivation:
> Suppose Alice has a document that she wants to make available to anybody who can meet some set of rules, but Alice will not be unavailable later to hand over the document. Alice has a plan though, she splits her data into pieces, or shares, and gives a single piece to anybody who she thinks will be available later and tells each person that if Bob shows up, to tell them their share. Then, each of these people have to make themselves known so that Bob can ask them for a secret. Bob appears, and they each give a copy to Bob and he is able to reassamble Alice's document and read her secret. However, suppose Charlie comes along before Bob and he doesn't want Alice to share her secret. Since Charlie knows who the shareholders are, he can bribe them to not give the key to Bob when he asks, or to get a key himself even though he isn't Bob. If sensitive information is to be secure this way, Alice needs a new plan to make sure Charlie can't do this.

### Project Details

We propose a DPoS blockchain, Cryptex, which uses a DKG based on the same [SASLE used by Sassafrass](https://eprint.iacr.org/2023/031.pdf) [1], in conjunction with an [aggregatable DKG](https://eprint.iacr.org/2021/005.pdf) [2], to enable a highly scalable decentralized KEM. We then define a smart contract based approach to define cryptographically verifiable conditions for data access along with a javascript/typescript SDK to provide tools for interacting with Cryptex and rule contracts. We intend to use the blockchain + contracts + SDK to enable dapps with general decentralized secret sharing capabilities (e.g. a fully-decentralized permissioned file system). Our initial use case will be centered around a decentralized custodial threshold wallet that enables non-interactive rule based secret sharing between addresses.

##### What this is not
- The main protocol does not store information about encrypted datasets, it only provides keypairs and key management capabilities. Data encryption and decryption must still be handled outside of the main protocol, and in fact many different encryption schemes could be implemented on top of the derived keys. We will build encryption functionality alongside the core protocol, both from the SDK and the dkg library.
- This protocol is not a decentralized storage solution. The management and storage of data that is encrypted with keys generated this way must still be handled offchain. This protocol will attempt to be storage agnostic.

### Aggregatable DKG based on SASLE 

We propose to use the semi-anonymous single leader election (SASLE) that is in use by the upcoming Sassafrass consensus mechanism [1] in order to facilitate an aggregatable distributed key generation mechanism. Using SASLE, we can semi-anonymously find a set of $k$ leaders, which will be the set of participants in the aggregatable DKG. As a **very** high level overview, during the protocol, participants use the output of a ring VRF and prepared 'tickets' to determine if they can be selected as the leader of a slot. Then each winner relies on a 'repeater', another node in the network who is responsible for running an algorithm called `VerifyPreparation` on the provided ticket, which checks the validity of the ticket and propogates it to the network upon verification. After a predetermined number of blocks ($\Delta_{live}$), when all tickets are finalized, tickets are sorted and the slot assignments/order is agreed on by the network. Leaders can then rely on the rVRF in order to verify that they are indeed the leader.

Now, we would like to make some modification in order to order to enable an aggregatable distributed key generator using SASLE. First, rather than epoch randomness, we will assume we have some 'session' randomness, which is unique to each instance of the protocol. We propose that for each participant with a winning ticket, they generate a secret (using scrape PVSS) and attach a DKG transcript as per [2], as $ass_j$, for example. Transcripts can then be verified by repeaters, who will aggregate transcripts if they receive more than one, taking the role of the 'gossip protocol' described in [2]. Repeaters then maintain the same responsibility, however, they also pass along the transcripts and a zk proof that they correctly verified the transcript. If the transcript fails to verify, the repeater should publicly announce the author of the transcript. Finally, the tickets can be sorted and each transcript and proof of correctness of the verification calculation can be publicly verified. 


#### Proposed Architecture

<p align="center">
 <img src="https://github.com/ideal-lab5/Grants-Program/blob/dkg/static/img/high_level_cryptex_user_centered.drawio.png?raw=true" alt="SDK interactions overview"/>
</p>

##### Libraries/Tech Stack

- Languages: rust, typescript/javascript
- Frameworks/Libs: substrate + tools (e.g. ink!, zombienet, telemetry), arkworks-rs for algebra and zk SNARKs.
- We will use [tarpaulin](https://github.com/xd009642/tarpaulin) for test coverage of our new pallets and dkg library.

##### Testing + Code Quality Approach

We intend to meet a minimum coverage of 80% on all code, as well as document manual test scenarios + zombienet scenarios, and to integrate code coverage and quality tools into our CI pipeline. Here is our testing tools/plans in brief:

- Pallets: For pallets, we will write unit tests, with coverage verified by tarpaulin. We'll also perform benchmarking for each pallet.
- E2E tests: E2E tests for the runtime will be done both manually and with zombienet. We intend to document manual testing steps, expected results, and verification for any reviewers.
- SDK: The sdk will be tested with some unit testing framework.

Code Quality: We haven't determined any code quality tools which we'll use, though we will follow all rust best practices, including ensuring code is properly formatted and passes cargo clippy. All code will be reviewed by the team before being released. We may also rely on the larger "PBA Alumni" community in some regard in order to receive external feedback as well.

Once completed, we will have security audits performed on our codebases as well (and resolve any issues).

##### Blind DKG Library

This is a standalone library that contains the code for the core blind DKG implementation. We plan on building this using arkworks-rs. This library will enable the Blind DKG protocol as detailed above. It will provide functionality to:

- generate secret polynomials, calculate shares and commitments
- verify key shares and prepare proofs of correctness of invalidity calculations
- blind messages, verify blind messages
- encrypt/decrypt using El Gamal (wasm)

The library will also compile to wasm and work with both std and no-std.

##### Pallets

These pallets may rely on other pallets that already exist in FRAME, which we don't mention here. (e.g. Grandpa, Babe, Balance, Timestamp, Assets, Contracts, etc.). The names and structure in the actual implementation may differ.

###### Coordinator Selection

We select a coordinator in the same way that a leader is selected in the consensus mechanism. That is, we select a coordinator using the same mechanism as BABE, via the election-provider-multi-phase module. 

###### DKG Pallet

The DKG pallet facilitates Blind DKG. That is, it contains the extrinsics needed to initate the protocol, submit blinded public keys, submit encrypted shares and commitments, and all other interactions as specified in the protocol. The pallet will allow for shared public key ownership to be validated, as well as contain the extrinsics needed to request decryption keys. The runtime storage of this pallet will store the blind public keys and allow for public verification that keys are correctly calcuated and that the participants were all valid. It will be able to trigger the coordinator selection process, which initates the blind dkg.

##### Authorization Pallet and Contracts

To facilitate the non-interactive secret sharing aspect of the network, we build functionality to allow the owner of a public key created via Blind DKG to specify other addresses as being able to 'authorize' reencryption of data for other parties. To make this a non-interactive process, we will use smart contracts that we refer to as 'rule' contracts. A rule contract is any contract that can call into the blockchain, via a chain extension, which, when authorized by the owner, can initate the reencryption process as discussed above.

To make this work, we will develop an 'authorization' pallet which allows the owner of a pubkey to specify addresses that can authorize data access. The extrinsic to authorize data access will be exposed through a chain extension and callable by contracts.

These contracts have no one-size-fits-all template, but to begin we will define a few nice primitives that could be used to build more complex rules. To start, we will define rule contracts that enable:

- password protected data
- identity-based data access
- data procetected by ownership of an asset

These will be implemented with ink!. In order to make this possible, we also implement a chain extension to authorize an address to decrypt data.

##### SDK

We will build an SDK with the following capabilities to allow developers and protocols to interact with cryptex:
 
  <p align="center">
    <img src="https://github.com/ideal-lab5/Grants-Program/blob/dkg/static/img/Cryptex-SDK.jpg?raw=true" alt="SDK Capabilities"/>
  </p>

  - **Encryption:** provides types and functions to encrypt and decrypt secrets. This functionality will rely on the wasm build of the [blind dkg library](#blind-dkg-library).
  - **VSS Client:** provides types and functions to interact with the protocol, some examples of such interactions are encryption key requests and secret sharing requests. We will build this with either polkadotjs or [Capi](https://github.com/paritytech/capi). It contains the calls needed to interact with the DKG pallet for key generation and the SNFT pallet for reencryption and for delegating decryption rights to other identities. This module also provides support for interacting with smart contracts (maybe with [useink!](https://github.com/paritytech/useink)). We would also like to explore using a light client, and potentially delve into [mobile compatibility](https://github.com/paritytech/trappist-extra).
  - **Rules DSL:** a domain specific language to define/model access rules required to get access to  a shared secret. Once defined, rules are packaged/translated to smart contracts. In future versions we will build a graphic editor to define these rules using our DSL as building block. The following diagram shows a high-level conceptual design of this capability:
  
  <p align="center">
    <img src="https://github.com/ideal-lab5/Grants-Program/blob/dkg/static/img/Cryptex-RulesDSL-final.jpeg?raw=true" alt="SDK Rules DSL Capability"/>
  </p>

  - **Storage:** provides types and functions to save/read/update ciphered documents through different datasource options. The first version will provide IPFS as data storage option but we are going to expand this module in the future to include centralized options as well such S3, Google Drive, between others. We will use a `MultiAddress` to locate data and a `CID` to identify it. The following diagram shows a high-level conceptual design of this capability:
  
  <p align="center">
    <img src="https://github.com/ideal-lab5/Grants-Program/blob/dkg/static/img/Cryptex-Storage-final.jpeg?raw=true" alt="SDK Storage Capability"/>
  </p>

  - **Graphql API:** provides types and functions to fetch data saved on-chain and off-chain related to Blind DKG in a developer friendly way. We may also explore the usage of [subquery](https://subquery.network/). This is not explicitly within the scope of this grant. The following diagram shows a high-level conceptual design of this capability:
  
  <p align="center">
    <img src="https://github.com/ideal-lab5/Grants-Program/blob/dkg/static/img/Cryptex-Graphql.jpeg?raw=true" alt="SDK Graphql Capability"/>
  </p>

Dapps built on the protocol would essentially be 'multiaddress and CID management' contracts which would be responsible for storage and curation of the multiaddresses and CIDs that are encrypted with some given public key. For example, a 'Netflix' dapp might look like some type of decentralized database mapping CIDs to some set of metadata (e.g. title, genre, rating), where the CIDs point to data encrypted with the Netflix public key generated via the DKG. The 'Netflix Rules' contract could be something as simple as checking if the caller owns the official 'Netflix NFT'. Dapps will most likely need to rely on some type of storage beyond what's available in the contract, as contract storage is limited and this data could potentially be huge. We leave the storage solution up to the implementer here via the storage module within the SDK. We intend to make this modular enough for a data owner to use any type of storage they choose, though to begin we limit this to only IPFS.


### Ecosystem Fit

Help us locate your project in the Polkadot/Substrate/Kusama landscape and what problems it tries to solve by answering each of these questions:

- Where and how does your project fit into the ecosystem?
  - Our project is a substrate-based chain that introduces new capabilities to the ecosystem, including using experimental cryptographic tools.
  - Our work will produce tools and knowledge that will be open source and openly available to the developer community.
  - We aim to become a parachain.
- Who is your target audience (parachain/dapp/wallet/UI developers, designers, your own user base, some dapp's userbase, yourself)?
  - **parachain**: ultimately, we strive to become a parachain. By taking advantage of XCM and continuing to evolve the cryptographic mechanisms surrounding delegation of decryption, we believe that very powerful and dynamic paradigms can be unleashed, such as gating access to data based on conditions of any state in any other parachain (e.g. data access via ownership of some specific asset on Statemint).
  - **users**: A long-term target audience will be our own user-base, as well as the user base of other parachains who need to share a secret. 
  - **Developers**: The tools we develop will be open source and available to other developers in the ecosystem. We intend to build a robust toolkit to enable devs to easily build on our new protocol/blockchain.
- What need(s) does your project meet?
  - Our project enables a decentralized an unstoppable secret key custodian. We see this as a foundational layer to building decentralized 'cryptographic data ownership' models, where by having a trustless key custodian, the network allows you to provably 'own' data, by proving that you own the keys to decrypt it. We think that this is a very powerful ownership model that can disrupt how data is secured today.
  - On top of that, the project also allows for a certain degree of governance within the system, and we have plans to add some type of 'decentralized moderation' tools in the future. 
- Are there any other projects similar to yours in the Substrate / Polkadot / Kusama ecosystem?
  - There are some projects that use threshold encryption:
    - [CESS](https://cess.cloud/)
    - [SkyeKiwi](https://github.com/skyekiwi/skyekiwi-protocol)
    - However, neither project uses a dkg for trustless keygen as we do. Further, SkyeKiwi is dependent on IPFS, which our protocol is not.
  - If so, how is your project different?
  - If not, are there similar projects in related ecosystems?
- We have found projects that are similar in other ecosystems as well:
  - For general "password manager/storage" type applications, we have:
    - [b.lock](https://github.com/BlockProject/b-lock)
    - [DaPassword](https://cardano.ideascale.com/c/idea/332494)
    - [Blockchain password](https://margatroid.github.io/blockchain-password/#/)
    - [You](https://medium.com/airgap-it/you-the-decentralized-password-manager-2f521cced7be)
  - [EthDKG](https://github.com/PhilippSchindler/EthDKG): EthDKG is a DKG protocol on built on ethereum, where its main contribution is the introduction of zk snarks during the disputes process.
  - [Share](https://share.formless.xyz/): which appears to use some type of threshold encryption but does not go into major detail (and which has dubious scalability)
  - [Lit Protocol](https://litprotocol.com/): LIT is a protocol that runs on a distributed network of mostly static nodes who each participate in a DKG to enable TSS *threshold signature scheme/threshold secret sharing. LIT, however, isn't a blockchain and isn't really decentalized, it is 'middleware' as their docs claim. Additionally, the DKG it uses is quite different than the one proposed here.

## Team :busts_in_silhouette:

### Team members

- Tony Riemer
- Carlos Montoya
- Juan Girini

### Contact

- **Contact Name:** Tony Riemer
- **Contact Email:** driemworks@idealabs.network
- **Website:** https://www.idealabs.network/

### Legal Structure

- **Registered Address:** 2451 Crystal Drive, 6th floor, Arlington, VA 22202, USA
- **Registered Legal Entity:** Ideal Labs, LLC

### Team's experience

Please describe the team's relevant experience. If your project involves development work, we would appreciate it if you singled out a few interesting projects or contributions made by team members in the past. 

If anyone on your team has applied for a grant at the Web3 Foundation previously, please list the name of the project and legal entity here.
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

- academic publications relevant to the problem
  - The VRF we use is the same as in [Babe](https://research.web3.foundation/en/latest/polkadot/block-production/Babe.html).
  - [Some further reading on VSS and DKG protocols](https://eprint.iacr.org/2012/377.pdf)
- links to your research diary, blog posts, articles, forum discussions or open GitHub issues,
  - I have already started a PoC to implement and simulate the blind dkg protocol here: https://github.com/ideal-lab5/dkg
  - In my substack I've published a few things about this, [here](https://driemworks.substack.com/p/blind-distributed-key-generation) and [here](https://ideallabs.substack.com/p/blind-dkg-part-1).
  - This work builds on the previous work done by Tony in his Iris project (see previous w3f grants).
  - While doing research for this proposal we uncovered we previously done by [protocol labs](https://research.protocol.ai/blog/2022/a-deep-dive-into-dkg-chain-of-snarks-and-arkworks/) on a deep dive of an implementation. Though it doesn't mirror our protocol, we see this as evidence for the feasibility of using arkworks for a DKG.
- references to conversations you might have had related to this project with anyone from the Web3 Foundation
  - We have spoken with several individuals involved with the grants program and with square one, specifically Coleman Maher and Nico Morgan, in a non-technical capacity, to discuss the high-level idea and potential. 
  - During an evaluation of the Iris grant, I spoke with the evaluator Diogo and he expressed scepticism around the security of the approach taken in Iris. While attending the PBA, after discussing ideas related to secret sharing schemes with instructors and  engineers from Parity and web3 foundation, I was able to reimagine the secret sharing implemented in Iris and redesign the system in order to fix the vulnerabilities inherent in the approach. I have already shared the draft whitepaper with some folks at web3 as well, though we haven't had a formal review of it.

## Development Roadmap :nut_and_bolt:

The general flow of our milestones are a two-pronged approach. In each milestone, we intend to develop the SDK capabilities as well as Blind DKG libary/runtime in parallel. We will also adhere to strict test coverage (>= 80%) and documentation requirements.

### Overview

- **Total Estimated Duration:** 24 weeks
- **Full-Time Equivalent (FTE):**  2.5 FTE
- **Total Costs:** 72,000 USD

There is a mandatory set of deliverables defined in the template. Since we aim to make similarly defined delvieries for each milestone, we present the list of pre-defined deliverables, expected per milestone, here: 

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | GPLv3 |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| **0e.** | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |

Note on estimates: 
- the estimates next to each deliverable are really high level, and we are attempting to break tasks down so that things can be worked on in parallel as much as possible. 
- Unless otherwise specified, assume that the estimate includes both any required research, develop, *and* testing.

### Milestone 1 — Integrate aggregated DKG into SASLE

- **Estimated duration:** 6 weeks
- **FTE:**  2.5
- **Costs:** 18,000 USD

The goal of this milestone is to build the core implementation of the DKG we proposed above. This will piggyback off of previous work done by Kobi Gurkan (for the DKG) as well as web3 teams (for the SASLE implementation). 

###### Goals
The goals of this milestone are:
- to integrate the aggregated DKG functionality into the SASLE process in order to:
  - prepare and verify DKG transcripts
  - submit zkp of correctness of transcript validity calculation
- a basic implementation of the SDK; the implementation should let users to initiate the dkg, check public key ownership, and query blockchain runtime storage.


| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1. | (4 weeks) Substrate Module | We integrate the existing SASLE into our runtime and make changes that allow  |
| 2. | (3 weeks) Substrate module: DKG Pallet | We build a pallet to integrate blind dkg into our runtime as described above. This will also include the calculation of a group public key. We will use a conjunction of runtime storage maps and logic within an on_initialize hook to accomplish this. |
| 3. | (2 weeks, done in parallel) SDK | The SDK should have basic functionality to: request to initate a round of blind dkg and query runtime storage, as well as setup the foundations of the SDK. |

### Milestone 2 - PoA Secret Sharing
- **Estimated Duration:** 6 weeks
- **FTE:**  2.5
- **Costs:** 18,000 USD

At the end of the previous milestone, we have accomplished the abilitiy for Alice to act as a static coordinator for a static set of participants, but we have yet to use those keys for any purpose. In this milestone, we don't yet solve the issue of the static authority set, but we do make the keys usable.

###### Goals
The outcome of milestone 2 is to enable secret sharing within the network. In order to do so, we develop functionality to encrypt, reencrypt, and decrypt data secured using the keys created through the mechanism implemented in milestone 1.

- shared public key derivation
- zk proofs of correctness to verify correctness of shares
- we begin the non-interactive secret sharing implementation by:
  - allowing users to publish a new public key to be used for encryption of secret shares 
  - allowing validators to encrypt shares and submit them onchain
- update the SDK to be able to leverage the secret sharing capabilities

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1a. | (2 weeks) Substrate Module: DKG Pallet: PK Derivation | We implement the public key derivation process as described above. |
| 1b. | (2 weeks) Substrate Module: DKG Pallet: SK derivation | We implement the blind mechanism for shareholders to submit keys as detailed [above](#non-interactive-secret-sharing). As part of this, we allow users to submit a public key to be used for encryption of their shares. |
| 2. | (2 weeks) Substrate Module: DKG Pallet | We enhance the pallet so that the coordinator (Alice) sends a zk proof of correctness that she validated a share and each participant verifies the proof. |
| 3. | (2 weeks) SDK | We add functionality to fetch public keys and verify their ownership, to encrypt/decrypt data, create and submit new public keys to the network for encryption, and to read a secret from the chain. |


### Milestone 3 - DPoS Network

- **Estimated Duration:** 6 weeks
- **FTE:**  2.5
- **Costs:** 18,000 USD

###### Goals

The goal of this milestone is to complete the implementation of blind dkg to make it truly blind. As part of this, we:
- upgrade to NPoS consensus
- implement coordinator selection
- implement blind participant selection
- slash/reward participants


| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1. | (1.5 weeks) | We upgrade our blockchain to use nominated proof of stake consensus. | 
| 2. | (3 weeks) Substrate Module: Coordinator Selection | We implement the coordinator selection mechanism, using the same approach used by BABE to select a coordinator (using a VRF). We also integrate this into the DKG pallet to trigger the process via extrinsic call. |
| 3. | (3 weeks) Substrate Module: Participant Selection (with VRF) | We implement the blind participant selection mechanism based on the work in the previous deliverable. Here, each validator submits a VRF and proof as detailed above. We also integrate this into the DKG pallet, so that once a coordinator is selected, the participant selection mechanism starts.  |
| 4. | (2 weeks) Substrate Module: DKG Pallet: Slashes and Rewards | We implement a basic slashing/reward scheme for participants in the DKG in order to incentivize honest participation. | 
| 5. | (2 weeks) SDK | We implement a storage adapapter in the SDK and verify a simple flow that keys can be created, used for encryption, and later on assembled and used for decryption. The reward mechanism will work by the coordinator distributing the reward to each participant. |

### Milestone 4 - Rule contracts
- **Estimated Duration:** 6 weeks
- **FTE:**  2.5
- **Costs:** 18,000 USD

###### Goals
Now that we have a complete implementation of Blind DKG, we will add a 'rule-based' access system on top of it, as defined using smart contracts. The intention is that the owner of data can define a proxy who can authorize data access, and that proxy is a smart cotnract deployed to the chain.
- enabling a rule based system to determine if an address can decrypt data
- develop and test a set of contracts to act as a base rule set
- develop tools to define and deploy these rules using a DSL
- showcase features and capabilities of everything developed thus far by building a dapp

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 1 .| (1.5 weeks) Substrate Module: Authorization Pallet + Chain Ext | We build a new pallet that provides functionality to authorize a proxy to delegate decryption rights to other addresses. Along with this, we build a chain extension that exposes this functionality for use in smart contracts. |
| 2. | (.5 weeks) Substrate Module: DKG Pallet | We modify the DKG pallet so that validators can publish new keys when the contract asks. |
| 2. | (2 weeks) Contracts | We use the chain extension and pallet from the previous deliverable to build several rule-based contracts for data that could be: password-protected, decryption based on identity (not exactly IBE*), and decryption based on ownership of an asset. |
| 3. | (3 weeks) SDK: DSL | We design and implement the DSL module of our SDK and enhance. The DSL module will allow users to design rules and deploy them as a contract, which can then be associated with their public key through the authorization pallet. |
| 4. | (3 weeks) SDK: Generic Secret Sharing Dapp | The final task of the final milestone is to use everything that has been developed thus far and to build a dapp on top of it. Our inital dapp will be a simple secret sharing platform that allows users to use our pre-defined contracts and DSL to build a decentralized, permission file system. We will build an interface that lets users create secrets, store them, define rules for access, share secrets, etc. We want this experience to showcase the full feature set that we have developed. |

## Future Plans

Please include here
- Short term intentions include: completion, review, and publication of our whitepaper, building an online presence, releasing cryptex as a testnet, promoting the protocol and SDK, enhancements and continued development, etc.
- Long Term:
  - We would like to become a parathread or parachain on Polkadot (probably parathread initially), which makes the next point more interesting.
  - We would like to explore the usage of XCM in order to accomplish cross-chain 'data locks', wherein the proof of a condition on chain A (e.g. owning some specific asset on Ajuna) would equate to decyryption rights being granted in Cryptex.
  - We would like to explore enabling a threshold signature scheme using the derived keys.
  - We would like to explore the usage of witness encryption within a blockchain. Some initial research has been done on this and it is a very promising concept, though a practical implementation would be very difficult at this point.

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** Web3 Foundation Website / Medium / Twitter / Element / Announcement by another team / personal recommendation / etc.
- Tony initially heard about this a year ago via the substrate website. Collectively, we all learned about the grants program at the polkadot blockchain academy.

Here you can also add any additional information that you think is relevant to this application but isn't part of it already, such as:

- As stated previously, Tony has worked on two grants previous to this one. The items in this grant are very much inspired by the Iris grant, however, it is intended to fix all of the vulnerabilities and issues (i.e. lack of scalability) that Iris failed to do.

# Resources 

[1] Sassafrass https://eprint.iacr.org/2023/031.pdf

[2] Aggregatable DKG: https://eprint.iacr.org/2021/005.pdf