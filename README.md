# Governance Contract Comparison

**Purpose**: compare the different smart contract systems which FIAT DAO can use to manage its governance

**Author**: @charlesndalton

**Factors considered (weighted)**:
- Ability to use sub-committees / not engage entire community for all decisions **(5)**
- Ability to lock FDT **(5)**
- How the code is written and maintained **(4)**
- How many audits have been performed & the quality of the auditors **(3)**
- An "exit lever" i.e., the ability to switch to another governance contract **(3)**
- Gas efficiency **(2)**

### **Element Finance**

#### <ins>Overview</ins>

Element's governance was designed with the following problems in mind:
- Lack of governance participation
- Disproportionate power of top governance holders
- Opportunity costs of locking tokens
- Lack of flexibility in existing solutions 

Their design contains three key concepts: a governance steering council, voting vaults, and voting quorums.

The governance steering council (GSC) is essentially a board of directors. Individuals are elected by the token-holders, and the council is responsible for managing off-chain governance and the treasury.

Voting vaults are an interesting concept which allow for extreme flexibility & extensibility in determining who is allowed to vote in on-chain governance decisions. Each vault needs to implement the following function:
```
function queryVotePower(
        address user,
        uint256 blockNumber,
        bytes calldata extraData
    ) external returns (uint256);
```
You can imagine many different implementations here. For example, there is already a vault implemented that would give voting power to people who lock their governance tokens (similar to ve models like Curve). But you could also have a vault for governance tokens that are deposited to lending markets like Compound/Aave or yield aggregators like Yearn. This would allow users to earn yield on their governance tokens while keeping the ability to vote. Of course, each voting vault needs to approved by the main DAO.

Quorums are the last important concept introduced by Element. Essentially, this is the idea that different types of votes have different levels of importance & risk, and that a proposal to upgrade a smart contract should not have the same voting quorum as a proposal to change an non-security critical parameter. As such, in Element's system custom quorums can be used for different function calls. See it in the code [here](https://github.com/element-fi/council/blob/820d40f18149bbe21b75d70f28b30c8b049935bd/contracts/CoreVoting.sol#L152-L159).

#### <ins>Analysis & Scoring</ins>

##### Ability to use sub-committees / not engage entire community for all decisions

The GSC would be able to control the treasury, so would not have to go to the broader community about decisions like budget, hiring, token buy-backs, etc.

However, the Element model does not natively include delegation or sub-committees which would have control over specific functions.

Score: 2

##### Ability to lock FDT

Yes, and there is already an [implementation](https://github.com/element-fi/council/blob/main/contracts/vaults/LockingVault.sol) that FIAT could use.

Score: 5

##### How the code is written and maintained

Code is very readable, concise, and well-structured. For example, the voting vault mechanism was designed such that the GSC could be implemented as just another voting vault, so there is no reference of the GSC in the *CoreVoting* contract.

Score: 5


##### How many audits have been performed & the quality of the auditors

Two audits have been performed, which were performed on both the core voting contracts and the standard voting vault contracts ([source](https://github.com/element-fi/council/tree/main/audits)). The auditors were Runtime Verification (has audited Uniswap, MakerDAO, Ethereum 2.0 Beacon chain, Gnosis Safe, many others) and ChainSafe (has audited Connext, Ribbon Finance, and the Graph).

Still, the system has not been in production which means it can't be classified as a highly secure system.

Score: 3

##### An "exit lever" i.e., the ability to switch to another governance contract

It would not be easy to switch the *CoreVoting* contract itself,  but many changes in the governance model & mechanics could be effectuated through additional voting vaults.

Score: 4

##### Gas efficiency

TBD

### **Gnosis Zodiac**

#### <ins>Overview</ins>

Gnosis Safe is used mainly to manage smart contracts by small groups of semi-trusted parties. However, signers are not the only ones who can execute transactions in the safe: "every Gnosis Safe account is controlled by two means. By the account owners using their signer keys and by optional modules that have their own custom access logic." Once a safe has approved a module, this module can directly execute transactions without any additional signers.

Zodiac uses this feature to allow DAOs to be built on top of Gnosis Safe. Today, there are three modules: exit, bridge, and reality. 

The reality module allows snapshot proposals which pass to automatically be executed on-chain. The way this works is the following:

1. Someone would create a Snapshot proposal
2. If it passes, an oracle (probably reality.eth, although you could use any oracle which implements their *RealitioV3* interface) would store this on-chain.
3. Anyone would be able to call the following function:
```
function executeProposalWithIndex(
        string memory proposalId,
        bytes32[] memory txHashes,
        address to,
        uint256 value,
        bytes memory data,
        Enum.Operation operation,
        uint256 txIndex
    )
```
4. Internally, it would make the following check, where questionId is deterministically generated by proposalId and txHashes:
```
require(
    oracle.resultFor(questionId) == bytes32(uint256(1)),
    "Transaction was not approved"
);
```
5. Once other checks had passed, it would call the *execTransactionFromModule* function of the Gnosis Safe with the calldata that was voted on.

Another module allows decisions to be bridged. For example, you could have your voting done on an L2 and then have that decision bridged to your L1 governance contract.

It's evident that modules could be used for many other applications, and that many other governance features (e.g., Element's voting vaults) could be built as Zodiac modules.

There are also special modules, called modifiers, which sit in between other modules and the Gnosis Safe. These modifiers implement the Gnosis Safe interface, *IAvatar*, so that other modules can treat them as a safe. For example, there is a delay modifier which would enforce a delay for all transactions coming out of a module before they were executed in the Gnosis Safe.

Overall, Gnosis Zodiac seems to be very extensible (you could build pretty much any DAO tool you can think of on top of it), but also somewhat lacking an ecosystem. There are not many modules, the existing modules don't seem to be heavily used. It could be used if FIAT DAO wanted maximum flexibility and didn't mind needing to build out some of the DAO tooling (and get audits) itself.

#### <ins>Analysis & Scoring</ins>

##### Ability to use sub-committees / not engage entire community for all decisions

Gnosis Safe works out of the box as a transaction executor for DAO team members, so you would not need to engage the community for all decisions. 

If you wanted a sub-committee structure, with specific committees responsible for specific types of transactions, that's something that you would need to build out as a new module.

Score: 4

##### Ability to lock FDT

This doesn't work out of the box, but would be possible to build.

Score: 3

##### How the code is written and maintained

Overall, the core Zodiac code is clean, concise, and fairly intuitive. An engineer could understand the entire core codebase, soup to nuts, in 2 hours or less.

Maintenance, especially of the modifiers and modules, is a different story. Most of these haven't recieved any committs in at least 3 months.

Score: 3


##### How many audits have been performed & the quality of the auditors

One audit, performed by the G0 group (relatively small auditor). Both core and modules are covered.

Score: 2.5

##### An "exit lever" i.e., the ability to switch to another governance contract

It depends. If you wanted to switch to another Zodiac module, this could be extremely easy. If it was a migration to a non-Zodiac-module governance contract, it would depend on the governance module itself and how easily it allows migration.

Score: 3.5

##### Gas efficiency

Depends entirely on the modules you use.

Score: N/A

### **Colony**

#### <ins>Overview</ins>

Relative to the prior two, Colony's product is much more expansive. Instead of just trying to solve governance of smart contracts, "Colony is designed for day-to-day operation of an organization." 

Colony focuses on these parts of DAOs:


##### **Domains and permissions**:

 Colony has the concept of domains, which can be created for the colony, similar to the concept of a business unit. The primary purpose of domains is to manage individuals' authority. For example, in a Colony DAO, you would be able to create a 'Development' domain and a 'Marketing' domain, and have some people be highly-trusted in development but not marketing and vice versa. People in the colony are assigned domain-specific permissions. There are 6 different permissions, each of which allows you to take action in the colony. For example, the 'administration' permission would allow an Ethereum account to create expenditures in a Colony. It's possible that FIAT DAO could use these permissions in other smart contracts. These contracts would use the below modifier: 
```
function hasUserRole(address _user, uint256 _domainId, ColonyRole _role)
```
For example, FIAT could have a 2/9 multisig safe responsible for adding collateral vaults, and give the safe's account the 'administration' permission in the vault domain. Then, you could change L17 of VaultFactory.sol ([src](https://github.com/fiatdao/vaults/blob/97bd2acededb9fafc11f1e8fa9a1ac1154c2f86e/src/VaultFactory.sol#L17)) to look like 
```
function createVault(address impl, bytes calldata params) external hasUserRole(msg.sender, 0, ColonyRole.Administrator) returns (address) {
```
if 0 was the domain ID for vaults.

##### **Funding and expenditures**:

Each domain has a 'funding pot' which it can manage. From this funding pot, tokens can be moved out (e.g., to pay someone) via 'expenditures,' which can be approved by those that hold the funding permission.

##### **Reputation**:

People can earn reputation in different domains. Reputation is earned by others' endorsements – think of this like the endorsement of skills on LinkedIn. 

##### **Motions & Disputes**:

When a colony member wants to change something (e.g., move money from one funding pot to another), they may make a 'motion', and then stake some of their reputation on that motion. Others may also stake their reputation on that motion. After a certain time period has elapsed, it automatically passes. If someone wants to stop the motion, they may raise a 'dispute.' At this point, it would move to reputation-weighted voting.

#### <ins>Analysis & Scoring</ins>

##### Ability to use sub-committees / not engage entire community for all decisions

It would be possible to use Colony's permission and domain system to manage arbitrary execution of smart contract functions, but permissions and domains were not designed with this in mind. As such, there are only 6 permissions, and new ones cannot be added (without changing Colony's code). 

Score: 2

##### Ability to lock FDT

Colony doesn't provide native support for token locking.

Score: 1

##### How the code is written and maintained

I wasn't a huge fan of the quality of the code. For example, look [here](https://github.com/JoinColony/colonyNetwork/blob/develop/contracts/colony/ColonyRoles.sol#L33-L83) to see code copy pasted when it could have been consolidated into a single internal function. 

There are 30 contributors to the core GitHub repo, but most of the core contracts have only been modified by two.

Score: 2


##### How many audits have been performed & the quality of the auditors

"We also make no guarantees about Colony’s security or fitness for purpose; the Colony Network smart contracts are large, complex, and under active development, which makes audit impractical." ([source](https://blog.colony.io/colony-v2-launch/))

Score: 1

##### An "exit lever" i.e., the ability to switch to another governance contract

The Colony contracts themselves follow the EtherRouter upgradeable pattern ([src](https://github.com/JoinColony/colonyNetwork/blob/806e4d5750dc3a6b9fa80f6e007773b28327c90f/contracts/common/EtherRouter.sol)). However, I can imagine that it might be tricky to migrate the FIAT contracts away from using the Colony permission system.

Score: 3

##### Gas efficiency

TBD

Score: TBD

### **Compound Governance**

#### <ins>Overview</ins>

##### High-level

Compound's governance system was arguably the first serious implementation of decentralized DeFi governance. Here's how it works:
- COMP token holders have a pro-rata share of control over the Compound Protocol. They can either keep this voting control or delegate it to an account (or accounts) of their choice.
- Anyone who has the voting power of 25k COMP, either through their holdings or voting power they've been delegated, can propose a governance action, which is a function call or a series of calls that can be made by the governance contract.
- Once a proposal achieves a threshold of votes, it is queued in the timelock, and can be executed after x days (x = 2 in Compound's case).

##### The code 💻

Under the hood, the governance contracts follow an upgradeable delegator-delegate pattern, where the [delegator](https://github.com/compound-finance/compound-protocol/blob/master/contracts/Governance/GovernorBravoDelegator.sol) is a typical proxy contract where an admin has the ability to swap out the implementation by calling the *_setImplementation* function.

Then, someone wanting to introduce a governance proposal would call the following function:

```
function propose(address[] memory targets, uint[] memory values, string[] memory signatures, bytes[] memory calldatas, string memory description) public returns (uint)
```

After that, people can vote by calling one of three functions: castVote, castVoteWithReason, or castVoteBySig. Once it achieves sufficient votes, anyone can call *queue(uint proposalId)* to move it into the timelock. 

Then comes the last stage: execution. Someone calls *execute(uint proposalId)*, which in turn calls *timelock.executeTransaction.* It's the timelock, then, that executes all of the function calls [here](https://github.com/compound-finance/compound-protocol/blob/3affca87636eecd901eb43f81a4813186393905d/contracts/Timelock.sol#L99). *To the rest of the system, the admin isn't the core governance module, it's the timelock contract* (e.g., see [cBAT](https://etherscan.io/address/0x6c8c6b02e7b2be14d4fa6022dfd6d75921d90e4e#readContract)).

All in all, Compound is a simple system. That carries advantages in terms of security and developability, but also means that any new features (e.g., vote-locked FDT) would need to be home-grown.

#### <ins>Analysis & Scoring</ins>

##### Ability to use sub-committees / not engage entire community for all decisions

No. The core Compound governance system is built such that the only way that calls can be executed by the timelock is through the voting & waiting process.

Score: 2

##### Ability to lock FDT

Possible to build, because the governance contract is upgradeable, but not natively built in.

Score: 2

##### How the code is written and maintained

As mentioned, the code is simple and easy-to-follow, and would be relatively easy to modify.

The key problem would be extensibility. The code isn't very modular – it's fairly tied to the Compound governance flow. For an example, look at this [block](https://github.com/compound-finance/compound-protocol/blob/3affca87636eecd901eb43f81a4813186393905d/contracts/Governance/GovernorBravoDelegate.sol#L208-L224) which decodes the state of a proposal.

Score: 3

##### How many audits have been performed & the quality of the auditors

OpenZeppelin and Trail of Bits audits on V1. We're now in V2, but not much has changed in the core code.

Score: 4

##### An "exit lever" i.e., the ability to switch to another governance contract

As mentioned earlier, it uses an upgradeable pattern that would allow changing / extending functionality.

If FIAT wanted to start with the existing code, but then change something (e.g., remove timelock, add vote-locking, etc.) that would be possible – you would just need the change to pass through the existing governance process, since you would need governance permission to change the delegator's (proxy's) implementation.

Score: 4

##### Gas efficiency

TBD

Score: TBD


### **Frax Governance**

#### <ins>Overview</ins>

Fitting of a hybrid collateral/seignorage based stablecoin, Frax's governance system is a combination of the Compound and the Curve systems. Essentially, you can think of this as 'Compound governance but with veGovToken.' Because they are not a novel system, they are more of a case study of a model that FIAT might implement.

On the Compound side, it has forked the exact *GovernorBravoDelegate* and *Timelock* contracts mentioned before. These contracts have all the necessary permissions to make changes in the underlying contracts (e.g., you can see [FXS](https://etherscan.io/address/0x3432B6A60D23Ca0dFCa7761B7ab56459D9C964D0#readContract) pointing to the timelock). These contracts are not currently used for governance decisions, however; the team currently uses Snapshot voting & implements decisions through a protocol multisig. 

This is accomplished through a simple modifier:

```
modifier onlyByOwnGov() {
    require(msg.sender == owner || msg.sender == timelock_address, "You are not an owner or the governance timelock");
    _;
}
```

When they want to fully decentralize, they plan to set the 'owner' value to a burn address, so that the timelock is the only one with control.

On the Curve side, it uses slightly modified version of veCRV (diff [here](https://www.diffchecker.com/Sqm7PfNt)). This works exactly like Curve's system, where the amount of veFXS you receive is proportional to (a) the amount of FXS you stake and (b) the length of escrow (e.g., 100 FXS locked for 4 years yields 400 veFXS), and the amount of veFXS you hold decays linearly until veFXS = FXS at 0 time remaining.

Overall, I think the Frax model is a strong contender because it meets a lot of the requirements laid out (ability for team to make decisions, ability to lock FDT, and highly-secure code).

There are also some problems, however. The most important one is extensibility: although it would be possible to build on this system, it's not engineered to be modular (unlike Zodiac, for example), which makes the system hard to maintain as it grows. For example, the Frax team is currently trying to figure out how they can have both veFXS and FXS vote in GovernorBravo, since this kind of functionality wasn't intended by the Compound team. 

There is also the more minor point of Curve being in Vyper and Compound being in Solidity. FWIW, Yearn uses both Vyper and Solidity, and it surprisingly doesn't cause many developer headaches, at least in Brownie and Foundry. Still a context-switching cost though.

#### <ins>Analysis & Scoring</ins>

Since we have already covered Compound, I'm going to answer all of these in reference to the Curve governance contracts, most notably VotingEscrow.vy.

##### Ability to use sub-committees / not engage entire community for all decisions

Definitely. It would be possible to use the same modifier they have, and have a FIAT multisig as one of the approved parties. Still, you don't lose to option to have the community make those decisions as well.

Score: 5

##### Ability to lock FDT

Yep, handled by the [VotingEscrow.vy](https://github.com/curvefi/curve-dao-contracts/blob/master/contracts/VotingEscrow.vy) contract.

One way to integrate veFDT natively into the Governance module would be to add the following function in VotingEscrow.vy:

```
function getPriorVotes(address account, uint blockNumber) external view returns (uint96);
```

This is what Bravo uses as a 'voter-counter' for a specific user. An implementation would be pretty simple, and would just wrap [this](https://github.com/curvefi/curve-dao-contracts/blob/4e428823c8ae9c0f8a669d796006fade11edb141/contracts/VotingEscrow.vy#L546) function. 

Alternatively, if FIAT wanted to implement arbitrary voting logic, similar to Element, it could implement a separate contract with this function, which could add separate logic for vote-tallying (e.g., allowing regular FDT holders to also vote).

Score: 5

##### How the code is written and maintained

We've already talked about Compound, so referring to the Curve ve contract:

It's reasonably well-written and very well-documented ([src](https://curve.readthedocs.io/dao-vecrv.html)), although it probably could have been a little cleaner. For example, to increase the escrow time of someone's governance token, they internally trigger a 0 deposit of the user ([src](https://github.com/curvefi/curve-dao-contracts/blob/4e428823c8ae9c0f8a669d796006fade11edb141/contracts/VotingEscrow.vy#L464)) which feels a little hacky.

3 people contributed, and there haven't been any changes since August 2020. Also Vyper.

Score: 3


##### How many audits have been performed & the quality of the auditors

Although the fact that there haven't been any changes since August 2020 is a sign of lack of maintenance, it's positive from a security perspective. The fact that about 50% of all CRV, or $1.2B, is locked in this contract, and that no significant bugs have been found is a positive signal stronger than an audit report.

Still, Curve hasn't hired any audit firms to specifically audit the governance contracts.

Score: 3.5

##### An "exit lever" i.e., the ability to switch to another governance contract

The ve code is immutable. However, the Compound Governance module would be able to simutaneously allow multiple tokens at once, which would allow for a migration process.

Score: 2

##### Gas efficiency

TBD

Score: TBD

### **Protocol**

#### <ins>Overview</ins>

Text

#### <ins>Analysis & Scoring</ins>

##### Ability to use sub-committees / not engage entire community for all decisions

Text

Score: TBD

##### Ability to lock FDT

Text

Score: TBD

##### How the code is written and maintained

Text

Score: TBD


##### How many audits have been performed & the quality of the auditors

Text

Score: TBD

##### An "exit lever" i.e., the ability to switch to another governance contract

Text

Score: TBD

##### Gas efficiency

Text

Score: TBD

### **References**

##### Element Finance:
- [An Introduction to Element's Governance Model](https://medium.com/element-finance/an-introduction-to-elements-governance-model-efea13d1c7ee)
- [Governing Element Finance, A Security Roadmap Update](https://medium.com/element-finance/governing-element-finance-a-security-roadmap-update-e9cf6816c5d3)
- [Voting Vaults: A New DeFi and Governance Primitive](https://medium.com/element-finance/voting-vaults-a-new-defi-and-governance-primitive-b4b2f6289d48?source=collection_home---4------2-----------------------)
- [The Governance Steering Council](https://medium.com/element-finance/the-governance-steering-council-63aea7732262)
- [The Core Principles of Element Finance](https://medium.com/element-finance/the-core-principles-of-element-finance-fdbaf6aa5f1d)
- [Element Governance: A Technical Architecture Overview](https://medium.com/element-finance/element-governance-a-technical-architecture-overview-2d0f72bd278a)
- [element-fi/council](https://github.com/element-fi/council)
##### Gnosis Zodiac
- [Zodiac: The expansion pack for DAOs](https://gnosisguild.mirror.xyz/OuhG5s2X5uSVBx1EK4tKPhnUc91Wh9YM0fwSnC8UNcg)
- [Gnosis Guild: Zodiac: A design system for DAOs](https://www.youtube.com/watch?v=SY0kLefcU7k)
- [What is a module?](https://help.gnosis-safe.io/en/articles/4934378-what-is-a-module#:~:text=Gnosis%20Safe%20Modules%20enable%20additional,their%20own%20custom%20access%20logic.)
- [GitHub.io page](https://gnosis.github.io/zodiac/)
- [g0-group/audits](https://github.com/g0-group/Audits)
- [gnosis/zodiac](https://github.com/gnosis/zodiac)
- [gnosis/zodiac-module-reality](https://github.com/gnosis/zodiac-module-reality)
- [gnosis/zodiac-module-bridge](https://github.com/gnosis/zodiac-module-bridge)
- [gnosis/zodiac-modifier-delay](https://github.com/gnosis/zodiac-modifier-delay)
- [gnosis/zodiac-modifier-roles](https://github.com/gnosis/zodiac-modifier-roles)
##### Compound Governance
- [Compound Governance: Steps toward complete decentralization](https://medium.com/compound-finance/compound-governance-5531f524cf68)
- [compound-finance/compound-protocol](https://github.com/compound-finance/compound-protocol)
- [Compound Alpha Governance System Audit](https://blog.openzeppelin.com/compound-alpha-governance-system-audit/)
- [Compound Governance Trail of Bits Asessment](https://github.com/trailofbits/publications/blob/master/reviews/compound-governance.pdf)
##### Frax Governance
- [Frax Docs: Governance](https://docs.frax.finance/smart-contracts/governance)
- [Frax Docs: veFXS](https://docs.frax.finance/vefxs/vefxs)
- [Frax Governance Forum](https://gov.frax.finance/)
- [Ceazor's Overview](https://youtu.be/ce9y8_bQCIE)
- [veCRV & veFXS contracts](https://www.diffchecker.com/Sqm7PfNt)
- [FXS contract state](https://etherscan.io/address/0x3432B6A60D23Ca0dFCa7761B7ab56459D9C964D0#readContract)
- [Curve Dune Analytics Dashboard](https://dune.xyz/mrblock_buidl/Curve.fi)
- [Curve Docs](https://curve.readthedocs.io/dao-vecrv.html)
- [Curve Audits page](https://curve.fi/audits)
