# Spartan v1 Design and Implementation Plan 

|                      |                                                                        |
| -------------------- | ---------------------------------------------------------------------- |
| Issue                | [Spartan](https://github.com/AztecProtocol/aztec-packages/issues/7482) |
| Owners               | @just-mitch @LHerskind @Maddiaa0                                       |
| Approvers            | @joeandrews @charlielye @iAmMichaelConnor @spalladino @PhilWindle      |
| Target Approval Date | 2024-07-24                                                             |

## Executive Summary

This design and implementation plan describes version 1 of "Spartan", a public network that will be operational on/before 2024-09-16.

We consider delivery of this network to be an engineering milestone; it remains to be seen if/how the network will be:
- communicated to the public
- used by developers

We have designed against a subset of the requirements for MainNet; those with checkmarks are part of this milestone:

- [x] have a "pending chain" with state updates at least every 30s
- [x] have a design that can support 10 TPS
- [x] "proposers" can build ahead on the "pending chain"
- [x] app developers and users can rely on the pending chain state for UX
- [x] have a mechanism to incentivize "proposers" to participate in the "pending chain"
- [x] have a CI/CD framework to easily deploy the network in different configurations for modeling, stress and regression tests
- [x] demonstrate building of full, proven rollup blocks on L1
- [x] have a design that can support forced inclusions
- [ ] support forced inclusions
- [ ] be able to select/rotate the set of "proposers" permissionlessly
- [ ] allow permissionless participation from "provers"
- [ ] a governance mechanism to support upgrades and integration with execution environment
- [ ] have an enshrined mechanism to punish "proposers" for misbehaving, and a quantifiable guarantee of the punishment
- [ ] do not depend on the "pending chain" for liveness
- [ ] do not require a hard fork to take advantage of most software updates
- [ ] integrated with the execution environment

Thus, Spartan v1 is effectively a stepping stone toward MainNet.

In a sense, part of Spartan v1 is a subset of [Option B-](https://hackmd.io/B1Ae2zp6QxaSDOQgdFKJkQ#Option-B-), with key differences:
- the validator set is small and defined by an Aztec Labs multisig (no PoS)
- all validators re-execute transactions
- TxEffects are committed to DA as opposed to TxObjects
- no built-in slashing
- no forced inclusions
- no optimistic signature verification
- no fallback to direct L1 sequencing (i.e. based sequencing)

**However, as stated, our design and implementation will be forward-compatible with the remaining MainNet requirements, and subsequent milestones will satisfy requirements based on their priority.**

To that end, there is a separate issue for [outlining MainNet](https://github.com/AztecProtocol/engineering-designs/pull/11).

## Introduction

We now introduce the broad design of Spartan v1.

### Spartan v1 Definitions

**Validator**
A node that is participating in consensus by producing blocks and attesting to blocks

**Prover**
A node that is responsible for generating proofs for blocks/epochs.

**Slot**
Time is divided into fixed length slots. Within each slot, exactly one validator is selected to propose a block. A slot might be empty if no block is proposed. Sometimes the validator selected to propose is called proposer or sequencer

**Epoch**
A fixed-length sequence of slots

**Committee**
A list of validators to propose/attest blocks for an epoch. The committee is stable throughout the epoch. Attesters are active throughout the entire duration, and one proposer per slot is active.

**Attestation**
A vote on the head on the chain.

**Confirmed**
Of a block, if it is included in a chain.

**Confirmation Rules**
Of a chain, the set of conditions that must be met for a block to be considered confirmed.


### Spartan v1 Chains

We will explicitly support multiple concurrent "chains":

- "Pending Chain" - Reflects blocks published to L1 with their state diffs, but not yet been proven.
- "Proven Chain" - Reflects blocks that have had their proof published and verified on L1.
- "Finalized Chain" - Reflects blocks in the "Proven Chain" that have been finalized on L1 (Casper).

The Finalized Chain is a prefix of the Proven Chain, which is a prefix of the Pending Chain.

Note: we do not need to "do" anything for the Finalized Chain, but it is relevant to users.

E.g., a front-end with low-value transactions may display the Pending Chain, but a DEX bridge might wait to release funds until the transaction is in the Finalized Chain.

In Spartan v1 the committee will solely be responsible for building the Pending and Proven Chains.

### Spartan v1 Sequencer Selection

The validator set will be selected by the multisig.

At the beginning of each epoch, we will assign proposers from the validator set to slots.

The exact number of validators will be determined via stress tests, modeling, and feedback from the community.

Part of the deliverable for this milestone will be an analysis of network performance based on the validator set size.

### Spartan v1 Prover Selection

The list of available provers will be determined by the multisig.

At the beginning of each epoch, a single prover will be selected to generate proofs for the epoch.

### Spartan v1 Incentives

For Spartan v1, we will create a simple incentives contract within the deployment:
- Whenever a block is added to the Proven Chain, rewards will be distributed to the participating proposer and prover.

### The Spartan v1 Pending Chain


#### Overview

```mermaid
sequenceDiagram
    participant PXE as PXE
    participant L2N as L2 Node
    participant MP as L2 P2P
    participant P as L2 Proposer
    participant V as L2 Validator
    participant L1R as L1 Rollup Contracts

    P->>L1R: Watches for new L2PendingBlockProcessed events
    P->>L1R: Download TxEffects
    P->>P: Apply TxEffects to the world state
    PXE->>L2N: Submit TxObject+Proof
    L2N->>MP: Add TxObject+Proof
    MP->>P: Broadcast TxObjects+Proofs
    MP->>V: Broadcast TxObjects+Proofs
    P->>P: Execute TxObjects, construct L2 block header
    P->>MP: Propose TxObjects and L2 block header for a slot 
    MP->>V: Broadcast proposal
    V->>V: Execute corresponding TxObjects, construct/compare header
    V->>MP: Signed proposal (if correct)
    MP->>P: Aggregate signatures
    P->>L1R: Submit TxEffects to L1 DA Oracle
    P->>L1R: Add L2 block to Pending Chain, supplying L2 Header and Signatures as call data
    L1R->>L1R: Check signatures, verify correct proposer, verify header/TxEffects available
    L1R->>L1R: Add L2 block to Pending Chain, emit L2PendingBlockProcessed
```

#### Confirmation Rules

A proposer may submit a block B to the rollup contract on L1 for inclusion in the Pending Chain.

The rollup contract will verify that:

- B is building on the tip of the Pending Chain:
  - The block number of B is one greater than the block number of the tip of the Pending Chain.
  - The slot of B is larger than the slot of the tip of the Pending Chain.
  - The slot of B is in the past
- B is submitted by the correct proposer for the slot.
- B contains proof it has signatures from 2/3 + 1 of the committee members attesting to the block.
- B's header has been made available on L1

After this, the block is confirmed in the Pending Chain.

### The Spartan v1 Proven Chain

#### Overview

It is the responsibility of the prover selected at the beginning of an epoch to submit the proof of the epoch.

The proof of epoch i must land after the proof of epoch i-1.

```mermaid
sequenceDiagram
    participant P as L2 Proposer
    participant MP as L2 P2P
    participant PN as L2 Prover Node
    participant L1R as L1 Rollup Contracts

    PN->>PN: startEpoch
    P->>MP: Propose TxObjects and L2 block header for a slot 
    MP->>PN: Broadcast proposal
    PN->>PN: Execute corresponding TxObjects, cache ProofRequests
    PN->>L1R: Watches for new L2PendingBlockProcessed events
    PN->>L1R: Download block header
    PN->>PN: Submit corresponding ProofRequests for pending block
    Note over PN: Cache root rollup proof for block
    Note over PN: After epoch
    PN->>PN: finalizeEpoch
    PN->>L1R: Submit proof of epoch to advance Proven Chain
    L1R->>L1R: Verify blocks in Pending Chain, verify proof
    L1R->>L1R: Add L2 block to Proven Chain, emit L2ProvenBlockProcessed
```

#### Confirmation Rules

A node may submit a transaction to the rollup contract on L1 to add blocks from the Pending Chain to the Proven Chain.

The rollup contract will verify that:

- The transaction is submitted by the correct proposer.
- The specified blocks are in the Pending Chain
- The TxEffects of all constituent transactions have been published
- A proof of the block's correctness has been verified on L1.

After this, all blocks in the epoch are confirmed in the Proven Chain.

### Spartan v1 Governance

The Spartan will have its own governance contract.

Version 1.0.0 will specify an L1 account owner by Aztec Labs that is able to add or remove sequencers and provers. 


## Interface

Who are your users, and how do they interact with this? What is the top-level interface?

### Smart Contracts
The contracts to support this milestone will provide minimal functionality, an overview of their arrangement is shown below.
![Alt text](./images/7482-contracts.png)

#### Teocalli (Multisig)
The Teocalli (Aztec great temple) contract will serve as governance for the testnet, for now it is sufficient that it be operated by Aztec Labs. In which case
its interface will follow that of an arbitrary multisig (gnosis safe will likely be used in practice).
```sol
interface ITeocalli {
  // Execute multisig operations
  function execute(bytes calldata cmd) external;
}
```


#### Rollup Contract

Much of the current rollup contract will remain unchanged.

It will maintain a simple validator registry owned by Teocalli. The addition and removal of validators / sequencers will be permissioned, governed by Teocalli. 


```sol
interface IRollup {
  // Append a block to the pending chain
  function processPendingBlock(bytes calldata header, bytes32 archive, bytes calldata signatures) external;

  // Prove the blocks in the pending chain
  function processProvenEpoch(bytes calldata header, bytes calldata proof) external;

  function getCurrentSlot() view external returns(uint256);

  function getCurrentEpoch() view external returns(uint256);


  ////////////////////////
  // Validator registry //
  ////////////////////////

  // Add a validator (who has previously staked) to the staking set.
  // @dev only admin
  function addValidator(address validator) external;

  // Remove a currently active validator from the staking set.
  // @dev only admin
  function removeValidator(address validator) external;

  // Return the currently active sequencer
  function getCurrentSequencer() view external returns (address);

  // Add a prover (who has previously staked) to the staking set.
  // @dev only admin
  function addProver(address prover) external;

  // Remove a currently active prover from the staking set.
  // @dev only admin
  function removeProver(address prover) external;

  // Return the currently active prover
  function getCurrentProver() view external returns (address);

}
```


### Prover Coordination

The Prover Node will not need to coordinate with sequencers, and will interface with the L2 P2P network and L1 to generate proof requests (and prove them).

## Change Set

- [ ] Cryptography
- [ ] Noir
- [ ] Aztec.js
- [ ] PXE
- [ ] Aztec.nr
- [ ] Enshrined L2 Contracts
- [ ] Private Kernel Circuits
- [x] Sequencer
- [ ] AVM
- [ ] Public Kernel Circuits
- [x] Rollup Circuits
- [x] L1 Contracts
- [x] Prover
- [x] Economics
- [x] P2P Network
- [x] DevOps

## Test Plan

### Local Clusters

We will be merging into master, so we will be adding unit and e2e test to the existing test suites that will run on every PR.

New e2e tests will be added which create an ephemeral, local network of nodes and test the basic functionality of the network.

### Parameterized clusters

We will have a set of parameters that can be passed to a deployment script to configure the network.

The parameters will be:
- Number of validators
- Number of provers
- Number of nodes
- Number of PXEs
- Slot time
- Epoch length
- Committee size
- P2P enabled
- Proving enabled
- Fees enabled
- Cluster type (local, spartan, devnet, etc.)

### Faults or Misbehaviors

We will need to configure components to be able to simulate different faults or misbehaviors.

We will need to be able to simulate:
- Nodes going offline
- Validators going offline
- Provers going offline
- Proposers going offline
- Proposers submitting invalid blocks
- L1 proposer misbehaving (e.g. censoring, or down)

### E2E Tests

The core cases we want to cover in our end-to-end tests are:

- A block is proposed and added to the Pending Chain
- A block is proven and added to the Proven Chain
- A block is finalized
- The network can tolerate a sequencer going offline
- The network can tolerate a prover going offline
- The network can tolerate a sequencer submitting an invalid block
- The network can tolerate sequencers/validators with slow connections

Additional attack scenarios we will test are:
- A block proposed without a signature from the current proposer should fail
- A prover submitting a proof with an invalid proof
- A coalition of dishonest sequencers submitting a block with an invalid state transition


### The Spartan Deployment

There will be a new cluster of nodes deployed in AWS called `spartan`.

We will create a new branch called `spartan` that we will merge master into whenever we want to redeploy the network.

This will be the cluster that we run stress tests on, which will be triggered whenever the network is redeployed.

**This will be a public deployment that external companies providing proving infrastructure can use.**


### Stress Tests

We will have a series of scenarios that we will run on the network to test its performance.

**It remains to be seen if we will run these tests on the public deployment or a separate deployment.**

We will run these scenarios across a range of parameters (number of nodes, validators, provers, slot duration, epoch duration) to assess the network's performance profile.

#### Basic Scenarios

We will want to determine the network's performance with various configurations under normal conditions:

- Sustained high TPS
- Burst TPS

#### Adverse Scenarios

We will also simulate conditions such as:

- Nodes, Validators, Proposers, Provers:
  - Going offline
  - Throttle compute / io / network
  - Ignore some requests (censorship)
    - Nodes not dispersing transactions
    - Validators ignoring some attestation requests
    - Proposers ignore some user transactions
    - Provers ignoring some proof requests

- DoS attacks on specific nodes, e.g.:
  - sending a large number of requests to a node with in/valid data
  - trying to trick the network into making a large number of requests to a node

#### Metrics

Some key metrics across all scenarios will be:
- Time for the proposer to advance their world state based on the Pending Chain
- Time for the proposer to prepare a block
- Time for the proposer to collect signatures
- Time for validators to verify a block
- Time for the proposer to submit a block to L1
- Number of simultaneous proofs that can be generated
- TPS of sequencer simulation
- TPS of the Pending Chain
- TPS of the Proven Chain
- TPS of a new node syncing
- TPS of a node following the chain

## Documentation Plan

We will add a README with information on how provers can participate in the Spartan.

We will also add a README with information on how to run the network locally.


