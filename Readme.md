# Account Abstraction: The Complete Guide

This is a comprehensive deep-dive into every major contribution to Account Abstraction (AA) on Ethereum. For each entry, you'll find: **what** it is, **why** it was proposed, **how** it works technically, **code/developer experience** changes, and **current status**.

---

## Table of Contents

- [The Core Problem](#the-core-problem)
- [Key Concepts](#key-concepts)
- [2015 - The Vision](#2015---the-vision)
- [2016 - Early Ideas](#2016---early-ideas)
- [2017 - First Concrete Proposals](#2017---first-concrete-proposals)
- [2018 - Gas Abstraction Era](#2018---gas-abstraction-era)
- [2019 - Simplification](#2019---simplification)
- [2020 - Two Competing Paths Emerge](#2020---two-competing-paths-emerge)
- [2021 - ERC-4337: The Game Changer](#2021---erc-4337-the-game-changer)
- [2022 - Refinement and Alternatives](#2022---refinement-and-alternatives)
- [2023 - The 4337 vs 3074 Debate](#2023---the-4337-vs-3074-debate)
- [2024 - EIP-7702: The Compromise](#2024---eip-7702-the-compromise)
- [2025 - Post-7702 Era](#2025---post-7702-era)
- [2026 - Native AA on the Horizon](#2026---native-aa-on-the-horizon)
- [Author Legend](#author-legend)
- [The Big Picture: How It All Connects](#the-big-picture-how-it-all-connects)

---

## The Core Problem

Ethereum has two types of accounts:

1. **EOA (Externally Owned Account)** -- controlled by a private key (your MetaMask wallet).
2. **Contract Account** -- controlled by code (smart contracts).

EOAs are rigid and limited. They can only:
- Sign transactions with one specific algorithm (ECDSA secp256k1)
- Pay gas in ETH only
- Execute one operation per transaction
- Have no programmable logic (no multisig, no recovery, no spending limits)

**Account Abstraction** = making accounts programmable so that the *validation logic* (who can authorize a tx) and *execution logic* (what happens) are both customizable. The goal: every account behaves like a smart contract, with flexible authentication, gas payment, and execution.

---

## Key Concepts

Before diving in, understand these building blocks:

| Concept | What It Means |
|---------|--------------|
| **Validation** | Checking if a transaction is authorized (signature verification, nonce check) |
| **Execution** | Actually performing the transaction's intended action |
| **Gas Abstraction** | Letting someone other than the tx sender pay for gas, or paying in tokens other than ETH |
| **Meta-transaction** | User signs a message, a relayer submits the actual on-chain tx and pays gas |
| **Counterfactual Deployment** | Computing a contract's address before deploying it, so you can send funds to it first |
| **Mempool** | The queue of pending transactions waiting to be included in a block |
| **DoS (Denial of Service)** | Flooding the network with invalid work to waste resources |
| **Bundler** | An off-chain node that packages UserOperations into real transactions (ERC-4337) |
| **Paymaster** | A smart contract that sponsors gas fees for users (ERC-4337) |
| **Invoker** | A smart contract that executes operations on behalf of an EOA (EIP-3074) |

---

## 2015 - The Vision

### EIP-101: Currency and Crypto Abstraction (Vitalik Buterin)

**What:** The most radical AA proposal ever. It proposes stripping Ethereum's protocol down to its bare essentials by removing three things hardcoded into the protocol: **ether balances**, **ECDSA signature verification**, and **nonce/sequence-number incrementing**. All three are pushed up into contract-level logic.

**Why:**
1. **Protocol simplification.** Fewer special cases = fewer bugs in client implementations.
2. **Generality.** Users are locked into ECDSA and sequential nonces. EIP-101 lets any account define its own signature scheme (ed25519, Lamport, ring signatures, multisig, etc.) and its own replay protection.
3. **Ether/token unification.** By making ETH itself a contract (at address `0x0`), ETH and ERC-20 tokens can be treated identically.

**How it works:**

Account structure is reduced to just two fields:
```
account = { code, storage }
// balance and nonce are REMOVED from protocol
```

Transactions are reduced to four fields:
```
tx = { to, startgas, data, code }
// No value, no nonce, no v/r/s signature, no gasprice
```

ETH becomes a contract at address `0x0`. Sending ETH with a call uses a three-step "cheque" mechanism:
1. Call the ETH contract to create a cheque for the amount
2. The callee cashes the cheque
3. If uncashed, the caller revokes it

A "normal" EOA becomes a contract that:
1. Extracts signature, nonce, target, value, data from calldata
2. Calls ECRECOVER precompile itself (at contract level) to verify the signer
3. Checks nonce against its own storage
4. Executes the sub-call
5. Pays the miner for gas via the ETH contract

**Developer experience:**
- Every wallet becomes a contract. Developers must deploy account code that handles sig verification and nonce management.
- `msg.value` disappears entirely. All contracts accepting ETH must rewrite to use cheques. Massive breaking change.
- Custom crypto is first-class. Multisig, Lamport sigs, ring signatures -- just implement whatever verification in your account contract.
- Gas payment is contract-level, making gas sponsorship and meta-transactions trivially expressible.

**Status:** Never implemented. Too radical and breaking. But the vision influenced every subsequent proposal.

---

### "On Abstraction" (Vitalik Buterin)

**What:** A philosophical essay arguing that Ethereum should abstract away implementation details at every layer -- not just accounts, but storage, computation, and consensus.

**Why:** Sets the intellectual foundation. The argument: protocols should provide the minimal possible set of hardcoded rules. Everything else should be customizable by users and developers.

**Key insight:** Hardcoding decisions (like "only ECDSA" or "only ETH for gas") limits the system's long-term potential and creates lock-in.

---

## 2016 - Early Ideas

### "Spending Gas from a Contract's Balance" (Buterin)

**What:** An early proposal that contracts should be able to pay for their own gas, rather than requiring an EOA to initiate and fund every transaction.

**Why:** Smart contract wallets need an EOA "helper" to relay transactions and pay gas. This is clunky. If contracts could pay gas directly, they could be first-class transaction originators.

**Key insight:** This is the seed of gas abstraction. Every later proposal (meta-transactions, paymasters, PAYGAS opcode) tries to solve this same problem in different ways.

---

## 2017 - First Concrete Proposals

### EIP-86: Abstraction of Transaction Origin and Signature (Vitalik Buterin)

**What:** A more pragmatic, incremental version of EIP-101. Introduces "null-sender" transactions that bypass the protocol's built-in ECDSA and nonce checks, allowing contract-based accounts to define their own authentication.

**Why:**
1. **ECDSA lock-in is a liability.** Prevents multisig wallets from paying their own gas, prevents privacy-preserving withdrawals (you need ETH to pay gas, which links your identity), blocks post-quantum signatures.
2. **Path toward "all accounts are contracts."**

**How it works:**

A new class of transaction where signature fields are `(v=CHAIN_ID, r=0, s=0)`. These are treated as valid, and the sender is set to a special `NULL_SENDER` address (essentially `0x0...0`). Constraints:
- `gasprice = 0`, `nonce = 0`, `value = 0`
- `NULL_SENDER` nonce is never incremented
- Anyone can submit such a transaction -- no private key controls `NULL_SENDER`
- The target contract is responsible for ALL authentication

Also introduces **CREATE2** opcode for deterministic contract addresses:
```
address = sha3(sender + salt + sha3(init_code)) % 2**160
```
This enables **counterfactual instantiation** -- compute an account's address before deployment, send funds to it, deploy later.

**Reference "forwarding contract" pattern:**
```
1. Accept call from entry point
2. Expect calldata: [signature, nonce, to, value, gasprice, data]
3. Verify signature (whatever scheme the contract implements)
4. Check and increment nonce (in own storage)
5. Pay miner (gasprice * gas_used)
6. Forward call to the target address
```

**DoS concern:** Miners waste at most ~50,000 gas verifying a null-sender transaction. If verification fails, the miner eats that cost.

**Developer experience:**
- Account contracts must be written and deployed with custom verification logic.
- Deterministic addresses via CREATE2 enable precomputing contract addresses (later shipped as EIP-1014 in Constantinople, Feb 2019).
- Multisig wallets simplify: the contract itself holds ETH and pays gas.
- Privacy-preserving withdrawals: mixer contracts can pay gas from withdrawn funds.
- Breaking change: contracts relying on `tx.origin == msg.sender` as an EOA check would break.

**Status:** Draft/Stagnant. CREATE2 was extracted and shipped separately as EIP-1014.

---

### "Tradeoffs in Account Abstraction Proposals" (Vitalik Buterin)

**What:** Analysis of the design space for AA, examining the tradeoffs between different approaches.

**Key tradeoffs identified:**
- Protocol simplicity vs. expressiveness
- DoS protection vs. validation flexibility
- Breaking changes vs. backward compatibility
- Single-tenant (one user per contract) vs. multi-tenant (shared contracts like Tornado Cash)

---

### "Account Abstraction, Miner Data and Auto-Updating Witnesses" (Jeff Coleman)

**What:** Explores how miners/validators interact with AA transactions, particularly around:
- How miners evaluate which AA transactions to include
- Witness data (proofs that a transaction is valid)
- Auto-updating witnesses that remain valid across blocks

---

## 2018 - Gas Abstraction Era

### EIP-859: Account Abstraction for Main Chain (Vitalik Buterin)

**What:** Takes EIP-86's ideas and adapts them specifically for the Ethereum mainchain. Key compromise vs. EIP-86: **mandatory protocol-level nonces are added back** to preserve the guarantee that each transaction appears at most once. Also introduces the **PAYGAS opcode**.

**Why:**
1. EIP-86's NULL_SENDER approach had DoS problems -- no cheap way to reject duplicate/replayed transactions.
2. The mainchain needs backward-compatible AA with minimal consensus risk.
3. Need a clean separation between verification and execution phases.

**How it works:**

Transaction format:
```
tx = { chain_id, target, nonce, data, start_gas, code, salt }
// No gasprice, value, or v/r/s -- all handled by account contract
```

Execution flow:
1. **Nonce check (protocol level):** Assert `tx.nonce == target_account.nonce`. Increment by 1.
2. **Account creation (if needed):** If target has no code, deploy using `code` and `salt` fields (CREATE2-style).
3. **Call the account contract:** Send message with `tx.data`. Contract runs verification logic.
4. **PAYGAS opcode:** When contract reaches PAYGAS, it signals verification succeeded and commits to paying gas.
5. **Execution phase:** After PAYGAS, the actual operation runs.
6. **Validity rule:** If exception occurs and PAYGAS was never called, the tx is INVALID (miners must not include it). If exception occurs AFTER PAYGAS, the tx is still valid -- gas is still paid.

**The PAYGAS opcode in detail:**
```
PAYGAS(gasprice):
  1. If already called, no-op (push 0)
  2. Subtract gasprice * start_gas from contract balance
  3. If insufficient funds, push 0
  4. Set PAYGAS_CALLED = true, PAYGAS_GASPRICE = gasprice
  5. Push 1 (success)

At end of tx: refund unused gas at PAYGAS_GASPRICE
Miner receives: PAYGAS_GASPRICE * gas_used
```

**State access restrictions (anti-DoS):** During verification (before PAYGAS), the contract can only read/write its OWN storage. This ensures a valid tx in the mempool can only be invalidated by another tx to the SAME account, not by unrelated state changes. This was a critical insight that survived into ERC-4337.

**Miner strategy:** Miners set a private `CHECK_LIMIT` (e.g., 200,000 gas). Execute the tx for at most CHECK_LIMIT gas. If PAYGAS is reached, accept the tx. If not, reject it.

**Developer experience:**
- Account contracts have a clear two-phase pattern: verification (before PAYGAS) then execution (after PAYGAS).
- Nonces remain familiar (unlike EIP-86).
- Gas pricing moves into the contract via PAYGAS -- contracts can even pay in tokens by swapping on a DEX during verification.
- The PAYGAS pattern foreshadows ERC-4337's `validateUserOp` / `execute` split.

**Status:** Never shipped. But ideas live on in EIP-2938 and ERC-4337.

---

### EIP-1613: Gas Stations Network (Yoav Weiss)

**What:** A decentralized relay network enabling "collect-call" transactions where the contract or a third party pays gas instead of the user. Requires **no protocol changes** -- everything via smart contracts and off-chain relays.

**Why:** Three adoption barriers:
1. Users must acquire ETH before using ANY dapp
2. Dapp UX suffers when users manage gas
3. No revenue model for relay operators

**How it works:**

Three core components:

**1. RelayHub Contract** (singleton mediator):
- Maintains relay registry
- Holds ETH stakes from relay operators and prepayments from dapp contracts
- Provides trusted `msg.sender`/`msg.data` to recipient contracts
- Compensates relays, penalizes malicious relays

**2. RelayRecipient Contracts** (dapps):
- Replace `msg.sender` with `getSender()`
- Implement three lifecycle functions:
  ```solidity
  function acceptRelayedCall()  // validate: should we accept this meta-tx?
  function preRelayedCall()     // setup before execution
  function postRelayedCall()    // cleanup after execution
  ```

**3. Relay Nodes** (off-chain operators):
- Listen for user requests
- Maintain funded hot wallets
- Submit transactions to RelayHub
- Charge configurable fee multipliers

**Transaction flow (7 stages):**
1. Relay registers with RelayHub, staking ETH
2. User selects a relay based on fees/reputation
3. User prepares a meta-transaction: `[address, recipient, encoded_call, relay_fee, gas_params, nonce, signature]`
4. Relay calls `RelayHub.canRelay()` (which calls recipient's `acceptRelayedCall()`) as a view function
5. Relay wraps the transaction, signs with its own key, submits on-chain, returns signed tx to user
6. User validates the wrapped transaction and can submit independently as backup
7. RelayHub executes: verifies relay + signature, calls `acceptRelayedCall`, `preRelayedCall`, the actual function, `postRelayedCall`, transfers payment from recipient deposit to relay

**Penalty system:** `penalizeRepeatedNonce()` confiscates 100% of relay's stake for nonce-reuse attacks (50% to reporter, 50% burned).

**Developer experience:**
- Minimal code changes: replace `msg.sender`/`msg.data` references, add three lifecycle functions.
- Maintain ETH deposit balance in RelayHub.
- No protocol upgrades needed. Works on existing Ethereum.
- Users need ZERO ETH to interact with dapps.

**Status:** Evolved into GSN v2/v3. Concepts live on in ERC-4337's Paymaster design.

---

### "A Recap of Where We Are at on Account Abstraction" (Vitalik Buterin)

**What:** Summary of progress on AA as of 2018. Maps the landscape: EIP-86, EIP-859, gas stations, and how they relate.

**Key takeaway:** There are two paths -- protocol-level changes (harder but cleaner) and application-level workarounds (deployable now but messier). The community is still searching for the right balance.

---

### "A New Account Type in Abstraction" (Vitalik Buterin)

**What:** Proposal for a new account type specifically designed for AA, sitting between EOAs and full contract accounts. Explores the design space of what "minimal" account abstraction could look like.

---

### "Application-Layer Account Abstraction" (Jeff Coleman)

**What:** The case for doing AA entirely at the application layer without ANY protocol changes. The argument: you can achieve most AA benefits with meta-transactions, relayers, and smart contract wallets.

**Key insight:** Protocol changes are slow and risky. Application-layer solutions can iterate faster. This philosophy eventually won with ERC-4337.

---

### "Layer 2 Gas Payment Abstraction" (Vitalik Buterin)

**What:** Extends gas abstraction concepts to Layer 2 systems. Since L2s have different fee models and execution environments, gas abstraction needs to work differently there.

---

## 2019 - Simplification

### "Maximally Simple Account Abstraction Without Gas Refunds" (Vitalik Buterin)

**What:** A stripped-down AA proposal that simplifies previous designs by removing gas refund mechanisms. The idea: make AA as simple as possible to reduce implementation risk.

**Key insight:** The hardest part of AA is gas -- who pays, how to prevent DoS, how to handle refunds. By removing refunds, the design becomes much simpler at the cost of some efficiency.

**Why this matters:** This thinking directly influenced EIP-2938's PAYGAS design, where gas is pre-paid and refunded at the end (a simpler model than per-opcode accounting).

---

### "1-800-Ethereum: Gas Stations Network for Toll-Free Transactions"

**What:** A practical guide and documentation for the Gas Stations Network (GSN) implementation. Explains how to build dapps where users don't need ETH.

**Key concepts:**
- Meta-transactions: user signs intent, relayer submits on-chain
- EIP-712 typed structured data signing
- ERC-2771: standard for extracting the real sender from relayed calls
- Trusted Forwarder pattern: a minimal contract that verifies signatures

**How meta-transactions work:**
```
1. User signs EIP-712 message: { target, function_call, nonce, params }
2. Relayer receives signed message
3. Relayer submits real tx calling TrustedForwarder
4. Forwarder verifies signature, appends real sender to calldata
5. Target contract uses ERC-2771 to extract real sender from last 20 bytes of msg.data
```

**Relationship to AA:** Meta-transactions are a precursor. They solve gas sponsorship but require every contract to explicitly support ERC-2771. AA solves this universally.

---

## 2020 - Two Competing Paths Emerge

### EIP-2938: Account Abstraction (Vitalik Buterin, Quilt Team)

**What:** Protocol-level AA. Allows a smart contract to be the top-level account that pays fees and initiates transaction execution. Extends EIP-86's vision with lessons learned from EIP-859.

**Why:**
- Only ECDSA signatures for EOAs -- no Schnorr, BLS, post-quantum, or multisig at protocol level
- Only the tx sender (an EOA) can pay gas in ETH
- No programmable validation (spending rules, timelocks, social recovery)
- Smart contract wallets are second-class citizens requiring EOA relayers

**How it works:**

New transaction type (EIP-2718 envelope):
```
tx = { nonce, target, data }
// No to, gas_price, gas_limit, or signature fields
// The contract itself determines gas pricing and validation
```

Two new opcodes:

**NONCE opcode:** Exposes the contract's nonce as `msg.nonce`. Used to verify transaction ordering and prevent replays.

**PAYGAS opcode:** The critical checkpoint. Takes `gasPrice` and `gasLimit` as arguments. Divides execution into two phases:
- **Before PAYGAS (Verification):** Validate signature, check nonce, authenticate. Restricted opcodes.
- **After PAYGAS (Execution):** Execute the actual operation. If this reverts, the revert only rolls back to PAYGAS -- gas is still consumed and paid.

```solidity
// Conceptual EIP-2938 account contract
account contract Wallet {
    address owner;

    function execute(
        uint gasPrice,
        uint gasLimit,
        address to,
        uint amount,
        bytes calldata payload
    ) external {
        // === VERIFICATION PHASE ===
        bytes32 hash = keccak256(abi.encodePacked(
            address(this), msg.nonce, gasPrice, gasLimit, to, amount, payload
        ));
        require(ecrecover(hash, v, r, s) == owner, "Invalid signature");

        // === PAYGAS CHECKPOINT ===
        paygas(gasPrice, gasLimit);
        // Contract is now committed to paying for gas

        // === EXECUTION PHASE ===
        (bool success,) = to.call{value: amount}(payload);
        require(success);
    }
}
```

**Verification phase restrictions (anti-DoS):**
Banned during verification: `BLOCKHASH`, `COINBASE`, `TIMESTAMP`, `NUMBER`, `DIFFICULTY`, `GASLIMIT`, `BALANCE`, external calls/creates to anything other than the target or precompiles.

The key insight: if validation can only access the account's own storage, then a valid tx in the mempool can only be invalidated by another tx to the SAME account. This bounds the DoS surface.

**Two-tier design:**
- **Single-tenant:** One contract per user (wallets). Simpler mempool rules.
- **Multi-tenant:** Shared contracts (Tornado Cash, Uniswap). More complex.

**Developer experience:**
- `account contract` keyword (new Solidity construct)
- `msg.nonce` for reading the NONCE opcode
- `paygas()` for the PAYGAS opcode
- `tx.origin` is set to `0xffff...ff` (entry point), not a real EOA

**Status:** Stagnant/Superseded. Required consensus changes that were politically infeasible during The Merge era. Effort shifted to ERC-4337.

---

### EIP-3074: AUTH and AUTHCALL Opcodes (Quilt Team)

**What:** Introduces two new EVM opcodes -- **AUTH** and **AUTHCALL** -- that allow an EOA to delegate its transaction authority to a smart contract (called an "invoker"). The invoker can then make calls where `msg.sender` appears as the original EOA.

**Why:**
- EOAs are limited to one operation per transaction, cannot batch, cannot sponsor gas
- Meta-transaction relayers require `_msgSender()` adoption across all contracts
- Smart contract wallets require users to migrate to new addresses
- EIP-3074 gives EOAs smart-account-like features without address migration

**How it works:**

**AUTH opcode (0xf6):**
Takes four stack inputs: `commit`, `yParity`, `r`, `s`.

```
signed_message = keccak256(
    MAGIC        // 0x03, prevents collision with EIP-191 (0x19) and EIP-712
    || chainId   // prevents cross-chain replay
    || nonce     // EOA's current nonce (prevents replay after any EOA tx)
    || invokerAddress  // binds authorization to specific invoker
    || commit    // hash of values the invoker will validate
)
```

The opcode recovers the signer via ecrecover. If valid, sets a frame-local `authorized` variable to that address. This variable only exists within the current execution frame.

**AUTHCALL opcode (0xf7):**
Identical to CALL except:
- `msg.sender` in the called contract is set to `authorized` instead of the invoker's address
- Value is deducted from the **invoker's** balance, not the EOA's
- Requires `authorized` to be set (AUTH must have been called first)

**Example flow:**
```
1. User wants to: approve USDC + swap on Uniswap (two operations, one tx)
2. User computes: commit = keccak256(abi.encode(calls[]))
3. User signs: keccak256(MAGIC || chainId || nonce || invokerAddress || commit)
4. Anyone (user, relayer, sponsor) submits tx calling the invoker contract
5. Invoker: recomputes commit from provided params, calls AUTH
6. AUTH: sets authorized = EOA's address
7. Invoker: iterates through calls using AUTHCALL
8. Each target sees msg.sender = the EOA (not the invoker)
```

```solidity
// Conceptual invoker contract
contract BatchInvoker {
    function execute(
        Call[] calldata calls,
        uint8 v, bytes32 r, bytes32 s
    ) external {
        bytes32 commit = keccak256(abi.encode(calls));

        // AUTH: verify EOA's signature, set `authorized`
        address authorized = auth(commit, v, r, s);
        require(authorized != address(0), "AUTH failed");

        // AUTHCALL: execute each call as the EOA
        for (uint i = 0; i < calls.length; i++) {
            authcall(gasleft(), calls[i].to, calls[i].value, calls[i].data);
        }
    }
}
```

**Security model:**
- The EOA FULLY trusts the invoker contract. A malicious invoker can drain the EOA.
- `invokerAddress` is in the signed message -- authorization can't be replayed through a different invoker.
- Any regular EOA transaction increments the nonce, invalidating all outstanding 3074 authorizations.

**Use cases:**
- Gas sponsorship (relayer sends the tx and pays gas)
- Batch transactions (approve + swap in one tx)
- Session keys (invoker enforces time/action limits)

**Status:** **Withdrawn.** Replaced by EIP-7702 in May 2024. Reasons:
- AUTH/AUTHCALL would become dead-weight opcodes in a post-EOA world
- Poor ERC-4337 compatibility (nonce-sharing problems)
- MEV vulnerability (commitment could be front-run)
- EIP-7702 achieves the same goals without new opcodes

---

### "DoS Attack Analysis" (Quilt Team)

**What:** Deep analysis of how attackers can exploit AA's custom validation to DoS the mempool.

**The core attack -- mass invalidation:**
1. Attacker submits thousands of UserOperations whose validation reads some on-chain state (e.g., a price oracle)
2. All pass validation when submitted
3. Attacker changes a single storage slot (one cheap tx)
4. ALL thousands of UserOps become invalid simultaneously
5. Bundlers/miners wasted massive computation on operations that will never pay fees

**Why custom validation is dangerous -- forbidden patterns:**
- `require(block.timestamp < deadline)` -- passes now, fails later
- `require(oracle.getPrice() > threshold)` -- one oracle update invalidates everything
- `CREATE`/`CREATE2` during validation -- side effects
- `BALANCE` of other accounts -- balance changes invalidate

**Solutions identified:**
1. Restrict what opcodes can run during validation (no `TIMESTAMP`, `BLOCKHASH`, etc.)
2. Restrict storage access to only the account's own storage
3. Staking/reputation system for entities that need broader access
4. Bound the gas cost of validation

These solutions became the foundation of ERC-7562.

---

### "Account Abstraction: Open Questions" (Quilt Team)

**What:** Catalogs unsolved problems in AA design:
- How to handle multi-tenant contracts (many users sharing one contract)?
- How to prevent front-running of AA transactions?
- How to handle storage access conflicts between concurrent UserOps?
- What's the right staking/reputation model for DoS prevention?

---

### "Account Abstraction Implementation, Rationale Document" (Quilt Team)

**What:** Technical documentation of the EIP-2938 implementation, explaining design decisions and their rationale.

---

### "Meta-transactions <-> AA" (Quilt Team)

**What:** Analysis of how meta-transactions (gasless txs via relayers) relate to full account abstraction.

**Key comparison:**

| Aspect | Meta-Transactions | Account Abstraction |
|--------|-------------------|---------------------|
| **Layer** | Application (requires contract changes) | Protocol/infrastructure |
| **Gas payment** | Relayer pays, dapp reimburses | Paymaster or contract pays natively |
| **Validation** | Still ECDSA only | Any signature scheme |
| **Contract support** | Must inherit ERC-2771 | Works universally |
| **Deployed contracts** | Cannot be retrofitted | Works with existing contracts |
| **Scope** | Only gas sponsorship | Gas + validation + batching + key management |

**Conclusion:** Meta-transactions are a partial, application-layer solution. AA subsumes and generalizes them.

---

### "Implementing Account Abstraction as Part of eth1.x" (Vitalik Buterin)

**What:** Roadmap for adding AA to the existing Ethereum mainchain (eth1.x) rather than waiting for eth2 (Serenity). Proposes incremental steps that are backward-compatible.

---

## 2021 - ERC-4337: The Game Changer

### EIP-4337: Account Abstraction via Entry Point Contract (Vitalik Buterin, Yoav Weiss)

**What:** Account abstraction **entirely without consensus-layer changes**. Uses a parallel system with a separate mempool, off-chain bundlers, and a singleton on-chain EntryPoint contract. This is the dominant AA standard in production today.

**Why:**
- EIP-2938 required protocol changes that were politically infeasible during The Merge
- EIP-3074 only enhanced EOAs, couldn't enable full smart account capabilities
- The ecosystem needed AA **now**, without waiting for consensus changes

**How it works -- The Six Components:**

#### 1. UserOperation (the pseudo-transaction)

```solidity
struct PackedUserOperation {
    address sender;              // The smart account address
    uint256 nonce;               // Anti-replay, managed by EntryPoint
    bytes   initCode;            // Factory address + init data (first-time deployment)
    bytes   callData;            // The actual operation(s) to execute
    bytes32 accountGasLimits;    // Packed: verificationGasLimit | callGasLimit
    uint256 preVerificationGas;  // Gas for bundler overhead (calldata cost)
    bytes32 gasFees;             // Packed: maxPriorityFeePerGas | maxFeePerGas
    bytes   paymasterAndData;    // Paymaster address + validation/postOp data
    bytes   signature;           // Validated by the smart account, not protocol
}
```

Key differences from regular transactions:
- `initCode`: Enables counterfactual deployment -- account deployed on first use
- `paymasterAndData`: Gas sponsorship by a third party
- `signature`: Validated by the smart account contract -- can be ANY scheme (ECDSA, passkeys, multisig, BLS, etc.)

#### 2. EntryPoint Contract (the singleton orchestrator)

A single, global, trusted contract deployed at a deterministic address. All 4337 wallets interact with it.

```solidity
function handleOps(
    PackedUserOperation[] calldata ops,
    address payable beneficiary  // the bundler's reward address
) external;
```

**The Two-Loop Architecture (critical for security):**

**Loop 1 -- Verification (for each UserOp):**
```
1. If sender contract doesn't exist, deploy it using initCode (call the factory)
2. Call sender.validateUserOp(userOp, userOpHash, missingAccountFunds)
   - Account verifies the signature
   - Account pays required prefund to EntryPoint
   - Returns: success/failure + optional time-range validity
3. If paymaster specified:
   - Check paymaster has sufficient EntryPoint deposit
   - Call paymaster.validatePaymasterUserOp(userOp, userOpHash, maxCost)
   - Paymaster decides whether to sponsor
4. If validation fails, skip this UserOp
```

**Loop 2 -- Execution (for each validated UserOp):**
```
1. Call sender with callData (account parses and executes)
2. Calculate actual gas used
3. Refund excess gas to account (or paymaster)
4. If paymaster returned context: call paymaster.postOp(context, actualGasCost)
5. Pay all collected fees to beneficiary (bundler)
```

The two-loop separation prevents a malicious UserOp from manipulating state to invalidate other UserOps in the same bundle.

#### 3. Bundler (off-chain relay node)

Bundlers collect UserOperations from the alt-mempool, validate them, package them into bundles, and submit to EntryPoint.

**Three-stage validation:**

```
Stage 1 - Sanity Checks:
  - Sender exists or valid initCode present
  - Gas limits below maximums (verification gas capped at 500,000)
  - Paymaster deposit sufficient
  - Valid nonce
  - Only one UserOp per sender (unless sender is staked)

Stage 2 - Simulation:
  - Call EntryPoint.simulateValidation
  - Trace EVM execution for banned opcodes (ERC-7562 rules)
  - Verify deterministic behavior

Stage 3 - Bundle Assembly:
  - Re-validate entire bundle against latest state
  - Exclude conflicting UserOps (same storage access)
  - Exclude UserOps whose paymaster deposit would be exhausted
```

Bundlers expose JSON-RPC API:
- `eth_sendUserOperation` -- submit a UserOp
- `eth_estimateUserOperationGas` -- estimate gas
- `eth_getUserOperationReceipt` -- get execution result

#### 4. Paymaster (gas abstraction)

Smart contracts that can sponsor gas fees for users.

```solidity
interface IPaymaster {
    // Called during verification. Decide whether to sponsor.
    function validatePaymasterUserOp(
        PackedUserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 maxCost
    ) external returns (bytes memory context, uint256 validationData);

    // Called after execution. Charge user in ERC-20, log analytics, etc.
    function postOp(
        PostOpMode mode,           // opSucceeded or opReverted
        bytes calldata context,
        uint256 actualGasCost,
        uint256 actualUserOpFeePerGas
    ) external;
}
```

**Two common paymaster types:**

**Verifying Paymaster:** Off-chain service signs UserOp data if it approves sponsorship. `validatePaymasterUserOp` checks that signature on-chain. Enables policies: per-user limits, time windows, allowlists, spending caps.

**ERC-20 Paymaster:** Users pay gas in tokens. `validatePaymasterUserOp` checks user has sufficient token balance. `postOp` transfers tokens from user at the current exchange rate (via oracle).

#### 5. Account Contract (smart contract wallet)

```solidity
interface IAccount {
    function validateUserOp(
        PackedUserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 missingAccountFunds
    ) external returns (uint256 validationData);
}
```

**Reference SimpleAccount:**
```solidity
contract SimpleAccount is BaseAccount {
    address public owner;

    function validateUserOp(
        PackedUserOperation calldata userOp,
        bytes32 userOpHash,
        uint256 missingAccountFunds
    ) external override returns (uint256 validationData) {
        require(msg.sender == entryPoint(), "not from EntryPoint");

        // Verify ECDSA signature (but could be ANY scheme)
        bytes32 hash = userOpHash.toEthSignedMessageHash();
        if (owner != hash.recover(userOp.signature))
            return SIG_VALIDATION_FAILED;

        // Prefund the EntryPoint
        if (missingAccountFunds > 0) {
            (bool success,) = payable(msg.sender).call{
                value: missingAccountFunds
            }("");
        }
        return 0; // success
    }

    function execute(address dest, uint256 value, bytes calldata func) external {
        require(msg.sender == entryPoint() || msg.sender == owner);
        (bool success, bytes memory result) = dest.call{value: value}(func);
        if (!success) assembly { revert(add(result, 32), mload(result)) }
    }
}
```

#### 6. Aggregator (optional -- signature aggregation)

Validates multiple signatures in a single step (e.g., BLS aggregation). Dramatically reduces on-chain verification cost.

**Complete flow:**
```
User --- signs UserOp ---> Bundler (alt-mempool)
                              |
                              |-- simulate validation
                              |-- check ERC-7562 rules
                              |-- bundle with other UserOps
                              |
                              +---> EntryPoint.handleOps(ops[], beneficiary)
                                      |
                                      |-- LOOP 1: Validate each op
                                      |     |-- sender.validateUserOp()
                                      |     +-- paymaster.validatePaymasterUserOp()
                                      |
                                      |-- LOOP 2: Execute each op
                                      |     |-- sender.execute(callData)
                                      |     +-- paymaster.postOp(context, gasCost)
                                      |
                                      +-- pay bundler (beneficiary)
```

**Developer experience -- before vs. after:**

```javascript
// BEFORE (EOA world): one operation per tx, must have ETH for gas
const tx = await wallet.sendTransaction({
    to: tokenContract.address,
    data: tokenContract.interface.encodeFunctionData('transfer', [to, amount])
});

// AFTER (ERC-4337): batch operations, paymaster sponsors gas
const userOp = await smartAccount.buildUserOperation({
    calls: [
        { to: tokenAddr, data: approveCalldata },    // approve
        { to: routerAddr, data: swapCalldata },       // + swap in one op
    ],
    paymaster: paymasterAddress,    // someone else pays gas
    paymasterData: "0x..."
});
const hash = await bundlerClient.sendUserOperation(userOp);
const receipt = await bundlerClient.waitForUserOperationReceipt({ hash });
```

**Gas overhead:** UserOperations cost 15,000-30,000 extra gas per operation for EntryPoint validation overhead vs. direct EOA transactions.

**EntryPoint versions:**
- **v0.6** (legacy)
- **v0.7** (current standard, at `0x0000000071727de22e5e9d8baf0edac6f37da032`)
- **v0.8** (latest, adds native EIP-7702 support)

**Status:** **Final, Live in Production.** Deployed March 2023. 40M+ smart accounts, 132M+ UserOperations processed, ~$5.7M gas sponsored by paymasters.

---

### "Moving Beyond EOAs" (Vitalik Buterin)

**What:** Vision piece arguing that EOAs should eventually be deprecated. All accounts should be smart contract wallets with programmable validation.

**Key arguments:**
- EOAs are a security liability (single key = single point of failure)
- EOAs can't be upgraded (no key rotation, no recovery)
- EOAs limit UX (no batching, no gas abstraction)
- The "endgame" is: every account is a smart contract

---

### "Validation Focused Smart Contract Wallets" (Quilt Team)

**What:** Design principles for building smart contract wallets, with emphasis on the validation phase.

**Key principles:**
- Validation must be cheap and deterministic
- Validation must not depend on external mutable state
- Wallet contracts should separate validation from execution
- Standard interfaces enable interoperability (leading to IAccount interface in ERC-4337)

---

## 2022 - Refinement and Alternatives

### EIP-5003: Insert Code into EOAs with AUTHUSURP (Quilt Team)

**What:** A new opcode, `AUTHUSURP`, that **permanently** deploys contract code at an EOA's address, irreversibly converting it from EOA to contract account.

**Why:** EIP-3074 lets EOAs delegate authority, but the original ECDSA private key always lingers as an attack vector. If the key leaks, the EOA is permanently compromised. No way to:
- Rotate keys
- Revoke a compromised key
- Migrate to more secure auth

EIP-5003 provides the "final migration path."

**How it works:**
1. EOA first authorizes a contract via EIP-3074's `AUTH` opcode
2. `AUTHUSURP(initcode_offset, initcode_size)` deploys code at the EOA's address
3. After deployment, EIP-3607 ensures the original ECDSA key can no longer send transactions
4. The account is now fully a smart contract -- the private key is dead

**Key difference from CREATE/CREATE2:** AUTHUSURP skips the nonce check because EOAs with transaction history have non-zero nonces.

**Concerns:**
- Irreversibility makes users hesitant
- `ecrecover` outside of transactions (e.g., ERC-2612 `permit`) would still accept the old key's signatures
- Dependency on EIP-3074 (which was superseded)

**Status:** Stagnant. Superseded by EIP-7702, which achieves the same goal reversibly.

---

### EIP-5792: Wallet Call API (Multiple Authors)

**What:** New JSON-RPC methods (`wallet_*` namespace) that replace `eth_sendTransaction` for modern wallet-dapp communication. Enables batching, capability discovery, and paymaster integration.

**Why:** `eth_sendTransaction` is a relic. It cannot:
- Submit batches
- Express atomicity requirements
- Communicate with paymasters
- Handle smart wallets where the tx hash is unknown at signing time

**How it works -- four new methods:**

```javascript
// 1. Discover what the wallet can do
const caps = await wallet.request({
    method: 'wallet_getCapabilities',
    params: ['0xUserAddress']
});
// Returns: { "0x1": { "atomic": { "status": "supported" },
//            "paymasterService": { "supported": true } } }

// 2. Send a batch of calls
const batchId = await wallet.request({
    method: 'wallet_sendCalls',
    params: [{
        version: '2.0.0',
        from: '0xUserAddress',
        chainId: '0x1',
        atomicRequired: true,  // all-or-nothing
        calls: [
            { to: '0xToken', data: '0xapprove...', value: '0x0' },
            { to: '0xDEX',   data: '0xswap...',    value: '0x0' }
        ],
        capabilities: {
            paymasterService: {
                url: 'https://paymaster.example.com',
                optional: true
            }
        }
    }]
});

// 3. Check status
const status = await wallet.request({
    method: 'wallet_getCallsStatus',
    params: [batchId]
});

// 4. Show status UI
await wallet.request({
    method: 'wallet_showCallsStatus',
    params: [batchId]
});
```

**Atomic capability values:**
- `"supported"` -- wallet executes atomically
- `"ready"` -- wallet CAN upgrade to atomic (pending user approval)
- `"unsupported"` -- no atomicity

**Developer experience:**
- Dapps call `wallet_sendCalls` instead of multiple `eth_sendTransaction`
- Capability discovery replaces ad-hoc feature detection
- Paymaster integration via ERC-7677 compliant URLs
- Gas savings: batching N txs saves `(N-1) * 21,000` base gas

**Status:** Draft/Review. Actively adopted by Coinbase Smart Wallet, thirdweb, WalletConnect, Dynamic.

---

### EIP-5806: Delegate Transaction (Hadrien Croubois)

**What:** New transaction type allowing EOAs to execute code via DELEGATECALL. The code runs in the EOA's context (its address, balance, storage).

**Why:** EOAs can't execute arbitrary code. Smart wallets require costly migration. EIP-5806 gives EOAs contract capabilities using the existing DELEGATECALL mechanism.

**How it works:** EOA sends a tx specifying a target contract. The tx delegate-calls it -- code runs as if it were the EOA.

**Restricted opcodes in delegate context:**
- `SSTORE`: Blocked (storage on EOA creates migration problems)
- `CREATE/CREATE2`: Blocked (alter nonce, breaking tx ordering)
- `SELFDESTRUCT`: Blocked

**Key use case:** Delegate-calling a multicall contract lets an EOA batch multiple calls while being `msg.sender` for all sub-calls.

**Status:** Stagnant. Superseded by EIP-7702 which provides a superset of functionality.

---

### "A Brief Note on the Future of Accounts" (Quilt Team)

**What:** Explores the long-term vision where all accounts are contracts, and EOAs are deprecated. Discusses the transition path and what the "endgame" account model looks like.

---

### "The Road to Account Abstraction" (Vitalik Buterin)

**What:** Vitalik's comprehensive roadmap tying together all AA efforts. Maps the path from current state to full native AA.

**Key roadmap:**
1. **Short-term:** ERC-4337 (no protocol changes, deployable now)
2. **Medium-term:** EOA enhancement (EIP-3074/7702 to bridge EOAs to smart accounts)
3. **Long-term:** Native protocol-level AA (EIP-2938/7701), deprecate EOAs

---

## 2023 - The 4337 vs 3074 Debate

### "ERC-4337 vs EIP-3074: False Dichotomy" (Yoav Weiss)

**What:** Argument that ERC-4337 and EIP-3074 are complementary, not competing.

**The debate:**
- ERC-4337 camp: "We need full smart accounts with custom validation, paymasters, bundlers."
- EIP-3074 camp: "Just let EOAs batch and delegate. Simpler, cheaper, backward-compatible."

**Yoav's argument:**
- 4337 provides the infrastructure (bundlers, paymasters, smart accounts)
- 3074/7702 provides the bridge (letting EOAs use 4337 infrastructure)
- They solve different problems and work together

---

### ERC-7562: Account Abstraction Validation Scope Rules (Yoav Weiss, Dror Tirosh)

**What:** Codifies the rules that bundlers must enforce during the validation phase of AA transactions. These rules protect the network from DoS attacks through unpaid computation.

**Why:** With AA, validation executes arbitrary EVM code. Without rules, an attacker can submit UserOps whose validation succeeds during simulation but reverts on-chain, wasting bundler computation without paying. This is the **mass invalidation attack**.

**Banned opcodes during validation:**

| Opcode | Why Banned |
|--------|-----------|
| `TIMESTAMP`, `NUMBER`, `COINBASE` | Block-dependent; change between simulation and inclusion |
| `BLOCKHASH`, `PREVRANDAO` | Different values per block |
| `GASPRICE`, `BASEFEE` | Network-condition dependent |
| `BALANCE`, `SELFBALANCE` | Only allowed for staked entities |
| `GASLIMIT`, `ORIGIN` | Vary between environments |
| `SELFDESTRUCT`, `INVALID` | Destructive/halting |
| `CREATE` | Banned except for factory creation patterns |
| `GAS` | Only allowed immediately before a `*CALL` |

**Storage access rules (the most critical part):**

"Associated storage" of address A = any slot where:
- The slot value is A, OR
- The slot was computed as `keccak256(A || x) + n` where n is 0..128

This maps to Solidity's `mapping(address => ...)` pattern.

```
Unstaked entities: can ONLY access own storage + associated storage of the account
Staked entities:   can access own storage + associated storage of ANY address
```

**Staking requirements:**
- Minimum stake: ~$1000 equivalent in native tokens
- Minimum unstake delay: 1 day
- Stake is NEVER slashed -- purely Sybil resistance

**Reputation system:**
```
Tracks per entity:
  opsSeen    = valid UserOps referencing this entity
  opsIncluded = UserOps actually included on-chain

Both decay hourly: value = value * 23 / 24

Three states:
  OK        = normal operation
  THROTTLED = max 4 mempool entries, max 4 bundle inclusions per block
  BANNED    = all UserOps rejected

Banning threshold: opsSeen/100 > opsIncluded + 50
  -> attacker can submit at most ~208 non-paying operations per hour before ban
```

**Developer impact:**
- Smart account devs must avoid banned opcodes in validation
- Paymaster/factory devs must stake to access broader storage
- Bundler implementations must trace validation via `debug_traceCall`

**Status:** Draft, but actively enforced by all production bundlers.

---

### EIP-6404: SSZ Transactions (Etan Kissling)

**What:** Proposal to use SSZ (Simple Serialize) format for Ethereum transactions, replacing RLP encoding.

**Why:** SSZ is more efficient, enables Merkle proofs of individual transaction fields, and aligns with the Beacon Chain's serialization format. This is infrastructure needed for future native AA transaction types.

**Relevance to AA:** Future native AA transaction types (EIP-7701, EIP-8141) will use SSZ encoding for cleaner parsing and proof generation.

---

## 2024 - EIP-7702: The Compromise

### EIP-7702: Set Code for EOAs (Vitalik Buterin, Sam Wilson, Ansgar Dietrichs)

**What:** A new transaction type (`0x04`) that allows any EOA to set its account code by pointing to an existing smart contract. The EOA doesn't store full bytecode; instead, a compact **delegation designator** (`0xef0100 || address`) is written into its code field, acting as a pointer to the implementation contract.

**Why:**
- EIP-3074's new opcodes (AUTH/AUTHCALL) would become dead weight in a post-EOA world
- 3074 had poor ERC-4337 compatibility (nonce-sharing problems with multi-tenant invokers)
- 3074 had MEV vulnerability (commitment could be front-run)
- 3074 delegation was per-transaction only; 7702 can persist

Vitalik reportedly drafted EIP-7702 in 22 minutes during a core dev call as a replacement.

**How it works:**

Transaction structure:
```
rlp([chain_id, nonce, max_priority_fee_per_gas, max_fee_per_gas,
     gas_limit, destination, value, data, access_list,
     authorization_list,  // <-- NEW
     signature_y_parity, signature_r, signature_s])
```

**Authorization list** -- each tuple: `[chain_id, address, nonce, y_parity, r, s]`

Processing for each authorization:
```
1. Verify chain_id is 0 (all chains) or current chain
2. Recover signer: ecrecover(keccak(0x05 || rlp([chain_id, address, nonce])))
3. Verify signer's code is empty OR already has a delegation designator
4. Verify signer's nonce matches
5. Increment signer's nonce
6. Write delegation: set signer's code = 0xef0100 || address
7. Special: if address is 0x00...00, CLEAR the delegation (revocation)
```

**After delegation:** When the EOA is called, the EVM loads code from the delegated address and executes it in the EOA's context. The EOA effectively "becomes" a smart contract while retaining its address and balance.

**Persistence:** Unlike EIP-3074 (per-transaction only), the delegation designator PERSISTS until explicitly revoked by delegating to `address(0)`.

**Self-sponsoring:** The `tx.origin` can set code on itself AND execute that delegated code within the SAME transaction.

```solidity
// OpenZeppelin pattern for EIP-7702 account
contract MyAccountERC7702 is Account, SignerEIP7702, ERC7821 {
    // SignerEIP7702 verifies signatures against address(this)
    // which is the EOA's own address
    // ERC7821 provides execute() for batch calls
}
```

**Breaking changes for existing contracts:**
- `extcodesize(addr)` returns 23 (not 0) for delegated EOAs -- breaks `isContract()` checks
- `require(tx.origin == msg.sender)` no longer reliably identifies EOAs
- Storage collisions possible when switching delegation targets

**Security note:** In the first week after Pectra, 65-70% of early EIP-7702 delegations were linked to phishing/scam activity. Wallets like MetaMask hard-code the delegation target to prevent abuse.

**Status:** **Live on Ethereum mainnet since May 7, 2025** (Pectra hard fork). 11,000+ authorizations in the first week. Supported by MetaMask, Coinbase Wallet, Trust Wallet, Ledger, Ambire.

---

### EIP-7701: Native Account Abstraction (Vitalik Buterin, Yoav Weiss)

**What:** Protocol-level (enshrined) AA by introducing a new transaction type that splits processing into distinct **validation**, **execution**, and **post-operation** phases with new EVM opcodes.

**Why:** ERC-4337 works but has overhead:
- ~20,000 extra gas per tx due to EntryPoint indirection
- Reliance on off-chain bundlers creates centralization risk
- EOAs still required to relay UserOperations

EIP-7701 moves AA into the protocol itself.

**New opcodes:**

| Opcode | Purpose |
|--------|---------|
| `ACCEPT_ROLE` | Signals that the current role (validation/execution/postOp) succeeded |
| `CURRENT_ROLE` | Returns which role is currently executing |
| `TXPARAMDLOAD` | Load transaction parameter data (like CALLDATALOAD but for tx fields) |
| `TXPARAMSIZE` | Size of a transaction parameter |
| `TXPARAMCOPY` | Copy transaction parameter data to memory |

**Context roles cycle through:**
```
ROLE_SENDER_DEPLOYMENT     -> deployer creates the sender contract
ROLE_SENDER_VALIDATION     -> sender validates the transaction
ROLE_PAYMASTER_VALIDATION  -> paymaster validates willingness to pay
ROLE_SENDER_EXECUTION      -> sender executes the actual operations
ROLE_PAYMASTER_POST_OP     -> paymaster bookkeeping
```

**Developer experience:**
- Smart contracts use `ACCEPT_ROLE` to signal successful validation
- `TXPARAM*` opcodes replace ABI-encoded calldata for transaction fields
- Paymasters become first-class protocol entities
- No bundlers required -- transactions go directly to the regular mempool

**Status:** Draft. Targeting the **Hegota fork (H2 2026)**. Partially consolidated into EIP-8141.

---

### EIP-7819: SETDELEGATE Instruction (Hadrien Croubois)

**What:** A new opcode that allows a contract to set its delegation target at the EVM level, complementing EIP-7702's transaction-level delegation.

**Why:** EIP-7702 sets delegation via a special transaction type. EIP-7819 provides the same capability as an EVM instruction, enabling contracts to programmatically delegate.

---

### "Why 4337 and 3074 Authors Are Disagreeing, and Who Got It Right" (Dror Tirosh)

**What:** Analysis of the philosophical split between the two camps:

**ERC-4337 camp (Yoav, Dror):**
- Smart contracts should be first-class accounts
- Custom validation is essential (not just ECDSA)
- Infrastructure (bundlers, paymasters) enables a rich ecosystem
- Worth the gas overhead for the flexibility

**EIP-3074 camp:**
- EOAs are here to stay; enhance them directly
- New opcodes are simpler than an entire parallel mempool
- Less gas overhead
- Don't break the existing model

**Resolution:** EIP-7702 was the compromise -- enhancing EOAs (like 3074 wanted) in a way compatible with 4337 infrastructure (like 4337 needed). Both camps eventually supported 7702.

---

## 2025 - Post-7702 Era

### "Provably Rootless EIP-7702 Proxy: PREP" (Multiple Authors)

**What:** A security pattern for EIP-7702 implementations that ensures the delegation target contract is "rootless" -- meaning no single key or entity can upgrade or change the contract's logic.

**Why:** If an EOA delegates to a contract controlled by a malicious party, that party can rug-pull all assets. PREP provides cryptographic proof that the delegation target is immutable and trustless.

**Key concern solved:** Users need assurance that the contract they're delegating to won't be changed underneath them.

---

### "Tempo Transactions" (Multiple Authors)

**What:** Explores time-based transaction features in the context of AA. Transactions with built-in time constraints (valid-after, valid-until) that the protocol enforces natively.

**Why:** Currently, time-based validity is checked during validation (wasting gas if expired). Native time bounds let nodes reject expired transactions before any execution.

**Relevance:** ERC-4337's `validAfter` and `validUntil` fields in validation data are an application-layer version of this. Tempo Transactions would make it protocol-native.

---

### "What Are the Motivating Goals of Full Account Abstraction?" (Vitalik Buterin)

**What:** Revisits the end-state vision now that EIP-7702 and ERC-4337 are live. Asks: what does "full" AA actually mean, and what's still missing?

**Key goals identified:**
1. **Signature flexibility:** Any algorithm (passkeys, post-quantum, multisig) -- largely achieved via 4337+7702
2. **Gas abstraction:** Someone else pays, or pay in tokens -- achieved via Paymasters
3. **Nonce abstraction:** Parallel nonces, no strict ordering -- partially achieved
4. **Key management:** Rotation, recovery, session keys -- achieved via smart accounts
5. **Batching:** Multiple operations atomically -- achieved via smart accounts + EIP-5792
6. **Protocol-level support:** No gas overhead from EntryPoint -- NOT yet achieved (needs EIP-7701/8141)

---

## 2026 - Native AA on the Horizon

### EIP-8130: Account Abstraction by Account Configuration (Multiple Authors)

**What:** A new transaction type achieving AA through **explicit owner registration and verifier contracts** stored on-chain. Rather than simulating wallet code to validate transactions, it makes verification deterministic.

**Why:** All prior AA proposals force nodes to simulate arbitrary wallet code before accepting transactions. EIP-8130 eliminates this burden by moving owner/verifier configuration into protocol-level storage.

**How it works:**

**Account Configuration Contract** at `ACCOUNT_CONFIG_ADDRESS` stores owner state:
```
ownerId (32 bytes) -> {
    verifier: address,    // contract that performs signature verification
    scope: uint8          // permission bitmask (SIGNATURE, SENDER, PAYER, CONFIG)
}
```

**Verifier contracts** implement a simple interface: receive hash + signature, return ownerId. A reserved `ECRECOVER_VERIFIER` (address(1)) triggers native secp256k1 without EVM overhead.

**2D Nonce System:** High 192 bits = nonce_key (2^192 parallel channels), low 64 bits = nonce_sequence. Special "nonce-free" mode with short-lived expiry timestamps.

**Transaction structure:**
```
sender identity + 2D nonce
account change entries (account creation, owner add/revoke, code delegation)
batched calls organized into atomic phases
gas sponsorship fields with independent payer authorization
```

**Phase-based execution:** Calls organized into phases. If any call in a phase reverts, that phase is discarded, but earlier phases persist. This means sponsor payment in phase 0 survives even if user actions in phase 1 fail.

**Account Lock Mechanism:** Accounts can be locked to freeze owner configuration with timelock for unlocking. Locked accounts get higher mempool rate limits.

**Developer experience:**
- EOA users gain AA immediately with no prior registration
- Smart contract wallets migrate via `importAccount()` reading ERC-1271 verification
- Gas sponsorship is built into the transaction type natively (no paymaster contracts needed)
- Key rotation (adding/removing owners with different verifiers) is a first-class operation

**Status:** Draft, targeting Hegota fork (H2 2026).

---

### EIP-8141: Frame Transaction (Vitalik Buterin, Others)

**What:** A new transaction type (`0x06`) that decouples authentication from ECDSA via a **frame-based execution model**. Transactions consist of ordered "frames," each with a mode, target, gas limit, and calldata.

**Why:** Post-quantum cryptography readiness. By replacing ECDSA-only authentication with flexible EVM-based validation, Ethereum can support quantum-resistant signature schemes without protocol changes.

**How it works:**

Transaction encoding:
```
[chain_id, nonce, sender, frames, max_priority_fee_per_gas,
 max_fee_per_gas, max_fee_per_blob_gas, blob_versioned_hashes]
```

Each frame: `[mode, target, gas_limit, data]`

**Three frame modes:**
```
VERIFY (1):  Validation frame (STATICCALL semantics, read-only)
             Must call APPROVE opcode to signal success
DEFAULT (0): General-purpose execution (contract deployment, etc.)
SENDER (2):  Executes as the sender after approval
```

**New opcodes:**
- `APPROVE (0xaa)`: Exits context and updates transaction-scoped approval (execution, payment, or both)
- `TXPARAM (0xb0)`: Introspects transaction fields and frame metadata
- `FRAMEDATALOAD / FRAMEDATACOPY`: Access frame input data

**Default EOA code supports two sig schemes:**
- SECP256K1 (0x0): Traditional ECDSA
- P256 (0x1): NIST P-256 for post-quantum preparedness

**Mempool safety:** Only four recognized validation frame prefixes for public relay. Validation gas capped at 100,000. Banned opcodes during validation: `ORIGIN`, `GASPRICE`, `BLOCKHASH`, `TIMESTAMP`, `BALANCE`, `SSTORE`, `CREATE/CREATE2`.

**Developer experience:**
- Custom signature algorithms without protocol forks (BLS, lattice-based, Schnorr)
- EOAs get gas sponsorship without deploying smart contracts
- ERC-20 fee payment via paymaster frames
- Deferred account deployment in first transaction

**Status:** Draft. Described by Vitalik as an "omnibus" proposal incorporating EIP-7701 plus remaining blockers. Targeting Hegota fork (H2 2026).

---

### "Mempool Strategies for EIP-8141" (Multiple Authors)

**What:** Technical analysis of how to safely manage the mempool for Frame Transactions. Addresses the DoS problem for the new validation model.

**Key challenges:**
- Frame-based validation is more complex than 4337's single-function validation
- Multiple frames with different gas budgets create new attack surfaces
- Need to define which frame patterns are safe for public relay

**Solutions explored:**
- Canonical frame prefixes that bundlers/validators recognize
- Strict gas limits per validation frame (100,000 max)
- Reputation tracking for non-canonical paymasters
- Limits on pending transactions per paymaster

---

## Author Legend

| Initial | Person | Notable Contributions |
|---------|--------|----------------------|
| **(V)** | Vitalik Buterin | EIP-101, 86, 859, 2938, 4337, 7702, 7701 |
| **(Q)** | Quilt Team (Sam Wilson, etc.) | EIP-3074, 5003, 2938, DoS analysis |
| **(Y)** | Yoav Weiss | EIP-1613 (GSN), ERC-4337, ERC-7562 |
| **(A)** | Ansgar Dietrichs | EIP-7701, 2938, 7702 |
| **(D)** | Dror Tirosh | ERC-4337, ERC-7562, analysis pieces |
| **(H)** | Hadrien Croubois (@Amxx) | EIP-5806, 7819 |
| **(S)** | Sam Wilson | EIP-5792, 3074, 86 |
| **(M)** | Multiple Authors | PREP |
| **(L)** | Multiple Authors | Mempool strategies |
| **(T)** | Multiple Authors | Tempo Transactions |
| **(J)** | Jeff Coleman | Application-layer AA, miner data |
| **(B)** | Buterin (early) | Gas from contract's balance |
| **(E)** | Etan Kissling | EIP-6404 (SSZ) |

---

## The Big Picture: How It All Connects

```
THE EVOLUTION OF ACCOUNT ABSTRACTION

2015  EIP-101 (radical vision: everything is a contract)
        |
2016  "Spending gas from contracts" (seed of gas abstraction)
        |
2017  EIP-86 (null-sender txs + CREATE2)
        |
2018  EIP-859 (PAYGAS opcode) ---- EIP-1613/GSN (relay network)
        |                              |
2019  "Maximally simple AA"      GSN production deployment
        |                              |
2020  EIP-2938 (protocol AA) --- EIP-3074 (AUTH/AUTHCALL)
        |         |                    |
        |    (too hard during     (security concerns)
        |     The Merge)               |
2021  ERC-4337 <--------------- takes ideas from both
        |  (no protocol changes,       |
        |   bundlers + paymasters)     |
2022  EIP-5003 (AUTHUSURP) --- EIP-5806 (delegate tx) --- EIP-5792 (wallet API)
        |                              |
2023  ERC-7562 (validation rules) --- "4337 vs 3074" debate
        |                              |
2024  EIP-7702 <----- the compromise (replaces 3074)
        |  (set code for EOAs,         |
        |   works with 4337)     EIP-7701 (native AA draft)
        |                              |
2025  LIVE: ERC-4337 + EIP-7702  ---- PREP, Tempo Transactions
        |  (Pectra hard fork)          |
2026  EIP-8130 + EIP-8141 <------- full native AA
        (targeting Hegota fork)

CURRENT STATE (2025):
  |-- ERC-4337: live, 40M+ accounts
  |-- EIP-7702: live since Pectra (May 2025)
  |-- EIP-5792: adopted by major wallets
  +-- ERC-7562: enforced by all bundlers

NEXT (2026):
  +-- EIP-8130/8141: native protocol-level AA (Hegota fork)
      -> eliminates EntryPoint overhead
      -> post-quantum ready
      -> every account is natively abstract
```

### The Three Eras of Account Abstraction

**Era 1: Application-Layer (2018-2023)**
Meta-transactions, GSN, relayers. Required every contract to opt in. Fragmented ecosystem.

**Era 2: Infrastructure-Layer (2023-2025)**
ERC-4337 + EIP-7702. No protocol changes needed. Bundlers and paymasters as off-chain infrastructure. 15-30K gas overhead per operation. The current standard.

**Era 3: Protocol-Layer (2026+)**
EIP-7701/8130/8141. AA built into Ethereum itself. Zero overhead. Post-quantum ready. Every account is natively abstract. The endgame.
