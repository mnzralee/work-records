# Hyperledger Besu, Solidity & the Diamond Pattern
## A Complete Technical Lecture for Fabric Developers

**Author**: For Manazir Ali — building on existing Hyperledger Fabric knowledge
**Context**: GX Protocol migration from Fabric to Besu
**Prerequisite**: Fabric architecture (orgs, orderers, peers, channels, chaincode lifecycle)

---

## Table of Contents

### Foundation (Parts 1-9)

1. [Part 1: The EVM Mental Model — From Fabric to Besu](#part-1-the-evm-mental-model)
2. [Part 2: Hyperledger Besu — The Node](#part-2-hyperledger-besu--the-node)
3. [Part 3: Solidity — The Language](#part-3-solidity--the-language)
4. [Part 4: Storage — How State Actually Works](#part-4-storage--how-state-actually-works)
5. [Part 5: The Diamond Pattern (EIP-2535)](#part-5-the-diamond-pattern-eip-2535)
6. [Part 6: How GX Protocol Uses All of This](#part-6-how-gx-protocol-uses-all-of-this)
7. [Part 7: Security Considerations](#part-7-security-considerations)
8. [Part 8: Development Tooling](#part-8-development-tooling)
9. [Part 9: CQRS Pipeline — Besu Edition](#part-9-cqrs-pipeline--besu-edition)

### Advanced (Parts 10-19)

10. [Part 10: Solidity Inheritance, Interfaces & Abstract Contracts](#part-10-solidity-inheritance-interfaces--abstract-contracts)
11. [Part 11: Assembly, Yul & Low-Level EVM](#part-11-assembly-yul--low-level-evm)
12. [Part 12: ABI Encoding — `encode` vs `encodePacked`](#part-12-abi-encoding--encode-vs-encodepacked)
13. [Part 13: Constructor vs Initializer Pattern](#part-13-constructor-vs-initializer-pattern)
14. [Part 14: Gas Optimization Patterns](#part-14-gas-optimization-patterns)
15. [Part 15: Testing with Hardhat](#part-15-testing-with-hardhat)
16. [Part 16: Deployment Scripts & Upgrade Procedures](#part-16-deployment-scripts--upgrade-procedures)
17. [Part 17: Error Handling on the Backend](#part-17-error-handling-on-the-backend)
18. [Part 18: Event Log Structure at the Byte Level](#part-18-event-log-structure-at-the-byte-level)
19. [Part 19: Libraries — Internal vs Deployed](#part-19-libraries--internal-vs-deployed)

---

## Part 1: The EVM Mental Model

### 1.1 The Fundamental Shift

In Fabric, you think in terms of **organizations**, **channels**, and **chaincode**. A transaction gets endorsed by peers, ordered by orderers, and committed to the channel ledger. Your chaincode reads/writes a key-value store via `GetState`/`PutState`.

In Besu (and all EVM chains), you think in terms of **accounts**, **transactions**, and **contracts**. There are no organizations, no channels, no orderers in the Fabric sense. Instead:

```
Fabric World                          EVM/Besu World
─────────────                         ──────────────
Organization (Org1, Org2)       →     Validator node (runs consensus)
Peer (endorses + commits)       →     Node (validates + executes)
Orderer (sequences txns)        →     Consensus protocol (QBFT/IBFT2)
Channel (private ledger)        →     Single shared world state
Chaincode (Go program)          →     Smart contract (Solidity bytecode)
GetState/PutState               →     Storage slots (256-bit slots)
TransactionContext              →     msg.sender, block.timestamp, etc.
Fabric CA (x509 certs)          →     Account addresses (20-byte hex)
```

### 1.2 Two Types of Accounts

This is the most fundamental concept. Everything in EVM revolves around accounts:

**Externally Owned Accounts (EOAs)** — controlled by private keys (like human wallets)
```
Address:  0x742d35Cc6634C0532925a3b844Bc9e7595f2bD8e
Balance:  1.5 ETH (or 0 in permissioned chains)
Nonce:    47 (number of transactions sent)
```
Think of this like a Fabric identity (x509 certificate), except it's just a cryptographic keypair. No CA needed. You generate a private key → derive the public key → hash it → that's your address.

**Contract Accounts** — controlled by code (your smart contracts)
```
Address:  0x1234567890AbcdEF1234567890aBcDEF12345678
Balance:  0
Nonce:    1
Code:     [compiled Solidity bytecode]
Storage:  [key-value pairs, 256-bit slots]
```
Think of this like your chaincode, except it lives AT an address on the chain. When someone calls it, the EVM loads its code and executes it. The contract has its own persistent storage (like your CouchDB state, but structured differently).

### 1.3 How a Transaction Executes

In Fabric:
```
Client → Endorsing Peers (simulate) → Orderer (sequence) → All Peers (commit)
```

In Besu:
```
Client → Any Node (broadcasts) → Validators (consensus) → All Nodes (execute + commit)
```

The critical difference: **every node executes every transaction independently and arrives at the same result**. There's no "endorsement" step — determinism is guaranteed by the EVM itself (it's a virtual machine with no access to randomness, network, filesystem, or clock beyond block.timestamp).

A transaction in EVM looks like:
```
{
  from:     0xSenderAddress,        // Who's sending (like ctx.GetClientIdentity())
  to:       0xContractAddress,      // Which contract to call (like chaincode name)
  data:     0x[function selector + encoded args],  // Which function + parameters
  value:    0,                      // ETH to send (usually 0 in permissioned)
  gas:      300000,                 // Max computation budget
  nonce:    47                      // Replay protection
}
```

### 1.4 Gas — The Computation Meter

This has NO equivalent in Fabric. In Fabric, your chaincode can run as long as it wants (within timeout). In EVM, every operation costs "gas":

```
Operation              Gas Cost
─────────              ────────
ADD (addition)         3
MUL (multiplication)   5
SLOAD (read storage)   2100       ← Reading state is EXPENSIVE
SSTORE (write new)     20000      ← Writing new state is VERY EXPENSIVE
SSTORE (update)        5000       ← Updating existing is cheaper
LOG (emit event)       375 + data
CREATE (new contract)  32000
```

Each transaction has a gas limit. If computation exceeds it, the transaction reverts (all changes undone, gas still consumed).

**In permissioned Besu**: Gas still exists but the gas PRICE can be set to 0. You still pay gas as a computation limit, but there's no real economic cost. Think of it as a safety mechanism to prevent infinite loops, not as a fee system.

For GX Protocol: Transaction fees are handled at the APPLICATION level (your TaxAndFeeFacet), not at the gas level.

### 1.5 Determinism: Built-In, Not Manual

In Fabric, you had to be careful:
```go
// WRONG in Fabric — non-deterministic
timestamp := time.Now()

// CORRECT in Fabric
timestamp := ctx.GetStub().GetTxTimestamp()
```

In EVM, you literally CANNOT access non-deterministic data. The Solidity language has no:
- `time.Now()` — only `block.timestamp` (set by the validator, same for all)
- `rand()` — no randomness source at all
- `http.Get()` — no network access
- `os.ReadFile()` — no filesystem access

The EVM is a sealed sandbox. Everything that goes in (transaction data, block info) is deterministic. This is why all nodes can independently execute and arrive at the same state — it's architecturally impossible to get different results.

---

## Part 2: Hyperledger Besu — The Node

### 2.1 What Besu Actually Is

Hyperledger Besu is an **Ethereum client** written in Java. Just like Fabric has peer binaries and orderer binaries, Besu has a single binary that can run in different modes:

```
Fabric                              Besu
─────                               ────
peer binary                    →    besu binary (single binary does everything)
orderer binary                 →    (consensus is built into the same binary)
Fabric CA                      →    (no CA needed — accounts are keypairs)
configtx.yaml                 →    genesis.json
docker-compose-dev.yaml        →    docker-compose.yaml (simpler)
```

### 2.2 Consensus: QBFT (Replaces Raft)

In Fabric, you used Raft for ordering. Besu uses **QBFT** (Quorum Byzantine Fault Tolerant):

```
Raft (Fabric)                       QBFT (Besu)
─────────────                       ────────────
Crash fault tolerant (CFT)     →    Byzantine fault tolerant (BFT)
Leader election                →    Round-robin proposer rotation
Tolerates F crashes (2F+1)     →    Tolerates F Byzantine (3F+1)
Fast (single leader)           →    Slightly slower (multi-round voting)
```

QBFT works in rounds:
1. **Proposer** (rotates each block): Creates a block with pending transactions
2. **PREPARE**: Validators verify the block and broadcast "I've seen it"
3. **COMMIT**: Once 2/3+ validators agree, they sign and commit
4. **Block finalized**: Immediate finality (no forks, no reorgs)

This is crucial for GX Protocol — **immediate finality** means once a transaction is in a block, it's permanent. No waiting for "confirmation blocks" like public Ethereum.

### 2.3 Network Topology — Drastically Simpler

Your Fabric DevNet:
```
3 Orderers + 2 Peers + 2 CouchDB + 2 CLI + 1 CA = 10 containers
Plus: crypto-config.yaml, configtx.yaml, channel creation, chaincode lifecycle
```

Besu DevNet equivalent:
```
4 Validator Nodes = 4 containers
Plus: genesis.json (single file)
```

That's it. No separate orderers. No CouchDB (state is built-in). No CA (accounts are keypairs). No channel creation. No chaincode lifecycle approval across orgs.

### 2.4 genesis.json — Your configtx.yaml Equivalent

The genesis file defines the initial state of the chain (illustrative example — GX DevNet uses `chainId: 197294`):

```json
{
  "config": {
    "chainId": 1337,
    "qbft": {
      "blockperiodseconds": 2,
      "epochlength": 30000,
      "requesttimeoutseconds": 4
    }
  },
  "nonce": "0x0",
  "timestamp": "0x0",
  "gasLimit": "0x1fffffffffffff",
  "difficulty": "0x1",
  "alloc": {
    "0xfe3b557e8fb62b89f4916b721be55ceb828dbd73": {
      "balance": "0xad78ebc5ac6200000"
    }
  },
  "extraData": "0x...[validator addresses encoded here]..."
}
```

Key fields:
- `chainId`: Unique network identifier (like Fabric channel name)
- `qbft.blockperiodseconds`: How often blocks are produced (2 seconds)
- `gasLimit`: Max gas per block (set very high for permissioned)
- `alloc`: Pre-funded accounts (like your genesis distribution, but for ETH/native currency)
- `extraData`: Encodes the initial validator set

### 2.5 Permissioning — Replacing Fabric's MSP

Fabric uses MSP (Membership Service Provider) with x509 certificates. If you're not in the MSP, you can't transact.

Besu has two permissioning layers:

**Node Permissioning** — which nodes can join the network:
```json
// permissions_config.toml
[nodes-allowlist]
nodes = [
  "enode://pubkey1@ip1:port1",
  "enode://pubkey2@ip2:port2"
]
```

**Account Permissioning** — which accounts can send transactions:
```json
// On-chain permissioning contract or local config
[accounts-allowlist]
accounts = [
  "0xfe3b557e8fb62b89f4916b721be55ceb828dbd73",
  "0x627306090abaB3A6e1400e9345bC60c78a8BEf57"
]
```

For GX Protocol: Your `AccessControlFacet` + `LibAccessControl` replaces Fabric's ABAC at the smart contract level, while Besu's node/account permissioning replaces MSP at the network level.

### 2.6 Privacy Groups (Replacing Channels)

Fabric uses channels for privacy — separate ledgers for separate groups. Besu uses **privacy groups** via Tessera (an off-chain privacy manager):

```
Fabric Channel                      Besu Privacy Group
──────────────                      ──────────────────
Separate ledger per channel    →    Private state stored off-chain in Tessera
Only members see data          →    Only group members see data
Chaincode deployed per channel →    Private contracts deployed per group
Join/leave channel             →    Add/remove from privacy group
```

**For GX Protocol**: You likely don't need privacy groups. Fabric channels made sense for multi-org consortiums where Org1 shouldn't see Org2's data. GX is a single-protocol system — all participants operate under the same rules on the same ledger. Your access control is at the smart contract level (roles), not at the ledger level (channels).

---

## Part 3: Solidity — The Language

### 3.1 Solidity vs Go — Mental Model

You wrote chaincode in Go. Solidity is fundamentally different:

```go
// Go (Fabric chaincode) — procedural, you manage state manually
func (tc *TokenomicsContract) Transfer(
    ctx contractapi.TransactionContextInterface,
    fromID string, toID string, amount int64,
) error {
    fromBalance, _ := ctx.GetStub().GetState("balance_" + fromID)
    // ... manually serialize/deserialize JSON
    ctx.GetStub().PutState("balance_" + fromID, newBalanceBytes)
    ctx.GetStub().SetEvent("TransferEvent", eventJSON)
    return nil
}
```

```solidity
// Solidity (Besu) — object-oriented, state is built into the contract
function transfer(
    string memory fromId, string memory toId, uint256 amount
) external {
    bytes32 fromKey = keccak256(abi.encodePacked(fromId));
    bytes32 toKey = keccak256(abi.encodePacked(toId));

    // State variables are persistent automatically — no GetState/PutState
    balances[fromKey] -= amount;   // Automatically saved to blockchain storage
    balances[toKey] += amount;     // Automatically saved to blockchain storage

    emit TransferEvent(fromKey, toKey, fromId, toId, amount, block.timestamp);
}
```

Key differences:
1. **No manual serialization** — Solidity variables persist automatically
2. **No GetState/PutState** — you just read/write variables
3. **No JSON** — data is stored in fixed-size 256-bit slots
4. **`emit` instead of `SetEvent`** — events are a language feature
5. **`msg.sender` instead of `ctx.GetClientIdentity()`** — caller identity is automatic

### 3.2 Data Types

```solidity
// Integers — the workhorse
uint256 amount;        // Unsigned 256-bit integer (0 to 2^256-1)
int256  delta;         // Signed 256-bit integer
uint8   status;        // Unsigned 8-bit (0-255, like an enum)
// No int64 like Go — Solidity uses uint256 as the standard size
// uint256 can hold numbers up to ~1.15 × 10^77 — more than enough for 1.25T × 10^6 Qirat

// Addresses — 20-byte account identifiers
address owner;                           // 0x742d35Cc6634C0532925a3b844Bc...
address payable recipient;               // Can receive ETH (not relevant for GX)

// Bytes — fixed-size byte arrays
bytes32 participantKey;                  // 32-byte hash (like a composite key in Fabric)
bytes4  functionSelector;                // First 4 bytes of function signature hash

// Strings — dynamic-size (EXPENSIVE in gas)
string fabricUserId;                     // "MY 506 ABI832 0PTNY 5758"
// Strings can't be compared directly: use keccak256(abi.encodePacked(a)) == keccak256(abi.encodePacked(b))

// Booleans
bool paused;

// Arrays
uint256[] dynamicArray;                  // Dynamic size (stored in storage)
uint256[10] fixedArray;                  // Fixed size

// Mappings — the KEY data structure (like CouchDB documents)
mapping(bytes32 => uint256) balances;    // participantKey → balance
mapping(bytes32 => bool) hasVoted;       // voterKey → voted?
mapping(bytes32 => mapping(bytes32 => bool)) nested;  // Can nest mappings
```

### 3.3 The `bytes32` Pattern — Why GX Uses It Everywhere

In Fabric, your state keys were strings:
```go
key := "USER_" + userID
ctx.GetStub().GetState(key)
```

In Solidity, string operations are **extremely expensive** (gas-wise). So GX Protocol converts all string identifiers to `bytes32` hashes for internal storage:

```solidity
// Convert string fabricUserId to bytes32 key (used throughout GX)
bytes32 participantKey = keccak256(abi.encodePacked(fabricUserId));

// Now use this fixed-size key for all storage lookups
balances[participantKey] = 1000000;  // Fast: single SLOAD/SSTORE
```

Why this matters:
- `mapping(string => uint256)` — every lookup hashes the string anyway internally
- `mapping(bytes32 => uint256)` — pre-hashing gives you control and consistency
- `bytes32` comparisons are single-instruction (`EQ` opcode = 3 gas)
- `string` comparisons require hashing both strings first (~200+ gas)

### 3.4 Functions — Visibility and Modifiers

```solidity
contract MyContract {

    // EXTERNAL — can only be called from outside the contract (like your exported Go functions)
    // This is what external callers (your backend) invoke
    function transfer(string memory fromId, string memory toId, uint256 amount) external {
        // ...
    }

    // PUBLIC — can be called externally AND internally
    function getBalance(string memory userId) public view returns (uint256) {
        // ...
    }

    // INTERNAL — can only be called within this contract or derived contracts
    // Like private Go functions (lowercase first letter)
    function _validateAmount(uint256 amount) internal pure {
        require(amount > 0, "Amount must be positive");
    }

    // PRIVATE — can only be called within THIS contract (not even derived)
    function _calculateHash(string memory s) private pure returns (bytes32) {
        return keccak256(abi.encodePacked(s));
    }

    // VIEW — reads state but doesn't modify it (free to call, no transaction needed)
    // Like a Fabric query that doesn't write
    function isPaused() external view returns (bool) {
        return _paused;
    }

    // PURE — doesn't even read state (pure computation)
    function add(uint256 a, uint256 b) external pure returns (uint256) {
        return a + b;
    }
}
```

**Fabric equivalent mapping:**
```
Fabric                              Solidity
──────                              ────────
Exported function (capital)    →    external / public
Unexported function (lower)   →    internal / private
Query (no PutState)            →    view
Pure computation               →    pure
```

### 3.5 `memory` vs `storage` vs `calldata`

This is unique to Solidity and has no Fabric equivalent. It controls WHERE data lives:

```solidity
function example(
    string memory input,        // MEMORY: temporary copy, exists during this call only
    string calldata rawInput    // CALLDATA: read-only reference to transaction input data
) external {
    // STORAGE: persistent on blockchain (like PutState)
    LibTokenomics.Layout storage s = LibTokenomics.layout();
    s.totalSupply = 1250000000000;  // This persists after the function ends

    // MEMORY: temporary, gone after function returns
    string memory temp = string(abi.encodePacked("hello", input));
    // temp disappears when function returns

    // You cannot assign storage to memory or vice versa without explicit copying
}
```

Rules of thumb:
- **`storage`** = blockchain state (persistent, expensive to read/write)
- **`memory`** = RAM (temporary, cheap, gone after function call)
- **`calldata`** = read-only input data (cheapest, used for `external` function parameters)

### 3.6 Events — Your `SetEvent` Replacement

In Fabric:
```go
eventJSON, _ := json.Marshal(event)
ctx.GetStub().SetEvent("TransferEvent", eventJSON)
```

In Solidity:
```solidity
// DECLARE the event shape (like defining a struct)
event TransferEvent(
    bytes32 indexed fromKey,      // "indexed" = filterable (up to 3 indexed fields)
    bytes32 indexed toKey,
    string  fromFabricUserId,     // Non-indexed = included in data, not filterable
    string  toFabricUserId,
    uint256 amountQirat,
    uint256 timestamp
);

// EMIT the event (like SetEvent)
emit TransferEvent(fromKey, toKey, fromId, toId, amount, block.timestamp);
```

**Why `indexed` matters**: Indexed fields go into "topics" — special log fields that can be efficiently filtered by Ethereum nodes. Your projector can subscribe to events and filter:
```javascript
// ethers.js — subscribe to TransferEvents for a specific sender
contract.on(
  contract.filters.TransferEvent(fromKey),  // Filter by indexed fromKey
  (fromKey, toKey, fromId, toId, amount, timestamp) => {
    console.log(`Transfer: ${fromId} → ${toId}: ${amount} Qirat`);
  }
);
```

In Fabric, you subscribed to ALL block events and filtered manually. In Besu, the node does the filtering for you.

### 3.7 Error Handling — `require`, `revert`, Custom Errors

In Fabric:
```go
if amount <= 0 {
    return fmt.Errorf("amount must be positive")
}
```

In Solidity, three patterns (from oldest to newest):

```solidity
// Pattern 1: require (old style — wastes gas on string storage)
require(amount > 0, "Amount must be positive");
require(msg.sender == owner, "Not authorized");

// Pattern 2: revert with message
if (amount == 0) revert("Amount must be positive");

// Pattern 3: Custom errors (GX Protocol uses this — cheapest, most informative)
error GX_Tokenomics_ZeroAmount();
error GX_Tokenomics_InsufficientBalance(bytes32 participantKey, uint256 required, uint256 available);

function transfer(string memory fromId, uint256 amount) external {
    if (amount == 0) revert GX_Tokenomics_ZeroAmount();

    bytes32 key = keccak256(abi.encodePacked(fromId));
    uint256 balance = balances[key];
    if (balance < amount) {
        revert GX_Tokenomics_InsufficientBalance(key, amount, balance);
    }
}
```

Custom errors are:
- **Cheaper** — error selector is 4 bytes vs full string storage
- **Type-safe** — parameters are encoded, not string-concatenated
- **Decodable** — your backend can parse the error and extract parameters

**GX Convention** (`GX_<FacetDomain>_<Condition>`):
```solidity
error GX_Tokenomics_ZeroAmount();
error GX_Tokenomics_WalletFrozen(bytes32 participantKey);
error GX_Governance_AlreadyVoted(bytes32 proposalKey, bytes32 voterKey);
error GX_LoanPool_AmountBelowMinimum(uint256 provided, uint256 minimum);
```

### 3.8 Modifiers — Reusable Guards

Modifiers are like middleware in Express.js or like wrapping every Fabric function with `RequireSuperAdmin(ctx)`:

```solidity
// Definition
modifier onlyAdmin() {
    require(msg.sender == admin, "Not admin");
    _;  // ← This is where the actual function body executes
}

modifier whenNotPaused() {
    require(!paused, "System is paused");
    _;
}

// Usage — modifiers chain left-to-right
function dangerousAction() external onlyAdmin whenNotPaused {
    // This code only runs if both modifiers pass
}
```

However, GX Protocol uses **library-based enforcement** instead of modifiers (for Diamond Pattern compatibility):

```solidity
function pauseSystem(string calldata reason) external {
    LibAccessControl.enforceRole(LibAccessControl.GX_SUPER_ADMIN);  // Like RequireSuperAdmin(ctx)
    LibAdmin.enforceNotPaused();                                     // Like checking system status
    // ... actual logic
}
```

### 3.9 Solidity Arithmetic — Safe by Default (Since 0.8.0)

In Go, integer overflow wraps silently:
```go
var x uint64 = math.MaxUint64
x = x + 1  // Wraps to 0 — silent bug!
```

In Solidity 0.8+ (which GX uses), overflow automatically reverts:
```solidity
uint256 x = type(uint256).max;
x = x + 1;  // REVERTS — transaction fails, state unchanged
```

This means you don't need manual overflow checks. The EVM does it for you. This is a massive security advantage over Go chaincode where you had to be careful with arithmetic.

For the rare case where you WANT wrapping (like hash computations):
```solidity
unchecked {
    uint256 hash = a + b;  // Won't revert on overflow
}
```

---

## Part 4: Storage — How State Actually Works

### 4.1 The Storage Model — Slots, Not Key-Value

This is the biggest conceptual shift from Fabric. In Fabric, state is a key-value store:
```
Key: "USER_abc123"  →  Value: {"name":"Alice","balance":1000} (JSON bytes)
Key: "USER_def456"  →  Value: {"name":"Bob","balance":500}
```

In EVM, each contract has **2^256 storage slots**, each 32 bytes wide:

```
Slot 0:  0x0000000000000000000000000000000000000000000000000000000000000001  (bool paused = true)
Slot 1:  0x0000000000000000000000000000000000000000000000000000000065a8f3c0  (uint256 pausedAt)
Slot 2:  0x000000000000000000000000742d35cc6634c0532925a3b844bc9e7595f2bd8e  (address pausedBy)
```

Simple state variables get sequential slots:
```solidity
contract Example {
    bool paused;           // Slot 0
    uint256 pausedAt;      // Slot 1
    address pausedBy;      // Slot 2
    uint256 totalSupply;   // Slot 3
}
```

### 4.2 How Mappings Work in Storage

Mappings don't use sequential slots. Instead, they use a hash to FIND the slot:

```solidity
mapping(bytes32 => uint256) balances;  // Declared at slot 5

// To find where balances[key] is stored:
// slot = keccak256(key . slot_number)
// slot = keccak256(abi.encode(key, 5))
```

So `balances[0xabc...]` might be at slot `0x7f3e...` (a hash). This means:
- Mappings are sparse — you can have 2^256 keys without pre-allocating anything
- You can't iterate over a mapping (no way to discover which slots have values)
- Each mapping entry is independent (reading one doesn't load others)

### 4.3 Why Storage Matters for Gas

```
Operation                    Gas Cost
─────────                    ────────
Read a storage slot (SLOAD)  2100 gas
Write new value (SSTORE)     20000 gas     ← 10x more expensive than reading!
Write to zero (SSTORE)       5000 gas      ← Cheaper if updating existing
Clear to zero (SSTORE)       Gets 4800 gas refund
```

This is why GX Protocol uses `bytes32` keys instead of `string` keys — fewer storage reads/writes, lower gas cost.

**Practical example** — a transfer in storage operations:
```solidity
function transfer(bytes32 from, bytes32 to, uint256 amount) internal {
    uint256 fromBalance = balances[from];    // SLOAD: 2100 gas
    uint256 toBalance = balances[to];        // SLOAD: 2100 gas
    balances[from] = fromBalance - amount;   // SSTORE: 5000 gas (update existing)
    balances[to] = toBalance + amount;       // SSTORE: 5000-20000 gas (depends if new)
    // Total: ~14,200 - 29,200 gas for a simple transfer
}
```

In Fabric, you don't think about this. In Besu, understanding storage cost informs every design decision.

### 4.4 Diamond Storage Pattern

This is critical for understanding how GX Protocol's facets share state. Normal contracts use sequential slots. But when you have 19 facets all sharing one Diamond's storage, slot collisions would be catastrophic.

Solution: **each library uses a unique, deterministic storage slot derived from a namespace string**:

```solidity
library LibTokenomics {
    // Storage slot = keccak256("gx.protocol.storage.tokenomics")
    // This gives a pseudo-random slot number that won't collide with other libraries
    bytes32 constant STORAGE_SLOT = keccak256("gx.protocol.storage.tokenomics");

    // Protocol-level constants (NOT in the struct — these are compile-time values)
    uint8   constant DECIMALS = 6;
    uint256 constant QIRAT_PER_GX = 1_000_000;
    uint256 constant MAX_SUPPLY = 1_250_000_000_000 * QIRAT_PER_GX;  // 1.25T GX in Qirat

    struct Layout {
        bool initialized;
        string name;
        string symbol;
        uint256 totalMinted;
        uint256 totalBurned;
        mapping(bytes32 => uint256) balances;
        mapping(bytes32 => bool) frozenWallets;
        mapping(bytes32 => bool) genesisDistributed;
        mapping(bytes32 => DemurrageInfo) demurrage;
        mapping(bytes32 => PoolInfo) pools;
        // ... + participantIdReverse, poolBalances, countryStats, globalCounters, etc.
    }

    function layout() internal pure returns (Layout storage s) {
        bytes32 slot = STORAGE_SLOT;
        assembly {
            s.slot := slot
        }
    }
}
```

When `TokenomicsFacet` calls `LibTokenomics.layout()`, it gets a pointer to a `Layout` struct starting at a specific storage slot. When `GovernanceFacet` calls `LibGovernance.layout()`, it gets a DIFFERENT struct at a DIFFERENT slot. No collisions.

```
Diamond Contract Storage:
═════════════════════════

Slot 0x7f3e...  ┌─── LibTokenomics.Layout ───┐
(illustrative)  │ initialized: true           │
                │ totalMinted: 500000000      │
                │ totalBurned: 0              │
                │ balances[key1]: 1000        │
                │ balances[key2]: 2000        │
                └─────────────────────────────┘

Slot 0xa1b2...  ┌─── LibGovernance.Layout ───┐
(illustrative)  │ initialized: true           │
                │ votingPeriodSeconds: 604800 │
                │ proposalCount: 5            │
                │ proposals[key]: {...}       │
                └─────────────────────────────┘

Slot 0xc3d4...  ┌─── LibAdmin.Layout ────────┐
(illustrative)  │ paused: false               │
                │ bootstrapped: true          │
                │ pausedAt: 0                 │
                │ parameters[key]: "10"       │
                └─────────────────────────────┘
```

Each library's storage is isolated by its unique position hash. This is what makes the Diamond Pattern possible — 19 facets, 13 libraries, zero storage collisions.

---

## Part 5: The Diamond Pattern (EIP-2535)

### 5.1 The Problem It Solves

In Fabric, upgrading chaincode is the full lifecycle:
```bash
# Package → Install on all peers → Approve per org → Commit
peer lifecycle chaincode package gxtv3.tar.gz ...
peer lifecycle chaincode install gxtv3.tar.gz
peer lifecycle chaincode approveformyorg ...
peer lifecycle chaincode commit ...
```

This replaces the ENTIRE chaincode. Every function, every contract. If you fix a bug in `TokenomicsContract`, you redeploy ALL 7 contracts.

In Besu without Diamond:
- Solidity contracts are immutable once deployed. You literally cannot change the code.
- To "upgrade," you deploy a NEW contract at a NEW address and tell everyone to use the new one.
- This breaks all references, requires migrating state, and is error-prone.

### 5.2 The Diamond Solution

The Diamond Pattern creates a single **proxy contract** (the Diamond) that delegates calls to **facet contracts**:

```
                                    ┌─────────────────────────┐
                                    │      Diamond Proxy      │
Client ─── calls ────────────────── │   Address: 0xDiamond    │
    transfer("alice","bob",1000)    │                         │
                                    │   Storage: ALL STATE    │
                                    │   Code: routing logic   │
                                    │                         │
                                    │   Function → Facet Map: │
                                    │   transfer() → 0xFacet1 │
                                    │   pause()    → 0xFacet2 │
                                    │   vote()     → 0xFacet3 │
                                    └───────┬─────────────────┘
                                            │
                              ┌─────────────┼─────────────┐
                              │             │             │
                    ┌─────────▼──┐  ┌───────▼────┐  ┌────▼──────────┐
                    │TokenomicsFacet│  │AdminFacet│  │GovernanceFacet│
                    │  0xFacet1  │  │  0xFacet2  │  │   0xFacet3    │
                    │            │  │            │  │               │
                    │ transfer() │  │ pause()    │  │ vote()        │
                    │ mint()     │  │ unpause()  │  │ submit()      │
                    │ burn()     │  │ bootstrap()│  │ execute()     │
                    └────────────┘  └────────────┘  └───────────────┘

All facets execute in the Diamond's storage context (delegatecall)
```

### 5.3 How `delegatecall` Works — The Magic

This is the key mechanism. Normal calls execute code in the CALLED contract's context. `delegatecall` executes code in the CALLER's context:

```
Normal call:
  Diamond calls TokenomicsFacet.transfer()
  → Executes in TokenomicsFacet's storage ← WRONG, we want Diamond's storage

delegatecall:
  Diamond delegatecalls TokenomicsFacet.transfer()
  → Executes TokenomicsFacet's CODE but uses Diamond's STORAGE ← CORRECT
  → msg.sender remains the original caller (your backend), not the Diamond
```

Think of it like this: `delegatecall` says "run this code AS IF it were my own code, using my own storage." The facet is just a library of functions — it has no state of its own.

### 5.4 The Routing Mechanism (Diamond Fallback)

When you call `transfer()` on the Diamond, how does it know to delegate to TokenomicsFacet?

Every function in Solidity has a **4-byte selector** — the first 4 bytes of the keccak256 hash of its signature:

```
transfer(string,string,uint256) → keccak256("transfer(string,string,uint256)") → 0x9b80b050
pause()                         → keccak256("pause()")                         → 0x8456cb59
```

The Diamond maintains a mapping: `selector → facet address`:

```solidity
// Inside the Diamond proxy (simplified)
mapping(bytes4 => address) selectorToFacet;

// When you call any function on the Diamond:
fallback() external payable {
    // 1. Extract the function selector from calldata
    bytes4 selector = bytes4(msg.data[:4]);

    // 2. Look up which facet handles this function
    address facet = selectorToFacet[selector];
    require(facet != address(0), "Function not found");

    // 3. Delegate the call to that facet
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

The `fallback()` function is a special Solidity function that executes when no other function matches. Since the Diamond itself doesn't define `transfer()` or `pause()`, every call hits the fallback, which routes to the correct facet.

### 5.5 DiamondCut — How You Upgrade

The `DiamondCut` operation is how you add, replace, or remove facets. This is your upgrade mechanism:

```solidity
enum FacetCutAction { Add, Replace, Remove }

struct FacetCut {
    address facetAddress;          // The facet contract address
    FacetCutAction action;         // Add, Replace, or Remove
    bytes4[] functionSelectors;    // Which functions this cut affects
}

// Example: Upgrade TokenomicsFacet with a bug fix
diamondCut([
    {
        facetAddress: 0xNewTokenomicsFacet,  // New version deployed
        action: FacetCutAction.Replace,       // Replace existing
        functionSelectors: [
            bytes4(keccak256("transfer(string,string,uint256)")),
            bytes4(keccak256("mint(string,uint256,string)")),
            // ... all TokenomicsFacet selectors
        ]
    }
]);
```

**What this means for GX Protocol:**
- Fix a bug in velocity tax? Deploy new `TaxAndFeeFacet`, DiamondCut to Replace. Zero downtime, no state migration.
- Add a new feature (e.g., escrow)? Deploy `EscrowFacet`, DiamondCut to Add. Existing facets untouched.
- Remove deprecated function? DiamondCut to Remove those selectors.
- **Storage is untouched** — because all state lives in the Diamond, swapping facets doesn't affect data.

### 5.6 DiamondLoupe — Introspection

The Loupe lets you inspect the Diamond's current state:

```solidity
interface IDiamondLoupe {
    // Get all facets and their selectors
    function facets() external view returns (Facet[] memory);

    // Get all selectors for a specific facet
    function facetFunctionSelectors(address facet) external view returns (bytes4[] memory);

    // Get all facet addresses
    function facetAddresses() external view returns (address[] memory);

    // Get which facet handles a specific selector
    function facetAddress(bytes4 selector) external view returns (address);
}
```

This is like querying `peer lifecycle chaincode queryinstalled` in Fabric, but at runtime and on-chain.

### 5.7 The Full GX Diamond Architecture

```
                    ┌──────────────────────────────────────┐
                    │          GX Diamond (0xDiamond)       │
                    │                                      │
                    │  ALL storage lives here:              │
                    │  ├── LibTokenomics.Layout             │
                    │  ├── LibAdmin.Layout                  │
                    │  ├── LibIdentity.Layout               │
                    │  ├── LibOrganization.Layout           │
                    │  ├── LibGovernance.Layout              │
                    │  ├── LibTaxAndFee.Layout              │
                    │  ├── LibLoanPool.Layout               │
                    │  ├── LibGovernment.Layout             │
                    │  ├── LibAccessControl.Layout          │
                    │  ├── LibNamingService.Layout          │
                    │  ├── LibThresholdEncryption.Layout    │
                    │  └── LibDiamond.Layout                │
                    │                                      │
                    │  fallback() → route by selector      │
                    └──────────────┬───────────────────────┘
                                   │ delegatecall
                    ┌──────────────┼───────────────────────────┐
                    │              │                           │
     ┌──────────────┼──────────────┼──────────────┐           │
     │              │              │              │           │
     ▼              ▼              ▼              ▼           ▼
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐
│Tokenomics│ │  Admin   │ │Identity  │ │  Org     │ │  + 15 more...    │
│  Facet   │ │  Facet   │ │  Facet   │ │  Facet   │ │  Governance      │
│          │ │          │ │          │ │          │ │  TaxAndFee        │
│ 21 fns   │ │ 10 fns   │ │ ~10 fns  │ │ ~13 fns  │ │  LoanPool        │
│          │ │          │ │          │ │          │ │  Government(x3)   │
│ Uses:    │ │ Uses:    │ │ Uses:    │ │ Uses:    │ │  NamingService    │
│ LibToken │ │ LibAdmin │ │ LibIdent │ │ LibOrg   │ │  ThresholdEncrypt │
│ LibAdmin │ │ LibAccess│ │ LibAdmin │ │ LibAdmin │ │  AuditLog         │
│ LibAccess│ │          │ │ LibAccess│ │ LibAccess│ │  AccessControl    │
│ LibReentr│ │          │ │          │ │ LibReentr│ │  DiamondCut(x2)   │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ │  DiamondLoupe     │
                                                     │  TokenomicsConfig │
                                                     └──────────────────┘
```

---

## Part 6: How GX Protocol Uses All of This

### 6.1 Mapping Fabric Concepts to Besu Implementation

Let's trace a complete flow — **genesis distribution** — in both systems:

**Fabric:**
```go
// 1. Backend calls chaincode via Fabric Gateway SDK
result, err := contract.SubmitTransaction("DistributeGenesis", userID, nationality)

// 2. Inside DistributeGenesis (Go):
func (tc *TokenomicsContract) DistributeGenesis(ctx contractapi.TransactionContextInterface,
    userID string, nationality string) error {

    // Check access
    if err := RequirePartnerAPI(ctx); err != nil { return err }

    // Read user state
    userBytes, _ := ctx.GetStub().GetState("USER_" + userID)
    var user User
    json.Unmarshal(userBytes, &user)

    // Check idempotency
    if user.GenesisMinted { return fmt.Errorf("already distributed") }

    // Calculate amount based on phase
    amount := calculateGenesisAmount(nationality)

    // Update balance
    balKey := "BALANCE_" + userID
    balBytes, _ := ctx.GetStub().GetState(balKey)
    currentBal := bytesToInt64(balBytes)
    newBal := currentBal + amount
    ctx.GetStub().PutState(balKey, int64ToBytes(newBal))

    // Mark as distributed
    user.GenesisMinted = true
    userBytes, _ = json.Marshal(user)
    ctx.GetStub().PutState("USER_" + userID, userBytes)

    // Emit event
    event := GenesisEvent{UserID: userID, Amount: amount}
    eventBytes, _ := json.Marshal(event)
    ctx.GetStub().SetEvent("GenesisDistributionEvent", eventBytes)

    return nil
}
```

**Besu:**
```solidity
// 1. Backend calls contract via ethers.js
const tx = await diamond.distributeGenesis(fabricUserId, nationality, phase, subPhase);
await tx.wait();  // Wait for block inclusion

// 2. Inside TokenomicsFacet.distributeGenesis (Solidity):
function distributeGenesis(
    string memory fabricUserId,
    string memory nationality,
    uint8 phase,
    string memory subPhase
) external {
    // Check access (replaces RequirePartnerAPI)
    LibAccessControl.enforceRole(LibAccessControl.GX_PARTNER_API);

    // Check not paused
    LibAdmin.enforceNotPaused();

    // Reentrancy guard (no Fabric equivalent — EVM-specific protection)
    LibReentrancyGuard.lock();

    // Get storage layout (replaces GetState)
    LibTokenomics.Layout storage s = LibTokenomics.layout();

    // Compute participant key (replaces string key construction)
    bytes32 participantKey = keccak256(abi.encodePacked(fabricUserId));

    // Check idempotency (replaces user.GenesisMinted check)
    if (s.genesisDistributed[participantKey]) {
        revert GX_Tokenomics_GenesisAlreadyDistributed(participantKey);
    }

    // Calculate amount (same business logic, different syntax)
    uint256 amount = _calculateGenesisAmount(nationality, phase, subPhase);

    // Update balance (replaces GetState + PutState)
    s.balances[participantKey] += amount;    // Automatically persisted!
    s.totalMinted += amount;

    // Mark as distributed
    s.genesisDistributed[participantKey] = true;

    // Emit event (replaces SetEvent)
    emit GenesisDistributionEvent(
        participantKey, fabricUserId, amount, nationality, phase, subPhase, block.timestamp
    );

    LibReentrancyGuard.unlock();
}
```

### 6.2 Access Control — ABAC vs Role-Based

**Fabric ABAC:**
```go
// access_control.go
func RequireSuperAdmin(ctx contractapi.TransactionContextInterface) error {
    // Read x509 certificate from transaction context
    identity, _ := ctx.GetClientIdentity().GetID()
    role, found, _ := ctx.GetClientIdentity().GetAttributeValue("gxc_role")
    if !found || role != "gx_super_admin" {
        return fmt.Errorf("unauthorized: requires gx_super_admin role")
    }
    return nil
}
```

**Besu Role-Based:**
```solidity
// LibAccessControl.sol
library LibAccessControl {
    error GX_Access_Unauthorized(address caller, bytes32 role);
    bytes32 constant STORAGE_SLOT = keccak256("gx.protocol.storage.access.control");

    // Role constants (like Fabric attribute values, but as bytes32 hashes)
    // Blockchain-level roles (from chaincode access_control.go)
    bytes32 constant GX_SUPER_ADMIN  = keccak256("GX_SUPER_ADMIN");
    bytes32 constant GX_ADMIN        = keccak256("GX_ADMIN");
    bytes32 constant GX_PARTNER_API  = keccak256("GX_PARTNER_API");

    // Protocol operations roles
    bytes32 constant PROTOCOL_ADMIN  = keccak256("PROTOCOL_ADMIN");
    bytes32 constant GOVERNANCE_ROLE = keccak256("GOVERNANCE_ROLE");
    bytes32 constant IDENTITY_ADMIN  = keccak256("IDENTITY_ADMIN");
    bytes32 constant TREASURY_ROLE   = keccak256("TREASURY_ROLE");
    bytes32 constant FSP_ROLE        = keccak256("FSP_ROLE");
    bytes32 constant VALIDATOR_ROLE  = keccak256("VALIDATOR_ROLE");
    // + 16 more: admin portal roles + government treasury roles (TREASURY_CONTROLLER, INITIATOR, etc.)

    struct Layout {
        mapping(bytes32 => mapping(address => bool)) roles;  // role → address → hasRole
        mapping(bytes32 => bytes32) roleAdmin;               // role → admin role
    }

    function hasRole(bytes32 role, address account) internal view returns (bool) {
        return layout().roles[role][account];
    }

    function enforceRole(bytes32 role) internal view {
        if (!hasRole(role, msg.sender)) revert GX_Access_Unauthorized(msg.sender, role);
    }

    function grantRole(bytes32 role, address account) internal {
        layout().roles[role][account] = true;
    }
}
```

The key difference: In Fabric, roles are embedded in x509 certificates issued by Fabric CA. In Besu, roles are stored ON-CHAIN in the Diamond's storage and managed by `AccessControlFacet`.

### 6.3 The fabricUserId Bridge

GX Protocol has a critical design choice: **participants are identified by `fabricUserId` strings, NOT by Ethereum addresses.** This is unusual for EVM contracts but essential for GX because:

1. Your backend (svc-wallet, svc-tokenomics) calls the contract on behalf of participants
2. Participants don't have Ethereum wallets — they use your mobile app
3. The `fabricUserId` is the canonical identity from your KYC system

So the flow is:
```
Participant → Mobile App → svc-wallet (Express.js) → ethers.js → Diamond → TokenomicsFacet

msg.sender = svc-wallet's Ethereum address (the PARTNER_API account)
fabricUserId = passed as a function parameter (the participant's identity)
```

This means `msg.sender` is always your backend service, not the participant. Access control verifies the SERVICE has the right role (PARTNER_API), and the `fabricUserId` parameter identifies WHICH participant the operation is for.

---

## Part 7: Security Considerations

### 7.1 Reentrancy — The EVM-Specific Attack

This has no equivalent in Fabric because Fabric's read-write sets prevent it. In EVM, a malicious contract can call back into your contract during execution:

```solidity
// VULNERABLE CODE
function withdraw(uint256 amount) external {
    require(balances[msg.sender] >= amount);
    (bool success, ) = msg.sender.call{value: amount}("");  // Sends ETH
    // ↑ If msg.sender is a contract, it can call withdraw() AGAIN here
    // before the balance is updated!
    balances[msg.sender] -= amount;  // Too late — already re-entered
}
```

**GX Protocol's protection** — `LibReentrancyGuard`:
```solidity
library LibReentrancyGuard {
    bytes32 constant STORAGE_SLOT = keccak256("gx.protocol.storage.reentrancy");

    error GX_Reentrancy_Locked();

    // Uses uint256 instead of bool — uninitialized storage (default 0) = UNLOCKED
    uint256 private constant UNLOCKED = 0;
    uint256 private constant LOCKED = 1;

    struct Layout {
        uint256 status;
    }

    function lock() internal {
        Layout storage s = layout();
        if (s.status == LOCKED) revert GX_Reentrancy_Locked();
        s.status = LOCKED;
    }

    function unlock() internal {
        layout().status = UNLOCKED;
    }
}

// Usage in every state-mutating financial function
function transfer(...) external {
    LibReentrancyGuard.lock();
    // ... do work ...
    LibReentrancyGuard.unlock();
}
```

### 7.2 Storage Collision — Diamond-Specific Risk

If two libraries accidentally use the same storage slot, they'll overwrite each other's data. GX prevents this with unique namespace strings:

```solidity
// Each library uses a DIFFERENT namespace string → DIFFERENT slot (illustrative hashes)
keccak256("gx.protocol.storage.tokenomics")       → 0x7f3e...
keccak256("gx.protocol.storage.admin")             → 0xa1b2...
keccak256("gx.protocol.storage.governance")        → 0xc3d4...
keccak256("gx.protocol.storage.access.control")    → 0xd5e6...
keccak256("gx.protocol.storage.reentrancy")        → 0xf7a8...
// Collision probability: 1 in 2^256 — effectively impossible
```

### 7.3 Facet Upgrade Safety

When you DiamondCut to replace a facet, the new facet MUST use the same storage layout as the old one. If you change the order of fields in a Layout struct, you'll corrupt data:

```solidity
// Version 1
struct Layout {
    bool initialized;       // Slot N+0
    uint256 totalMinted;    // Slot N+1
    uint256 maxSupply;      // Slot N+2
}

// Version 2 — WRONG (inserted field shifts everything)
struct Layout {
    bool initialized;       // Slot N+0
    bool paused;            // Slot N+1 ← NEW FIELD
    uint256 totalMinted;    // Slot N+2 ← WAS AT N+1, NOW READS maxSupply!
    uint256 maxSupply;      // Slot N+3 ← SHIFTED
}

// Version 2 — CORRECT (append only)
struct Layout {
    bool initialized;       // Slot N+0 (unchanged)
    uint256 totalMinted;    // Slot N+1 (unchanged)
    uint256 maxSupply;      // Slot N+2 (unchanged)
    bool paused;            // Slot N+3 ← NEW FIELD APPENDED AT END
}
```

**Rule: Only append new fields to the end of Layout structs. Never reorder, remove, or insert.**

### 7.4 Access Control on DiamondCut

The biggest security risk in a Diamond is: who can call `diamondCut`? If anyone can swap facets, they can replace your `TokenomicsFacet` with a malicious version that drains all balances.

GX Protocol has TWO cut facets:
- `DiamondCutFacet` — standard, owner-only
- `GXDiamondCutFacet` — GX-specific, with governance controls

Only the contract owner (initially the deployer, later the governance system) can perform cuts.

---

## Part 8: Development Tooling

### 8.1 Hardhat — Your Development Framework

Hardhat is to Besu what the Fabric SDK + Docker Compose is to Fabric:

```
Fabric                              Hardhat (Besu)
──────                              ───────────────
deploy.sh                     →    hardhat deploy script
go test                        →    hardhat test (Mocha + Chai)
peer chaincode invoke          →    hardhat console / scripts
docker-compose up              →    hardhat node (local EVM)
cryptogen                      →    hardhat accounts (auto-generated)
```

### 8.2 ethers.js — Your Fabric Gateway SDK Replacement

In Fabric:
```javascript
// Fabric Gateway SDK
const gateway = new Gateway();
await gateway.connect(ccp, { wallet, identity: 'user1' });
const network = await gateway.getNetwork('gxchannel');
const contract = network.getContract('gxtv3');
const result = await contract.submitTransaction('Transfer', fromID, toID, amount);
```

In Besu:
```javascript
// ethers.js
const provider = new ethers.JsonRpcProvider('http://besu-node:8545');
const wallet = new ethers.Wallet(privateKey, provider);
const diamond = new ethers.Contract(DIAMOND_ADDRESS, ABI, wallet);
const tx = await diamond.transfer(fromId, toId, amount);
const receipt = await tx.wait();  // Wait for block inclusion
```

### 8.3 Event Subscription — Replacing Fabric Block Events

In Fabric:
```javascript
// Subscribe to block events
const listener = await network.addBlockListener(async (event) => {
    for (const tx of event.getTransactionEvents()) {
        for (const contractEvent of tx.getContractEvents()) {
            if (contractEvent.eventName === 'TransferEvent') {
                // Process event
            }
        }
    }
});
```

In Besu:
```javascript
// Subscribe to specific events with filtering
diamond.on('TransferEvent', (fromKey, toKey, fromId, toId, amount, timestamp, event) => {
    console.log(`Transfer: ${fromId} → ${toId}: ${amount} Qirat`);
    console.log(`Block: ${event.blockNumber}, Tx: ${event.transactionHash}`);
});

// Or query historical events
const filter = diamond.filters.TransferEvent(specificFromKey);
const events = await diamond.queryFilter(filter, fromBlock, toBlock);
```

This is significantly cleaner than Fabric's block event subscription. You get:
- Type-safe event parameters
- Built-in filtering by indexed fields
- Historical event queries
- No manual JSON deserialization

### 8.4 ABI — The Contract Interface

ABI (Application Binary Interface) is how external code knows what functions a contract has and how to encode/decode calls. It's like your chaincode API reference, but machine-readable:

```json
[
    {
        "type": "function",
        "name": "transfer",
        "inputs": [
            { "name": "fromId", "type": "string" },
            { "name": "toId", "type": "string" },
            { "name": "amount", "type": "uint256" }
        ],
        "outputs": [],
        "stateMutability": "nonpayable"
    },
    {
        "type": "event",
        "name": "TransferEvent",
        "inputs": [
            { "name": "fromKey", "type": "bytes32", "indexed": true },
            { "name": "toKey", "type": "bytes32", "indexed": true },
            { "name": "amount", "type": "uint256", "indexed": false }
        ]
    }
]
```

The ABI is generated by the Solidity compiler. Your backend uses it to:
1. Encode function calls (what bytes to send in the transaction)
2. Decode return values (what the function returned)
3. Decode events (what happened during execution)

For a Diamond, you combine the ABIs of ALL facets into a single "Diamond ABI" that your backend uses.

---

## Part 9: CQRS Pipeline — Besu Edition

### 9.1 The Architecture Stays the Same

Your CQRS pattern doesn't fundamentally change:

```
Fabric CQRS:
Service → OutboxCommand → PostgreSQL → outbox-submitter → Fabric SDK → Blockchain
                                                                            ↓
                                            projector ← Fabric block events ←

Besu CQRS:
Service → OutboxCommand → PostgreSQL → outbox-submitter → ethers.js → Diamond
                                                                          ↓
                                          projector ← ethers.js events ←
```

### 9.2 What Changes in the Outbox Submitter

The outbox-submitter currently uses the Fabric Gateway SDK. For Besu, it switches to ethers.js:

```typescript
// BEFORE (Fabric)
async submitToChaincode(command: OutboxCommand): Promise<string> {
    const contract = this.fabricNetwork.getContract('gxtv3');
    const result = await contract.submitTransaction(
        command.functionName,
        ...command.args
    );
    return Buffer.from(result).toString();
}

// AFTER (Besu)
async submitToChaincode(command: OutboxCommand): Promise<string> {
    const tx = await this.diamond[command.functionName](...command.args);
    const receipt = await tx.wait();
    return receipt.hash;
}
```

The command resolution (`resolveChaincode`) maps outbox command types to Solidity function names instead of Go function names — but since GX designed Besu function names to match Fabric, this is mostly a 1:1 swap.

### 9.3 What Changes in the Projector

The projector switches from Fabric block event subscription to ethers.js event subscription:

```typescript
// BEFORE (Fabric)
async startListening(): Promise<void> {
    const listener = await this.network.addBlockListener(async (blockEvent) => {
        for (const txEvent of blockEvent.getTransactionEvents()) {
            for (const contractEvent of txEvent.getContractEvents()) {
                await this.handleEvent(contractEvent.eventName, contractEvent.payload);
            }
        }
    });
}

// AFTER (Besu) — ethers.js v6
async startListening(): Promise<void> {
    // Query events block-by-block (more resilient than subscription)
    const latestBlock = await this.provider.getBlockNumber();
    const events = await this.diamond.queryFilter('*', lastProcessedBlock, latestBlock);

    for (const event of events) {
        if (event instanceof ethers.EventLog) {
            await this.handleEvent(event.eventName, event.args);
        }
    }

    // Or use WebSocket subscription for real-time
    this.diamond.on('TransferEvent', async (...args) => {
        const event = args[args.length - 1]; // Last arg is the ContractEventPayload
        await this.handleEvent('TransferEvent', args.slice(0, -1));
    });
}
```

### 9.4 Event Name Compatibility

GX Protocol deliberately uses the SAME event names in Besu as in Fabric:

```
Fabric Event Name              Besu Event Name
─────────────────              ───────────────
TransferEvent            →    TransferEvent
GenesisDistributionEvent →    GenesisDistributionEvent
SystemPaused             →    SystemPaused
VelocityTaxApplied       →    VelocityTaxApplied
LoanApprovedEvent        →    LoanApprovedEvent
```

This means your projector handlers can stay largely the same — only the event delivery mechanism changes, not the event processing logic.

### 9.5 Key Difference: fabricUserId Resolution

In Fabric, events carry `fabricUserId` strings directly. In Besu, events carry `bytes32 participantKey` (indexed) AND `string fabricUserId` (non-indexed):

```solidity
event TransferEvent(
    bytes32 indexed fromKey,       // For efficient on-chain filtering
    bytes32 indexed toKey,
    string  fromFabricUserId,      // For projector to resolve to profileId
    string  toFabricUserId,
    uint256 amountQirat,
    uint256 timestamp
);
```

The projector still does the same `fabricUserId → profileId` resolution via `UserProfile.findUnique({ where: { fabricUserId } })`. This mapping is unchanged.

---

## Quick Reference Card

### Fabric → Besu Concept Map

```
INFRASTRUCTURE
  Org1, Org2              →  Validator nodes
  Peer                    →  Besu node
  Orderer                 →  QBFT consensus (built into node)
  Channel                 →  Single chain (privacy groups if needed)
  Fabric CA               →  Account keypairs (no CA)
  MSP                     →  On-chain permissioning + AccessControlFacet
  CouchDB                 →  EVM storage slots (built-in)

CHAINCODE / SMART CONTRACTS
  Go chaincode            →  Solidity facets
  contract.go embedding   →  Diamond proxy + DiamondCut
  GetState/PutState       →  Storage variables (automatic persistence)
  SetEvent                →  emit Event(...)
  GetClientIdentity       →  msg.sender + LibAccessControl
  TransactionContext       →  msg.sender, block.timestamp, msg.data
  JSON marshal/unmarshal  →  ABI encoding/decoding (automatic)
  Composite keys          →  keccak256(abi.encodePacked(...)) → bytes32
  InvokeChaincode         →  Shared Diamond storage / cross-facet reads

LIFECYCLE
  Package + Install       →  Compile (solc / hardhat compile)
  Approve + Commit        →  Deploy (hardhat deploy)
  Upgrade chaincode       →  DiamondCut (add/replace/remove facets)
  Init function           →  Constructor + initialize() functions

OPERATIONS
  peer chaincode invoke   →  ethers.js contract.functionName()
  peer chaincode query    →  ethers.js contract.functionName() (view)
  Block event listener    →  ethers.js contract.on('EventName', ...)
  Fabric Gateway SDK      →  ethers.js library
```

### GX Protocol Besu File Map

```
blockchain-besu/
├── contracts/
│   ├── diamond/
│   │   └── Diamond.sol              # The proxy — all calls go through here
│   ├── facets/                      # Business logic (19 facets)
│   │   ├── TokenomicsFacet.sol      # 21 functions — transfers, minting, balances
│   │   ├── AdminFacet.sol           # 10 functions — pause, bootstrap, params
│   │   ├── IdentityFacet.sol        # ~10 functions — participants, relationships
│   │   ├── OrganizationFacet.sol    # ~13 functions — orgs, multi-sig
│   │   ├── GovernanceFacet.sol      # 7 functions — proposals, voting
│   │   ├── TaxAndFeeFacet.sol       # 5 functions — velocity tax, fees
│   │   ├── LoanPoolFacet.sol        # 5 functions — interest-free lending
│   │   ├── GovernmentFacet.sol      # Government treasury operations
│   │   ├── GovernmentLendingFacet.sol
│   │   ├── GovernmentTransitionFacet.sol
│   │   ├── AccessControlFacet.sol   # Role management (grant/revoke)
│   │   ├── AuditLogFacet.sol        # On-chain audit trail
│   │   ├── NamingServiceFacet.sol   # Human-readable name resolution
│   │   ├── TokenomicsConfigFacet.sol
│   │   ├── ThresholdEncryptionFacet.sol
│   │   ├── ThresholdEncryptionConfigFacet.sol
│   │   ├── DiamondCutFacet.sol      # Standard upgrade mechanism
│   │   ├── GXDiamondCutFacet.sol    # GX governance-gated upgrades
│   │   └── DiamondLoupeFacet.sol    # Introspection
│   ├── libraries/                   # Shared storage + logic (13 libraries)
│   │   ├── LibTokenomics.sol        # Tokenomics storage layout + helpers
│   │   ├── LibAdmin.sol             # Admin storage layout + enforceNotPaused
│   │   ├── LibIdentity.sol          # Identity storage layout
│   │   ├── LibOrganization.sol      # Organization storage layout
│   │   ├── LibGovernance.sol        # Governance storage layout + helpers
│   │   ├── LibTaxAndFee.sol         # Tax storage + velocity tax bands
│   │   ├── LibLoanPool.sol          # Loan pool storage layout
│   │   ├── LibGovernment.sol        # Government storage layout
│   │   ├── LibAccessControl.sol     # Role storage + enforcement
│   │   ├── LibNamingService.sol     # Naming service storage
│   │   ├── LibThresholdEncryption.sol
│   │   ├── LibReentrancyGuard.sol   # Reentrancy protection
│   │   └── LibDiamond.sol           # Diamond core (selector mapping, cuts)
│   └── interfaces/
│       ├── IDiamondCut.sol          # DiamondCut interface standard
│       └── IDiamondLoupe.sol        # DiamondLoupe interface standard
├── hardhat.config.js                # Build + test configuration
└── package.json
```

---

## Summary: What You Need to Remember

1. **Everything is an account** — EOAs (people) and contracts (code) both live at addresses
2. **Storage is slots, not key-value** — but mappings give you key-value semantics
3. **Gas limits computation** — but in permissioned Besu, it's just a safety net
4. **Determinism is guaranteed by the EVM** — you literally can't write non-deterministic code
5. **The Diamond is a proxy** — one address, many facets, shared storage via libraries
6. **delegatecall is the magic** — facet code runs in Diamond's storage context
7. **DiamondCut is your upgrade path** — swap facets without touching state
8. **Storage layouts are append-only** — never reorder fields in Library structs
9. **Events are first-class** — indexed fields enable efficient filtering
10. **Your CQRS pipeline mostly stays the same** — swap Fabric SDK for ethers.js

---

## Part 10: Solidity Inheritance, Interfaces & Abstract Contracts

### 10.1 Inheritance — `is` Keyword

In Go, you embedded structs to compose behavior:
```go
type SmartContract struct {
    contractapi.Contract
    IdentityContract      // Embeds all IdentityContract methods
    TokenomicsContract    // Embeds all TokenomicsContract methods
}
```

Solidity uses the `is` keyword for inheritance:

```solidity
// Base contract
contract Pausable {
    bool private _paused;

    modifier whenNotPaused() {
        require(!_paused, "Paused");
        _;
    }

    function _pause() internal {
        _paused = true;
    }
}

// Derived contract — inherits everything from Pausable
contract TokenomicsFacet is Pausable {
    function transfer(...) external whenNotPaused {
        // Can use whenNotPaused because we inherited it
    }
}
```

**Multiple inheritance** is allowed (unlike Java, like Python):
```solidity
contract MyContract is Pausable, Ownable, ReentrancyGuard {
    // Inherits from ALL three
}
```

**Why GX Protocol does NOT use inheritance for facets**: In a Diamond, all facets share the Diamond's storage via `delegatecall`. If `TokenomicsFacet` inherited from `Pausable`, the `_paused` variable would be at slot 0 of the facet — but `delegatecall` maps it to slot 0 of the Diamond, which might already be used by `LibDiamond`. This causes storage collisions.

Instead, GX uses **library calls** that explicitly access namespaced storage:
```solidity
// Instead of: contract TokenomicsFacet is Pausable { ... whenNotPaused ... }
// GX does:
function transfer(...) external {
    LibAdmin.enforceNotPaused();  // Reads from LibAdmin's namespaced storage slot
    // ...
}
```

### 10.2 Interfaces — Contracts Without Implementation

An interface defines WHAT functions exist without HOW they work:

```solidity
// Interface — no function bodies, no state variables, no constructor
interface IDiamondCut {
    enum FacetCutAction { Add, Replace, Remove }

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}
```

Think of interfaces like Go interfaces — they define a contract (pun intended) that implementing contracts must fulfill:

```go
// Go equivalent concept
type IDiamondCut interface {
    DiamondCut(cuts []FacetCut, init common.Address, calldata []byte) error
}
```

**GX uses interfaces for:**
- `IDiamondCut` — the standard Diamond upgrade interface
- `IDiamondLoupe` — the standard Diamond introspection interface
- Type-safe interaction between contracts

When your backend creates a contract instance:
```typescript
// ethers.js uses the interface (ABI) to know what functions exist
const diamond = new ethers.Contract(DIAMOND_ADDRESS, IDiamondCut_ABI, wallet);
```

### 10.3 Abstract Contracts — Partially Implemented

Between interfaces (no implementation) and full contracts (all implemented), abstract contracts provide some implementation:

```solidity
abstract contract AccessControlBase {
    // Implemented — derived contracts get this for free
    function _checkRole(bytes32 role) internal view {
        require(hasRole(role, msg.sender), "Missing role");
    }

    // NOT implemented — derived contracts MUST implement this
    function hasRole(bytes32 role, address account) public view virtual returns (bool);
}

contract MyAccessControl is AccessControlBase {
    // MUST implement hasRole or contract won't compile
    function hasRole(bytes32 role, address account) public view override returns (bool) {
        return _roles[role][account];
    }
}
```

The `virtual` keyword means "this can be overridden." The `override` keyword means "I'm overriding a virtual function." The compiler enforces that all `virtual` functions without a body are implemented somewhere in the inheritance chain.

GX Protocol doesn't use abstract contracts much because the Diamond Pattern replaces the need for inheritance-based code sharing. Libraries (`LibAccessControl`, `LibAdmin`, etc.) serve the same purpose more safely.

### 10.4 The `super` Keyword and Linearization

When multiple contracts define the same function, Solidity uses **C3 linearization** (same algorithm as Python) to determine which version runs:

```solidity
contract A {
    function foo() public virtual returns (string memory) { return "A"; }
}

contract B is A {
    function foo() public virtual override returns (string memory) { return "B"; }
}

contract C is A {
    function foo() public virtual override returns (string memory) { return "C"; }
}

contract D is B, C {
    // Must override because both B and C define foo()
    function foo() public override(B, C) returns (string memory) {
        return super.foo();  // Calls C.foo() — rightmost parent in "is" list wins
    }
}
```

**For GX Protocol**: You won't encounter this complexity because facets don't inherit from each other. Each facet is a standalone contract that delegates to libraries. This avoids the "diamond problem" (the inheritance one, not EIP-2535) entirely.

---

## Part 11: Assembly, Yul & Low-Level EVM

### 11.1 Why GX Protocol Uses Assembly

You've seen this pattern in every library:
```solidity
function layout() internal pure returns (Layout storage s) {
    bytes32 slot = STORAGE_SLOT;
    assembly {
        s.slot := slot
    }
}
```

And in the Diamond proxy:
```solidity
fallback() external payable {
    // ...
    assembly {
        calldatacopy(0, 0, calldatasize())
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
        returndatacopy(0, 0, returndatasize())
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

Assembly (technically "inline Yul") gives direct access to EVM opcodes. It's used when Solidity's high-level syntax can't express what you need.

### 11.2 The Storage Slot Assignment

```solidity
assembly {
    s.slot := slot
}
```

This is the ONLY way to set a storage pointer to an arbitrary slot. Solidity normally assigns slots sequentially (0, 1, 2...). The `s.slot := slot` instruction says "make this storage reference point to slot `slot` instead of wherever Solidity would normally put it."

Breaking it down:
- `s` is a `storage` reference to a `Layout` struct
- `.slot` is a special Yul property that controls which storage slot the reference points to
- `:=` is Yul assignment (different from Solidity's `=`)
- `slot` is the `bytes32` constant (`keccak256("gx.protocol.storage.tokenomics")`)

Without assembly, you'd have no way to implement Diamond Storage — Solidity would always put struct fields at sequential slots starting from 0.

### 11.3 The Diamond Fallback — Opcode by Opcode

Let's decode the Diamond's fallback function from [Diamond.sol](blockchain-besu/contracts/diamond/Diamond.sol):

```solidity
fallback() external payable {
    // Step 1: Get the facet address for this function selector
    LibDiamond.DiamondStorage storage ds;
    bytes32 position = LibDiamond.DIAMOND_STORAGE_POSITION;
    assembly {
        ds.slot := position  // Point to Diamond's routing table
    }
    address facet = ds.selectorToFacetAndPosition[msg.sig].facetAddress;
    require(facet != address(0), "Diamond: Function does not exist");

    // Step 2: Forward the call via delegatecall
    assembly {
        // Copy all calldata (function selector + arguments) to memory position 0
        calldatacopy(0, 0, calldatasize())

        // Execute the facet's code using Diamond's storage
        // delegatecall(gasToForward, targetAddress, inputOffset, inputSize, outputOffset, outputSize)
        let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)

        // Copy the return data to memory position 0
        returndatacopy(0, 0, returndatasize())

        // If delegatecall failed (result=0), revert with the returned error data
        // If succeeded (result=1), return the data
        switch result
        case 0 { revert(0, returndatasize()) }
        default { return(0, returndatasize()) }
    }
}
```

**Step-by-step flow when your backend calls `diamond.transfer("alice", "bob", 1000000)`:**

1. Transaction arrives at Diamond's address
2. `msg.sig` = first 4 bytes of calldata = function selector for `transfer`
3. Diamond looks up `selectorToFacetAndPosition[msg.sig]` → gets TokenomicsFacet's address
4. `calldatacopy` copies the entire call (selector + encoded args) to memory
5. `delegatecall` executes TokenomicsFacet's `transfer` function using Diamond's storage
6. `msg.sender` inside `transfer` is still your backend's address (not the Diamond)
7. `returndatacopy` captures whatever `transfer` returned
8. `return` sends it back to your backend

### 11.4 `msg.sig` vs `msg.data`

```solidity
// When you call: diamond.transfer("alice", "bob", 1000000)
// The raw transaction data looks like:

msg.data = 0x9b80b050                          // bytes4: function selector
         + 0000...0020                          // uint256: offset to "alice" string
         + 0000...0060                          // uint256: offset to "bob" string
         + 0000...000f4240                      // uint256: 1000000 (the amount)
         + 0000...0005 616c696365               // string: "alice" (length 5 + utf8)
         + 0000...0003 626f62                   // string: "bob" (length 3 + utf8)

msg.sig  = 0x9b80b050    // Just the first 4 bytes (the selector)
```

### 11.5 Other Assembly You'll See in GX

**Address code size check** (is this address a contract?):
```solidity
assembly {
    size := extcodesize(account)  // Returns 0 for EOAs, >0 for contracts
}
```

**Efficient revert with custom error data:**
```solidity
assembly {
    let ptr := mload(0x40)           // Get free memory pointer
    mstore(ptr, errorSelector)       // Store 4-byte error selector
    mstore(add(ptr, 4), param1)      // Store first parameter
    revert(ptr, 36)                  // Revert with 4 + 32 bytes
}
```

**You don't need to write assembly** — the existing GX codebase has it where needed. But understanding what these blocks do helps you read the Diamond infrastructure code with confidence.

### 11.6 `receive()` and `payable` — ETH Handling

You'll notice the Diamond has both `fallback() external payable` and `receive() external payable {}`:

```solidity
// Diamond.sol
fallback() external payable { ... }  // Handles function calls (routes to facets)
receive() external payable {}         // Handles plain ETH transfers (no calldata)
```

- **`fallback()`** executes when calldata is present (i.e., a function is being called)
- **`receive()`** executes when ETH is sent with NO calldata (a plain transfer)
- **`payable`** means the function can receive ETH — without it, sending ETH reverts

**For GX Protocol**: Since Besu is permissioned with `gasPrice: 0`, native ETH is irrelevant. The `payable` keyword is present for EIP-2535 compliance, and `msg.value` is always 0. GX units are tracked in `LibTokenomics.Layout.balances`, not as native ETH.

### 11.7 Cross-Facet Calls — How Facets Interact

In Fabric, cross-contract calls used `InvokeChaincode()`. In a Diamond, facets share the same storage, so there are two patterns:

**Pattern 1: Shared storage reads (preferred in GX)**
```solidity
// GovernanceFacet reads admin state directly — no "call" needed
function submitProposal(...) external {
    LibAdmin.enforceNotPaused();  // Reads LibAdmin.Layout directly
    // Both GovernanceFacet and AdminFacet access the SAME Diamond storage
}
```

**Pattern 2: Call through Diamond address (when you need another facet's FUNCTION logic)**
```solidity
// If a facet needs to call another facet's function (rare in GX):
// It would call the Diamond address, which routes to the correct facet
// GX avoids this — the projector handles cross-facet coordination off-chain
```

GX Protocol primarily uses Pattern 1 (shared library storage). For operations that span multiple facets (e.g., velocity tax distribution touching both `TaxAndFeeFacet` and `TokenomicsFacet`), the projector coordinates off-chain by observing events from one facet and calling another.

---

## Part 12: ABI Encoding — `encode` vs `encodePacked`

### 12.1 Why This Matters

GX Protocol uses `abi.encodePacked` throughout for key derivation:
```solidity
bytes32 key = keccak256(abi.encodePacked(fabricUserId));
```

Understanding the difference between `encode` and `encodePacked` prevents subtle hash collision bugs.

### 12.2 `abi.encode` — Padded, Unambiguous

`abi.encode` follows the standard ABI encoding — every value is padded to 32 bytes:

```solidity
abi.encode("abc", "def")
// Result (hex):
// 0000...0020           // offset to first string (32)
// 0000...0060           // offset to second string (96)
// 0000...0003           // length of "abc" (3)
// 6162630000...0000     // "abc" padded to 32 bytes
// 0000...0003           // length of "def" (3)
// 6465660000...0000     // "def" padded to 32 bytes
```

This is **unambiguous** — you can always decode back to the original values. The function `abi.decode(data, (string, string))` would recover "abc" and "def" perfectly.

### 12.3 `abi.encodePacked` — Tight, Compact, Potentially Ambiguous

`abi.encodePacked` concatenates values without padding:

```solidity
abi.encodePacked("abc", "def")
// Result: 0x616263646566  (just "abcdef" — no padding, no length prefixes)
```

**The collision risk:**
```solidity
// These produce IDENTICAL output:
abi.encodePacked("ab", "cdef")   // → 0x616263646566
abi.encodePacked("abc", "def")   // → 0x616263646566
abi.encodePacked("abcd", "ef")   // → 0x616263646566
abi.encodePacked("abcdef")       // → 0x616263646566
```

If you hash these, you get the same hash. This is a hash collision.

### 12.4 Why GX Protocol Uses `encodePacked` Safely

GX uses `encodePacked` for **single-value** key derivation:
```solidity
// SAFE — single value, no collision possible
bytes32 key = keccak256(abi.encodePacked(fabricUserId));
bytes32 poolKey = keccak256(abi.encodePacked(poolId));
bytes32 countryKey = keccak256(abi.encodePacked(countryCode));
```

With a single value, there's nothing to collide with. The risk only arises with multiple dynamic-type values.

**If you ever need to hash multiple values, use `abi.encode`:**
```solidity
// SAFE — padded encoding prevents collisions
bytes32 compositeKey = keccak256(abi.encode(fabricUserId, orgId));

// DANGEROUS — could collide with different splits
bytes32 compositeKey = keccak256(abi.encodePacked(fabricUserId, orgId));
```

### 12.5 Key Derivation Consistency — Solidity ↔ TypeScript

The key derivation MUST be byte-identical between Solidity and your backend:

```solidity
// Solidity (on-chain)
bytes32 key = keccak256(abi.encodePacked("MY 506 ABI832 0PTNY 5758"));
```

```typescript
// TypeScript (backend — ethers.js)
const key = ethers.keccak256(ethers.toUtf8Bytes("MY 506 ABI832 0PTNY 5758"));
```

Both produce the same `bytes32`. This is documented in [LibTokenomics.sol:171-174](blockchain-besu/contracts/libraries/LibTokenomics.sol#L171-L174):
```
// MUST be byte-for-byte identical to TypeScript:
//   ethers.keccak256(ethers.toUtf8Bytes(fabricUserId))
```

If these ever diverge, your projector will look up the wrong participant and corrupt data.

---

## Part 13: Constructor vs Initializer Pattern

### 13.1 Why Constructors Don't Work with Diamonds

In normal Solidity, a constructor runs once when the contract is deployed:

```solidity
contract SimpleToken {
    string public name;
    uint256 public totalSupply;

    constructor(string memory _name, uint256 _supply) {
        name = _name;
        totalSupply = _supply;
    }
}
```

But in a Diamond, facets are deployed as **logic contracts** — their constructor runs in THEIR OWN storage context, not the Diamond's. Since all state lives in the Diamond, constructor-set values are invisible to the Diamond.

```
Deploy TokenomicsFacet:
  constructor sets name = "GX" in TokenomicsFacet's storage (slot 0)
  → But delegatecall reads from Diamond's storage (slot 0)
  → Diamond's slot 0 is still empty
  → name = "" when accessed through Diamond
```

### 13.2 The Initializer Pattern — GX Protocol's Approach

Instead of constructors, each facet has an `initialize` function that sets up state in the Diamond's storage:

```solidity
// TokenomicsFacet.sol
function initializeTokenomics(
    string calldata tokenName,
    string calldata tokenSymbol
) external {
    _enforceRole(LibAccessControl.PROTOCOL_ADMIN);         // Access control
    LibTokenomics.Layout storage s = LibTokenomics.layout(); // Diamond storage
    if (s.initialized) revert GX_Tokenomics_AlreadyInitialized(); // One-time guard

    s.name            = tokenName;
    s.symbol          = tokenSymbol;
    s.storageVersion  = 1;
    s.initialized     = true;    // Prevents re-initialization
}
```

Key properties:
1. **Called AFTER deployment** — not during construction
2. **Writes to Diamond storage** — via `LibTokenomics.layout()`, not local state
3. **One-time guard** — `initialized` flag prevents re-initialization
4. **Access controlled** — only `PROTOCOL_ADMIN` can initialize

### 13.3 The Deployment Sequence

This is how GX Protocol bootstraps the Diamond — from your actual [deploy.ts](blockchain-besu/scripts/deploy.ts):

```
Step 1: Deploy DiamondCutFacet (standalone contract)
Step 2: Deploy Diamond proxy (constructor wires DiamondCutFacet)
Step 3: Deploy all other facets (standalone contracts)
Step 4: DiamondCut — register all facet selectors with the Diamond
Step 5: Call initialize() on each facet THROUGH the Diamond
```

```typescript
// Step 1-2: Deploy infrastructure
const diamondCutFacet = await DiamondCutFacet.deploy();
const diamond = await Diamond.deploy(owner.address, await diamondCutFacet.getAddress());

// Step 3: Deploy business facets
const facets = ["DiamondLoupeFacet", "AccessControlFacet", "AdminFacet", ...];
const cuts = [];
for (const name of facets) {
    const facet = await ethers.getContractFactory(name).then(f => f.deploy());
    cuts.push({
        facetAddress: await facet.getAddress(),
        action: 0,  // Add
        functionSelectors: getSelectors(facet),
    });
}

// Step 4: Register all selectors
const cut = await ethers.getContractAt("DiamondCutFacet", diamondAddress);
await cut.diamondCut(cuts, ethers.ZeroAddress, "0x");

// Step 5: Initialize through Diamond
const tokenomics = await ethers.getContractAt("TokenomicsFacet", diamondAddress);
await tokenomics.initializeTokenomics("GX Protocol", "GX");
//                                    ↑ This writes to DIAMOND's storage
```

### 13.4 The Diamond Constructor — The Only Real Constructor

The Diamond itself has a constructor — it's the one contract that IS deployed directly (not delegated to):

```solidity
// Diamond.sol — from your actual codebase
contract Diamond {
    constructor(address _contractOwner, address _diamondCutFacet) payable {
        // Set owner in Diamond storage
        LibDiamond.setContractOwner(_contractOwner);

        // Wire up the DiamondCut function selector
        IDiamondCut.FacetCut[] memory cut = new IDiamondCut.FacetCut[](1);
        bytes4[] memory functionSelectors = new bytes4[](1);
        functionSelectors[0] = IDiamondCut.diamondCut.selector;
        cut[0] = IDiamondCut.FacetCut({
            facetAddress: _diamondCutFacet,
            action: IDiamondCut.FacetCutAction.Add,
            functionSelectors: functionSelectors
        });
        LibDiamond.diamondCut(cut, address(0), "");
    }
}
```

This is the chicken-and-egg solution: the Diamond needs DiamondCut to add facets, but DiamondCut is itself a facet. So the constructor manually wires the first facet, and then DiamondCut can add all the others.

### 13.5 `_init` and `_calldata` — DiamondCut Initialization Hook

When you do a DiamondCut, you can optionally pass an initialization contract and calldata:

```solidity
function diamondCut(
    FacetCut[] calldata _diamondCut,
    address _init,        // Contract with initialization logic
    bytes calldata _calldata   // Encoded function call on _init
) external;
```

This runs `_init.delegatecall(_calldata)` AFTER the cut, allowing atomic "add facet + initialize" in one transaction:

```typescript
// Add TokenomicsFacet AND initialize in one atomic operation
const initData = tokenomicsFacet.interface.encodeFunctionData(
    "initializeTokenomics", ["GX Protocol", "GX"]
);
await cut.diamondCut(
    [{ facetAddress: tokenomicsAddr, action: 0, selectors: [...] }],
    tokenomicsAddr,    // _init = the facet itself
    initData           // _calldata = encoded initializeTokenomics call
);
```

If anything fails (cut OR init), the entire transaction reverts — no half-initialized state.

---

## Part 14: Gas Optimization Patterns

### 14.1 Storage Packing — Fitting Multiple Values in One Slot

Each storage slot is 32 bytes. Solidity packs consecutive small types into a single slot:

```solidity
// UNOPTIMIZED — 3 slots (96 bytes of storage)
struct Bad {
    uint256 amount;     // Slot 0 (full 32 bytes)
    bool active;        // Slot 1 (wastes 31 bytes!)
    uint256 timestamp;  // Slot 2 (full 32 bytes)
}

// BETTER ORDERED — 3 slots, but bool is at end for future packing
struct Good {
    uint256 amount;     // Slot 0 (full 32 bytes)
    uint256 timestamp;  // Slot 1 (full 32 bytes)
    bool active;        // Slot 2 (1 byte — future small fields can pack here)
}

// BEST — packs bool + uint8 + address into one slot
struct Best {
    uint256 amount;     // Slot 0
    uint256 timestamp;  // Slot 1
    bool active;        // Slot 2 (1 byte)
    uint8 status;       // Slot 2 (1 byte — packs with active!)
    address owner;      // Slot 2 (20 bytes — packs with both!)
    // Total: 22 bytes out of 32 used in slot 2
}
```

**GX example** — from `LibTokenomics.Layout`:
```solidity
struct Layout {
    bool initialized;       // 1 byte  ─┐
    // gap                              │ Slot N (31 bytes wasted — but that's OK
    //                                  ─┘ because next field is uint256)
    string name;            // Slot N+1 (string pointer)
    string symbol;          // Slot N+2
    uint256 totalMinted;    // Slot N+3
    uint256 totalBurned;    // Slot N+4
    // ...
}
```

The packing isn't aggressive here because Diamond storage uses hash-derived slots where packing savings are minor compared to code clarity. But knowing this matters when you design new structs.

### 14.2 `calldata` vs `memory` for Parameters

```solidity
// CHEAPER — calldata is read-only, no copy needed
function transfer(string calldata fromId, string calldata toId) external { ... }

// MORE EXPENSIVE — memory copies the entire string from calldata
function transfer(string memory fromId, string memory toId) external { ... }
```

Rule: Use `calldata` for `external` function parameters you don't need to modify. Use `memory` only when you need to construct or modify strings inside the function.

GX Protocol uses `calldata` on most external functions (e.g., `initializeTokenomics(string calldata tokenName, string calldata tokenSymbol)`) and `memory` when the function signature was inherited from an interface or when internal manipulation is needed.

### 14.3 `unchecked` Blocks — Skipping Overflow Checks

Since Solidity 0.8, every arithmetic operation includes an overflow check (~3 gas per operation). When you KNOW overflow is impossible, you can skip it:

```solidity
// With check (default) — ~5 gas
uint256 newBalance = balance + amount;  // Reverts if overflows

// Without check — ~2 gas
unchecked {
    uint256 newBalance = balance + amount;  // Won't revert on overflow
}
```

**When it's safe to use `unchecked`:**
```solidity
// Safe: we just checked balance >= amount, so subtraction can't underflow
require(balance >= amount, "Insufficient");
unchecked {
    balances[from] = balance - amount;
}

// Safe: loop counter can't overflow uint256 in practice
for (uint256 i = 0; i < length;) {
    // ... process item ...
    unchecked { i++; }  // Saves ~3 gas per iteration
}
```

GX Protocol uses this sparingly — correctness over gas savings in financial code.

### 14.4 Short-Circuiting Reads

Reading storage is 2100 gas. If you read the same slot multiple times, cache it in memory:

```solidity
// BAD — reads storage 3 times (6300 gas for SLOADs)
function check(bytes32 key) external view {
    if (balances[key] > 0) {           // SLOAD 1
        uint256 fee = balances[key] / 100;  // SLOAD 2
        emit Balance(balances[key]);        // SLOAD 3
    }
}

// GOOD — reads storage once (2100 gas)
function check(bytes32 key) external view {
    uint256 bal = balances[key];       // SLOAD 1 (cached in memory)
    if (bal > 0) {
        uint256 fee = bal / 100;       // memory read (3 gas)
        emit Balance(bal);             // memory read (3 gas)
    }
}
```

GX Protocol follows this pattern — see how `TokenomicsFacet` reads the Layout once and operates on the `s` reference throughout each function.

### 14.5 The 24KB Contract Size Limit

Ethereum has a hard limit: deployed contract bytecode cannot exceed **24,576 bytes** (24KB). This is called the Spurious Dragon limit (EIP-170).

This is actually ONE REASON the Diamond Pattern exists. A single monolithic contract with 90+ functions would easily exceed 24KB. By splitting into facets:

```
TokenomicsFacet:     ~15KB (21 functions)
AdminFacet:           ~4KB (11 functions)
GovernanceFacet:      ~8KB (7 functions)
OrganizationFacet:   ~12KB (13 functions)
// Each facet stays under 24KB independently
```

The Diamond proxy itself is tiny (~2KB) — it's just the fallback router.

If a single facet grows too large, split it into two facets. GX already did this:
- `TokenomicsFacet` (core operations) + `TokenomicsConfigFacet` (configuration)
- `GovernmentFacet` + `GovernmentLendingFacet` + `GovernmentTransitionFacet`

---

## Part 15: Testing with Hardhat

### 15.1 The Test Framework

GX Protocol uses Hardhat + Chai + ethers.js for testing. From your actual test at [test/SPEC04.test.ts](blockchain-besu/test/SPEC04.test.ts):

```
Fabric Testing                      Hardhat Testing
──────────────                      ───────────────
go test -v                     →    npx hardhat test
testify/assert                 →    chai expect
mock.TransactionContext        →    ethers.getSigners()
stub.GetStateReturns(...)      →    (real contract execution — no mocks!)
```

The critical difference: **Hardhat tests run against a real EVM**. No mocks for state, no simulated endorsement. Your contract code executes exactly as it would on Besu.

### 15.2 Test Structure — From Your Codebase

```typescript
import { expect } from "chai";
import { ethers } from "hardhat";

describe("SPEC-04: Diamond Foundation + Identity + Tokenomics", function () {
    let diamond: any;
    let tokenomics: any;
    let owner: any;
    let user1: any;

    // Helper: convert GX units to Qirat
    const QIRAT = 1_000_000n;
    const gx = (n: number) => BigInt(n) * QIRAT;

    // Role constants — MUST match Solidity exactly
    const PROTOCOL_ADMIN = ethers.keccak256(ethers.toUtf8Bytes("PROTOCOL_ADMIN"));
    const TREASURY_ROLE  = ethers.keccak256(ethers.toUtf8Bytes("TREASURY_ROLE"));

    beforeEach(async function () {
        // 1. Get test accounts (Hardhat auto-generates 20 funded accounts)
        [owner, user1] = await ethers.getSigners();

        // 2. Deploy Diamond + all facets (full deployment for each test)
        const diamondCutFacet = await ethers.getContractFactory("DiamondCutFacet")
            .then(f => f.deploy());
        const diamond = await ethers.getContractFactory("Diamond")
            .then(f => f.deploy(owner.address, await diamondCutFacet.getAddress()));

        // 3. Deploy and register facets via DiamondCut
        // ... (see SPEC04.test.ts for full deployment)

        // 4. Get typed contract references AT Diamond's address
        tokenomics = await ethers.getContractAt("TokenomicsFacet", diamondAddr);

        // 5. Initialize
        await tokenomics.initializeTokenomics("GX Protocol", "GX");
    });

    it("should mint tokens with TREASURY_ROLE", async function () {
        await tokenomics.connect(treasuryAdmin).mint(
            "USER_001", gx(1000), "Genesis allocation"
        );
        expect(await tokenomics.getBalance("USER_001")).to.equal(gx(1000));
    });

    it("should revert on unauthorized mint", async function () {
        await expect(
            tokenomics.connect(user1).mint("USER_001", gx(1000), "Unauthorized")
        ).to.be.reverted;
    });
});
```

### 15.3 Key Testing Patterns

**Testing custom error reverts:**
```typescript
// Test that a specific custom error is thrown
await expect(
    tokenomics.connect(user1).mint("USER_001", gx(1000), "Unauthorized")
).to.be.revertedWithCustomError(tokenomics, "GX_Access_Unauthorized");

// With specific parameters
await expect(
    tokenomics.mint("USER_001", 0, "Zero amount")
).to.be.revertedWithCustomError(tokenomics, "GX_Tokenomics_ZeroAmount");
```

**Testing events:**
```typescript
// Verify event emission with specific parameters
await expect(
    tokenomics.mint("USER_001", gx(1000), "Genesis")
).to.emit(tokenomics, "MintEvent")
 .withArgs(
     ethers.keccak256(ethers.toUtf8Bytes("USER_001")),  // participantKey (indexed)
     "USER_001",                                         // recipientId
     gx(1000),                                        // amountQirat
     "Genesis",                                          // remark
     anyValue                                            // timestamp (any uint256)
 );
```

**Testing access control:**
```typescript
// Verify role-based access
it("should reject non-admin pause", async function () {
    await expect(
        admin.connect(user1).pauseSystem("maintenance")
    ).to.be.revertedWithCustomError(admin, "GX_Access_Unauthorized")
     .withArgs(user1.address, GX_SUPER_ADMIN);
});
```

**Time manipulation (for velocity tax testing):**
```typescript
// Advance blockchain time by 360 days
await ethers.provider.send("evm_increaseTime", [360 * 24 * 60 * 60]);
await ethers.provider.send("evm_mine", []);

// Now velocity tax should be applicable
const result = await taxAndFee.checkVelocityTaxEligibility("USER_001", gx(500));
expect(result.eligible).to.be.true;
```

**Snapshot and revert (fast test isolation):**
```typescript
let snapshotId: string;

beforeEach(async function () {
    snapshotId = await ethers.provider.send("evm_snapshot", []);
});

afterEach(async function () {
    await ethers.provider.send("evm_revert", [snapshotId]);
});
// Each test starts from the same state — faster than full redeployment
```

### 15.4 Running Tests

```bash
cd blockchain-besu

# Run all tests
npx hardhat test

# Run specific test file
npx hardhat test test/SPEC04.test.ts

# Run with gas reporting
REPORT_GAS=true npx hardhat test

# Run with verbose output
npx hardhat test --verbose

# Compile contracts
npx hardhat compile

# Start local EVM node (for manual testing)
npx hardhat node
```

### 15.5 Helper Function: `getSelectors`

Every test needs to extract function selectors from facets for DiamondCut:

```typescript
function getSelectors(contract: any): string[] {
    const selectors: string[] = [];
    for (const fragment of contract.interface.fragments) {
        if (fragment.type === "function") {
            selectors.push(contract.interface.getFunction(fragment.name)!.selector);
        }
    }
    return selectors;
}
```

This reads the compiled ABI, finds all function fragments, and extracts their 4-byte selectors. Used in EVERY deployment and test.

---

## Part 16: Deployment Scripts & Upgrade Procedures

### 16.1 Initial Deployment Flow

From your actual [scripts/deploy.ts](blockchain-besu/scripts/deploy.ts), the deployment follows a strict sequence:

```
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: Deploy DiamondCutFacet (standalone)                     │
│ Step 2: Deploy Diamond(owner, diamondCutFacetAddress)           │
│ Step 3: Deploy all business facets (standalone contracts)       │
│ Step 4: DiamondCut — register all selectors at Diamond address  │
│ Step 5: Initialize each domain through Diamond                  │
│ Step 6: Grant roles via AccessControlFacet                      │
│ Step 7: Configure system parameters                             │
│ Step 8: Verify deployment (Loupe introspection)                 │
└─────────────────────────────────────────────────────────────────┘
```

### 16.2 Upgrade Procedure — Adding/Replacing a Facet

When you fix a bug or add a feature:

```typescript
// 1. Deploy the NEW facet version
const NewTokenomicsFacet = await ethers.getContractFactory("TokenomicsFacet");
const newFacet = await NewTokenomicsFacet.deploy();
await newFacet.waitForDeployment();

// 2. Get selectors for the new version
const selectors = getSelectors(newFacet);

// 3. DiamondCut — Replace old selectors with new facet
const cut = await ethers.getContractAt("IDiamondCut", diamondAddress);
await cut.diamondCut(
    [{
        facetAddress: await newFacet.getAddress(),
        action: 1,  // Replace
        functionSelectors: selectors,
    }],
    ethers.ZeroAddress,
    "0x"
);

// 4. Verify via Loupe
const loupe = await ethers.getContractAt("DiamondLoupeFacet", diamondAddress);
const facets = await loupe.facets();
console.log("Facets after upgrade:", facets.length);
```

### 16.3 Selector Clash — The Silent Killer

A **selector clash** occurs when two different functions produce the same 4-byte selector:

```solidity
// These have DIFFERENT names but could theoretically produce the same selector:
function transfer(string memory a, uint256 b) external;  // selector: 0x9b80b050
function someOtherFunc(bytes32 x) external;               // selector: 0x9b80b050 (collision!)
```

The probability is ~1 in 4 billion per pair, but with 90+ functions across 19 facets, collisions become possible.

**Detection**: Hardhat's Diamond plugin and manual checks during DiamondCut. If you try to Add a selector that already exists, `LibDiamond.addFunctions` will revert with "Can't add function that already exists."

**Prevention**: Before any DiamondCut, verify no selector clashes:
```typescript
const allSelectors = new Set<string>();
for (const facet of allFacets) {
    for (const sel of getSelectors(facet)) {
        if (allSelectors.has(sel)) {
            throw new Error(`Selector clash: ${sel}`);
        }
        allSelectors.add(sel);
    }
}
```

### 16.4 Progressive Immutability — GX's Unique Feature

Your [GXDiamondCutFacet.sol](blockchain-besu/contracts/facets/GXDiamondCutFacet.sol) implements a feature unique to GX Protocol — **facet locking**:

```
Announce Lock → 7-day cooling period → Execute Lock → IRREVERSIBLE
```

Once a facet is locked, its functions can NEVER be replaced or removed. This maps directly to the protocol specification's governance evolution:

| Governance Epoch | Diamond State |
|-----------------|---------------|
| Genesis (Years 0-15) | All facets unlocked — rapid iteration |
| Tutelage (Years 15-25) | Core facets (TokenomicsFacet, IdentityFacet) locked |
| Maturity (Year 25+) | Most facets locked — only governance-gated changes |

This is how GX Protocol achieves the protocol specification's promise of "immutable core rules" while allowing early-phase flexibility.

---

## Part 17: Error Handling on the Backend

### 17.1 How Custom Errors Reach Your Backend

When a Solidity function reverts with a custom error:
```solidity
revert GX_Tokenomics_InsufficientBalance(participantKey, required, available);
```

The EVM returns encoded error data:
```
0x[4-byte error selector][32-byte participantKey][32-byte required][32-byte available]
```

Your backend receives this as a transaction revert.

### 17.2 Decoding Errors with ethers.js

```typescript
try {
    const tx = await diamond.transfer(fromId, toId, amount);
    await tx.wait();
} catch (error: any) {
    // ethers.js v6 wraps revert data in error object
    if (error.code === 'CALL_EXCEPTION') {
        // Try to decode the custom error
        const iface = diamond.interface;
        const decodedError = iface.parseError(error.data);

        if (decodedError) {
            console.log('Error name:', decodedError.name);
            // "GX_Tokenomics_InsufficientBalance"

            console.log('Args:', decodedError.args);
            // [participantKey, required, available]

            // Map to your backend's error response
            switch (decodedError.name) {
                case 'GX_Tokenomics_InsufficientBalance':
                    throw new InsufficientBalanceError(
                        decodedError.args.required.toString(),
                        decodedError.args.available.toString()
                    );
                case 'GX_Tokenomics_WalletFrozen':
                    throw new WalletFrozenError(decodedError.args.participantKey);
                case 'GX_Access_Unauthorized':
                    throw new UnauthorizedError();
            }
        }
    }
    throw error;
}
```

### 17.3 Mapping to GX Backend Error Classes

Your Express.js services use domain-typed errors. Map Solidity custom errors to these:

```typescript
// In your outbox-submitter or service layer
const SOLIDITY_TO_DOMAIN_ERROR: Record<string, (args: any) => DomainError> = {
    'GX_Tokenomics_ZeroAmount':
        () => new ValidationError('Transfer amount must be positive'),
    'GX_Tokenomics_InsufficientBalance':
        (args) => new InsufficientBalanceError(args.required, args.available),
    'GX_Tokenomics_WalletFrozen':
        (args) => new WalletFrozenError(args.participantKey),
    'GX_Tokenomics_GenesisAlreadyDistributed':
        (args) => new ConflictError('Genesis already distributed for participant'),
    'GX_Tokenomics_MaxSupplyExceeded':
        (args) => new BusinessRuleError(`Mint would exceed max supply`),
    'GX_Access_Unauthorized':
        (args) => new AuthorizationError(`Account ${args.caller} lacks required role`),
};
```

### 17.4 View Call Errors vs Transaction Errors

**View calls** (read-only, no transaction) revert immediately:
```typescript
// This reverts INSTANTLY — no gas spent, no block mined
try {
    const balance = await diamond.getBalance("NONEXISTENT");
    // Returns 0 (GX getBalance returns 0 for unknown accounts)
} catch (error) {
    // Only if the function itself reverts
}
```

**State-changing calls** have two failure points:
```typescript
// Point 1: Estimation failure — reverts before sending
try {
    const tx = await diamond.transfer(from, to, amount);
    // Point 2: Execution failure — tx mined but reverted
    const receipt = await tx.wait();
    if (receipt.status === 0) {
        // Transaction was mined but execution reverted
    }
} catch (error) {
    // Could be estimation failure OR execution failure
}
```

For GX Protocol's CQRS outbox submitter, you need to handle BOTH — estimate errors (the outbox command is invalid) and execution errors (on-chain state changed between queuing and submission).

---

## Part 18: Event Log Structure at the Byte Level

### 18.1 How Events Are Stored On-Chain

When `TokenomicsFacet` emits:
```solidity
emit TransferEvent(fromKey, toKey, fromId, toId, amount, block.timestamp);
```

The EVM creates a **log entry** in the transaction receipt:

```
Log {
    address:  0xDiamondAddress              // Which contract emitted it
    topics: [                               // Up to 4 topics (32 bytes each)
        0x[keccak256 of event signature],   // Topic 0: event selector
        0x[fromKey],                        // Topic 1: indexed param 1
        0x[toKey],                          // Topic 2: indexed param 2
    ],
    data: abi.encode(fromId, toId, amount, timestamp)  // Non-indexed params
}
```

### 18.2 Topics — The Indexed Fields

```solidity
event TransferEvent(
    bytes32 indexed fromKey,      // → topics[1]
    bytes32 indexed toKey,        // → topics[2]
    string  fromFabricUserId,     // → data (not in topics)
    string  toFabricUserId,       // → data
    uint256 amountQirat,          // → data
    uint256 timestamp             // → data
);
```

- `topics[0]` is ALWAYS the event signature hash: `keccak256("TransferEvent(bytes32,bytes32,string,string,uint256,uint256)")`
- `topics[1..3]` are the `indexed` parameters (max 3 indexed per event)
- Everything else goes into `data` (ABI-encoded, no size limit)

**Why this matters for your projector**: Topics are searchable by Besu nodes. When you subscribe:
```typescript
// Filter by specific fromKey — Besu node only sends matching events
const filter = diamond.filters.TransferEvent(specificFromKey);
```

Besu scans `topics[1]` to match. This is O(1) per block, not O(n) per event — extremely efficient.

### 18.3 Anonymous Events

Events can be `anonymous` — they skip the signature in `topics[0]`, giving you 4 indexed fields instead of 3:

```solidity
event Transfer(
    bytes32 indexed a,
    bytes32 indexed b,
    bytes32 indexed c,
    bytes32 indexed d    // 4 indexed fields possible with anonymous
) anonymous;
```

GX Protocol does NOT use anonymous events because:
1. They can't be filtered by name (no `topics[0]` signature)
2. ethers.js can't auto-parse them
3. Your projector needs event names for routing

### 18.4 How Your Projector Receives Events

```typescript
// Method 1: Real-time subscription (WebSocket)
const provider = new ethers.WebSocketProvider("ws://besu-node:8546");
const diamond = new ethers.Contract(DIAMOND_ADDRESS, ABI, provider);

diamond.on("TransferEvent", (fromKey, toKey, fromId, toId, amount, ts, event) => {
    // event.log contains the raw log entry
    // event.blockNumber, event.transactionHash available
    processTransfer(fromId, toId, amount);
});

// Method 2: Polling historical events (HTTP)
const provider = new ethers.JsonRpcProvider("http://besu-node:8545");
const diamond = new ethers.Contract(DIAMOND_ADDRESS, ABI, provider);

// Get all TransferEvents in blocks 1000-2000
const events = await diamond.queryFilter(
    diamond.filters.TransferEvent(),
    1000,  // fromBlock
    2000   // toBlock
);

for (const event of events) {
    const { fromKey, toKey, fromFabricUserId, toFabricUserId, amountQirat } = event.args;
    processTransfer(fromFabricUserId, toFabricUserId, amountQirat);
}
```

### 18.5 The JSON-RPC Layer

Under the hood, ethers.js calls Besu's JSON-RPC API:

```json
// eth_getLogs — query historical events
{
    "jsonrpc": "2.0",
    "method": "eth_getLogs",
    "params": [{
        "address": "0xDiamondAddress",
        "fromBlock": "0x3E8",
        "toBlock": "0x7D0",
        "topics": [
            "0x[TransferEvent signature hash]",
            "0x[optional fromKey filter]"
        ]
    }]
}

// eth_subscribe — WebSocket real-time subscription
{
    "jsonrpc": "2.0",
    "method": "eth_subscribe",
    "params": ["logs", {
        "address": "0xDiamondAddress",
        "topics": ["0x[TransferEvent signature hash]"]
    }]
}
```

Your projector can use either method. Real-time subscription is lower latency; polling is more resilient to connection drops (you can always replay missed blocks).

---

## Part 19: Libraries — Internal vs Deployed

### 19.1 Two Types of Libraries in Solidity

**Internal libraries** (what GX Protocol uses):
```solidity
library LibTokenomics {
    function participantKey(string memory fabricUserId) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(fabricUserId));
    }
}
```

The `internal` keyword means the library's code is **inlined** into every contract that uses it. When `TokenomicsFacet` calls `LibTokenomics.participantKey("alice")`, the compiler copies the function's bytecode directly into TokenomicsFacet. No external call happens.

**Deployed libraries** (NOT used by GX):
```solidity
library MathLib {
    function sqrt(uint256 x) public pure returns (uint256) { ... }
}
```

The `public` keyword means the library is deployed as a separate contract. Contracts that use it perform a `DELEGATECALL` to the library's address at runtime. This requires "linking" — the library's address must be injected into the contract's bytecode before deployment.

### 19.2 Why GX Uses Internal Libraries

1. **No linking required** — deployment is simpler (no library addresses to manage)
2. **No external call overhead** — inlined code is cheaper (no DELEGATECALL gas cost)
3. **Compatible with Diamond storage** — the `assembly { s.slot := position }` pattern works because the code executes in the calling contract's context
4. **Simpler upgrades** — when you replace a facet, the new facet contains its own copy of library code

The trade-off: every facet includes its own copy of library code, increasing bytecode size. But since each facet stays under 24KB, this isn't a problem.

### 19.3 How Libraries Enable Diamond Storage

The key insight: a library with `internal` functions executes in the calling contract's storage context (because the code is inlined). Combined with the assembly slot trick, this gives each library its own namespaced storage:

```
TokenomicsFacet calls LibTokenomics.layout()
  → Code is inlined into TokenomicsFacet
  → TokenomicsFacet is delegatecalled by Diamond
  → Therefore LibTokenomics.layout() reads from Diamond's storage
  → The assembly trick points to the namespace-specific slot
  → Result: TokenomicsFacet reads/writes Diamond's tokenomics storage
```

This chain — `library (inlined) → facet (delegatecalled) → Diamond (storage owner)` — is the architectural backbone of the entire GX Besu implementation.

---

## Updated Summary: Complete Knowledge Map

### Parts 1-9 (Foundation)
1. EVM mental model — accounts, transactions, gas
2. Besu node — QBFT, genesis.json, permissioning
3. Solidity basics — types, functions, errors, events
4. Storage — slots, mappings, Diamond storage
5. Diamond Pattern — proxy, delegatecall, DiamondCut
6. GX implementation — fabricUserId bridge, access control
7. Security — reentrancy, storage collisions
8. Development tooling — Hardhat, ethers.js, ABI
9. CQRS pipeline — Besu edition

### Parts 10-19 (Advanced)
10. Inheritance & interfaces — why Diamond uses libraries instead
11. Assembly & Yul — slot assignment, fallback decoding
12. ABI encoding — `encode` vs `encodePacked`, collision risks
13. Constructor vs initializer — why Diamond facets need `initialize()`
14. Gas optimization — storage packing, calldata, unchecked, 24KB limit
15. Testing with Hardhat — test structure, custom errors, time manipulation
16. Deployment & upgrades — deploy sequence, DiamondCut, selector clashes, progressive immutability
17. Backend error handling — decoding custom errors, mapping to domain errors
18. Event log structure — topics, data, JSON-RPC, projector integration
19. Libraries — internal vs deployed, how they enable Diamond storage

---

*This document is a complete technical reference for understanding the GX Protocol Besu implementation. Parts 1-9 provide the foundation. Parts 10-19 cover the advanced topics you'll encounter during active development and migration. Revisit sections as needed — the concepts build on each other.*
