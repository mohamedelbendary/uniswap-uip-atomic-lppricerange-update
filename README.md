# UIP-X: Atomic Update of Liquidity Position Price Range

**Author**: Mohamed ElBendary  
**Date**: April 24, 2025  
**Status**: Draft  
**Category**: Core Protocol Enhancement

## 🧠 Abstract
This proposal advocates for a core protocol feature that elevates the liquidity event of modifying a liquidity position’s active price range to be treated as a first-class citizen by the pool contract, alongside its sibling operations (`swap`, `mint`, `burn`, `donate`, etc.).

The primary motivations are to:
- Enhance operational robustness through **atomicity**
- Improve **impermanent loss (IL)** management by avoiding forced crystallization
- Achieve significant **gas savings** compared to the current two-transaction (`burn`/`mint`) process

We propose the addition of a new core function, `updateLPPriceRange`, to the Uniswap Protocol Core `PoolManager.sol` contract, along with corresponding hooks `beforeUpdateLPPriceRange` and `afterUpdateLPPriceRange` in the `IHook` interface.

This function is designed to reuse existing, optimized internal library functions (e.g., `Tick.update`, `Position.update`) for state changes and calculations.

The benefits of this feature extend to potentially realizing a more robust implementation of the widely adopted **Position Manager**, by exposing a wrapper function of the one proposed for this feature.


## 🎯 Motivation

Currently, modifying the price range of a Uniswap v3/v4 concentrated liquidity position requires two separate transactions (`burn`/`removeLiquidity` followed by `mint`/`addLiquidity`). This presents several challenges:

- **Operational Risk**: The two-step process can fail between transactions, leaving capital unexpectedly out of the market and requiring complex recovery logic.
- **Impermanent Loss (IL) Crystallization**: The `burn` step forces realization of IL at the moment of withdrawal, which can be disadvantageous during temporary market volatility.
- **Complexity for Automated Managers**: Systems must coordinate two distinct transactions and handle intermediate failures.
- **High Gas Costs**: Executing two separate transactions incurs significant gas costs, including two base transaction fees and potentially redundant storage reads/writes. This makes frequent active management expensive, especially on L1.

The proposed `updateLPPriceRange` function addresses these issues by performing the range adjustment **atomically within a single transaction**, leveraging core internal logic for efficiency. This provides:
- **Atomicity**
- **IL crystallization avoidance**
- **Intent-revealing primitive**
- **Substantial gas savings**


## 🛠️ Proposed Change

### New Core Function

- Add `updateLPPriceRange` to `PoolManager.sol`.

### New Hook Callbacks

- Add `beforeUpdateLPPriceRange` and `afterUpdateLPPriceRange` to the `IHook` interface.
- Integrate their execution within `updateLPPriceRange`.

### Use Cases for `before` / `afterUpdateLPPriceRange` Hook Callbacks

The proposed hook functions enable critical functionality that contributes to compliance, safety, observability, and automation.

#### ✅ Validation & Policy Enforcement

Hooks could enforce specific rules before allowing a range update, such as:

- Is the new range width acceptable?
- Is the caller authorized (beyond basic ownership)?
- Is the update happening within allowed time windows?
- Does the new range align with external oracle data (e.g., prevent updates to clearly unprofitable ranges based on volatility)?
- Does the `mustContinueTrading` flag align with current pool conditions?

#### ❌ Reverting Invalid Actions

- Hooks can reject updates if any condition fails, reverting the transaction.

#### 🔔 Notifications

- Emit custom events to signal the successful update.
- Enables off-chain systems (e.g., automated liquidity manager backends) to listen for these events.

#### 📊 Logging & Analytics

- Record range update data for internal hook state or external consumption via events.

#### ⚙️ Triggering Secondary Actions

- Enable follow-up processes (on-chain or off-chain) post-confirmation.

#### 🔄 State Synchronization

- Allow hooks to update their own internal state as necessary in sync with the pool.


## Benefits

✅ Atomicity: No half-executed LP transitions

✅ IL Avoidance: No forced crystallization

✅ Gas Savings: Estimated 30–50% reduction vs burn + mint

✅ Simplified Automation: One call, fewer edge cases

✅ Compliance/Policy Control: Via hook validation

## 👥 Stakeholder Impact Analysis

### 🧑‍🌾 Liquidity Providers (LPs)
- ✅ Atomic adjustments reduce the risk of capital being stuck between steps.
- ✅ Avoid forced IL crystallization during routine adjustments.
- ✅ Significant potential gas savings for active range management.

### 🧠 Automated Liquidity Managers (Vaults, Strategies)
- ✅ Simplified automation logic – manage one atomic call instead of coordinating two.
- ✅ Reduced operational complexity and failure handling requirements.
- ✅ Enhanced ability to offer strategies that minimize IL realization and operate more frequently due to lower costs.
- ✅ Lower operational costs due to reduced gas usage for rebalancing.

### 📈 Traders
- ✅ More efficient liquidity provision can lead to deeper liquidity and potentially better pricing.
- ✅ More reliable pool operation (fewer \"stuck\" LP adjustments).

### 🧩 Pool Deployers / Hook Developers
- ✅ Gain new, specific hook points (`beforeUpdateLPPriceRange`, `afterUpdateLPPriceRange`) for validation, policy enforcement, notifications, etc.
- ⚠️ Requires implementing the new hook callbacks if custom logic is desired.

### 🦄 Uniswap Protocol
- ✅ Enhanced functionality addressing major LP pain points (cost, risk, IL).
- ✅ Increased robustness for liquidity management actions.
- ✅ Increased capital efficiency across the ecosystem due to cheaper liquidity management.
- ⚠️ Increased complexity of the core `PoolManager` contract and `IHook` interface (manageable through reuse of existing internal logic).


## 🧪 Technical Details

### 🔧 5.1. Function Signature (`PoolManager.sol`)

The function takes as input:
- A position identifier
- A new price range
- A flag to ensure continued liquidity exposure if desired
- Optional hook data

Upon success, it returns `Position.Info`.

```solidity
function updateLPPriceRange(
    PositionKey calldata key,
    UpdateLPPriceRangeParams calldata params
) external override returns (Position.Info memory positionInfo);

struct UpdateLPPriceRangeParams {
    int24 tickLower;              // The new lower tick boundary
    int24 tickUpper;              // The new upper tick boundary
    bool mustContinueTrading;     // If true AND current tick is outside the new range, revert
    bytes data;                   // Optional data to pass to hooks
}
```


### ⚙️ 5.2. Core Logic (Execution Steps within `updateLPPriceRange`)

The function executes the following steps atomically:

1. **Decode Keys & Params**: Extract identifiers and new range parameters.
2. **Load Position**: Read current `Position.Info`. Validate ownership/operator. Ensure liquidity > 0.
3. **Range Check**: Return early if unchanged. Validate `newTickLower < newTickUpper`.
4. **Load Pool State (`slot0`)**: Retrieve `sqrtPriceX96`, `currentTick`, and `feeGrowthGlobal`.
5. **`mustContinueTrading` Check**: Ensure current tick falls within new range if flag is set.
6. **Call `beforeUpdateLPPriceRange` Hook**: Run pre-update logic.
7. **Fee Calculation & State Update**:
   - Use `Position.update` to calculate accrued fees.
   - Update `tokensOwed` in a temporary copy of `Position.Info`.
8. **Tick Updates (Remove Old)**:
   - `Tick.update` for old `tickLower` with `-liquidity`
   - `Tick.update` for old `tickUpper` with `+liquidity`
9. **Tick Updates (Add New)**:
   - `Tick.update` for new `tickLower` with `+liquidity`, store `feeGrowthOutside`
   - `Tick.update` for new `tickUpper` with `-liquidity`, store `feeGrowthOutside`
10. **Calculate New Snapshots**:
    - Use `Position.getFeeGrowthInside` with `feeGrowthOutside` to compute new snapshot.
11. **Prepare Final Position State**:
    - Update memory copy of `Position.Info` with new tick range and fee snapshot.
12. **Call `afterUpdateLPPriceRange` Hook**: Run post-update logic with final state.
13. **Write Final State**: Persist updated `Position.Info` to storage (`SSTORE`).
14. **Emit Event**: `UpdateLPPriceRange`.
15. **Return**: Return updated `Position.Info`.


### 🧩 5.3. Hook Interface Changes (`IHook.sol`)

```solidity
function beforeUpdateLPPriceRange(
    address sender,
    PoolKey calldata key,
    PoolManager.UpdateLPPriceRangeParams calldata params,
    bytes calldata data
) external returns (bytes4);

function afterUpdateLPPriceRange(
    address sender,
    PoolKey calldata key,
    PoolManager.UpdateLPPriceRangeParams calldata params,
    Position.Info calldata positionInfo,
    bytes calldata data
) external returns (bytes4);
```


### ⛽ 5.4. Gas Considerations

By reusing optimized internal functions and performing atomic execution:

- Saves one base transaction fee (~21k gas)
- Reduces `SLOADs`: Reads `slot0` and position data once
- Reduces `SSTOREs`: Updates storage once instead of twice
- Avoids duplicate access control checks from separate public calls

**Estimated gas cost**:
- `updateLPPriceRange`: ~80k – 140k gas
- Equivalent `burn + mint`: ~140k – 260k+ gas

This implies a **30–50% savings** in range adjustment scenarios. Benchmarks pending.


### 🚫 5.5. Error Handling

The function **should revert** in the following cases:

- Invalid `positionId` or `PoolKey`
- Unauthorized caller (not owner or approved operator)
- Zero liquidity in the position
- Invalid tick range (`newTickLower >= newTickUpper`)
- `mustContinueTrading` check fails
- Hook reverts or returns failure (`before` or `after`)
- Internal overflow/underflow
- Gas limit exceeded


## 🧰 Improved Position Manager Implementation

The proposed atomic update feature would significantly enhance the **Position Manager's** batched operation capabilities by replacing the current `burn` + `mint` sequence with a more efficient, single-step approach.

In Uniswap v4, the Position Manager uses a **batched command pattern** to execute multiple actions in one transaction. However, when modifying a position's price range, it still must orchestrate two discrete core operations — `burn` and `mint`. This results in the same challenges faced by external liquidity managers:

### ❌ Current Challenges

- **Increased Gas Costs**: Despite batching, the core operations still involve redundant state reads/writes and two separate liquidity modifications.
- **Implementation Complexity**: Intermediate state must be manually handled to ensure funds from the `burn` are applied to the `mint`.
- **Error Handling Complexity**: Failures between the two steps require additional recovery logic, even within a single transaction.


### ✅ Benefits of Atomic `updateLPPriceRange` Integration

If adopted, the Position Manager could replace its two-step logic with a single atomic command. This would:

- **Simplify the Command Structure**: One command instead of two.
- **Reduce Gas Usage**: Avoids redundant operations and saves gas.
- **Improve Reliability**: Removes failure risk between dependent operations.
- **Maintain Liquidity Continuity**: The position stays live throughout the update.


This enhancement makes the Position Manager more **efficient**, **robust**, and **developer-friendly**, ultimately benefiting **all users** who interact with Uniswap through this interface.

## 🏦 Institutional Benefits Discussion

The combination of an atomic `updateLPPriceRange` function and the `mustContinueTrading` flag offers targeted advantages for **institutional and regulated liquidity providers** and **quantitative traders**.


### 🛡️ Reduced Operational Risk

- The current two-step (`burn` + `mint`) process introduces the risk that `burn` succeeds but `mint` fails (e.g., due to gas spikes or network congestion), leaving capital temporarily withdrawn from the market.
- An atomic update ensures the adjustment is **all-or-nothing**, significantly reducing failure scenarios and eliminating the need for complex recovery logic.


### 📉 Improved Risk Management & P&L Smoothing (IL Avoidance)

- Avoiding IL crystallization during routine adjustments supports smoother profit and loss (P&L) profiles.
- Helps regulated entities meet reporting requirements and align with strategies focused on continuous market exposure rather than reactive liquidity repositioning.


### 🎯 Enhanced Execution Control

- Institutional mandates often require **precise, slippage-aware execution**.
- The `mustContinueTrading = true` flag ensures that updates **only execute when conditions still align** with the intention to maintain an active position.
- Prevents execution based on stale signals or market divergence from preconditions.


### 📋 Compliance and Policy Adherence

- The `mustContinueTrading` flag allows programmatic enforcement of mandates that require **continuous active participation**.
- Enables automated audit trails that show policy-aligned behavior even in volatile conditions.


### 📊 Support for Sophisticated Strategies

- Institutions may use advanced tactics like **anticipatory liquidity placement** or **limit-range strategies**.
- The flag ensures strategies can programmatically differentiate between proactive rebalancing and passive liquidity positioning.


### 🤖 Simplified Automation & Reconciliation

- A single atomic call reduces error-prone complexity in automated trading infrastructure.
- Eases integration, reconciliation, and reporting by replacing two operations with one intent-aligned action.


Together, these features deliver **greater operational robustness, reduced economic risk, and improved automation clarity**, all of which are especially critical in **institutional DeFi environments**. Significant gas cost savings are also likely to be welcomed by high-volume trading desks.

## 🔐 Hook Callback Functions Permissions

Adding the two new hook functions — `beforeUpdateLPPriceRange` and `afterUpdateLPPriceRange` — to the `IHook` interface requires updates to the existing **hook permissions mechanism** in the Uniswap protocol.


### 🧩 Hook Permissions Structure

- Uniswap v4's `PoolManager` maintains a `Pool.HookPermissions` structure for each pool.
- This structure contains **bitwise flags** that determine which hook functions the `PoolManager` is authorized to call on a designated hook contract.
- The existing structure must be **extended** with new flags to control access to the proposed `beforeUpdateLPPriceRange` and `afterUpdateLPPriceRange` callbacks.


### 🏗️ Pool Initialization

- Upon pool creation, the deployer must be able to specify whether the new hook callbacks are **permitted**, using the updated `HookPermissions` structure passed to the `initialize` function.
- This ensures pool-level configurability of callback behavior for range updates.


### ⚙️ PoolManager Logic Update

- Before invoking `hook.beforeUpdateLPPriceRange(...)` or `hook.afterUpdateLPPriceRange(...)`, the `PoolManager` must:
  1. Check the corresponding permission flag.
  2. Skip the call if the flag is not enabled.

Without these changes:
- The `PoolManager` would either **unconditionally call the new hooks** (breaking the permissions concept), or
- Be unable to call them at all (limiting functionality).

By extending `HookPermissions` to cover the new callbacks:
- **Pool deployers retain fine-grained control** over allowed hook interactions.
- The system preserves **gas efficiency** and **security isolation** for unauthorized or unnecessary hook paths.

## 🛡️ Security Considerations

Ensuring the security and integrity of the `updateLPPriceRange` function — and its integration into the Uniswap Protocol ecosystem — is paramount. Security relies on:

- Robust access control
- Standard reentrancy protection
- Careful orchestration of logic and state transitions
- Correct handling of hook interactions
- Proven correctness of core Uniswap v4 libraries (`Tick.sol`, `Position.sol`)

Rigorous auditing and comprehensive testing are essential before deployment.


### 🔁 Reentrancy Protection

- The addition of `beforeUpdateLPPriceRange` and `afterUpdateLPPriceRange` introduces **new reentrancy vectors**.
- If a hook callback re-enters `PoolManager` (e.g., via `collect` or even `updateLPPriceRange` itself), it could compromise state consistency.
- **Mitigation**: Apply Checks-Effects-Interactions and `nonReentrant` patterns. Hook developers must also implement safe patterns.


### 🔐 Access Control

- Only the **position owner or approved operator** must be able to call `updateLPPriceRange`.
- Must strictly enforce existing `PoolManager` ownership/approval logic to prevent unauthorized updates.


### 🧱 Core Library Security & Correctness

- The function **relies heavily** on `Tick.update`, `Position.update`, and `Position.getFeeGrowthInside`.
- These components:
  - Handle fee calculations
  - Return critical accumulator values
  - Update snapshots
- Any vulnerability in these libraries will propagate to `updateLPPriceRange`.

✅ These libraries are widely used and audited, but their reuse in this new sequence **must be re-verified** as part of this feature’s audit scope.


### ⛓️ Tick Manipulation & Fee Integrity

- Fee calculations must be based on immutable snapshots **before range change**.
- Atomic execution prevents manipulation **during** the operation, but reading accumulator values at the wrong time could yield incorrect fee values.


### 🧩 Complexity and Edge Cases

- Orchestrating updates across multiple ticks and position states introduces potential for **edge-case errors**.
- Must test for:
  - `MIN_TICK` / `MAX_TICK` boundaries
  - Low-liquidity rounding errors
  - Tick spacing misalignments


### 🧪 Hook Interaction & Validation

- Must check the return values of both hooks:
  - `beforeUpdateLPPriceRange(...)`
  - `afterUpdateLPPriceRange(...)`
- If either hook fails or returns an unexpected selector, the **entire transaction must revert**.
- Gas limits within hook execution must also be considered to avoid mid-call failures.


### ✅ Input Validation

- Validate:
  - `newTickLower < newTickUpper`
  - Liquidity > 0
  - `mustContinueTrading` behavior against `currentTick`
- Ensure invalid configurations do not waste gas or enter invalid state.


Thorough testing, edge case analysis, and full integration audits are required before this feature can be safely adopted in production.


## 🔄 Backward Compatibility

- Requires a protocol upgrade (e.g., Uniswap v5).
- No changes to existing v4 pools.
- New hook functions remain opt-in via permission flags.

## 🚀 Rollout / Deployment

This feature introduces an implementation-level change to the core `PoolManager` contract and `IHook` interface. As such, it will likely require:

- A **coordinated protocol upgrade**
- Inclusion in a **future major version** of the Uniswap protocol (e.g., v5)

It is not deployable as a runtime feature for existing pools without a broader protocol version bump, due to breaking interface changes and required permission structure extensions.


## 🔭 Open Questions / Future Work

### 🧪 Formal Verification

- Apply formal methods to `updateLPPriceRange` and its interaction with `Tick` and `Position` libraries
- Prove correctness and prevent edge-case logic errors or vulnerabilities

### ⚙️ Integration with Dynamic Fees & Advanced Features

- Analyze potential edge cases in pools using:
  - Dynamic fees
  - Hooks with stateful logic
  - Other advanced v4 extensibility features

### 📚 Library Interface Finalization

- Define stable interfaces for:
  - `Tick.update`
  - `Position.update`
  - `Position.getFeeGrowthInside`
- Consider future-proofing and minimal API surface area changes

### 🧑‍💻 Developer Experience Exploration

- Explore SDK and Position Manager integration
- Design intuitive APIs and UI abstractions to surface this functionality for LPs and power users

### ❌ Error Reporting & UX Feedback

- Improve validation error codes for:
  - `mustContinueTrading` failures
  - Hook rejections
  - Invalid tick range inputs
- Ensure clear revert messages and predictable contract behavior

### ⛽ Gas Benchmarking

- Run empirical benchmarks across:
  - L1 and L2 deployments
  - Different range widths
  - Warm vs. cold tick updates
- Validate estimated **30–50% gas savings** over `burn + mint`

### 📖 Documentation

- Create comprehensive developer documentation covering:
  - Function signature and parameter behavior
  - Expected use cases and UX considerations
  - Hook integration guidance
  - Gas performance expectations


Together, these areas will help mature the proposal into a production-grade feature aligned with Uniswap’s roadmap and contributor needs.

## 🧾 Authorship Verification

This UIP is authored by:

**Mohamed ElBendary**  
GitHub: [https://github.com/mohamedelbendary](https://github.com/mohamedelbendary)  
Uniswap Discourse: **mbendary**  
Ethereum address: `0xf475608bBE46cBC5E9B1d42F731b5BbfC2C27930`  
Date: April 24, 2025

_Signed message from the Ethereum address will be added._
## License
This proposal is shared under CC0 1.0 Public Domain Dedication. Attribution is appreciated.
