# W3F Grant Proposal

- **Project Name:** Iris
- **Team Name:** Ideal Labs
- **Payment Address:** Ethereum: 0x6ec0D6c005797a561f6F3b46Ca4Cf43df3bF7888 (DAI)
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 2

## Project Overview :page_facing_up:

Followup to: https://github.com/w3f/Grants-Program/pull/812

### Overview

Iris is a decentralized network that provides a secure data storage, delivery, and ownership layer for Web 3.0 applications. It is infrastructure for the decentralized web, providing a storage and data exchange which enables the transfer and monetization of access to and ownership of data across chains, smart contracts and participants in the network or connected through a relay chain. Iris provides security, availability, reputation, and governance on top of IPFS, enabling data ownership, access management, and the commodification of latent storage capacity and content delivery. It applies defi concepts to data, reputation, storage capacity and availability to synthesize computation and storage and to represent off-chain assets in an on-chain context. Ideal Labs wants to be the forefront of the next data revolution, and to help build the tools needed for a transparent and fair data economy.

Ideal Labs wants to be at the forefront of the next data revolution.

### Project Details

The intention of this followup grant is to implement several features that enable Iris to be a secure, social, and highly-available storage network without compromising the highest degrees of decentralization. In our system, data owners associate their data with any number of 'data spaces' which each have specific rule sets and inclusion policies. To enhance security, we use a novel proxy re-ecryption mechanism that does not expose any secret keys as well as introduce the concept of the proxy node which enables ciphertext to be re-encrypted. Additionally we introduce "composable access rules", which allow data owners to specify rules which are implicitly enforced when consumers access their data. Lastly, we will build a javascript SDK to allow user interfaces for dapps built on Iris to easily build applications and interface with Iris.

This proposal makes several improvements on top of the existing Iris blockchain, specifically in terms of security, extensibility, data organiziation, and data ingestion/ejection.

To summarize, in the following we propose:

- the introduction of "data spaces"
- the implementation of "composable access rules" to apply custom business logic to data
- the introduction of "proxy nodes" that enable proxy re-encryption within the network
- a javascript SDK to allow dapp developers to easily build front ends for smart contracts on Iris

For further details, we direct the reader to view our whitepaper here: [https://www.idealabs.network/docs](https://www.idealabs.network/docs)

#### Tech Stack

- IPFS (specifically rust-ipfs)
- Substrate
- Javascript/Typescript

#### Terminology

- Data Asset Class and Data Asset: A **data asset class** is an asset class which is associated with some owned data, as identified by its CID, which has been ingested into the Iris storage system at some point in the past (does not necessarily mean it is currently available in Iris). A **data asset** is an asset minted from the data asset class.

- Data Owner: A **data owner** is any node that has initiated the ingestion of some data and owns a data asset class. In the following, we may refer to data owners as "Alice".

- Data Consumer: A **data consumer** is any node that owns a data asset minted from a data asset class. In the following, we may refer to data consumers as "Bob".

- Proxy Node: A **proxy node** is any node that has the minimum requried hardware requirements and has staked the minimum required amount IRIS tokens to become a proxy node.

- Storage Node: A **storage node** is any node that has staked the minimum required amount IRIS tokens to become a storage node.

#### Data Spaces

A data space is a user-defined configuration and rule set that allows users to group data together into curated collections. More explicitly, the data space lays the foundation to enable user-defined moderation rules within the network and provides a means to securely associate disparate data sets with one another (i.e. under an ‘organization’). All data in Iris exists within the context of a space, however, there is no limit to the number of spaces a data owner may associate their data with.

Under the hood, the 'data space' is an asset class which is mapped to a set of configuration items that determine the inclusion and moderation policies of the space. These asset classes exist much the same as the concept of 'data assets' from the original Iris proposal. In a data space with a private inclusion policy, only nodes holding tokens minted from the data space asset class are authorized to associate data with the data space. We see this as an additional market opportunity for the data economy, as ownership of a space may become important based on the sum of data stored within. Data spaces may also be subject to  [composable access rules](#composable-access-rules) at some point in the future.

Further, data spaces form the basis for moderation, or curation, within the network. Each data space may have a set of rules associated with it which limits not only the type of data that can be associated with the space, but also with the contents of the data. We intend to accomplish this through the execution of machine learning models, bayesian filters, and more, within a trusted execution environment. However, that work is outside the scope of this proposal.

![data spaces diagram](https://github.com/ideal-lab5/iris-docs/blob/master/resources/data_spaces.drawio.png)

#### Composable Access Rules

Iris enables data owners to determine rules that consumers must follow when requesting their data from the network. These rules could include: limited use access tokens, minimum token requirements, time sensitive tokens, and many other use cases. We accomplish this through a set of smart contracts called composable access rules (CARs). They are composable in the sense that the data owner may specify any number of rules, each of which must be validated when a consumer requests data. In general, each rule stipulates a condition which, when met, results in the consumer being blocked from accessing data or in their token(s) being burned. If the token is not burned by the end of execution of each CAR, then the consumer can proceed to fetch the data. In general, the greater the number of CARs, the more costly it will be for consumers to access the data, since they must pay for each one to be executed.

A composable access rule must implement the following functions:

- an `execute(asset_id)` function that produces some boolean output. This function accepts the asset id of an asset class in Iris. It encapsulates the ‘access rule’ logic.
- a `register_asset(asset_id)` function. This function accepts the asset id of an asset class in Iris. When executed successfully, it results in the asset id being ‘registered’ in the rule contract’s storage.

With these two functions implemented and the contract deployed to the chain, data owners can then specify, using the contract address, which rules they want to be applied to their data. When a consumer node fetches data from the network, each of the specified rules must be executed by invoking the execute function in each registered contract. The result of contract failure results in the denial of data access. We use the `bare_call` function from the contracts pallet to invoke contract functions and wait for execution to complete.

#### Proxy/Gateway Nodes

Proxy. or gateway. nodes form the basis of secure data ingestion and ejection from the network. Ultimately, these nodes will be responsible for powering Iris' proxy re-encryption mechanism. This mechanism is out of the scope of the current proposal, however, we will create this node role as to provide the foundation for subsequent work in which the proxy re-encryption scheme will be implemented. In the implementation proposed here, these nodes simply act as ingress and egress points to/from the IPFS network. Additionally, we will implement zk proofs once the offchain worker adds data to IPFS and publishes the cid onchain, which results in the creation of a data asset.

Putting data spaces and proxy nodes together, we arrive at the following design:

![proxy nodes](https://github.com/ideal-lab5/iris-docs/blob/master/resources/proxy.drawio.png)

Additionally, this impacts the data ingestion and ejection workflows. We modify data ingestion to first encrypt data before it is ingested into Iris and to decrypt data when it is retrieved from Iris.

![data ingestion](https://github.com/ideal-lab5/iris-docs/blob/master/resources/data_ingestion.png)

![data ejection](https://github.com/ideal-lab5/iris-docs/blob/master/resources/data_ejection.drawio.png)

#### Proxy Node Reward Structure

Each data ingestion and ejection transaction has an additional transaction fee which is then pooled together and distributed to proxy nodes at the end of a session based on their online 

#### Availability-Encouraging Replication Mechanism

For a mathematical treatment of this mechanism, we defer to section 4 of the Iris whitepaper.

To maximize the decentralized nature of the storage layer of the network, we propose a game theoretic availability-encouraging mechanism based on the "stochastic replication game" as proposed [here](https://www.researchgate.net/publication/282894916_Game-Theoretic_Mechanisms_to_Increase_Data_Availability_in_Decentralized_Storage_Systems). The intention behind our mechanism is to build a system where highly-available, high capacity storage nodes are incentivized to provide assistance to lower-capacity nodes with less availability without sacrificing total storage availability. That is, we want to build a storage system that won't eventually become dominated by large data warehouses.

In our mechanism, each storage nodes maintains four sets of other storage nodes int he network,

- The **replica set** (for a data asset D): The set of storage nodes (excluding self) which have previously agreed to replicate some data asset D within the current session.
- The **taboo set**: A set of storage nodes who have rejected replication proposals.
- The **metric set**: A set of nodes that have been scored (by a scoring function) and meet some minimum score. It has size sm < 0. This set is maintained and updated by the T-MAN protocol.
- The **random set**: A set of randomly chosen storage nodes, with maximum size s r > 0. This set is maintained and updated by the T-MAN protocol.

The replica set and metric set are both maintained by the [T-Man protocol](https://www.researchgate.net/publication/225403352_T-Man_Gossip-Based_Overlay_Topology_Management). Through two distinct phases, our mechanism allows a data owner to choose a minimum desired availability and a number of replicas, and the storage system autonomously seeks replicas which satisfy her needs. In the first phase, nodes update their replica and metric sets, which reflect the replica candidates that the node will request replication from in the next round. In the subsequent round, nodes who have been requested by data owners to store and replicate data use the metric and random sets to gossip with peers and request replication. Nodes who agree to replication are granted a share of rewards as provided by data owners. The taboo set maintains a list of candidates who have rejected replication proposals and the replica set a list of candidates who have accepted them.

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

- **Registered Address:** TBD, in progress
- **Registered Legal Entity:** Ideal Labs, Ltd.

### Team's experience

Iridium Labs is composed of a group of individuals coming from diverse backgrounds across tech, business, and marketing.

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
https://www.linkedin.com/in/brianthamm/
https://www.linkedin.com/in/sebastian-s-253502159/

## Development Status :open_book:

As stated earlier in the [prior work](#prior-work) section, we have delivered the three milestones detailed as part of the intiial Iris grant proposal. Additionally, we have also synced the Iris repository with the latest substrate master branch as well as have upgraded to use the latest rust-ipfs version (see here: TODO).

## Development Roadmap :nut_and_bolt:

### Overview

- **Total Estimated Duration:** 5 months
- **Full-Time Equivalent (FTE):**  2.5 FTE
- **Total Costs:** 90,000

### Milestone 1 — Implement Data Spaces and Composable Access Rules

- **Estimated duration:** 1 month
- **FTE:**  2.5
- **Costs:** 10,000 USD

This milestone delivers the creation of data spaces, the ability to manage data spaces associated with data, and lays the groundwork for future data-space moderation capabilities. It also delivers composable access rules to the Iris network, allowing data owners to specify unique business models that consumers must adhere to across any number of data spaces.

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 1. | Substrate module: Iris-Spaces | Create a pallet, similar to iris-assets pallet, that acts as a wrapper around the assets pallet and allows nodes to construct new data spaces and mint data space access tokens. |  
| 2. | Substrate module: Iris-Assets | Modify the iris-assets pallet to allow nodes to specify data spaces with which to associate their data. Additionally, we implement the logic required to verify that the data owner holds tokens to access the space. |  
| 3. | Contracts | Create the trait definition for a Composable Access Rule and develop the following composable access rules: limited use token: allow a token associated with an asset id to be used only 'n' times perishable token: allow a token associated with an asset id to be used only before some specific date |  
| 4. | Substrate Module: Iris-Assets | Execute composable access rules associated with an asset id when requesting data from Iris. |  

### Milestone 2 - Proxy/Gateway Nodes

- **Estimated Duration:** 1 months
- **FTE:**  2.5
- **Costs:** 30,000 USD

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |

### Milestone 3 - Storage System

- **Estimated Duration:** 2 months
- **FTE:**  2.5
- **Costs:** 30,000 USD

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |

### Milestone 4 - Javascript SDK and other apps

- **Estimated Duration:** 1 months
- **FTE:**  2.5
- **Costs:** 8,000 USD

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |

## Future Plans

- work with dapp developers to assist in develop of applications on Iris. We are already in communication with several projects that are interested in leveraging Iris as their storage system.
- strive to become a parachain on Kusama and Polkadot
- continue to expand the features available in Iris, such as partial ownership of assets
- a

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** Web3 Foundation Website

Here you can also add any additional information that you think is relevant to this application but isn't part of it already, such as:

- Work you have already done.
- If there are any other teams who have already contributed (financially) to the project.
- Previous grants you may have applied for.
