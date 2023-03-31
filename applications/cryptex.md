# Cryptex

- **Team Name:** Ideal Labs
- **Payment Address:** 0x0TODO
- **[Level](https://github.com/w3f/Grants-Program/tree/master#level_slider-levels):** 2

## Project Overview :page_facing_up:

### Overview

**Cryptex** is an unstoppable, scalable, and decentralized ‘secret key custodian’ built using *blind DKG*. This proposal introduces the "blind DKG" protocol and elaborates on how it can be used to secure secret key shares using a proof of stake consensus system. The scope of this proposal includes the implementation of blind DKG in a standalone library, new pallets, including one to enable a verifiable secret sharing scheme within the blockchain, and an SDK to interact with Cryptex and handle encryption/decryption along with calls to fetch data from external storage.

Now, suppose Alice has a document that she wants to make available to anybody who can meet some set of rules, but Alice will not be available in the future to hand over the document. Since Alice will not be available, she splits her secret data into pieces, or shares, and gives a single piece to anybody who she thinks will be available later and tells each person some condition that someone should prove to get a share. Then, if Bob appears later and can prove that they have met Alice's rules and enough of the people who have shares believe Bob, then they give a copy to Bob and Bob is able to reassamble Alice's document and read her secret.

This type of sharing is made possible thanks to threshold secret sharing, first introduced by [Shamir](https://web.mit.edu/6.857/OldStuff/Fall03/ref/Shamir-HowToShareASecret.pdf). However, TSS by itself has several issues, namely that it requires a trusted setup, or key generation, phase. So, verifiable secret sharing schemes have been developed to allow for the verification of shares (such as [Feldman's scheme](https://www.cs.umd.edu/~gasarch/TOPICS/secretsharing/feldmanVSS.pdf), which ours closely mirrors). The setup, however, still depends on a trusted party to generate the shares. Thus, a distributed key generation protocol is a protocol to generate shares in a trustless way, with no third party needed. 

In this proposal, we introduce our protocol, blind DKG, where validators are incentivized to participate in a distributed key generation protocol. Our protocol allows for a verifiable secret sharing scheme with an on-chain proof system to gate access to data. Unlike a traditional VSS scheme however, it eliminates the need for a trusted setup by using a distributed key generator, as well as introduces a mechanism for validators to communicate 'blindly'. It allows for a trustless key generation and reencryption (i.e. secret sharing) process. When implemented as part of a decentralized network (such as a blockchain), it enables a system where you can create a distributed secret and derive numerous provably owned (by onchain identity) public keys that can then be used to encrypt data, while the network becomes responsible for reencrypting it. Additionally, we extend the protocol to enable a 'cryptographic gate' to data access, wherein an owner of some data can define rules to delegate decryption rights. The mechanism is both non-interactive, in that Alice does not need to interact with Bob in order to delegate decryption rights, and also has no trusted setup. Alice does not need to trust the set of participants holding the pieces of her secret (nor even know their identitites).

### Project Details

To start, we provide a brief overview of secret sharing and distributed key generation. Then, we will use this to explain the idea for our protocol. For additional motivation and details, we direct the reader to a [substack](https://ideallabs.substack.com/p/blind-dkg-part-1) we wrote that elaborates on the reason that VSS by itself isn't enough for an unstoppable application.

#### Background: Secret Sharing and Distributed Key Generation

##### Secret Sharing

Threshold secret sharing is a cryptographic protocol to encrypt a secret so that it can only be recovered if a threshold of participants participate. To be explicit, it can be thought of as the secret key being split into 'shares' and distributed to a set of shareholders. When a minimum threshold of shareholders reveal their shares, the secret is revealed. The basis of this scheme is thanks to interpolation of Lagrange polynomials. A LaGrange polynomial is any n-degree polynomial of the form $f(x) = \sum_{i=0}^n a_ix^i$ over some field. In a secret sharing scheme, we assume that this polynomial is over a finite field of prime order $q$, with coefficients $a_i \in \mathbb{Z}_q$.

The basis of a TSS protocol is thanks to Lagrange interpolation, which basically says that if you have $n$ points $(x_1, y_1), ..., (x_n, y_n)$, then there is a **unique** polynomial $f(x)$ whose degree is less than $n$ which passes through them. The degree of the polynomial $f$ is what we will mean by the 'threshold'. In a secret sharing scheme, a polynomial of degree $t < n$ is used, where $t$ can be configured as an input parameter.

To share a secret, a polynomial $f$ of degree $t$ can be created, with coefficients randomly sampled over the integers modulo a large prime, and where the first coefficient is the secret to be shared. The polynomial can then be evaluated at $n$ points and each value, or 'share' can be transmitted to a shareholder. Later on, the secret can be recovered when a threshold of the shares that were distributed are collected by reconstructing the polynomial $f$ through interpolation, and evaluating $f(0)$, which is our secret. With this, Alice can distribute the shares among $n$ participants, and later on Bob only needs to collect $t$ shares from them to recover the secret. 

This basic secret sharing scheme can be extended in order to make the distributed shares 'verifiable', in the sense that the recipient of a share can verify if what it received from another participant was truly a share from some polynomial or if it is some other value. This is done by publishing a commitment to the polynomial along with the shares. This type of scheme is called a 'verifiable secret sharing' scheme, and is what we will use in our construction.

##### Distributed Key Generation

By itself, VSS still requires a 'dealer' to generate the secret polynomial $f$ and distribute shares. This places a large degree of trust on the dealer. Also, there is not a clear way to make the sharing of a secret both trustless and non-interactive. To make it non-interactive, a semi-trusted set of  proxies (shareholders) are needed to reencrypt the data. To make it trustless would require that the owner of some data 'sign off' in order to make a secret available to another participant, which provides no clear way to scale. 

Some solutions, such as the Lit protocol, use a similar mechanism, using a DKG to start a TSS process. Additionally, they perform the computations using keys within a TEE. However, in their protocol, each member of the validator set is a shareholder, and so generating new keys is very expensive. I doubt that the network could scale to handle reencryption hundreds of keys within a small timeframe, whereas our solution is highly scalable.

Distributed key generation, or DKG, allows for shares (which can be used in a threshold encryption scheme), to be generated in a trustless way. Our scheme, inspired by the [ethDKG scheme](https://eprint.iacr.org/2019/985), is composed of three major phases: the sharing phase, the disputes phase, and the key derivation phase. During the sharing phase, a subset of the participants, called the dealer set, randomly selects a polynomial and calculates secret key shares for each member of the dealer set (other than itself). Then it encrypts each share for each member of the dealer set and publishes a commitment to the polynomial and the encrypted share onchain. After a dealer shares its generated shares with other dealers, the disputes phase begins, in which dealers prepare zero-knowledge proofs of the invalidity of any shares they are claiming as invalid. If a threshold of dealers believe the proof, then the share is marked is disqualified from the rest of the protocol. Finally, in the key derivation phase, a qualified subset of the initial dealer set is determined and a master public key and secret is able to be determined. 

Once a master public key is known, it can be used to encrypt some secret. Later, through the VSS scheme, the secret can be made available to another participant by 'reencrypting' the shares used for the master public key used and sharing with the other participant. In our network, we will assume that each address has a public/secret keypair associated with it. Thus, by reencryption, we mean that each shareholder decrypts their share, then encrypts it again using some public key (e.g. the pubkey of the address who should receive shares) and publishes it.

#### Our Proposal

Though a DKG scheme might solve the problem of securely generating the keys, it still does not allow for a 'fully decentralizable' protocol. First is the issue of choosing which participants in the network are selected to act as dealers. If this set is centralized or static, then the network is more centralized, and potentially less secure. Secondly, assuming that through some mechanism that the set of participants in the DKG are randomly distributed among the available candidates, how can we be sure that the selected participants will be willing to participate in the DKG to begin with? Or if they do, how can we be sure that our secret can be reencrypted later? EthDKG solved this with smart contracts, but contracts provide limited capabilities.

We propose consensus-backed blind DKG protocol, where it is blind in the sense that participants identities are unknown prior to participation. It enables a DKG process where participation is incentivized by rewards provided by the network, and shares are secured by the stake of each participant in the DKG protocol. The mechanisms allows for the formation of 'societies' whose members, each a validator, participated in the DKG process together. 

[ok redo this whole idea.. it really sounds half baked imo] Subsequently, a 'society' can be represented by an onchain asset class, owned by some participant of the network who requested that the secret key be created. By itself, the asset class simply points to ownership/leadership of the society. However, by minting a token, we enable a process by which a new distributed public key can be generated by the society and encoded in an onchain asset class. These public keys can then be used to encrypt data. The encrypted data can then be uploaded to some external storage (e.g. IPFS), and it's location can be encoded onchain. 

Finally, the last leg of our proposal concerns data accessibility. That is, we propose a mechanism that allows the owner a public key to control how data encrypted with that key can be created. We have several potential options in mind that we would like to explore further, and as yet we do not feel comfortable commiting to one or the other. So, we will present the options that we are considering below, and we will ammend this proposal once the decision is made.

##### What this is not
- This protocol does not encrypt or decrypt data for you, only keys. Data encryption and decryption must still be handled outside of the main protocol, though we scope out a user interface that demonstrates how handle that in the browser.
- This protocol is not a decentralized storage solution. The management and storage of data that is encrypted with keys generated this way must still be handled offchain. This protocol will attempt to be storage agnostic.

### Blind DKG Protocol

To begin, our protocol has two main DKG algorithms. The first is among a publicly known validator set, and the second is among a 'blind' validator set.

#### Session Validators and Session Public Key Derivation

In the first phase of the protocol, a fixed amount of *session validators* are selected. In fact, we will be contstructing a (t, n)-VSS scheme among the session validator set. This idea has been used in [HoneyBadgerBFT](https://eprint.iacr.org/2016/199.pdf) previously, so we may want to explore atomic broadcast in the future. The idea is that this set of validators can reencrypt data for other validators later on without the validators knowing each others identities. This forms the basis for the 'blindness' of our ultimate keygen construction, and also ensure that we are given a unique, randomly sampled session public key for each session. At the end of each session, rewards will be calcualted and distributed based on session validators' stake and performance during the session. The values for $\tau$ and $\epsilon$ are TBD. but will be well known among all validators.

1. A session's planning phase begins.
2. Using a commonly known number $\tau$ and $\epsilon$, each validator calculates a random number using a VRF and commits to it. For the calculated value, $d$, 
3. If $|\tau - d| \lt \epsilon$, then the validator can consider itself 'chosen' and begins to participate in the DKG process by generating a secret share, as well as a secret polynomial, commitment to that poly, and secret shares. 
4. When ready, participants publish their proof that their VRF output makes them a session validator and commitments to the polynomial. 
5. When another node *publicly* announces its ability to participate and commits to its poly (i.e. step (3)), then encrypt a secret key share for that identity and publish it onchain along with its proper commitment.
6. The validators dispute invalid shares. This is elaborated on [here](#disputes-phase).
7. If a threshold of session validators participate honestly, then a session public key can be calculated.
 

<p align="center">
 <img src="../static/img/spk.png" alt="Secret Society high-level overview"/>
</p>

The values for the size of the session validator set, $n$ and the thresold $t$ are a function of the total size of the network. That is, if there are more keys that need to be held and reencrypted, then the number of session validators should be increased in order to process each. However, this poses an issue in regards to scalability. It implies that the validator set will need to continuously grow in order to facilitate more 'reencryption bandwidth', unless we limit the bandwidth to some certain number of keys generated or reencrypted per block. We do not yet have a solution for this, but will actively research potential approaches.

#### Ad-Hoc DKG and Secret Societies

Now that the session validators have provided a session public key, we can instantiate ad-hoc *blindly* dkg using other members of the validator set. During each session, a participant can issue a 'request' to start the DKG by proving a number of shares and a threshold, $(t, n)$. Then, 'secret societies' can be formed, wherein each member has a secret key share. The protocol (for a single request) is as follow:

1. A participant issues a request to start the DKG, $(t, n)$
2. Validators can choose to 'bid' on being in the group by generating a VRF output and publishing a commitment to it onchain.
3. If $|\tau - d| \lt \epsilon$, then the validator can consider itself 'chosen' and begins to participate in the DKG process by generating a secret share, as well as a secret polynomial, commitment to that poly, secret key shares. That is, they calculate $\{(F(j), f(j))\}_{j=1}^n$. 

Up until now, this has pretty much been the same as the previous section. But now, we make a change. In order to share the secret key shares, they need to be encrypted. However, since we want to continue to maintain the anonymity of the validators who think they've won, we can't possibly encrypt the shares because we don't have any public keys. This is where the session public key comes in. Instead of encrypting for a specific validator, the keys are encrypted using the session public key.

4. Encrypt each key share using the session public key, generating secret shares $\overline{s_j} := E_{spk}(f(j))$ for $j = 1, ..., n$.
5. Generate your slot number, $sl$. Slot numbers will be important later on when the session validators are reencryption secret shares. Since we are doing $(t,n)-VSS$, there are a total of $n$ slots that must be assigned. Since we are only choosing validators in the range $|\tau - d| \lt \epsilon$, then we need to ensure that there are $n$ discrete slots in that range as well.  I need to put a little more thought into this, but for now just assume they are each able to randomly assign themselves a slot, and that this is done in a way so the slots tend to be non-overlapping. This probably implies we will need to have more than $n$ participants in each DKG to begin.
6. Publicly announce the encrypted shares, commitments to the polynomial, and proof that you could participate, however, this proof should only be verifiable by a member of the session validator set -> I'm unsure if this means we need to make changes to the VRF itself? Technically, I think this is the weakest step. Here, you could probably just look at who sent commitments and figure out the society's members. Need to come back to this.
7. When other participants announce the information above, the session validator set becomes responsible for acting as a 'secret key transport layer' in a way. When a validator submits a valid proof of ability to participate, along with its selected slot number, each SV (session validator) validates the proof and reencrypts shares for the validator. That is, they reencrypt up to $n$ shares that were published by the other members of the secret society. We will assume that each validator reserved $f(sl)$ as its own share, and so each session validator participates in the reencryption process, reencrypting its individual share and publishing it onchain, along with the identity of the recipient.
8. Finally, each participant receives a set of encrypted key shares that they can then decrypt, derive public keys from, and reencrypt.
9. For a subset of the society that passed the disputes phase, a default public key can be derived. We elaborate on that [here](#key-derivation).

Now, we need to explore how to look at this onchain. My inital thoughts is that the secret society is like an asset class, and public keys are like assets. This could actually work I think. But, the problem is how to keep the societies secret? If you're in a society, you need to have some secret knowledge. And you do, that's the secret key share.

<p align="center">
 <img src="../static/img/secret_society.png" alt="Secret Society high-level overview"/>
</p>

#### Disputes Phase

The disputes phase occurs as a subprocess of both of the above keygen algorithms. When a share is invalid, is must be ejected from the protocol. If an adversary issues an invalid share, then honest participants should issue a dispute against it. In order to ensure an adversary cannot falsely dispute the validity of a share, we place the burden of proving invalidity onto the party issuing a dispute. This is accomplished by using zk SNARKs. There are two flavors of this algorithm that we use. First, among the session validators, and secondly among a secret society.

##### Disputes Phase for Session Validators

The disputes phase for validators is very simple.

1. Since session validators all agree on the same 'group generator', when a session validator receives a share from another validator it can verify if it is a valid share. Each session validator verifies each share it recieves. If a share is valid, then there is no dispute.
2. If a share is invalid, the prepare a (zk) proof that the share is invalid and publish it.
3. If a threshold $t$ of other validators verify the proof, then the share is rejected from the protocol. Here, the identity of the validator who provided a bad share can be publicly announced as well.

At the end of this, if any validator submitted an invalid share we consider them an adversary, and their stake will be slashed.

##### Disputes Phase for Secret Societies

The disputes phase for secret societies is quite a bit different than for session validators. In effect, the society relies on the session validators to ensure the validity of shares that they receive. During the DKG, when a participant proves its valid VRF output to the session validator set, the session validator set will facilitate the disputes process by verifying each share when it is asked to reencrypt it. This functions identically to the disputes phase for session validators.

#### Encryption

Encryption is accomplished by using some type of public key encryption (over the same curve used by the blind dkg protocol) using the public key as derived above. Subsequently, the resulting ciphertext is added to some storage system and a multiaddress is published onchain.


#### Session Changes/New Sessions

TODO

For the time being, we will assume that the input and output of the blind dkg must happen within a single session. In the future, we will resolve this issue. 

#### Secret Sharing

After encrypting some data using the public key that you own, and which is emergent from some secret society, you may want to share that information with someone else, identified as another participant in the network. To do so, we need to the secret society to reencrypt the shares for the new participant. In one sense, a secret society is really just any set of nodes who can prove all of their shares came from the same polynomial. So when we need to reencrypt data using this secret society, we really want to issue a 'request' that asks "If you can prove your share came from the polynomial whose commitment is given by $F(x)$, then please reencrypt a key".

So, to enable secret sharing with a secret society, we create a struct, similar conceptually to an asset class, with some associated commitment to a polynomial, $F(x)$, and an owner, the participant who initiated the DKG. By attesting its ownership of the commitment, the owner can issue commands to the network (i.e. via an extrinsic) to both `derive` new public keys and to `reencrypt` secret keys. This is done by issuing a transaction that encodes the commitment, $F(x)$ and rewards any participants who provide a valid share. How can this be done in a way that preserves the secrecy of the society? As long as the formation is secret, keeping it forever secret isn't that big a deal.

Finally, the recipient of the shares can verify each share, rejecting invalid ones, and only rewarding participants who provided a valid share. At this point, the identities of the society are revealed. We will explore 'member rotation' in the future, however, for now we will just assume this is acceptable.

#### Data Access/Decryption Rights Delegation

In this section, we explore several potential ideas that we can implement in order to let the owner of a public key define how access to that public key is defined in terms of the blockchain's state. 

##### Option 1
Now, we propose an extension to the DKG/VSS mechanism above. In our new construction, we allow the owner of a society to prepare a ZK SNARK to encode a statement in the blockchain's state, for example, using R1CS to encode a requirement for ownership of some specific NFT, or having a minimum balance, etc. This is a publicly verifiable SNARK. Along with the (multi)location if the ciphertext, the encryptor also shares the common reference string (CRS) and the relation used (i.e. the condition in the blockchain's state) to generate the CRS. This is then associated with the public key that encrypted it. For example, a mapping like: $PK \to (R, \sigma, /ip4/.../QmX99dAd...)$. It should be publicly verifiable that an owner 'owns' the public key, but there is no direct mapping between the public key and the society. However, the members of the society are still able to determine if the public key belongs to them.

When a third party, say Bob, wants to get access to some ciphertext, he prepares a merkle proof of his state that he claims meets the conditions defined by Alice. Then, he prepares a zk proof claiming that his Merkle proof satisfies Alice's condition. If a threshold of shareholders can verify this proof, they each reencrypt a share for Bob. After a threhsold are reencrypted, Bob can recover the secret.

TODO.
Using arkworks r1cs, we can construct a zk SNARK based on a state in a Merkle tree. 

##### Option 2
Using Smart contracts, a similar way as was done in Iris. When you create a pubkey, you also provide an address. When a caller requests reencryption, you first check that the given address has authorized it, and then you can provide a share. This option is probably a lot easier than option 1. But, it's not privacy preserving at all.


#### Proposed Architecture

##### Libraries/Tech Stack

- Languages: rust, typescript/javascript
- Frameworks/Libs: substrate + tools (e.g. zombienet, telemetry), arkworks-rs,
- We will use [tarpaulin](https://github.com/xd009642/tarpaulin) for test coverage of our new pallets and node.

##### Blind DKG Library

This is a standalone library that contains the code for the main blind dkg protocol. We plan on building this using arkworks-rs unless we find a better alternative. This library will enable the Blind DKG protocol as detailed above. In essence, it will provide functionality to:

- generate secret shares, commitments
- verify key shares and prepare proofs of correctness of invalidity
- encrypt and decrypt key shares using the curve used by the dkg 
- encrypt plaintext using a public key on the curve used by the dkg
- decrypt ciphertext using a secret key on the curve

##### Pallets

At a *really* high level, our runtime will use the following pallets at a bare minimum (we will use other 'standard' pallets too, e.g. timestamp, tx fees, etc.)

![high-level-pallets](../static/img/high_level_pallets.png)

###### DKG  Pallet

The DKG pallet will:
- request dkg, manage request queue
- submit encrypted shares, commitments to polynomials (i.e. interact with blind dkg library)
- manage zk verification
- track participation/rewards

###### 

###### Other Pallets

We need to explore the best way to represent the 'secret society' onchain in a way that doesn't expose their identities. 
I think an interesting idea to explore is to use a DID, decentralized identifier, where the 'identity' that's owned is a public key derived from some society. We will most likely not use curve25519 in our VSS scheme, since it isn't a field of prime order (so we don't have the same security guarantees). We will probably use BLS12-381. Now, let's say we consider that as a DID that can be owned by a substrate address, say whichever node paid to have the blind dkg run. In a DID  scheme, the owner of the DID can use its ownership to perform certain tasks. For us, we want the DID to track parties that are authorized to get a reencryption key. So basically in a really simplistic way, we could do something like:

have a contract deployed, a rule executor done the same way as in Iris. Then, when deriving a public key you also give each shareholder the address of the rule executor. When some address calls the executor, it should call into the chain (via a chain extension) to temporarily give the caller the rights to get a decryption key. So basically, in the "DID" way, by owning the DID or by being a temorary 'delegate', it authorizes the society to give you a share. 

##### SDK
We will build a SDK with the following capabilities to allow developers and protocols to interact with cryptex:
  - **Encryption:** provides types and functions to encrypt and decrypt secrets. 
  - **VSS Client:** provides types and functions to interact with the protocol, some examples of such interactions are encryption key requests and secret sharing requests.
  - **ZK:** provides types and functions to define zk-SNARKS and Zero-knowledge proofs. They are required as part of the Blind DKG flow to define restrictions/rules that must be meet to get access to shared secrets, and to provide a proofs that requirements to access a shared secret are satisfied.
  - **Rules** **DSL:** a domain specific language to define/model access rules required to get access to  a shared secret. Once defined, rules are packaged/translated to zk-SNARKS. In future versions we will build a graphic editor to define these rules using our DSL as building block.
  - **Storage:** provides types and functions to save/read/update cyphered documents through different datasource options. The first version will provide IPFS as data storage option but we are going to expand this module in the future to include centralized options as well such S3, Google Drive, between others.
  - **Graphql API:** provides types and functions to fetch data saved on-chain related to Blind DKG in a developer friendly way.

### Ecosystem Fit

Help us locate your project in the Polkadot/Substrate/Kusama landscape and what problems it tries to solve by answering each of these questions:

- Where and how does your project fit into the ecosystem?
  - Our project is a substrate-based chain that introduces new capabilities to the ecosystem.
  - Our work will produce tools and knowledge that will be open source and openly available to the developer community.
  - We aim to become a parachain.
- Who is your target audience (parachain/dapp/wallet/UI developers, designers, your own user base, some dapp's userbase, yourself)?
  - **parachain**: ultimately, we strive to become a parachain. By taking advantage of XCM and continuing to evolve the cryptographic mechanisms surrounding delegation of decryption, we believe that very powerful and dynamic paradigms can be unleashed, such as gating access to data based on conditions of any state in any other parachain (e.g. data access via ownership of some specific asset on Statemint).
  - **users**: A long-term target audience will be our own user-base, as well as the user base of other parachains who need to share a secret. 
  - **Developers**: The tools we develop will be open source and available to other developers in the ecosystem. We intend to build a robust toolkit to enable devs to easily build on our new protocol/blockchain.
- What need(s) does your project meet?
  - Our project enables a decentralized an unstoppable secret key custodian. We see this as a foundational layer to building decentralized 'cryptographic data ownership' models, where by having a trustless key custodian, the network allows you to provably 'own' data, by proving that you own the keys to decrypt it.
  - On top of that, the project also allows for a certain degree of governance within the system, and we have plans to add some type of 'decentralized moderation' tools in the future. 
- Are there any other projects similar to yours in the Substrate / Polkadot / Kusama ecosystem?
  - There are some projects that use threshold encryption:
    - [CESS](https://cess.cloud/)
    - [SkyeKiwi](https://github.com/skyekiwi/skyekiwi-protocol)
    - However, neither project uses a dkg for trustless keygen as we do. Further, SkyeKiwi is dependent on IPFS, which our protocol is not. We see our protocol as being more highyl scalable and secure. 
  - If so, how is your project different?
  - If not, are there similar projects in related ecosystems?
- We have found projects that are similar in other ecosystems as well:
  - For general "password manager/storage" type applications, we have:
    - [b.lock](https://github.com/BlockProject/b-lock)
    - [DaPassword](https://cardano.ideascale.com/c/idea/332494)
    - [Blockchain password](https://margatroid.github.io/blockchain-password/#/)
    - [You](https://medium.com/airgap-it/you-the-decentralized-password-manager-2f521cced7be)
  - [EthDKG](https://github.com/PhilippSchindler/EthDKG): Our protocol uses the same DKG scheme as ethDKG, though we use a different SNARK system and have several other distinctions (e.g. consensus-backed security for secret shares).
  - [Share](https://share.formless.xyz/): which appears to use some type of threshold encryption but does not go into major detail (and which has dubious scalability)
  - [Lit Protocol](https://litprotocol.com/): We share many similarities with this protocol as it is built on the same underlying technology, using DKG and TSS. However, lit only enables on layer of TSS, similar to our 'session public key'. Additionally, it does not use zk SNARKs or other privacy preserving tools. I believe that Lit would not scale well, whereas this protocol does. 

## Team :busts_in_silhouette:

### Team members

- Tony Riemer
- Carlos Montoya
- Juan Girini

### Contact

- **Contact Name:** Full name of the contact person in your team
- **Contact Email:** Contact email (e.g. john@duo.com)
- **Website:** https://www.idealabs.network/

### Legal Structure

- **Registered Address:** 2451 Crystal Drive, 6th floor, Arlington, VA 22202, USA
- **Registered Legal Entity:** Ideal Labs, LLC

### Team's experience

Please describe the team's relevant experience. If your project involves development work, we would appreciate it if you singled out a few interesting projects or contributions made by team members in the past. 

If anyone on your team has applied for a grant at the Web3 Foundation previously, please list the name of the project and legal entity here.

### Tony Riemer

I am an engineer and (aspiring) mathematician with a passion for exploring new ideas. I studied mathematics at the University of Wisconsin and subsequently went to work at Fannie Mae (Federal National Mortgage Association) and Capital One as a full stack engineer where I mainly worked on fintech products (e.g. systems for loan servicing and pricing algorithms). For the previous year and a half, I've been working exclusively in the web3 space, including having worked on two web3 foundation grants [here]() and [here](). Beyond that, I have dabbled in many open source projects as well as have built my own, ranging from computer vision, machine learning, to blockchains and IoT communications.  Most recently, I attended the Polkadot Blockchain Academy in Buenos Aires, and this new proposal is an application of ideas I learned there applied to my previous grant. 

### Carlos Montoya
Carlos has been doing software for more than 20 years now, most recently in the startup world. 
- **Education**
  - Carnegie Mellon University
    Master of Science Information Technology, 2011 - 2013
  - Tecnológico de Monterrey
    Master in Information Technology Management, 2011 - 2013
  - Universidad Pontificia Bolivariana
    Innovation and Technoogy Management, 2009 - 2010
  - Universidad Autónoma de Manizales
    Systems Engineer, 1997 - 2002
- **Blockchain Experience**
Through 2022 Carlos had the chance to learn a lot about building smart contracts with solidity, and took part of some encode-club bootcamps and ETH Global hackathons. During this period of time he built several apps, one of them a decentralized job-board app and protocol called [web3Jobs](https://ethglobal.com/showcase/web3jobsfevm-inz64) ([Repo](https://github.com/encode-g2-project)). At the same time he started to learn rust and recently took part of the polkadot academy in Buenos Aires with very good performance. During the polkadot academy he was able to build a uniswap V2 Dex pallet, and XCM communication scenario, and some other cool use cases.
- **Software Engineering Experience**
  - Between 2004 and 2015, at [MVM Software Engineering](https://www.mvm.com.co/?lang=en), a technology firm with a deep focus in the energy industry, he was in charge of defining the way of doing great software for the entire company, leading the most skilled people, building the most complex software products and managing hundreds of initiatives for helping the company to expand its operations in Colombia, Dominican Republic, and Mexico. 
  - Between 2016 and 2020 he was completely focused on building [StellarEmploy](https://www.stellaremploy.com) with his co-founders, where we had the opportunity to take part of NY ERA accelerator, and got institutional Money. StellarEmploy technology was recently acquired by Learning Collider.  
  - Since early 2021 Carlos has been focused mainly on [TeamClass](https://www.teamclass.com), a b2b marketplace for helping companies with their team-building initiatives through virtual events. We bootstrapped TeamClass ourselves and made sales by 3.8M in our first year.


### Juan Girini

### Team Code Repos

- https://github.com/ideal_lab5/cryptex
- https://github.com/ideal_lab5/blind-dkg

Please also provide the GitHub accounts of all team members. If they contain no activity, references to projects hosted elsewhere or live are also fine.

- https://github.com/driemworks
- https://github.com/carloskiron
- https://github.com/juangirini

### Team LinkedIn Profiles (if available)

- https://www.linkedin.com/in/tony-riemer/
- https://www.linkedin.com/in/cmonvel/
- https://www.linkedin.com/in/juan-girini/


## Development Status :open_book:

If you've already started implementing your project or it is part of a larger repository, please provide a link and a description of the code here. In any case, please provide some documentation on the research and other work you have conducted before applying. This could be:

- links to improvement proposals or [RFPs](https://github.com/w3f/Grants-Program/tree/master/docs/RFPs) (requests for proposal),
- academic publications relevant to the problem,
- links to your research diary, blog posts, articles, forum discussions or open GitHub issues,
  - share my white paper here
  - share previous grants
- references to conversations you might have had related to this project with anyone from the Web3 Foundation,
- previous interface iterations, such as mock-ups and wireframes.

- We have started an article series to try to elaborate on these topics more in depth: https://ideallabs.substack.com/p/blind-dkg-part-1
- we have started a whitepaper to formally define and analyze these ideas. We intend to complete this paper as part of this grant as well.


## Development Roadmap :nut_and_bolt:

### Overview

- **Total Estimated Duration:** 16 weeks
- **Full-Time Equivalent (FTE):**  2.5 FTE
- **Total Costs:** 49,999 USD

The outcome of our milestones is threefold:
1. Cryptext: A blockchain that uses the protocol above within
2. A DKG/VSS library
3. An SDK to build user interfaces and client side logic, to define secret sharing access rules, and to perform offchain encryption and decryption.

### Milestone 1 — Session Validator DKG

- **Estimated duration:** 5 weeks
- **FTE:**  2.5
- **Costs:** 10,000 USD

Goals:
- TSS Library development and testing
- Basic session validator set creation and session public key derivation
- Beginnings of SDK: encryption, decryption, IPFS interactions
-- 
- we omit the disputes phase until milestone 3

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | Apache 2.0 / GPLv3 / MIT / Unlicense |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |
| 1. | Library: DKG | We implement and test a DKG protocol implemented with arkworks (unless we find a better solution). The first version of the protocol will have abilities to generate a key/poly, commit to it, to encrypt secret shares (for transmission) and to decrypt them (the combination of these two enables reencryption). This will be the main library we integrate into our runtime to enable the dkg. We will experiment with a few potential curves, specifically BLS12-381 and BLS12-377 (among others). There is some learning that will be needed as part of this. |
| 2. | Substrate module: DKG Pallet | We build a pallet to enable the system mentioned [here](#session-validators-and-session-public-key-derivation) minus the disputes phase. Under the hood, we will use Babe and Grandpa for authorship and finalization, and this pallet will be based on the [staking pallet](https://github.com/paritytech/substrate/blob/master/frame/staking/README.md). We will use the same VRF that is used by BABE. |
| 3. | SDK | Setup encryption and decryption capabilities, setup capabilities to interact with IPFS to add/read data. |

### Milestone 2 — Ad-Hoc DKG/Secret Societies

- **Estimated Duration:** 5 weeks
- **FTE:**  2.5
- **Costs:** 8,000 USD

Overview: At the end of this milestone, we will have a functional decentralized 'secret recovery' network, wherein an address can initiate the dkg, own a derived public key, encrypt data with it, request a reencryption key, and use that to decrypt the data.

Goals:
- Ability to perform ad-hoc DKG and derive a public key
  - Use session public key to securely transmit shares
- Ability to use public key to encrypt data
- Ability to decrypt data by getting a decryption key from the society
- At the end of this milestone, we should have basic (though not very secure) functionality to generate a new public key using the dkg, encrypt data with it, then get a reencryption key from the network and decrypt the data.
--
- no disputes phase (milestone 3)
- no secret sharing

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | Apache 2.0 / GPLv3 / MIT / Unlicense |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |
| 1. | Library: DKG | We make any changes as needed. There should be none, but you never know with these things. |
| 2. | Substrate module: DKG Pallet - ad-hoc dkg | We extend the functionality of the DKG pallet to submit requests to start the [ad-hoc dkg](#ad-hoc-dkg-and-secret-societies) process and to enable a 'bidding' phase where validators can calculate a VRF output and participate in the DKG as above. The derived public key will be encoded on chain. |
| 3. | Substrate module: DKG Pallet - share transmission | We implement an encryption mechanism so that validators can encrypt shares using the session public key. Further, we add functionality wherein validators are incentivized to reencrypt these shares correctly when asked (this enables the transmission of the share). Finally, we add functionality so that the receiving validator is able to decrypt the share. |
| 4. | SDK | We integrate with the blockchain and enable users to initiate the dkg process by submitting a threshold and shares value to the dkg pallet. Users should be able to view their public keys created via the dkg, as well as be able to encrypt data with those public keys and to request decryption keys from the network. |

### Milestone 3 - zkSNARKs and Disputes Phase
- **Estimated Duration:** 1 month
- **FTE:**  2.5
- **Costs:** 8,000 USD

Goals:
- implement the disputes phase for session validators and societies
- use zk SNARKs to encode the proof of correctness of disputed shares
- implement economic incentives to participate honestly in DKG (slashing + rewarding)

| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | Apache 2.0 / GPLv3 / MIT / Unlicense |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |
| 1. | Library: DKG | We implement functionality to verify/unverify secret key shares. Additionally, we will use [groth16](https://github.com/arkworks-rs/groth16) to prepare a zero knowlede proof when a share is calculated as invalid. |
| 2. | Substrate module: DKG Pallet: Session Validator Disputes Phase | We implement the disputes phase as detailed [here](#disputes-phase). We integrate the changes made as part of (1) in order to verify shares and construct proofs of their invalidity that can be shared with the network. We also implement the verification of these proofs. We do this using the arkworks [r1cs library](https://github.com/arkworks-rs/r1cs-std). |
| 3. | Substrate module: DKG Pallet: Secret Society Disputes Phase | We implement the disputes phase for societies as detailed [here](#disputes-phase-for-secret-societies). |
| 4. | SDK | General enhancements, will need to add more features, make it prettier, idk really.  |

### Milestone 4 - Decryption Delegation
- **Estimated Duration:** 1 month
- **FTE:**  2.5
- **Costs:** 8,000 USD

Goals:
- enable a rule based system to determine if an address can decrypt data


| Number | Deliverable | Specification |
| -----: | ----------- | ------------- |
| **0a.** | License | Apache 2.0 / GPLv3 / MIT / Unlicense |
| **0b.** | Documentation | We will provide both **inline documentation** of the code and a basic **tutorial** that explains how a user can (for example) spin up one of our Substrate nodes and send test transactions, which will show how the new functionality works. |
| **0c.** | Testing and Testing Guide | Core functions will be fully covered by comprehensive unit tests to ensure functionality and robustness. In the guide, we will describe how to run these tests. |
| **0d.** | Docker | We will provide a Dockerfile(s) that can be used to test all the functionality delivered with this milestone. |
| 0e. | Article | We will publish an **article**/workshop that explains [...] (what was done/achieved as part of the grant). (Content, language and medium should reflect your target audience described above.) |
| 1. | TODO | TODO |

...


## Future Plans

Please include here

- We would like to explore the usage of XCM in order to accomplish cross-chain 'data locks', wherein the proof of a condition on chain A (e.g. owning some specific asset on Ajuna) would equate to decyryption rights being granted in Cryptex.
- We intend to further enhance the protocol, to continue to research ways to make it more performant, more secure, and to make it more privacy preserving. In essence, we ultimate want to ensure that the 'society' that is formed to hold shares is a 'secret society', or rather a blind society.
- We intend to build tools on top of this functionality to enable developers to make use of the protocol.
- Short term intentions include further testing and enancements of the core protocol, building out simple tools and applications, such as a general secret sharing platform, and to work with engineers in web2 industries to prevent ways to decentralize their data architecture.

## Additional Information :heavy_plus_sign:

**How did you hear about the Grants Program?** Web3 Foundation Website / Medium / Twitter / Element / Announcement by another team / personal recommendation / etc.

Here you can also add any additional information that you think is relevant to this application but isn't part of it already, such as:

- Work you have already done.
- If there are any other teams who have already contributed (financially) to the project.
- Previous grants you may have applied for.
