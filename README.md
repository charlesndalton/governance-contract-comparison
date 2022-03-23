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

Score: 3

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

### **References**

##### Element Finance:
- [An Introduction to Element's Governance Model](https://medium.com/element-finance/an-introduction-to-elements-governance-model-efea13d1c7ee)
- [Governing Element Finance, A Security Roadmap Update](https://medium.com/element-finance/governing-element-finance-a-security-roadmap-update-e9cf6816c5d3)
- [Voting Vaults: A New DeFi and Governance Primitive](https://medium.com/element-finance/voting-vaults-a-new-defi-and-governance-primitive-b4b2f6289d48?source=collection_home---4------2-----------------------)
- [The Governance Steering Council](https://medium.com/element-finance/the-governance-steering-council-63aea7732262)
- [The Core Principles of Element Finance](https://medium.com/element-finance/the-core-principles-of-element-finance-fdbaf6aa5f1d)
- [Element Governance: A Technical Architecture Overview](https://medium.com/element-finance/element-governance-a-technical-architecture-overview-2d0f72bd278a)
- [element-fi/council](https://github.com/element-fi/council)