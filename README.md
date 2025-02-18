# Technical Specification: Babylon BTC Staking Integration for Taiko

## Table of Contents
1. Introduction
2. System Architecture
3. Component Design
4. Integration Flows
5. Security Model
6. Implementation Plan
7. Technical Appendix

## 1. Introduction

### 1.1 Purpose
This specification outlines the technical implementation for integrating Babylon's Bitcoin staking protocol with Taiko's based rollup architecture. 

### 1.2 Scope
The integration encompasses:
- BTC staking mechanisms
- Finality provider infrastructure
- Preconfirmation enhancement
- Security protocols
- State synchronization
- Cross-chain communication

### 1.3 Design Goals
- Maintain Taiko's 12-second block time
- Achieve sub-3-second preconfirmations
- Ensure slashable safety
- Preserve decentralization
- Enable trustless BTC staking

## 2. System Architecture

### 2.1 High-Level Architecture

![](architecture.png)

### Cross-Chain Verification System

```mermaid
graph TB
    subgraph Bitcoin Chain
        BTCN[Bitcoin Network]
        BTCV[BTC Validator Set]
        SPV[SPV Verification]
        
        BTCN --> SPV
        BTCV --> SPV
    end
    
    subgraph Babylon Protocol
        subgraph Finality Module
            FSM[Finality State Machine]
            SV[Signature Verification]
            VP[Voting Power]
            
            FSM --> SV
            SV --> VP
        end
        
        subgraph Stake Management
            SM[Stake Manager]
            VM[Vault Manager]
            PM[Power Manager]
            
            SM --> VM
            VM --> PM
            PM --> VP
        end
        
        subgraph ZK Bridge
            ZG[ZK Generator]
            ZV[ZK Verifier]
            
            ZG --> ZV
        end
    end
    
    subgraph Taiko Network
        subgraph L2 Protocol
            BP[Block Proposer]
            BV[Block Verifier]
            PC[Preconfirmation]
            
            BP --> BV
            BV --> PC
        end
        
        subgraph State Machine
            SS[State Sync]
            FV[Finality Verifier]
            
            SS --> FV
            FV --> PC
        end
    end
    
    SPV --> VM
    VP --> FV
    ZV --> SS
```


### 2.2 Core Components

#### 2.2.1 Babylon Components
- **Babylon Node**: Customized for OP-stack support
- **Finality Contract**: Manages BTC-backed signatures
- **Staking Protocol**: Handles BTC delegation and slashing
- **State Bridge**: Facilitates cross-chain communication

#### 2.2.2 Taiko Components
- **Based Rollup Core**: Manages L2 block production
- **Preconfirmation System**: Handles fast confirmations
- **Finality Gadget**: Processes BTC-backed signatures
- **State Manager**: Tracks finalized states

## 3. Component Design

### 3.1 Finality Provider Implementation

```typescript
interface FinalityProvider {
    // Core functionality
    submitSignature(block: Block): Promise<void>;
    verifyStake(provider: Address): Promise<BigNumber>;
    trackPreconfirmations(): void;
    
    // State management
    updateState(state: State): void;
    syncWithBabylon(): Promise<void>;
    
    // Security features
    detectEquivocation(): Promise<boolean>;
    reportViolation(proof: Proof): Promise<void>;
}
```

### 3.2 BTC Staking Contract

#### Clarification and Deployment Details:
The **BTC staking contract** will be deployed on the **Babylon chain**, utilizing the CosmWasm smart contract framework. The Babylon chain is specifically designed to handle BTC staking and finality-related logic while ensuring trustless interactions between the Bitcoin network and OP stack-based chains like Taiko.

##### Why CosmWasm Instead of Solidity?
Although the contract is represented in Solidity syntax for illustrative purposes, the actual implementation will use **CosmWasm**. This is because:
1. **Compatibility with Babylon**: CosmWasm is natively supported on Babylon, which ensures seamless integration with the chain’s core architecture.
2. **Trustless BTC Staking**: Babylon allows Bitcoin holders to stake BTC in a self-custodial vault on the Bitcoin network without bridging assets. The CosmWasm environment enforces this trustless staking.
3. **Interoperability with OP Stack**: Babylon’s staking contracts serve as the backbone for BTC-backed finality on OP-stack chains.

The BTC staking contract will:
- Verify BTC transactions using SPV proofs, ensuring that BTC is staked natively on the Bitcoin blockchain.
- Manage BTC delegations, including slashing mechanisms to maintain security.
- Act as the bridge between Bitcoin and Babylon for cross-chain finality processes.

#### Staking Contract Interface:
The following interface demonstrates the core functionality of the BTC staking contract:

```solidity
contract BTCStaking {
    struct Stake {
        bytes32 btcTxHash;        // Hash of the BTC transaction locking funds
        uint256 amount;           // Amount of BTC staked
        address provider;         // Address of the staking provider
        uint256 lockPeriod;       // Locking period for the stake
    }

    struct Signature {
        bytes32 blockHash;        // Hash of the finalized block
        bytes sig;                // Finality signature
        uint256 timestamp;        // Timestamp of the signature
    }

    mapping(address => Stake) public stakes;                 // Active stakes
    mapping(bytes32 => Signature[]) public blockSignatures;  // Finality signatures for blocks

    function stake(bytes32 txHash, bytes proof) external;    // Stake BTC
    function submitSignature(bytes32 blockHash, bytes sig) external; // Submit finality signature
    function verifyStake(address provider) external view returns (uint256); // Verify stake
    function slash(address provider, bytes proof) external;  // Slash provider for violations
}


### 3.3 Preconfirmation Enhancement

```solidity
contract EnhancedPreconfirmation {
    struct Preconfirmation {
        bytes32 blockHash;
        uint256 timestamp;
        bytes btcStakingProof;
        Signature[] signatures;
    }
    
    mapping(bytes32 => Preconfirmation) public preconfirmations;
    
    function submitPreconfirmation(
        bytes32 blockHash,
        bytes btcProof,
        Signature[] memory sigs
    ) external;
    
    function verifyPreconfirmation(bytes32 blockHash) 
        external view returns (bool, uint256);
}
```

#### Core Operations:

- **Staking BTC:** Users lock BTC in a self-custodial vault on the Bitcoin network. The stake function validates the staking proof using SPV.
- **Finality Signatures:** The `submitSignature` function allows providers to sign L2 blocks. These signatures are submitted to the Babylon chain for verification and tallied for quorum achievement.
- **Slashing Violations:** If a provider misbehaves (e.g., submits invalid signatures or equivocates), the slash function enforces penalties, reducing the staked BTC as per the slashing conditions.


## 4. Integration Flows

### 4.1 Staking Flowrrr

```mermaid
sequenceDiagram
    participant BTC as Bitcoin Network
    participant BS as BTC Staker
    participant FP as Finality Provider
    participant BC as Babylon Chain
    participant TK as Taiko Network
    
    BS->>BTC: Lock BTC in vault
    BS->>BC: Submit stake proof
    BC->>BC: Verify BTC proof
    BC->>FP: Register stake
    FP->>TK: Begin block signing
    TK->>BC: Verify provider status
```

### 4.2 Block Finalization Flow

```mermaid
sequenceDiagram
    participant P as Proposer
    participant FP as Finality Provider
    participant BC as Babylon Chain
    participant TK as Taiko Network
    
    P->>TK: Propose block
    TK->>FP: Request signatures
    FP->>BC: Submit signatures
    BC->>BC: Verify signatures
    BC->>TK: Confirm finality
    TK->>TK: Update state
```

## 5. Security Model

### 5.1 Slashing Conditions

```solidity
interface ISlashing {
    enum SlashingReason {
        Equivocation,
        InvalidSignature,
        PreconfirmationViolation
    }
    
    struct SlashingProof {
        bytes32 violation;
        bytes proof;
        uint256 timestamp;
    }
    
    function submitSlashingProof(
        address provider,
        SlashingReason reason,
        SlashingProof calldata proof
    ) external;
}
```

### 5.2 Security Parameters

```typescript
const SecurityConfig = {
    MIN_STAKE: "1.0 BTC",
    SLASH_AMOUNT: "0.1 BTC",
    QUORUM_THRESHOLD: "66%",
    FINALITY_BLOCKS: 100,
    MAX_REORG_DEPTH: 100,
    PRECONF_WINDOW: "3s"
};
```

## 6. Implementation Plan

### 6.1 Phase 1: Core Integration
1. Babylon node customization
2. Basic staking implementation
3. Signature verification system
4. Initial testing framework

### 6.2 Phase 2: Security Enhancement
1. Slashing mechanism implementation
2. Preconfirmation security
3. State sync optimization
4. Security auditing

### 6.3 Phase 3: Production Readiness
1. Performance optimization
2. Monitoring systems
3. Documentation
4. Mainnet preparation

## 7. Technical Appendix

### 7.1 API Specifications

#### 7.1.1 Finality Provider API

```typescript
interface FinalityProviderAPI {
    // Staking Management
    stake(amount: BigNumber, duration: number): Promise<void>;
    withdraw(amount: BigNumber): Promise<void>;
    
    // Block Processing
    signBlock(blockHash: string): Promise<Signature>;
    submitSignature(sig: Signature): Promise<void>;
    
    // State Management
    getStakingStatus(): Promise<StakingStatus>;
    getSignatureQuorum(blockHash: string): Promise<QuorumStatus>;
}
```

#### 7.1.2 Preconfirmation API

```typescript
interface PreconfirmationAPI {
    // Submission
    submitPreconfirmation(block: Block): Promise<void>;
    
    // Verification
    verifyPreconfirmation(
        blockHash: string,
        signatures: Signature[]
    ): Promise<boolean>;
    
    // Monitoring
    getPreconfirmationStatus(
        blockHash: string
    ): Promise<PreconfirmationStatus>;
}
```

### 7.2 State Transition Rules

```typescript
interface StateTransition {
    // State Types
    type BlockState = "Proposed" | "Preconfirmed" | "Finalized";
    type SignatureState = "Pending" | "Valid" | "Invalid";
    
    // Transition Functions
    function validateTransition(
        currentState: BlockState,
        newState: BlockState,
        proof: TransitionProof
    ): boolean;
    
    function applyTransition(
        state: BlockState,
        transition: Transition
    ): Promise<BlockState>;
}
```

### 7.3 Network Parameters

```typescript
const NetworkConfig = {
    // Timing Parameters
    BLOCK_TIME: "12s",
    PRECONF_INTERVAL: "3s",
    SIGNATURE_TIMEOUT: "6s",
    
    // Network Parameters
    MAX_BLOCK_SIZE: "2MB",
    MAX_SIGNATURES_PER_BLOCK: 100,
    MIN_FINALITY_CONFIRMATIONS: 6,
    
    // Staking Parameters
    MIN_STAKE_DURATION: "30 days",
    MAX_STAKE_DURATION: "365 days",
    SLASH_LOCK_PERIOD: "14 days"
};
```
