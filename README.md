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