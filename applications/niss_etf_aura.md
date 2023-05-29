# Cryptex: Non-Interactive Secret Sharing in an EtF Network

- **Team Name:** Ideal Labs
- **Payment Address:** TODO
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 3

## Project Overview :page_facing_up:

Cryptex is an EtF (encryption to the future) network using Aura. Our proposal builds the EtF network and a non-interactive secret sharing scheme on top of it.

### Overview

Cryptex is blockchain that uses a modified Aura which uses a treshold identity based encryption scheme to seal blocks. We then implement an encryption-to-the-future (EtF) scheme, where messages can be encrypted for arbitrary slots in the future. Our runtime will contain a pallet which will enable slot selection and verification (via authentication from past (AfP)). Finally, we propose a modification which enables a non-interactive cryptographically-verifiable-rule-based secret sharing mechanism on the EtF network.

- An indication of how your project relates to / integrates into Substrate / Polkadot / Kusama.

Cryptex introduces several new cryptographic primitives to the ecosystem which would be beneficial. By developing EtF capabilities, other networks can likewise benefit from EtF-based consensus. 

- An indication of why your team is interested in creating this project.

We want to build more extensive and secure decentralized data tools which allow general decentralized secret sharing.

### Project Details

There are four major components of the proposal. Firstly, modifying Aura to use TIBE block seals. The major piece will be the EtF implementation. 

#### IBE Aura

For simplicity, our network will use Aura for block authoring. In the future, we intend to migrate to Sassafrass. For now, we assume that there is a static set of validators defined on network genesis. In Aura consensus, each validator defined in the validator set authors a block in sequential (round robin) order. When a block is authored, the author signs the block, which is referred to as sealing the block. We implement this as a fork of Aura, wherein blocks are sealed using nugget BLS. In particular, we will add IBE capabilities to nugget BLS as per [this comment](https://github.com/w3f/Grants-Program/pull/1660#issuecomment-1563616111). Though we will not be taking advantage of many properties of nugget BLS that make is beneficial, our future plans include functionality which will, so we opt to just use it from the start. 

##### Overview

1. Genesis
  - Keygen: Each validator generates a private key and public key and the public key is encoded in the genesis block, with the private key stored by the validator (will need to highlight key management practices later on, should be strictly enforced for validators).
2. Block Sealing: 
3. 

- `initialize_authorities`


-> define how this will look on genesis
-> should this be rerun each new session?

#### Encryption-to-Future Slots

We propose a simple EtF scheme on top of Aura consensus with IBE block seals. Since slot owners in any given session are well known, we can devise a mechanism that allows for encryption to a 'time' in the future. The mechanism we propose is based on the well-known and static nature of the validator set for the time being.


#### Non-Interactive Rule-Based Secret Sharing

Once we have achieve EtF, we will implement a scheme on top of it in order to enable non-interactive rule-based secret sharing. The general idea is that some party $U$ has a secret $m$, to which they want to grant decryption rights if any given participant meets a condition $C$. In our case, this condition is some specific transaction in the blockchain, for example, a transaction that deposits one token to $U$'s wallet. In a substrate 

#### Light Client

We present a light client based on smoldot. The light client will connect to specific nodes, as we do not need a mempool. It should also ensure that users clocks are set correctly (i.e. block number), else give an error. We need to make sure requests are properly synchronized to ensure accurate and predicible time of decryption.

#### SDK



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
