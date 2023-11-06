# ETF Network

- **Team Name:** Ideal Labs
- **Payment Address:** TODO (DAI)
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 3

## Project Overview :page_facing_up:

This is a followup to [cryptex](https://github.com/w3f/Grants-Program/pull/1660).

### Overview

The ETF Network is a substrate-based blockchain which uses a `proof-of-extract` consensus, enabling decentralized timelock encryption and keygen capabilities.

The ETF Network uses identity-based encryption and DLEQ proofs to implement a proof-of-extract consensus mechanism, wherein validators leak `slot secrets` with each block produced. In a sense, this can be thought of as building an ever-growing table of publicly verifiable keypairs (with all public keys already known) backed by consensus. In our inital grant, we delivered a proof of authority version of our consensus mechanism, along with rust/typescript SDKs for enabling timelock encryption on top of the network. In this followup grant, we aim to ensure the security and scalability of the network by implementing a proof-of-stake version of the network. 

Beyond being a substrate-based chain, our project also makes heavy use of polkadotjs and smoldot. Additionally, the network is capable of providing benefits across the ecosystem, providing chain-agnostic timelock encryption capabilties.

We are interested in creating, or extending, this project as we see it as an important step in a more secure, decentralized online world which requires less trust between participants.


### Project Details

> We expect the teams to already have a solid idea about your project's expected final state. Therefore, we ask the teams to submit (where relevant):
> - Mockups/designs of any UI components
> - Data models / API specifications of the core functionality
> - An overview of the technology stack to be used
> - Documentation of core components, protocols, architecture, etc. to be deployed
> - PoC/MVP or other relevant prior work or research on the topic
> - What your project is _not_ or will _not_ provide or implement
>   - This is a place for you to manage expectations and to clarify any limitations that might not be obvious


This is a followup to our previous grant, in which we built the foundational layers of the ETF network. In this grant, we aim to make the network more decentralized and secure, and to approach production readiness for our system. To be more specific, the scope of this grant is:


1) to implement a proof of stake version of the proof-of-extract consensus mechanism
   1) ensures greater decentralization, security, and scalability
   2) uses verifiable proactive secret sharing (VPSS) + Babe + IBE 
2) to implement an oracle for reading etf network slots and slot secrets
   1) allows for secure reading of slots/secrets without syncing with the chain, opens up to easy non-web3-native integrations
   2) allows for use cases beyond the scope of etf network itself (i.e. can use in contracts not deployed to the ETF network chain)
   3) this is essentially a randomness beacon + allows for temporal decision making 
3) Atomic, (cross-chain) trustless interactions
   1) leverage timelock encryption via the etf network, along with the oracle and zk-rollups to enable a 'trustless private asset swap' mechanism
   2) revisit the sealed-bid auction, use new 'transactions to the future' style to bid on auctions, makes the reveal non-interactive AND trustless (currently, this phase requires some trust).
   3) This can be used for a wide array of use cases beyond auctions, secure MPC protocols, voting + governance, MEV/front-running protection, DeFi: DEX/NFT marketplaces

#### Stake-Backed Proof of Extract

In order to ensure the scalability and security of the network, we will implement a proof-of-stake version of our proof of extract mechanism, modeled after Babe. Babe functions in sequential epochs, where each epoch consists of a static number of sequantial slots, with epoch changes announced one epoch in advance (i.e. the new validator set). Potential block authors from the validator set use a VRF, using on-chain randomness, to generate a random number which determines if they can propose a block in any given slot. Along with this, validators will also calcualte the IBE.EXTRACT function and provide a DLEQ proof, which block importers must verify along with the VRF signature.

In the initial version of the network, all authorities are also IBE master key custodians, each having complete knowledge of the IBE master secret. Not only does this require trust that all authorities are honest, but also requires that each authority securely stores the secret. This adds a degree of centalization to the network which could make it easily compromisable. To counteract this issue, we propose using [verifiable proactive secret sharing]() in order to decentralize knowledge of the IBE master secret. A  verifiable proactive secret sharing scheme allows for a dynamic committee (i.e. validator set) that refreshes with each epoch change to receive new shares for the epoch, while the IBE master key remains unchanged and unrevealed. A threshold of authorities must collude in order to reveal the secret.

To be explicit, our VPSS scheme will rely on a semi-trusted dealer on genesis. The dealer will be responsible for defining the initial polynomial $f^0(x)$ that will be used to generate the initial shares and commitments (i.e. shamir). The VPSS scheme provides us with three PPT algorithms:

- Share
- Refresh
- Open


Should we use CHURP? https://eprint.iacr.org/2019/017.pdf


Our network will then reward honest participants with our inflationary native token, else slash their stake if they behave adversarially.

![](../static/img/babe_to_etf.drawio.png)

Using substrate's onchain randomness via [Babe](https://github.com/paritytech/polkadot-sdk/tree/master/substrate/client/consensus/babe), after ending an epoch N, when authorities and randomness for epoch N+2 are already announced, each announced authority can asynchronously participate in the proactive secret sharing scheme's **Refresh** algorithm, in which the new epoch key is created.

Additionally, we will add a secondary keystore, which stores these new secret shares. => we should have a formal specification for this keystore:
- store bls keys
- refresh each epoch + drop keys

https://medium.com/polkadot-network/polkadot-consensus-part-3-babe-dcc2e0dd8878

At the beginning of each epoch, each participant issues a commitment to their epoch secret. Each issued public commitment can then be used to derive the `epoch public commitment`, which we can then use to perform tlock encryption as usual.

#### Slot Secrets Oracle (Randomness Beacon)

A major shortcoming of the network is that dapps that may want to use timelock encryption are relegated to existing on the chain itself. Not only does this limit adoption, as there may be a barrier to usage, but it also adds unnecessary load on the ETF network itself. In order to counteract this, we propose to develop an oracle which can provide slots and slot secrets.

#### Trustless Atomic Asset Swaps

Benefits/Value statement: 

- Front Running resistance
- non-custodial control
- no counterparty risk
- privacy preserving

We define a trustless atomic swap to be a protocol between at least two participants, Alice and Bob. Here, the idea is that Alice and Bob both will construct unique transactions with agreed upon inputs (for example, Alice sends asset A to Bob, Bob sends asset B to Alice). 

We require that...
- no information about the transactions is publicly revealed until the swap is already complete
- either all txs involved in the scheme are valid, or else none are. That is, if any one transaction included in the swap is invalid, then no other transactions should be executed. 
- should be chain-agnostic  


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

- **Contact Name:** Tony Riemer
- **Contact Email:** driemworks@idealabs.network
- **Website:** https://idealabs.network

### Legal Structure

- **Registered Address:** TBD
- **Registered Legal Entity:** Ideal Labs

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

      1) zkiam => simplified version is a password-based secrets vault: based on the idea that there's some minimum N such that given a random selection of N slot secrets, by randomly selected M < K <= N of them, for some M > 0, by aggregating the secrets revealed at each of the K slots to get SK, and doing the same to get a pubkey PK, then this new keypair (SK, PK) is random + secure and can be used for IBE. I'll have to revise this a little to take into account the VPSS, but that's the basic idea. This could be extended with a fuzzy extractor to do a biometrics based secrets vault, and then with a ring vrf to do zk iam, but those last two would be out of scope of the grant I think. This could, however, be a passwordless secure IBE based communication tool quite easily without the biometrics and stuff.

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
