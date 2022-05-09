# W3F Grant Proposal

- **Project Name:** Iris
- **Team Name:** Ideal Labs
- **Payment Address:** Ethereum: 0x6ec0D6c005797a561f6F3b46Ca4Cf43df3bF7888 (DAI)
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 3

## Project Overview :page_facing_up:

Followup to: https://github.com/w3f/Grants-Program/pull/812

### Overview

Iris is a decentralized network that provides a secure data storage, delivery, and ownership layer for Web 3.0 applications. It is infrastructure for the decentralized web, providing a storage and data exchange which enables the transfer and monetization of access to and ownership of data across chains, smart contracts and participants in the network or connected through a relay chain. Iris provides security, availability, reputation, and governance on top of IPFS, enabling data ownership, access management, and the commodification of latent storage capacity and content delivery. It applies defi concepts to data, reputation, storage capacity and availability to synthesize computation and storage and to represent off-chain assets in an on-chain context. Ideal Labs wants to be the forefront of the next data revolution, and to help build the tools needed for a transparent and fair data economy.

Ideal Labs wants to be at the forefront of the next data revolution.

### Project Details

The intention of this followup grant is to implement several features that enable Iris to be a secure, social, and highly-available storage network without compromising the highest degrees of decentralization. Firstly, we will implement an availability-encouraging storage system to optimize data availability and storage costs. We also will add a social aspect to Iris by implementing an anonymous reputation system that allows data consumers to rate transactions (and data itself) with data owners using non-interactive zero-knowledge proofs. In our system, data owners associate their data with any number of 'data spaces' which each have specific rule sets and inclusion policies. To enhance security, we use a novel proxy re-ecryption mechanism that does not expose any secret keys as well as introduce the concept of the proxy node which enables ciphertext to be re-encrypted. Additionally we introduce "composable access rules", which allow data owners to specify rules which are implicitly enforced when consumers access their data. Lastly, we will build a javascript SDK to allow user interfaces for dapps built on Iris to easily build applications and interface with Iris.

To summarize, in the following we propose:

- the introduction of "data spaces"
- the implementation of "composable access rules" to apply custom business logic to data
- the introduction of "proxy nodes" that enable proxy re-encryption within the network
- implementation of an anonymous reputation system to create reputations for data owner, data spaces, and data iteself
- implementation of an availability-encouraging storage system with a low barrier to enter
- a javascript SDK to allow dapp developers to easily build front ends for smart contracts on Iris

For further details, we direct the reader to view our whitepaper here: [https://www.idealabs.network/docs](https://www.idealabs.network/docs)

#### Tech Stack

- IPFS (specifically rust-ipfs)
- Intel SGX
- Substrate
- Javascript

#### Terminology

- Data Asset Class and Data Asset: A **data asset class** is an asset class which is associated with some owned data, as identified by its CID, which has been ingested into the Iris storage system at some point in the past (does not necessarily mean it is currently available in Iris). A **data asset** is an asset minted from the data asset class.

- Data Owner: A **data owner** is any node that has initiated the ingestion of some data and owns a data asset class. In the following, we may refer to data owners as "Alice".

- Data Consumer: A **data consumer** is any node that owns a data asset minted from a data asset class. In the following, we may refer to data consumers as "Bob".

- Storage Node: A **storage node** is any node that has staked the minimum required amount IRIS tokens to become a storage node.

- Proxy Node: A **proxy node** is any node that has the minimum requried hardware requirements and has staked the minimum required amount IRIS tokens to become a proxy node.

#### Data Spaces

A data space is a user-defined configuration and rule set that allows users to group data together into curated collections. More explicitly, the data space lays the foundation to enable user-defined moderation rules within the network and provides a means to securely associate disparate data sets with one another (i.e. under an ‘organization’). All data in Iris exists within the context of a space, however, there is no limit to the number of spaces a data owner may associate their data with.

Under the hood, the 'data space' is an asset class which is mapped to a set of configuration items that determine the inclusion and moderation policies of the space. These asset classes exist much the same as the concept of 'data assets' from the original Iris proposal. In a data space with a private inclusion policy, only nodes holding tokens minted from the data space asset class are authorized to associate data with the data space. We see this as an additional market opportunity for the data economy, as ownership of a space may become important based on the sum of data stored within. Data spaces may also be subject to  [composable access rules](#composable-access-rules) at some point in the future.

Further, data spaces form the basis for moderation, or curation, within the network. Each data space may have a set of rules associated with it which limits not only the type of data that can be associated with the space, but also with the contents of the data. We intend to accomplish this through the execution of machine learning models, bayesian filters, and more, within a trusted execution environment. However, that work is outside the scope of this proposal.

![data spaces diagram](https://github.com/ideal-lab5/iris-docs/blob/master/resources/data_spaces.drawio.png)

#### Proxy/Gateway Nodes and Proxy Re-Encryption Mechanism

Proxy Re-encryption is a scheme through which a third party (the proxy) is able to alter the ciphertext of some encrypted data so that it can be decrypted by an authorized party.  Iris leverages proxy re-encryption to secure data, allowing a data-owner to encrypt data without knowing the identity of a potential consumer, and for consumers to receive encrypted data without exposing or sharing their secret keys. Specifically, Iris uses a unidirectional proxy re-encryption mechanism as outlined in https://eprint.iacr.org/2005/028.pdf. Data owners encrypt their data offchain using their secret key prior to ingestion. When an authorized consumer requests data, a proxy node re-encrypts the ciphertext for the consumer and delivers it offchain, after which the data owner can decrypt the data using their secret keys.

Iris uses two levels of encryption: a first-level encryption that cannot be re-encrypted by the proxy and a second-level encryption that can be re-encrypted by the proxy and then decrypted by delegatees. That is, a user Alice uses her secret key to encrypt data, the proxy node uses a proxy key to re-encrypt it for another user Bob, and Bob uses his secret key to decrypt it. Though our approach does not fully expose a data owner’s secret key to the proxy node, data owners implicitly delegate to consumers re-encryption abilities. Since we already trust consumers (by ultimately providing access to data offchain), and since Bob could theoretically just transfer the data offchain, we do not see this as an issue.

Proxy nodes form the first layer of security for the Iris network as well as allow data owners and consumers to ingest and eject data. They operate in two contexts, as a gateway and as a proxy. As a gateway to data ingestion, they allow data owners to add data to the Iris network and create an asset class while also enforcing a minimal set of data verification rules prior to allowing data to be ingested, such as ensuring the data does not contain malicious content such as a virus. Conversely, as a proxy, proxy nodes form an ‘invisible’ proxy layer that re-encrypts data for authorized callers when requested. That is, they are the proxy layer in the proxy re-encryption mechanism. After re-encryption, the data is streamed to the authorized caller who requested the re-encryption.

Proxy nodes must meet a minimum hardware requirement, specifically as determined by the ability to execute offchain services within a TEE (intel 6+ gen). As proxy nodes are the layer between IPFS and the external world, they define the bandwidth of the network, and so we require a minimum bandwidth to become a proxy node, which should be at least as much as the minimum allowable bandwidth of the network as a whole. To become a proxy, a node stakes a minimum number of IRIS tokens. When staked, a proxy routing service routes requests to ingest data, to re-encrypt data, and to eject data to proxy nodes. To be eligible for rewards, a proxy must be online for a predetermined minimum number of sessions. The proxy routing layer collects fees from callers and subsequently distributes fees to proxy nodes as required, based on uptime and bandwidth.

Putting data spaces and proxy nodes together, we arrive at the following design:

![proxy nodes](https://github.com/ideal-lab5/iris-docs/blob/master/resources/proxy.drawio.png)

Additionally, this impacts the data ingestion and ejection workflows. We modify data ingestion to first encrypt data before it is ingested into Iris and to decrypt data when it is retrieved from Iris.

![data ingestion](https://github.com/ideal-lab5/iris-docs/blob/master/resources/data_ingestion.png)

![data ejection](https://github.com/ideal-lab5/iris-docs/blob/master/resources/data_ejection.drawio.png)

#### Composable Access Rules

Iris enables data owners to determine rules that consumers must follow when requesting their data from the network. These rules could include: limited use access tokens, minimum token requirements, time sensitive tokens, and many other use cases. We accomplish this through a set of smart contracts called composable access rules (CARs). They are composable in the sense that the data owner may specify any number of rules, each of which must be validated when a consumer requests data. In general, each rule stipulates a condition which, when met, results in the consumer being blocked from accessing data or in their token(s) being burned. If the token is not burned by the end of execution of each CAR, then the consumer can proceed to fetch the data. In general, the greater the number of CARs, the more costly it will be for consumers to access the data, since they must pay for each one to be executed.

A composable access rule must implement the following functions:

- an `execute(asset_id)` function that produces some boolean output. This function accepts the asset id of an asset class in Iris. It encapsulates the ‘access rule’ logic.
- a `register_asset(asset_id)` function. This function accepts the asset id of an asset class in Iris. When executed successfully, it results in the asset id being ‘registered’ in the rule contract’s storage.

With these two functions implemented and the contract deployed to the chain, data owners can then specify, using the contract address, which rules they want to be applied to their data. When a consumer node fetches data from the network, each of the specified rules must be executed by invoking the execute function in each registered contract. The result of contract failure results in the denial of data access. We use the `bare_call` function from the contracts pallet to invoke contract functions and wait for execution to complete.

#### Storage Node Eligibility and Inclusion

The goal of the storage network design is to ensure that the storage layer is able to remain decentralized as storage demand increases while also retaining high availability. That is, we want to minimize barriers to enter the network as a storage node while also encouraging nodes to maintain a minimum level of availability. In this section, we first discuss the conditions for becoming a storage node and the on-chain representation of storage space in the network.

![storage_provider](https://github.com/ideal-lab5/iris-docs/blob/master/resources/storage_prov_token_flow_ideas.drawio%20(2).png)

First, we introduce a new token to the Iris ecosystem, the OBOL, which is derived from staked IRIS tokens and acts as an on-chain representation of data promised to the network. We define a conversion rate between IRIS and OBOL, which scales based on the ratio of total available storage capacity to total reserved storage capacity, as characterized by an 'ideal storage capacity' ratio. The OBOL is minted on demand and immediately added to an 'availability pool', a special kind of staking pool, which represents an individual node's available storage capacity. In order to prove storage capacity, Iris generates a unique file whose size reflects the amount of OBOL staked, megabyte for megabyte (i.e. 1 OBOL ~ 1MB storage), which the storage node must store. Each piece of data that the storage node commits to storing (i.e. ipfs pin 'cid'), as explained in the Replication Mechanism section, results in a new such file being generated and its storage confirmed. Additionally, tokens are moved from the node's availability pool to a "reserved pool for data asset D".

#### Availability-Encouraging Replication Mechanism

For a mathematical treatment of this mechanism, we defer to section 4 of the Iris whitepaper.

To maximize the decentralized nature of the storage layer of the network, we propose an availability encouraging mechanism based on the "stoachastic replication game" as proposed here https://www.researchgate.net/publication/282894916_Game-Theoretic_Mechanisms_to_Increase_Data_Availability_in_Decentralized_Storage_Systems. The intention behind our mechanism is to build a system where highly-available, high capacity storage nodes are incentivized to provide assistance to lower-capacity nodes with less availability without sacrificing total storage availability. That is, we want to build a storage system that won't eventually become dominated by large data warehouses.

In our mechanism, each storage nodes maintains four sets of other storage nodes int he network,

- The **replica set** (for a data asset D): The set of storage nodes (excluding self) which have previously agreed to replicate some data asset D within the current session.
- The **taboo set**: A set of storage nodes who have rejected replication proposals.
- The **metric set**: A set of nodes that have been scored (by a scoring function) and meet some minimum score. It has size sm < 0. This set is maintained and updated by the T-MAN protocol.
- The **random set**: A set of randomly chosen storage nodes, with maximum size s r > 0. This set is maintained and updated by the T-MAN protocol.

The replica set and metric set are both maintained by the [T-Man protocol](https://www.researchgate.net/publication/225403352_T-Man_Gossip-Based_Overlay_Topology_Management). Through two distinct phases, our mechanism allows a data owner to choose a minimum desired availability and a number of replicas, and the storage system autonomously seeks replicas which satisfy her needs. In the first phase, nodes update their replica and metric sets, which reflect the replica candidates that the node will request replication from in the next round. In the subsequent round, nodes who have been requested by data owners to store and replicate data use the metric and random sets to gossip with peers and request replication. Nodes who agree to replication are granted a share of rewards as provided by data owners. The taboo set maintains a list of candidates who have rejected replication proposals and the replica set a list of candidates who have accepted them.

#### Anonymous Reputation System

We plan to implement an anonymous reputation and feedback system to allow nodes to provide feedbac  to data assets, data owners, and data spaces. We accomplish this by minting a new token for users after a (cryptographically verified) interaction has occurred between a buyer and a seller, called a repcoin, which allows nodes to submit a non-interactive zero-knowledge proof to a "bulletin board".

![repcoin awarding](https://github.com/ideal-lab5/iris-docs/blob/master/resources/repcoin_reward%20(1).png)

Users generate a cryptogram containing a binary decision, either 0 or 1, to indicate that the interaction with the seller or their data positive (+1) or negative (0). Any node in the network can then reconstruct the reputation of the seller, data, or data space by taking the product of each cryptogram within the 'bulletin board'. The restructuring of reputation scores can be computed by any node with access to the bulletin board.

![voting](https://github.com/ideal-lab5/iris-docs/blob/master/resources/anonymous_feedback_system.drawio.png)

For a mathematical description of this process, please refer to section 5 of the Iris whitepaper (link here)

#### IRIS SDK

We intend to adapt the user interface developed as part of the initial Iris grant proposal for usage as an SDK for dapp developers on Iris. The SDK will facilitate development of user interfaces for contracts running on the Iris blockchain. As an exmaple, the Iris Asset Exchange UI from the initial grant proposal is one such application. Additionally, we intend to develop a user interface to allow users to easily find and use smart contracts deployed on Iris through a 'hub-like' portal.

#### Prior Work

This proposal is a followup to a completed web3 foundation grant which laid the foundation for these enhancements. With our initial proposal, we delivered:

- a layer on substrate that builds a  cryptographic relationship between on-chain asset ownership and actual off-chain data storage
- a naïve validator-based storage system  that encourages storage of 'valuable' data
- support for contracts and a rudimentary decentralized exchange to purchase and sell data access rights

#### Limitations and Expectations

Iris is *not* intended to act as a decentralized replacement for traditional cloud storage or to be a competitor to existing centralized data storage solutions. We are aware that Amazon could artifically deflate the price of S3 buckets down to $0 and still make billions a year, while this project would inevitably implode.

### Ecosystem Fit

Help us locate your project in the Polkadot/Substrate/Kusama landscape and what problems it tries to solve by answering each of these questions:

- Where and how does your project fit into the ecosystem?
- Who is your target audience (parachain/dapp/wallet/UI developers, designers, your own user base, some dapp's userbase, yourself)?
- What need(s) does your project meet?
- Are there any other projects similar to yours in the Substrate / Polkadot / Kusama ecosystem?
  CESS, Crust
  - If so, how is your project different?
    - embedded IPFS
    - availability encouruaging storage system
    - proxy re-encryption mechanism and reputation
    - Iris believes that not all data is equal and though we do not intend to impose any type of authority in terms of moderation or censorship, we provide mechanisms for data owners to create curated data enclaves.

  - If not, are there similar projects in related ecosystems?

## Team :busts_in_silhouette:

### Team members

Tony Riemer: co-founder/CTO/developer
Developer X: developer
Sebastian Spitzer: co-founder/CEO
Brian Thamm: co-founder/COO

### Contact

- **Contact Name:** Tony Riemer
- **Contact Email:** driemworks@idealabs.network
- **Website:** https://idealabs.network

### Legal Structure

- **Registered Address:** 123 Main St, Arlington, VA 22046, USA
- **Registered Legal Entity:** Ideal Labs, Ltd.

### Team's experience

[Brief overview of who we are here]
- Something about how we all come from different backgrounds

**Tony Riemer**
Tony is a full-stack engineer with 6 years of experience building enterprise applications for fortune 500 companies, including both Fannie Mae and Capital One. Notably, he was the lead engineer of Capital One's "Bank Case Management" system, has lead several development teams to quickly and successfully bringing products to market while adhering to the strictest code quality and testing practices, and has acted as a mentor to other developers. Additionally, he holds a breadth of knowledge in many languages and programming paradigms and has built several open-source projects, including Mercury, a proof of work blockchain written in Go, an OpenCV-based augmented reality application for Android, as well as experiments in home automation and machine learning. More recently, he is the founder and lead engineer at Ideal Labs, where he single-handedly designed the Iris blockchain and implemented the prototype, which was funded via a web3 foundation grant. He is passionate about decentralized technology and strives to make the promises of decentralization a reality.

**Developer X**
TODO.

**Sebastian Spitzer**
Intrapreneur, business builder, networker and technology scout.

Originally started in marketing, Sebastian ventured into innovation and has built up innovation processes and networks in 3 different verticals. He manages the funnel from ideation to pilots and scaling, and successfully transitioned companies to agile innovation using lean startup and design thinking concepts.

Besides pitching 30+ business concepts at CxO level, he scaled-up 2 concepts as an intrapreneur, adding 2m+ of annual turnover within the first 2 years to market. 

He loves working with creative minds and supports start-ups on their journey through value proposition, business plan, strategy and market fit.

He is actively immersed in global VC and start-up ecosystems and has built a strong footprint in the Blockchain industry since 2016 as a tech scout, advisor and consultant. He supported 2 projects from inception to reaching 200m+ market caps and assists dealflow activities for a VC in the space, analyzing projects along their tech, team and market and working closely with founders to ensure their success.

**Brian Thamm**
Brian Thamm has over a decade of experience leading large data initiatives in both the Commercial and U.S. Federal Sectors.

After several years working with companies ranging from the Fortune 500 to startups, Brian decided to focus on his passion for enabling individuals to use data to improve their decision-making by starting Sophinea. Since its founding in late 2018, Sophinea has lead multi-national Data Analytics Modernization efforts in support of the United State’s Department of State and the National Institutes of Health.

Brian believes that data-driven success is something that often requires top level support, but is only achieved when users of the information are significantly engaged and are able to seamlessly translate data into decision-making. This requires expertise, but often also novel datasets. Brian believes that blockchain ecosystems represent a generational opportunity to develop these novel data sets by providing a fairer incentive structure to reward data providers for their efforts and willingness to share.

This is a follow-up grant to Iris: https://github.com/w3f/Grants-Program/blob/master/applications/iris.md

### Team Code Repos

- https://github.com/ideal-lab5
- https://github.com/ideal-lab5/substrate
- https://github.com/ideal-lab5/contracts
- https://github.com/ideal-lab5/ui

Please also provide the GitHub accounts of all team members. If they contain no activity, references to projects hosted elsewhere or live are also fine.

- https://github.com/driemworks
- https://github.com/<team_member_2>

### Team LinkedIn Profiles (if available)

- https://www.linkedin.com/in/tony-riemer/
- https://www.linkedin.com/<person_2>

## Development Status :open_book:

As stated earlier in the [prior work](#prior-work) section, we have delivered the three milestones detailed as part of the intiial Iris grant proposal. Additionally, we have also synced the Iris repository with the latest substrate master branch as well as have upgraded to use the latest rust-ipfs version (see here: TODO).

## Development Roadmap :nut_and_bolt:

### Overview

- **Total Estimated Duration:** 4 months
- **Full-Time Equivalent (FTE):**  3 FTE
- **Total Costs:** 120,000

### Milestone 1 — Implement Data Spaces and Composable Access Rules

- **Estimated duration:** 1.5 months
- **FTE:**  2.5
- **Costs:** 20,000 USD

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 / GPLv3 / MIT / Unlicense |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.)
| 1. | Substrate module: X | We will create a Substrate module that will... (Please list the functionality that will be implemented for the first milestone) |  
| 2. | Substrate module: Y | We will create a Substrate module that will... |  
| 3. | Substrate module: Z | We will create a Substrate module that will... |  
| 4. | Substrate chain | Modules X, Y & Z of our custom chain will interact in such a way... (Please describe the deliverable here as detailed as possible) |  

### Milestone 2 — Implement Storage System

- **Estimated Duration:** 1.5 month
- **FTE:**  2.5
- **Costs:** 20,000 USD

### Milestone 3 - Implement Proxy Re-encryption Scheme

- **Estimated Duration:** 1.5 month
- **FTE:**  2.5
- **Costs:** 20,000 USD

### Milestone 4 - Implement Anonymous Reputation System

## Future Plans

- work with dapp developers to assist in develop of applications on Iris. We are already in communication with several projects that are interested in leveraging Iris as their storage system.
- strive to become a parachain on Kusama and Polkadot
- continue to expand the features available in Iris, such as partial ownership of assets
- a

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** Web3 Foundation Website / Medium / Twitter / Element / Announcement by another team / personal recommendation / etc.

Here you can also add any additional information that you think is relevant to this application but isn't part of it already, such as:

- Work you have already done.
- If there are any other teams who have already contributed (financially) to the project.
- Previous grants you may have applied for.
