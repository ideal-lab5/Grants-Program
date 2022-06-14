# W3F Grant Proposal

- **Project Name:** Iris
- **Team Name:** Ideal Labs
- **Payment Address:** 0xB7E92CCDE85B8Cee89a8bEA2Fd5e5EE417d985Ad (DAI)
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 3

## Project Overview :page_facing_up:

This is a follow-up grant to Iris:

- https://github.com/w3f/Grants-Program/pull/812
- https://github.com/w3f/Grants-Program/blob/master/applications/iris.md

### Overview

Iris is a decentralized **data exchange protocol** that enables a secure data storage, delivery, and ownership layer for Web 3.0 applications. It is infrastructure for the decentralized web, providing a data exchange which enables the transfer and monetization of access to and ownership of data across chains, smart contracts and participants in the network or connected through a relay chain, by building a cryptographically verifiable relationship between storage and ownership. Iris provides security, availability, reputation, and governance on top of storage, enabling data ownership, access management, smart contract support for data, and content delivery. It applies defi concepts to data, reputation, storage capacity and availability to synthesize computation and storage and to represent off-chain assets in an on-chain context.

### Project Details

The intention of this follow-up grant is to implement several features that enable Iris to be a secure, social, and highly-available data exchange protocol without compromising decentralization. In our system, data owners associate their data with any number of 'data spaces' which each have specific rule sets and inclusion policies. We also lay the foundations for our encryption scheme (we will use a threshold encryption mechanism, though the implementation of this is out of scope for this proposal) by introducing the concept of the proxy node which will act as the linchpin for re-encryption in the future, as well as allows data owners and data consumers to run light clients (as they no longer are required to run the full node and add data to the embedded IPFS node). Additionally we introduce "composable access rules", which allow data owners to specify rules which are implicitly enforced when consumers access their data. Lastly, we will build a javascript SDK to allow user interfaces for dapps built on Iris to easily build applications and interface with Iris.

This proposal makes several improvements on top of the existing Iris blockchain, specifically in terms of security, extensibility, data organiziation, and data ingestion/ejection.

To summarize, in the following we propose:

- the introduction of "data spaces"
- the implementation of "composable access rules" to apply custom business logic to data
- the introduction of "proxy nodes" that enable threshold encryption within the network
- implementation of an offchain client to send/receive data to/from proxy nodes
- introduce a configurable storage layer to support storage in multiple backends as well as usage of go-ipfs for hot storage
- a javascript SDK to allow dapp developers to easily build front ends for smart contracts on Iris

For further details, we direct the reader to view our whitepaper draft here: [https://www.idealabs.network/docs](https://www.idealabs.network/docs)

Note: The whitepaper is somewhat outdated, specifically in sections where it references the storage system.

#### Tech Stack

Frameworks/Libraries:

- go-ipfs
- Substrate
- Crust Network

Languages:

- Rust
- Go
- Javascript/Typescript

#### Terminology

- Data Asset Class and Data Asset: A **data asset class** is an asset class which is associated with some owned data which has been ingested into the Iris storage system at some point in the past. A **data asset** is an asset minted from the data asset class.

- Data Owner: A **data owner** is any node that has initiated the ingestion of some data and owns a data asset class. In the following, we may refer to data owners as "Alice".

- Data Consumer: A **data consumer** is any node that owns a data asset minted from a data asset class. In the following, we may refer to data consumers as "Bob".

- Proxy Node: A **proxy node** is any node that has the minimum required hardware requirements and has staked the minimum required amount IRIS tokens to become a proxy node.

#### Data Spaces

A data space is a user-defined configuration and rule set that allows users to group data together into curated collections. More explicitly, the data space lays the foundation to enable user-defined moderation rules within the network and provides a means to securely associate disparate data sets with one another (i.e. under an ‘organization’). All data in Iris exists within the context of a space, however, there is no limit to the number of spaces a data owner may associate their data with.

Under the hood, the 'data space' is an asset class which is mapped to a set of configuration items that determine the inclusion and moderation policies of the space. These asset classes exist much the same as the concept of 'data assets' from the original Iris proposal. In a data space with a private inclusion policy, only nodes holding tokens minted from the data space asset class are authorized to associate data with the data space. We see this as an additional market opportunity for the data economy, as ownership of a space may become important based on the sum of data stored within. Data spaces may also be subject to  [composable access rules](#composable-access-rules) at some point in the future.

Further, data spaces form the basis for moderation, or curation, within the network. Each data space may have a set of rules associated with it which limits not only the type of data that can be associated with the space, but also with the contents of the data. We intend to accomplish this through the execution of machine learning models, bayesian filters, and more, within a trusted execution environment. However, that work is outside the scope of this proposal.

![data spaces diagram](https://github.com/ideal-lab5/Grants-Program/blob/iris_followup/src/data_spaces.drawio.png)

#### Composable Access Rules

Iris enables data owners to determine rules that consumers must follow when requesting their data from the network. These rules could include: limited use access tokens, minimum token requirements, time sensitive tokens, and many other use cases. We accomplish this through a set of smart contracts called composable access rules (CARs). They are composable in the sense that the data owner may specify any number of rules, each of which must be validated when a consumer requests data. In general, each rule stipulates a condition which, when met, results in the consumer being blocked from accessing data or in their token(s) being burned. For example, a "limited use token contract" would track the number of times a token is used to fetch data, and forces that token to be burned once the threshold is reached. If the token is not burned by the end of execution of each CAR, then the consumer can proceed to fetch the data. In general, the greater the number of CARs, the more costly it will be for consumers to access the data, since they must pay for each one to be executed.

A composable access rule must implement the following functions:

- an `execute(asset_id)` function that produces some boolean output. This function accepts the asset id of an asset class in Iris. It encapsulates the ‘access rule’ logic.
- a `register_asset(asset_id)` function. This function accepts the asset id of an asset class in Iris. When executed successfully, it results in the asset id being ‘registered’ in the rule contract’s storage.

With these two functions implemented and the contract deployed to the chain, data owners can then specify, using the contract address, which rules they want to be applied to their data. When a consumer node fetches data from the network, each of the specified rules must be executed by invoking the `execute` function in each registered contract. The result of contract failure results in the denial of data access. We use the `bare_call` function from the contracts pallet to invoke contract functions and wait for execution to complete.

#### Proxy Nodes and offchain clients for data ingestion/ejection

Proxy nodes form the basis of secure data ingestion and ejection from the network. Ultimately, these nodes will be responsible for powering Iris's threshold encryption mechanism. This mechanism is out of the scope of the current proposal, however, we will create this node role to provide the foundation for subsequent work in which the threshold encryption scheme will be implemented, as well as make data ingestion/ejection workflows easier for data owners and consumers. In the implementation proposed here, these nodes simply act as ingress and egress points to/from the IPFS network. Though proxy nodes do not increase the security of the network at this point, they do remove the dependency on data consumers and data owner to run a full Iris node to enable data ingestion and ejection.

Putting data spaces and proxy nodes together, we arrive at the following design:

![proxy nodes](https://github.com/ideal-lab5/Grants-Program/blob/iris_followup/src/proxy_data_spaces_io.drawio.png)

##### Offchain Client

The inclusion of proxy nodes impacts data ingestion and ejection workflows. We will build a secure offchain client using Go in order to serve files for ingestion by proxy nodes and to listen for data sent by proxy nodes when requested from the network.

In the initial design of Iris, data ingestion functioned by allowing a data owner, who is running a full iris node, to add some data to their embedded IPFS node to gossip with a validator node, who would then add the data to their own embedded IPFSS node, which is connected with other validator nodes. This poses several issues. Not only is it insecure, but also it forces data owners to always run a full node which limits the ease of use of the system. Similarly, data ejection functioned by allowing a data consumer to directly connect their embedded IPFS node to a validator node to retrieve data from the 'validator network', which introduced a similar set of issues that data owners would face. Proxy nodes allow us to circumvent both of these issues by acting as a gateway to the IPFS network. To accomplish data ingestion, nodes will run an offchain client, which acts as a file server that only authorized proxy nodes can call into.

![data ingestion](https://github.com/ideal-lab5/Grants-Program/blob/iris_followup/src/data_injection.drawio.png)

For data ejection, data consumers run the offchain client which will listen for connections from proxy nodes, accept authorized connections, and provide data consumers the ability to fetch data without running a full node.

![data ejection](https://github.com/ideal-lab5/Grants-Program/blob/iris_followup/src/data_ejection.drawio.png)

#### Proxy Node Reward Structure

Each data ingestion and ejection transaction has an additional transaction fee which is then pooled together and distributed to proxy nodes at the end of a session based on their participation during the session. This will function similarly to how proof of stake consensus algorithms reward validator nodes when new blocks are accepted into the chain.

#### Storage System Integration

There are two storage layers in Iris, a 'hot' storage layer which is supported by the proxy nodes, and a 'cold' storage layer which exists offchain.

We will build a generic pallet that allows for any given storage backend to be configured for use with Iris. The intention behind this is that it may allow the network to function independently of any one given storage solution. Initially, we will build a pallet which acts a connector to the Crust network via XCM. The pallet will expose two main extrinsics, a 'read' and a 'write' extrinsic. Additionally, we will demonstrate how this can be used to allow connections to other storage backends by also building a connector to an external, centralized data store.

In order to properly test and verify this, we will also setup our own relay chain with both Iris and Crust deployed as parachains.

#### IRIS SDK

We intend to adapt the user interface developed as part of the initial Iris grant proposal for usage as an SDK for dapp developers on Iris. The SDK will facilitate development of user interfaces for contracts running on the Iris blockchain. As an example, the Iris Asset Exchange UI from the initial grant proposal is one such application. We will adapt the current Iris UI to use this new SDK as well as demonstrate how to develop a simple data marketplace.

Specifically, we intend to adapt the SDK from the functions available in these files: [https://github.com/ideal-lab5/ui/tree/master/src/services](https://github.com/ideal-lab5/ui/tree/master/src/services).

#### Prior Work

This proposal is a followup to a completed web3 foundation grant which laid the foundation for these enhancements. With our initial proposal, we delivered:

- a layer on substrate that builds a  cryptographic relationship between on-chain asset ownership and actual off-chain data storage
- a naïve validator-based storage system  that encourages storage of 'valuable' data
- support for contracts and a rudimentary decentralized exchange to purchase and sell data access rights

In addition to these items, the Iris fork of substrate has been upgraded to by in sync with the latest substrate master branch and we have also upgraded the runtime to use the latest version of rust-ipfs (see here: [d9e14294c89abd2c085e91d8312041245b5d3b35](https://github.com/ideal-lab5/substrate/commit/d9e14294c89abd2c085e91d8312041245b5d3b35)).

#### Limitations and Expectations

Iris is *not* intended to act as a decentralized replacement for traditional cloud storage or to be a competitor to existing data storage solutions. In fact, Iris is not really a storage layer at all. Further, this proposal does not address security (i.e. threshold encryption) in the network, nor does it appproach moderation capabilities, though it does lay the foundation for those items to be implemented in the next phase of development.

### Ecosystem Fit

- Where and how does your project fit into the ecosystem?

  - Iris intends to be infrastructure for dapps that leverage decentralized storage and ownership capabilities. Further, our long term intention is to participate in a parachain auction once Iris is feature complete, after which we will be able to provide cross-chain data ownership that is cryptographically linked with their stored data.

- Who is your target audience (parachain/dapp/wallet/UI developers, designers, your own user base, some dapp's userbase, yourself)?

  - The target audience is very wide ranging as we aim to provide infrastructure for dapps and potentially other parachains to allow them to easily take advantage of decentralized storage capabilities. We also aim to provide tools for data owners and content creators to share or sell their data without a middleman, determining their own prices and business models. Additionally, data spaces allow organizations to control which data can be associated with them, which may allow Iris to be a used in B2B or middleware within existing centralized applications.

- What need(s) does your project meet?
  - The basis of Iris is the creation of a cryptographically verifiable relationship between data ownership, accessibility, and availability. We aim to treat data as an on-chain asset as well as provide a robust governance and moderation framework, which can assist in IP protections and safeguarding the network from being used for abusive or malicious purposes (such as hosting malware, illicit or illegal materials, plagiarized materials, etc.).

- Are there any other projects similar to yours in the Substrate / Polkadot / Kusama ecosystem?


## Team :busts_in_silhouette:

### Team members

Tony Riemer: co-founder/engineer
Tom Richards: engineer
Sebastian Spitzer: co-founder
Brian Thamm: co-founder

### Contact

- **Contact Name:** Tony Riemer
- **Contact Email:** driemworks@idealabs.network
- **Website:** https://idealabs.network

### Legal Structure

- **Registered Address:** TBD, in progress
- **Registered Legal Entity:** Ideal Labs (TBD)

### Team's experience

Ideal Labs is composed of a group of individuals coming from diverse backgrounds across tech, business, and marketing, who all share a collective goal of realizing a decentralized storage layer that can facilitate the emergence of dapps that take advantage of decentralized storage.

**Tony Riemer**

Tony is a full-stack engineer with over 6 years of experience building enterprise applications for fortune 500 companies, including both Fannie Mae and Capital One. Notably, he was the lead engineer of Capital One's "Bank Case Management" system, has lead several development teams to quickly and successfully bring products to market while adhering to the strictest code quality and testing practices, and has acted as a mentor to other developers. Additionally, he holds a breadth of knowledge in many languages and programming paradigms and has built several open-source projects, including a proof of work blockchain written in Go, an OpenCV-based augmented reality application for Android, as well as experiments in home automation and machine learning. More recently, he is the founder and lead engineer at Ideal Labs, where he single-handedly designed the Iris blockchain and implemented the prototype, which was funded via a web3 foundation grant. He is passionate about decentralized technology and strives to make the promises of decentralization a reality.

**Tom Richards**

A programming fanatic with six years of professional experience who always thrives to work on emerging technologies, especially in substrate and cosmos to create an impact in the web 3.0 revolution. I believe WASM is the future and has great leverage over EVM, and that is the reason why I started building the Polkadot ecosystem, currently developing smart contract applications and tools with Rust.

**Sebastian Spitzer**

Sebastian started his career in marketing, from where he ventured into innovation management and built up innovation units and startup ecosystems in 3 verticals.

Essentially working as an intrapreneur in the automotive sector for the last 6 years, he managed the entire funnel from ideation to piloting and ultimately scaling digital solutions. In the last 3 years he scaled up 2 projects, adding 2m+ of annual turnover within the first 2 years to market.

In 2016 he first came into contact with the Blockchain space when scouting new tech and got deeply immersed when he saw the disruptive potentials. Since then he has become an advisor for two projects, supporting them in their growth to 200m+ market caps, worked as a dealflow manager/analyst for a crypto VC and helped founders on their early stage journey in the space.

In 2019 he authored a research paper on why the data economy in the automotive industry fails. Since then he was looking for an adequate solution and co-founded Ideal Labs 2 years later after discovering Tony's initial work.

**Brian Thamm**

Brian Thamm has over a decade of experience leading large data initiatives in both the Commercial and U.S. Federal Sectors.

After several years working with companies ranging from the Fortune 500 to startups, Brian decided to focus on his passion for enabling individuals to use data to improve their decision-making by starting Sophinea. Since its founding in late 2018, Sophinea has lead multi-national Data Analytics Modernization efforts in support of the United State’s Department of State and the National Institutes of Health.

Brian believes that data-driven success is something that often requires top level support, but is only achieved when users of the information are significantly engaged and are able to seamlessly translate data into decision-making. This requires expertise, but often also novel datasets. Brian believes that blockchain ecosystems represent a generational opportunity to develop these novel data sets by providing a fairer incentive structure to reward data providers for their efforts and willingness to share.

### Team Code Repos

- https://github.com/ideal-lab5/substrate
- https://github.com/ideal-lab5/iris
- https://github.com/ideal-lab5/contracts
- https://github.com/ideal-lab5/ui

Please also provide the GitHub accounts of all team members. If they contain no activity, references to projects hosted elsewhere or live are also fine.

- https://github.com/driemworks
- https://github.com/crytodevkj
- https://github.com/bgt


### Team LinkedIn Profiles (if available)

- https://www.linkedin.com/in/tony-riemer/
- linkedin.com/in/haulerkonj
- https://www.linkedin.com/in/brianthamm/
- https://www.linkedin.com/in/sebastian-s-253502159/

## Development Status :open_book:

As stated earlier in the [prior work](#prior-work) section, we have delivered the three milestones detailed as part of the intiial Iris grant proposal. Additionally, we have also synced the Iris repository with the latest substrate master branch as well as have upgraded to use the latest rust-ipfs version (see here: [d9e14294c89abd2c085e91d8312041245b5d3b35](https://github.com/ideal-lab5/substrate/commit/d9e14294c89abd2c085e91d8312041245b5d3b35)).

## Development Roadmap :nut_and_bolt:

### Overview

- **Total Estimated Duration:** 4 months (16 weeks)
- **Full-Time Equivalent (FTE):**  2.5 FTE
- **Total Costs:** 73,000

### Milestone 1 — Implement Data Spaces and Composable Access Rules

- **Estimated duration:** 1 month
- **FTE:**  1.5
- **Costs:** 9,000 USD

This milestone delivers two distinct deliverables.

1. We abandon the integration with rust-ipfs which was used as part of the original iris proposal in place of go-ipfs. There are several reasons to do this, but the main reason is that rust-ipfs is still under development and is not yet feature complete, which severely limits its capabilities and security. Additionally, integration with rust-ipfs by embedding it within the substrate runtime adds significant development and maintenance overhead while the benefits do not outweight usage of an external ipfs instance with go-ipfs and http bindings/communication via offchain workers.

2. We introduce data spaces, the ability to manage data spaces associated with data, and lay the groundwork for future data-space moderation capabilities. We also delivers composable access rules to the Iris network, allowing data owners to specify unique business models that consumers must adhere to across any number of data spaces. Specifically, we will implement a composable access rule and associated rule executor contract which allows a token to be 'redeemed' only a limited number of times (e.g. ownership of a token implies you can fetch associated data only N times from the network).

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 1. | Separate Iris node from substrate runtime fork | We migrate the existing pallets used in iris to a new repository based on the substrate node template |
| 2. | Replace offchain::ipfs bindings with http bindings | We abandon the integration with the embedded rust-ipfs instance in favor of using http bindings which communicate with go-ipfs |
| 3. | Substrate module: DataSpaces | Create a pallet, similar to iris-assets pallet, that acts as a wrapper around the assets pallet and allows nodes to construct new data spaces and mint data space access tokens. |  
| 4. | Substrate module: Iris-Assets | Modify the iris-assets pallet to allow nodes to specify data spaces with which to associate their data. Additionally, we implement the logic required to verify that the data owner holds tokens to access the space. |
| 5. | Contracts | Create the trait definition for a Composable Access Rule and develop the following composable access rules: limited use token: allow a token associated with an asset id to be used only 'n' times perishable token: allow a token associated with an asset id to be used only before some specific date |  
| 6. | Substrate Module: Iris-Assets | Execute composable access rules associated with an asset id when requesting data from Iris. |
| 7. | User Interface | We update the iris-ui repository so as to keep calls to extrinsics in sync with changes to parameters. |

### Milestone 2 - Proxy/Gateway Nodes

- **Estimated Duration:** 1 month
- **FTE:**  2.5
- **Costs:** 18,000 USD

This milestone delivers the infrastructure that provides data owners and data consumers the freedom to run light clients while still benefiting fromt he ability to ingest and eject data to/from the network. In particular, it implements proxy nodes and the accompanying offchain client.

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 1. | Substrate Module: Iris-Proxy: Proxy Node creation | Implement mechanism to allow nodes to act as a proxy, including verification of network connection speed. |
| 2. | Substrate Pallet: Iris-Proxy | Implement a layer to assign incoming commands in the DataQueue to be processed by specific proxy nodes. This will function similarly to how validators are selected in a Proof of Stake system. |
| 3. | Offchain Module: Data Ingestion + Reception Server | Build an offchain client using Go that allows data owners to make data available to proxy nodes and data consumers to receive data streams from proxy nodes |
| 4. | Substrate Module: Iris-Proxy | Implement offchain service to fetch data from a data-owners offchain server and stream bytes to data-consumers |
| 5. | User Interface | We modify the user interface to use the offchain client for data ingestion as well as to view data that has been made available to the offchain client. |

### Milestone 3 - Storage System

- **Estimated Duration:** 1 months
- **FTE:**  2.5
- **Costs:** 18,000 USD

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 1. | Runtime: IPFS integration | We make the initialization of the embedded rust-IPFS node optional, required only by proxy nodes. |
| 2. | Substrate Module: Generic Storage Service pallet | We build a generic pallet with read and write capabilities which can be modified to support multiple storage systems |
| 3. | Substrate Module: Centralized Storage System | We build a storage system connector based on (2) which can read and write data to a centralized storage system (i.e. an AWS S3 or equivalent local file server). |
| 4. | Substrate Module: Integration with Crust | We use the pallet developed during part 2 to use XCM to store and retrieve data in the Crust network. |
| 5. | Test Environment Setup | We deploy a relay chain with Iris and Crust as parachains. |

## Milestone 4 - Threshold Encryption

- **Estimated Duration:** 1 months
- **FTE:**  2.5
- **Costs:** 18,000 USD

TODO

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 1. | Threshold Encryption Module | | 

## Milestone 5 - Javascript SDK

- **Estimated Duration:** 1 month
- **FTE:**  2
- **Costs:** 12,000 USD

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| 0a. | License | Apache 2.0 |
| 0b. | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| 0c. | Testing Guide | Core functions will be fully covered by unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. We will provide a demo video and a manual testing guide, including environment setup instructions. |
| 0d. | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 1. | Javascript SDK | We will develop the javascript SDK to provide easy accessbility for front end developers to hook into dapps built on top of Iris as well as easily accomplish data ingestion, ejection, allocation/inclusion to data spaces, and more. We will document the full functionalities and specification of the SDK as part of this milestone. |
| 2. | Demonstration of the SDK | We will develop an NFT-marketplace type application on top of Iris to demonstrate the usage of the SDK. This will be adapted from the Iris Asset Exchange from the previous grant proposal for Iris. Further, we will demonstrate how this can be adapted by app developers to categorize data into their own data spaces. |

## Future Plans

- We intend to continue development after this proposal is completed. Specifically, we plan to consult with cryptography experts to determine the best approach for implementing a robust, performant, and secure **threshold ecnryption** scheme. Additionally, we intend to implement an **anonymous reputation system** for submitting feedback for data, data owners, and data spaces. We also plan to implement a robust **moderation and governance** protocol, which includes the execution of machine learning models and other security/verification checks (such as bayseian filters) within a **trust execution environment**.
- We will work with dapp developers to assist in develop of applications on Iris. We are already in communication with several projects that are interested in leveraging Iris as their storage system.
- We will strive to become a parachain on Kusama and Polkadot
- We will continue to expand the features available in Iris, such as partial ownership of assets, selling ownership of asset classes, and more

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?**
We first heard about the grants program from the Web3 Foundation Website.

Previous work on Iris is known the W3F grants team.
