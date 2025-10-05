# Book Of Red Flags

- [ ]  **Caller-context mismatch bypasses per-vault oracle overrides**
    - **Symptom:** Slippage/daily-loss hooks use the **default oracle** even after a vault opts into a new feed; risk checks pass/fail on stale prices (e.g., hook reads 1× while vault expects 2×).
    - **Where to look (Solidity/DeFi stacks):**
        - `src/periphery/hooks/slippage/*`: `_convertToNumeraire`, `_handleBefore*`, `handleBeforeExactInput*`; UniswapV3/KyberSwap/Odos hooks.
        - `src/periphery/OracleRegistry.sol`: `getQuote`, `_getOracleForVault`, `getQuoteForUser`, and `oracleOverrides[vault][base][quote]`.
        - State access sites: `_vaultStates[msg.sender].oracleRegistry` **vs** `_vaultStates[vault].oracleRegistry`.
    - **Red flags:**
        - Registry functions key lookups by `msg.sender` and are invoked **from hooks/adapters** (not the vault).
        - Hooks call `oracleRegistry.getQuote(...)` (caller-scoped) instead of `getQuoteForUser(..., vault)`.
        - Silent fallback to default when `oracleOverrides[hook][...]` is empty.
        - Tests only exercise direct registry calls from the vault (no intermediary path).
    - **Fix pattern:** Pass the vault explicitly and fetch the registry from the vault’s state.
        
        ```solidity
        // Before
        IOracleRegistry R = _vaultStates[msg.sender].oracleRegistry;
        return IOracle(R).getQuote(amount, token, _getNumeraire());
        
        // After
        IOracleRegistry R = _vaultStates[vault].oracleRegistry;
        return R.getQuoteForUser(amount, token, _getNumeraire(), vault);
        
        ```
        
        Add an integration test that sets a per-vault override and asserts the **hook path** uses it (no silent default); prefer revert over fallback when override is required; never use `tx.origin`.
        
- [ ]  **Unchecked multi-source swap receivers siphon vault funds**
    - **Symptom:** Aggregation swaps accept calldata with `srcReceivers[]/srcAmounts[]` (and often `feeReceivers[]/feeAmounts[]`). If hooks/adapters don’t validate these arrays, the router can `transferFrom(vault, receiver[i], amount[i])` to arbitrary addresses while slippage/daily-loss checks still pass.
    - **Where to look (Solidity/DeFi stacks):**
        - Periphery hooks/adapters around DEX aggregation: pre-swap handlers (e.g., `_handleBefore*`, `_convertToNumeraire`) and any parsing of `SwapExecutionParams`/`SwapDescription`like structs.
        - Router interfaces and internal helpers that loop over receivers and call `safeTransferFrom`/`_doTransferERC20` with `from = vault` and `to = receiver[i]`.
        - Vault execution/policy path (Merkle/allowlist/guardian approvals) that authorizes the *target* but does **not** bind array contents of the swap description.
    - **Red flags:**
        - No assertion that every `srcReceivers[i]` is an **approved** executor (router/known system address) or on an allowlist; EOAs permitted.
        - Only `total <= amount` checks (allows partial diversion) instead of strict `sum(srcAmounts) == amount`.
        - `dstReceiver` not constrained to the vault (or an authorized recipient).
        - Unrestricted `approveTarget/flags` combinations that enable third-party spenders.
        - Tests cover direct swaps but don’t mutate `srcReceivers/srcAmounts/fee*` to simulate theft.
    - **Fix pattern:** Enforce receiver and amount invariants at the hook/adapter boundary **before** invoking the router.
        - Require: `srcReceivers.length == 1 && srcReceivers[0] == approvedRouter` (or enforce an allowlist); forbid EOAs unless explicitly approved.
        - Require: `sum(srcAmounts) == desc.amount` and `desc.dstReceiver == vault` (or a pre-approved recipient).
        - Bind calldata in policy: include a hash of `srcReceivers/srcAmounts/dstReceiver/flags/approveTarget` in the authorization/proof; reject on mismatch.
        - Prefer a **pull/collected** model: pre-transfer to the router (or use Permit2) so the router pulls from the hook, not directly from the vault.
        - Revert on unknown `approveTarget`/`flags`; add tests that tamper with receivers/amounts and expect revert.
    - **Related bug cases:**
        - Fee leakage via unchecked `feeReceivers/feeAmounts`.
        - Other aggregators with multi-recipient sources or internal `transferFrom` loops.
        - Multicall/clipboard frameworks where policy authorizes the target/selector but not the **array fields** of calldata.
        - Permit/allowance scopes that let third-party spenders move funds without recipient validation.
- [ ]  **Incorrect path parsing for `exactOutput` swaps flips `tokenIn/tokenOut`**
    - **Symptom:** Hooks/adapters parse `params.path` identically for `exactInput` and `exactOutput`. For `exactOutput`, the path is reversed, so first 20 bytes are **tokenOut** and last 20 bytes are **tokenIn**. Mis-parsing swaps the roles, causing slippage/daily-loss checks and oracle lookups to run on the wrong assets (bypass or false reverts).
    - **Where to look (Solidity/DeFi stacks):**
        - Uniswap V3 integration hooks/adapters: `exactOutput`, `exactOutputSingle`, and their `_handleBefore*` helpers.
        - Path parsing sites that do `tokenIn = address(bytes20(path[:20]))` and `tokenOut = address(bytes20(path[path.length-20:]))` without considering direction.
        - Slippage/accounting code that prices `amountInMaximum` with `tokenIn` and `amountOut` with `tokenOut` using the parsed endpoints.
        - (If used) Uniswap V3 `Path` library helpers; ensure direction-aware decoding.
    - **Red flags:**
        - Same endpoint-extraction logic used for both `exactInput` and `exactOutput`.
        - No assertion that decoded `(tokenIn, tokenOut)` match the direction implied by the swap type.
        - Multihop not normalized (fees/tokens not iterated from correct end), or tests only cover `exactInput`.
        - Nonsense valuations: e.g., pricing `amountInMaximum` in the **output** token and `amountOut` in the **input** token.
    - **Fix pattern:** Decode endpoints based on swap **direction**, then pair amounts with the correct tokens.
        
        ```solidity
        // exactInput (forward): tokenIn = first20, tokenOut = last20
        address tokenIn  = address(bytes20(params.path[:20]));
        address tokenOut = address(bytes20(params.path[params.path.length - 20:]));
        
        // exactOutput (reverse): tokenOut = first20, tokenIn = last20
        address tokenOut = address(bytes20(params.path[:20]));
        address tokenIn  = address(bytes20(params.path[params.path.length - 20:]));
        
        ```
        
        - For multihop, use a direction-aware iterator (e.g., Uniswap V3 `Path` lib) to derive **first token** and **last token** correctly.
        - Validate invariants: `valueBefore = toNumeraire(amountInMaximum, tokenIn)`, `valueAfter = toNumeraire(amountOut, tokenOut)`.
        - Add tests for both single-hop and multihop `exactOutput` paths to prevent regressions.
- [ ]  **Router ABI/struct mismatch breaks swap decoding (exactInput/Output)**
    - **Symptom:** Hooks/adapters expect a swap params struct layout (e.g., includes `deadline`), but callers encode a different router version (e.g., layout **without** `deadline`). The hook’s `abi.decode` reverts or mis-reads fields, causing swap failure or incorrect slippage/accounting.
    - **Where to look (Solidity/DeFi stacks):**
        - Hook/adapter entrypoints: `exactInputSingle/exactOutputSingle` function signatures and their param struct definitions.
        - Imported router interfaces vs configured router address (versioned routers with differing tuple layouts).
        - Call sites that build calldata for these hooks (vault/guardian/ops encoders).
        - Network-specific deployments (per-chain router variants).
    - **Red flags:**
        - Single hardcoded struct type used across multiple router versions or chains.
        - No runtime compatibility checks (codehash/selector/calldata length) for the configured router.
        - Hook signatures take a **typed struct** (forcing a specific ABI) instead of normalizing from `bytes`.
        - Test coverage only on one network/variant; no cross-variant decode tests.
    - **Fix pattern:** Align ABI to the actual router variant and normalize before policy checks.
        - Standardize on one router version across deployments **or** provide versioned overloads: `...ParamsV1` (with `deadline`) and `...ParamsV2` (without).
        - Prefer `bytes swapData` at the hook boundary; decode using the configured router’s schema, then map into an internal canonical struct.
        - Add invariants/guards: assert expected function selector, tuple length, and router codehash at initialization.
        - Add per-chain tests that encode/decode both variants and run slippage/accounting on the **canonical** struct.
- [ ]  **Gross-amount slippage checks ignore protocol fees (false rejections)**
    - **Symptom:** Price checkers compute `expectedOut` from the **full** `amountIn` but settlement only swaps **netIn = amountIn − fee**. Slippage thresholds are applied to a larger notional than actually traded, so borderline-valid orders are rejected.
    - **Where to look (Solidity/DeFi stacks):**
        - Price-checker hooks/adapters for RFQ/orderflow (e.g., CoW/Milkman-style): `checkPrice(...)`, `validateQuote(...)`.
        - Call sites that pass both `amountIn` **and** a `feeAmount` but feed `amountIn` to oracle quotes/conversions.
        - Bot/solver code that pre-deducts fees to form `minOut`, vs on-chain checker that uses gross `amountIn`.
    - **Red flags:**
        - `expectedOut = oracle.quote(amountIn, fromToken, toToken)` while a `feeAmount` parameter is present/known.
        - Slippage math compares solver `minOut` (computed from **netIn**) against `expectedOut` (from **gross**).
        - “Tight slippage at fee boundary” reverts; effective slippage shrinks linearly with fee.
    - **Fix pattern:** Base quotes and slippage on **net input**.
        - Compute `netIn = amountIn − feeAmount` (clamp at 0); then `expectedOut = quote(netIn, fromToken, toToken)`.
        - Apply tolerance to `expectedOut` derived from `netIn`, and compare against solver `minOut`.
        - Keep fee handling explicit (separate var/event), add tests for varying fees at the slippage boundary, and align off-by-rounding rules (ceil/floor) across bot and on-chain checks.
- [ ]  **Compensate/Rollover lets underwater borrowers rotate debt IDs to evade liquidation**
    - **Symptom:** An underwater borrower front-runs `liquidate()` with a “compensate/rollover/split” action that mints a new debt/credit pair and marks the old debt **repaid** (e.g., `futureValue=0`) without reducing **net** debt or improving CR. Liquidators targeting the old debt ID revert; the borrower can repeat to keep DoS-ing liquidations until tenure/other gates force action.
    - **Where to look (Solidity/DeFi lending stacks):**
        - **Debt ops:** `compensate()`, `rollover()`, `refinance()`, `split/merge/migratePosition()`, multicall paths; any flow that can **create a new debt ID** while zeroing the old one.
        - **Liquidation logic:** `isLiquidatable/isUserUnderwater`, `liquidate()`, overdue rules; check whether liquidation targets **position IDs** instead of **borrower-level exposure**.
        - **Guards/modifiers:** “not underwater after” checks that don’t assert **strict** CR improvement or **net debt reduction** post-state.
        - **Position accounting:** places where old positions are marked repaid/closed while value is transferred to a new ID without enforcing repayment.
    - **Red flags:**
        - Compensation allowed with a **placeholder/RESERVED** ID to mint a fresh debt/credit pair, with **no requirement** that `netDebtAfter < netDebtBefore` or `CR_after > CR_before`.
        - Liquidation keyed to a **static debtPositionId** (no re-resolution to the borrower’s current active debt set).
        - Ability to call compensate while **eligible for liquidation**, with no freeze/cooldown or atomic repay requirement.
        - Tests prove “partial repay” but **don’t** test compensate-only front-runs vs liquidation.
    - **Fix pattern:** Make structural debt moves conditional on **real deleveraging** and/or borrower-level liquidation.
        - Enforce **post-state** invariants in compensate/rollover:
        `require(netFutureValueAfter < netFutureValueBefore)` **or** `require(CR_after > CR_before + ε)`.
        - Replace “mint-pair + optional repay” with a single **`partialRepay(...)`** that mints/splits **and** repays atomically; forbid compensate-only when underwater.
        - When `isLiquidatable(borrower)` is true, **block** structure-only moves (or require they bring CR above threshold in the same tx).
        - Let liquidators target the **borrower** (aggregate exposure) or resolve the **current active debt ID(s)** inside `liquidate(borrower, …)` at execution time.
- [ ]  **Factory uses `owner()` after renounce; post-upgrade creates always revert**
    - **Symptom:** After an upgrade that renounces ownership (`transferOwnership(address(0))`/`renounceOwnership()`), factory methods (e.g., `createMarket`, `createBorrowAToken*`) pass `owner()` into initializers. Since `owner()==address(0)`, downstream `validateOwner(owner)` / Ownable constructors revert, blocking new market/token creation.
    - **Where to look (Solidity/DeFi stacks):**
        - Factory functions that call libraries like `FactoryLibrary.create*(..., owner(), ...)`.
        - Initializer paths that `require(owner != address(0))` (e.g., `validateOwner`, Ownable/AccessControl setup).
        - Upgrade/reinitialization scripts that set `owner` to zero and any mixed use of `OwnableUpgradeable` + `AccessControl`.
    - **Red flags:**
        - Factory relies on `owner()` for **new instance** ownership after governance has renounced.
        - No alternate source of governance address (e.g., timelock) for passing to initializers.
        - Creates work on earlier versions but **always** fail post-upgrade.
    - **Fix pattern:** Decouple “factory admin” from “new instance owner”.
        
        ```solidity
        // Accept explicit owner (governance/timelock) or read from a stored `deploymentOwner`.
        function setDeploymentOwner(address gov) external onlyRole(ADMIN_ROLE) { deploymentOwner = gov; }
        
        function createMarket(/*...*/) external {
            address _owner = deploymentOwner; // must be nonzero
            require(_owner != address(0), "deployment owner unset");
            market = MarketFactoryLibrary.createMarket(sizeImplementation, _owner, /*...*/);
        }
        
        function createBorrowATokenV1_5(IPool pool, IERC20Metadata u) external {
            address _owner = deploymentOwner;
            require(_owner != address(0), "deployment owner unset");
            borrowATokenV1_5 = NonTransferrableScaledTokenV1_5FactoryLibrary
                .createNonTransferrableScaledTokenV1_5(impl, _owner, pool, u);
        }
        
        ```
        
        - Alternatively: require a nonzero `_owner` param on each call; or use a timelock/governance address stored during upgrade.
        - Add tests that simulate post-renounce state and assert creations succeed with the explicit owner.
- [ ]  **Mixed native/on-behalf deposits can leak operator ETH in batched calls**
    - **Symptom:** In a batched/multicall flow, `depositOnBehalfOf` inherits a generic `if (msg.value > 0)` branch that wraps **caller’s** ETH to WETH and deposits it, crediting the **onBehalfOf** account. If the operator also includes their own `deposit{value:...}` in the same multicall, their ETH can be unintentionally used/credited to someone else.
    - **Where to look (Solidity/DeFi stacks):**
        - Shared deposit libraries used by both `deposit()` and `depositOnBehalfOf()` (e.g., `executeDeposit`).
        - Branches gating native ETH: `if (msg.value > 0) { weth.deposit{value: ...}(); from = address(this); }`.
        - Validations like `if (msg.value != 0 && (msg.value != params.amount || params.token != WETH)) revert`.
        - Multicall implementations that forward a single `msg.value` and where `address(this).balance` may reflect prior calls in the same batch.
    - **Red flags:**
        - `depositOnBehalfOf` does **not** require `msg.value == 0`.
        - ETH path allowed without `onBehalfOf == msg.sender`.
        - Use of `address(this).balance` inside a shared deposit routine (stateful across the multicall) rather than using the exact per-call `msg.value`.
        - Tests only cover separate txs, not `depositOnBehalfOf` and `deposit{value:...}` within the **same** multicall (and in different orders).
    - **Fix pattern:** Decouple native-ETH handling from on-behalf flows and enforce invariants.
        - Make `depositOnBehalfOf` **reject** native ETH: `require(msg.value == 0, "no-native-onBehalf");` and always follow the ERC20 `transferFrom` path.
        - Provide a dedicated `depositNative(...)` (and optionally `depositNativeOnSelf(...)`) that is only valid when `onBehalfOf == msg.sender`.
        - If keeping a shared lib, split into `executeDepositNativeETH` (used by `deposit` only) and `executeDepositERC20` (used by both).
        - Prefer using **per-call** `msg.value` (not `address(this).balance`) and assert `params.token == WETH` and `msg.value == params.amount` in native path.
        - Add multicall tests covering both orders (on-behalf first vs second) to ensure the operator’s ETH can’t be redirected.
- [ ]  **Fee keyed to “partial vs full” lets borrowers split lenders’ credit without paying claim fees**
    - **Symptom:** A borrower uses compensate/offset with an **existing** credit position where `amountToCompensate == creditToCompensate.credit` (full exit of that position) but **`amountToCompensate < creditWithDebt.credit`**. The borrower avoids the “fragmentation/claim” fee, yet the lender ends up owning **more distinct credit positions** (must claim multiple times). Repeating this griefs keepers/lenders without fee payment.
    - **Where to look (Solidity/DeFi lending stacks):**
        - Debt/credit maintenance flows: `compensate/offset`, `split/merge/migrate`, and credit emission paths (e.g., `createCreditPosition`).
        - Fee gating booleans like `shouldChargeFragmentationFee` computed from **only one side** of the transfer (e.g., “is exiting position fully redeemed?”).
        - Borrow/origination paths that **mint new credit** but don’t charge a claim/fragmentation fee.
    - **Red flags:**
        - Fee charged **only** when the *exiting* position is partially redeemed, not when the *recipient lender* gains extra positions to claim.
        - Logic that treats “full exit == no fee” even when it **reassigns ownership** of a whole position to another lender (increasing that lender’s claim count).
        - Ability to create cheap “compensate fuel” via self-borrows/sybil borrows (e.g., arbitrary APR curves → near-zero swap fees), then loop compensate to fan-out positions.
    - **Fix pattern:** Charge based on **claim cost creation**, not just “partial vs full”.
        - Set fee if the operation **increases the number of claimable credit positions for any lender** (before/after delta by lender), or when ownership of a nonzero position is reassigned to a different lender.
        - Alternatively (and additionally), levy the “claim/fragmentation” fee at **loan origination** whenever new credit is minted (the moment future claim cost is created).
        - Defensive rule of thumb for compensate:
        `shouldChargeClaimFee = (amountToCompensate != creditWithDebt.credit) || (existingCompPos && creditToCompensate.lender != creditWithDebt.lender)`; and/or compute per-lender position-count delta.
        - Add tests that: (1) full-exit + partial-reduce **does** charge; (2) repeated compensate splits are fee-bearing; (3) borrow-mint increases fee accrual even without immediate splits.
- [ ]  **No signature expiry on proxy upgrades enables indefinite/time-shift & cross-chain replays**
    - **Symptom:** `setImplementation(...)` accepts an EOA signature whose typed hash lacks a `deadline/expiration`. If `allowCrossChainReplay` (or chainId=0 domain) is used and the nonce remains unused, the same signed message can be replayed **years later** or on **other chains** to upgrade a proxy unexpectedly.
    - **Where to look (Solidity/proxy stacks):**
        - Upgrade endpoints on proxies (e.g., EIP-1967/EIP-7702 wrappers): `setImplementation`, `upgradeToAndCall`, validator/authorizer modules.
        - Typed-data/Keccak encodes: `_IMPLEMENTATION_SET_TYPEHASH` fields (chainId, proxy, nonce, currentImpl, newImpl, callData, validator) — check if **deadline** is included and enforced.
        - Nonce manager: `NonceTracker` use (per-proxy? per-chain?), and whether “cross-chain” mode zeroes `chainId` in the hash.
    - **Red flags:**
        - No `deadline` or `validAfter/validBefore` in the signed struct; no `require(block.timestamp <= deadline)`.
        - “Cross-chain replay” flag zeros `chainId` (or uses a shared domain) with **global** nonces that are not synchronized across chains.
        - Guidance suggesting users “burn the nonce” to invalidate old signatures (impractical across multiple chains).
    - **Fix pattern:** Bind signatures to **time** and **domain**; fail closed when stale.
        
        ```solidity
        // Add deadline to the struct and enforce it
        bytes32 constant _IMPLEMENTATION_SET_TYPEHASH =
          keccak256(
            "ImplementationSet(uint256 chainId,address proxy,uint256 nonce,address currentImpl,address newImpl,bytes callData,address validator,uint256 deadline)"
          );
        
        function setImplementation(
          address newImpl,
          bytes calldata callData,
          address validator,
          bytes calldata sig,
          bool allowCrossChainReplay,
          uint256 deadline
        ) external {
          require(block.timestamp <= deadline, "sig expired");
          bytes32 hash = keccak256(abi.encode(
            _IMPLEMENTATION_SET_TYPEHASH,
            allowCrossChainReplay ? 0 : block.chainid,
            _proxy,
            nonceTracker.useNonce(), // or preview nonce then consume on success
            ERC1967Utils.getImplementation(),
            newImpl,
            keccak256(callData),
            validator,
            deadline
          ));
          // recover & verify...
        }
        
        ```
        
        - Prefer **per-chain** nonce spaces; if cross-chain replay is desired, let users set `deadline = type(uint256).max` explicitly.
        - Consider EIP-712 domain separation (name/version/chainId) and consuming the nonce **only on successful verification** to avoid accidental desync.
- [ ]  **Chain-agnostic upgrade/init signatures bind only addresses (cross-chain backdoor)**
    - **Symptom:** A replayable, chain-agnostic signature authorizes components by **address only** (e.g., implementation, nonce tracker, receiver, validator). On other chains those same addresses may host **different code** (proxy under new admin, metamorphic contract, CREATE instead of CREATE2/3), so the signed tuple can be replayed to wire in malicious logic.
    - **Where to look (Solidity/proxy stacks):**
        - Hash construction for upgrade/init: fields like `newImplementation`, `nonceTracker`, `_receiver`, `validator`, `allowCrossChainReplay`, `chainId==0`.
        - Deployment assumptions: use of `create` vs deterministic `CREATE2/CREATE3`, proxy vs logic contracts, upgradability, chains allowing `SELFDESTRUCT`.
        - Validators/nonce managers not domain-separated per chain.
    - **Red flags:**
        - Signature schema includes **addresses only** (no code identity) and permits `allowCrossChainReplay`.
        - No EIP-712 domain separation (name/version/chainId) and **no deadline**.
        - Dependencies are proxies or non-deterministic deployments; lack of on-chain allowlist/attestation for expected code.
    - **Fix pattern:** Bind signatures to **code identity + domain**; fail closed when ambiguous.
        - Include **code identity** for each dependency in the signed data: `extcodehash(addr)`; or better, **factory commitments** (`factory`, `salt`, `initCodeHash`) so the same bytecode must be used cross-chain.
        - Require EIP-712 domain with **chainId**; if `allowCrossChainReplay` is supported, **also** require code identity fields (addresses alone are insufficient).
        - Add optional **deadline** to limit time-shift replays.
        - On execution, assert current state: `ERC1967.implementation()==expected` **and** `extcodehash(implementation)==expectedHash`; reject if contract is a proxy with unknown admin or if code is metamorphic.
        - Maintain an on-chain **codehash/version allowlist** (governed) that signatures reference via a single `bundleHash` to simplify checks.
- [ ]  **Escrow/treasury ignores non-asset reward tokens → rewards get stuck**
    - **Symptom:** External integrations (ERC-4626 vaults, lending markets, reward distributors) pay ERC20 rewards in tokens **other than the escrow’s asset**. The escrow can receive them (via `claim/skim` or third-party transfers) but has **no logic to account, forward, or sweep** these tokens, so rewards accumulate and become inert/lost yield.
    - **Where to look (Solidity/DeFi stacks):**
        - Escrow/fee collector/treasury contracts: accepted asset type, accounting paths, and any token-rescue logic.
        - Adapters/integrations around ERC-4626 vaults and reward distributors (`claim`, `skimRecipient`, “rewards” modules).
        - Initialization/config that routes rewards to an escrow address (e.g., “skim recipient”) without downstream handling.
    - **Red flags:**
        - Hardcoded single `ASSET_TOKEN`; no handling for arbitrary `ERC20`.
        - No `claim()` implementation for external rewards or it sends to escrow with **no** follow-up processing.
        - Missing authenticated **sweep/rescue** function; no events for non-asset inflows.
        - Tests only verify principal/asset flows; no scenario where a different reward token lands in escrow.
    - **Fix pattern:** Add authenticated sweeping and explicit reward handling.
        
        ```solidity
        // Auth: only trusted roles (governance/ops)
        function sweepToken(address token, address to) external onlyAuthorized {
            require(token != address(0) && to != address(0), "bad params");
            if (token != address(ASSET_TOKEN)) {
                uint256 bal = IERC20(token).balanceOf(address(this));
                if (bal > 0) IERC20(token).safeTransfer(to, bal);
            }
            // If token == ASSET_TOKEN, treat via normal profit/accounting path.
        }
        
        ```
        
        - Implement/route `claim()` for integrated reward sources; emit events; allow optional auto-swap/forward to incentive module or treasury.
        - Guardrails: allowlist `to`/destinations; reentrancy guard; unit tests covering foreign-token receipt, sweeping, and “asset == reward” profit accounting.
- [ ]  **Unscaled decimals (eBTC 18 vs BTC assets 8) distort mint/burn math**
    - **Symptom:** `sellAsset/buyAsset` use raw `_assetAmountIn/_ebtcAmountIn` for `_ebtcAmountOut/_assetAmountOut` and slippage, ignoring decimal delta. With 8-dec assets (cbBTC/LBTC/etc.), users get wrong eBTC amounts or spurious slippage reverts.
    - **Where to look (Solidity/DeFi stacks):**
        - `EbtcBSM.sellAsset/buyAsset/_sellAsset/_buyAsset`: lines setting `ebtcAmountOut = assetAmountIn - fee` and `assetAmountOut = redeemed - fee`.
        - Fee math and constraints: `_checkMintingConstraints`, minOut checks, rate-limit/price constraints expecting 18-dec inputs.
        - Escrow integration: `ERC4626Escrow.onDeposit/onWithdraw` and any assumptions about 1e18 units.
        - Init paths: caching `EBTC_TOKEN.decimals()` and `ASSET_TOKEN.decimals()` (or lack thereof).
        - Tests: scenarios that only use 18-dec mock assets.
    - **Red flags:**
        - Direct comparisons like `assetAmountIn >= ebtcAmountOut` without unit conversion.
        - No `assetDecimals`/`ebtcDecimals` reads or stored scaler; hardcoded 1e18 arithmetic.
        - Oracle/constraint quotes assumed 18-dec while fed asset-native decimals.
        - Fees computed in one unit and subtracted from the other.
    - **Fix pattern:** Normalize amounts to a common unit before math.
        
        ```solidity
        uint8 ad = IERC20Metadata(ASSET_TOKEN).decimals();
        uint8 ed = IERC20Metadata(EBTC_TOKEN).decimals(); // expect 18
        int8  d  = int8(ed) - int8(ad);
        
        function toEbtc(uint256 assetAmt) internal view returns (uint256) {
            if (d >= 0) return assetAmt * 10 ** uint8(d);
            // round toward zero or up per spec for slippage
            return assetAmt / 10 ** uint8(-d);
        }
        function toAsset(uint256 ebtcAmt) internal view returns (uint256) {
            if (d >= 0) return ebtcAmt / 10 ** uint8(d);
            return ebtcAmt * 10 ** uint8(-d);
        }
        // sell: netIn in asset units → eBTC
        uint256 netAssetIn = _assetAmountIn - feeAsset;              // fee in ASSET units
        uint256 ebtcOut    = toEbtc(netAssetIn);                     // convert, then check caps/slippage in eBTC units
        // buy: burn eBTC → asset out
        uint256 netEbtcIn  = _ebtcAmountIn - feeEbtc;                // fee in eBTC units
        uint256 assetOut   = toAsset(netEbtcIn);
        
        ```
        
        - Compute **fees in the same unit** as the amount being netted; do not mix units.
        - Apply price/slippage limits on **normalized** values; document rounding (floor/ceil) choices.
        - Cache decimals at init; add tests for 6/8/18-dec assets and cross-check oracle/escrow paths.
- [ ]  **Inverted tBTC/BTC ratio flips minting guardrails**
    - **Symptom:** Adapter labeled “tBTC/BTC” actually returns **BTC/tBTC**, so `minPrice` checks in the constraint are evaluated in reverse (e.g., tBTC < peg → price > 1 → minting wrongly allowed).
    - **Where to look (oracles/adapters):**
        - Chainlink adapter math (`_convertAnswer`): ordering of `tBtcUsdPrice` vs `btcUsdPrice`.
        - Precision terms (`TBTC_USD_PRECISION`, `BTC_USD_PRECISION`, `ADAPTER_PRECISION`) and `description()` vs returned ratio.
        - Downstream consumers (e.g., `OraclePriceConstraint.canMint`) assuming **tBTC/BTC** in 1e18.
    - **Red flags:**
        - Formula shaped like `(btcUsd * …)/(… * tBtcUsd)` while docs/UI say “tBTC/BTC”.
        - `description()` claims tBTC/BTC but unit tests pass when `answer * reciprocal != 1e18`.
        - Constraint compares against peg (1e18) but real-market “below peg” scenarios still allow minting.
    - **Fix pattern:** Return **tBTC/BTC**, not its inverse, and assert consistency.
        
        ```solidity
        // Expect 18-dec tBTC/BTC
        int256 price = (BTC_USD_PRECISION * tBtcUsdPrice * ADAPTER_PRECISION)
                     / (btcUsdPrice * TBTC_USD_PRECISION);
        
        ```
        
        - Add invariant tests: `adapterPrice * inverseAdapterPrice ≈ 1e18` (within epsilon) and “below-peg blocks mint / above-peg allows mint.”
- [ ]  **Composite oracle uses one freshness window + `min(updatedAt)` → stale-leg acceptance/DoS**
    - **Symptom:** Aggregated price (e.g., asset/USD with BTC/USD) returns a single `updatedAt = min(feedA, feedB)` and checks it against one `oracleFreshnessSeconds`. This either (a) accepts a stale leg (long-heartbeat feed passes) or (b) reverts valid quotes (short-heartbeat feed forces too-strict window).
    - **Where to look (oracles/adapters):**
        - Adapter `latestRoundData()`: how `updatedAt` is derived across multiple feeds (min/max/one feed only).
        - Constraint code that consumes the adapter (e.g., `canMint → _getAssetPrice`): single global freshness param, single timestamp check.
        - Chainlink read paths: `latestRoundData` usage, decimals/precision normalization, `answeredInRound` checks.
    - **Red flags:**
        - `updatedAt = _min(feed1UpdatedAt, feed2UpdatedAt)` (or any single timestamp) with only one `oracleFreshnessSeconds`.
        - No per-feed staleness windows (BTC feed 1h heartbeat vs asset feed 24h).
        - Missing checks: `answeredInRound < roundId`, negative/zero price, per-feed revert reasons.
        - Tests only cover equal heartbeats or “both fresh” paths.
    - **Fix pattern:** Validate **each leg independently** with its **own freshness window**, then combine.
        
        ```solidity
        // per-feed configs
        uint256 maxAgeAsset = assetFreshnessSecs;  // e.g., 24h
        uint256 maxAgeBTC   = btcFreshnessSecs;    // e.g., 1h
        
        ( , int256 pAssetUsd, , uint256 tAsset, uint80 rA) = ASSET_USD.latestRoundData();
        ( , int256 pBtcUsd,  , uint256 tBtc,   uint80 rB) = BTC_USD.latestRoundData();
        
        require(pAssetUsd > 0 && pBtcUsd > 0, "Bad oracle price");
        require(rA >= /*roundId ok*/ 0 && rB >= 0, "Bad round");
        require(block.timestamp - tAsset <= maxAgeAsset, "Asset feed stale");
        require(block.timestamp - tBtc   <= maxAgeBTC,   "BTC feed stale");
        
        // compute asset/BTC with proper precision after both legs pass
        uint256 price = uint256(pAssetUsd) * ADAPTER_PREC / uint256(pBtcUsd) * …; // normalize precisions
        // Optionally surface both timestamps; if you must return one, keep internal per-feed checks and return min(tAsset, tBtc) only for display.
        
        ```
        
        - Make `assetFreshnessSecs` and `btcFreshnessSecs` **separately configurable**; default them to each feed’s heartbeat/deviation policy.
        - Add unit tests for: (asset fresh, BTC stale) → revert; (asset stale, BTC fresh) → revert; (both fresh but different ages) → pass; boundary cases at exact thresholds.
- [ ]  **Migration assumes ERC4626 `redeem()` never reverts → stuck funds / blocked upgrades**
    - **Symptom:** `updateEscrow`/migration reverts when the external vault can’t redeem all shares (paused/frozen, insufficient liquidity so `maxRedeem(owner) < shares`, or `previewRedeem == 0`). Funds sit in the old vault; migrations and sometimes buys that call internal “ensure liquidity” start failing indefinitely.
    - **Where to look (4626 integrations/migration flows):**
        - Escrow migration hooks: `onMigrateSource/_beforeMigration` and any call that does `redeem(EXTERNAL_VAULT.balanceOf(this), …)`.
        - External vault adapters (Aave/Euler/Morpho): pause/freeze flags, `maxRedeem/maxWithdraw` logic, liquidity checks, exit fees.
        - Buy/sell paths that implicitly redeem to raise liquidity.
    - **Red flags:**
        - Blind `redeem(balanceOf(this))` with no check against `maxRedeem`, no partial/streamed exit, no retry window.
        - Migration hard-requires redemption in a single tx; vault address change is impossible if `redeem` reverts.
        - No handling for `assets == 0` / zero-shares cases; no circuit breaker or alternate path.
        - Escrow can’t transfer share tokens (ERC20) to a new escrow/migrator; only redeems.
    - **Fix pattern:** Decouple “who holds shares” from “when to redeem”.
        - First move **shares**, not assets: transfer all 4626 shares to the new escrow (or a designated migrator) and switch the pointer; redeem later when vault is liquid/unpaused.
        - Support **partial exits**: cap to `min(balanceOf(this), maxRedeem(this))`, loop/queue/redrive over time; handle `0` cases gracefully.
        - Add emergency paths: admin share transfer, pause buy paths that force redeem, and clear, observable migration states with retries/backoff.
- [ ]  **Contradictory array-length guard makes Permit2 order creation unreachable**
    - **Symptom:** Function requires `details.length == 0` and then immediately reads `details[0]`, so it always reverts and the Permit2 “create order” path can’t be used.
    - **Where to look (routers/permit flows):**
        - Order creation helpers that accept `PermitBatch`/`PermitSingle` structs.
        - Parameter validation blocks before any array indexing.
        - Any code that both checks `.length == 0` (or `!= 0`) and then indexes `[0]`.
    - **Red flags:**
        - Guards like `if (arr.length != 0) revert ...;` followed by `arr[0]` access.
        - Mismatched invariants: using `!= 0` instead of `== 1` for single-element batches.
        - Comments saying “must be 1 element” while code enforces “0”.
        - Lack of fuzz/unit tests for `len ∈ {0,1,>1}`.
    - **Fix pattern:** Align the guard with the access pattern and add tests.
        - If single-item permit is intended: `require(details.length == 1, ArrayLengthsMismatch());`
        - If variable size is allowed: `require(details.length > 0, EmptyArray());` then loop; never index `[0]` without a prior `> 0` check.
        - Add invariant tests for `0/1/N` elements and property tests that assert “no revert on valid lengths; revert on invalid”.
- [ ]  **Native refund to non-payable receiver enables dust-DoS of order fills**
    - **Symptom:** Order fill reverts on the final “refund leftover ETH to `receiver`” step; a griefer can front-run by dusting the executor so `address(this).balance > 0`, and if `receiver` lacks a payable `receive/fallback`, the refund fails → permanent DoS for that order.
    - **Where to look (routers/executors):**
        - Post-execution “refund native balance to receiver” blocks in proxy/multicall routers and settlement hooks.
        - Any function that does `uint256 bal = address(this).balance; if (bal != 0) receiver.call{value: bal}("")`.
        - Order structs allowing arbitrary `receiver` without capability checks.
    - **Red flags:**
        - Unconditional native refund to `receiver` with `revert` on failure.
        - No WETH wrapping / sweep fallback; no per-receiver crediting.
        - No validation that `receiver` can accept ETH; no dust-grief mitigation.
        - Assumption that the executor’s native balance is always zeroed by downstream calls.
    - **Fix pattern:** Make refunds non-blocking and dust-proof.
        - Attempt refund; on failure, **wrap to WETH and transfer** to `receiver` (preferred) or **sweep to a protocol escrow** for later claim.
        `if (bal != 0) { (ok,) = receiver.call{value:bal}(""); if (!ok) { WETH.deposit{value:bal}(); WETH.safeTransfer(receiver, bal); } }`
        - Or adopt a **pull model**: credit native owed and expose `claimNative(receiver)`; never revert the fill on refund failure.
        - **Preempt dust grief:** immediately wrap any incidental ETH into WETH before/after external calls so terminal native balance is 0; or add a minimal-dust sweep to treasury instead of reverting.
        - **Order hygiene:** optionally restrict/validate `receiver` (must be EOA or payable) at order creation; emit events on fallback path for operability.
- [ ]  **Incorrect signer flag for non-signer account blocks Solana order reverts**
    - **Symptom:** Revert-order txs fail with “missing required signature / signature verification failed,” leaving orders stuck unless the trader also signs (which the flow never supplies).
    - **Where to look (Solana/SVM orchestration & clients):**
        - Tx builders that craft `AccountMeta[]` for revert/cancel flows (TypeScript/JS SDKs).
        - Keys passed to `VersionedTransaction`/`TransactionMessage` for “revert”/“cancel” instructions.
        - Anchor IDL / program accounts for the same instruction (`pub trader: AccountInfo<'info>` or `isSigner: false`).
    - **Red flags:**
        - `AccountMeta { pubkey: trader, isSigner: true, ... }` while program expects a plain account (`AccountInfo<'info>`, not `Signer<'info>`).
        - Client key list not generated from the IDL; hard-coded metas drifting from on-chain constraints.
        - Only orchestrator signs, yet client marks additional metas as `isSigner: true`.
    - **Fix pattern:** Make client metas match the program.
        - Set trader (and any non-signer accounts) to `isSigner: false`; require only the intended signer(s) (e.g., orchestrator PDA/EOA).
        - Prefer generating `AccountMeta`s from the Anchor IDL to avoid drift; add a unit test that asserts `ix.keys` signer flags match IDL.
        - Add a simulation check in CI that builds the revert tx and `simulateTransaction` passes without the trader signature.
- [ ]  **User-supplied `order_hash` on Solana isn’t validated → hash drift DoS on fills/reverts**
    - **Symptom:** Orders created on Solana can use an arbitrary `order_hash` that doesn’t match the canonical hash of the order fields. The orchestrator computes the hash from data (EVM-style) and then:
        - Source/dest status checks don’t line up (`Created/Nonexistent` filter fails), so fills are skipped.
        - Revert TX builder derives a PDA from the computed hash that doesn’t exist → revert cannot be submitted.
        - An attacker can front-run by pre-creating a conflicting order account at the expected hash, causing persistent DoS for the real order.
    - **Where to look (Solana + orchestrator):**
        - Solana program: `CreateOrder` instruction logic and account layout (stores `order_hash` without recomputing).
        - PDA derivation helpers (e.g., `AddressUtils.getOrderAddress(orderHash, programId)`).
        - Orchestrator: `SolverFactory.getOrderHashesToCheck`, `getOrderStatuses`, and any `GeniusEvmVault.getOrderHash` usage for Solana lookups; revert builder `getRevertOrderTx`.
    - **Red flags:**
        - `order_hash` accepted as a parameter and written on-chain; PDA seeds use that value directly.
        - No on-chain recomputation of `keccak256(canonical(orderFields))` and equality check.
        - Cross-chain code assumes a single canonical hash but clients create/read Solana accounts by a user-provided hash.
        - Filters like `sourceStatus === Created && destStatus === Nonexistant` silently excluding affected orders.
    - **Fix pattern:** Make Solana the source of truth for the hash; clients must follow it.
        - In the Solana program, **drop `order_hash` as an input**; recompute a **canonical hash** on-chain from the exact ordered fields (include chain IDs, tokens, amounts, minOut, seed, receiver, trader). Derive the **order PDA from that computed hash**, store the same hash in the account, and **emit it in an event**.
        - Ensure **identical serialization** across chains (define a single spec: field order, types, endian, and whether it’s `abi.encode`/packed/EIP-712 or Borsh). Add tests that EVM and Solana hashes match for fuzzed inputs.
        - Update orchestrator/SDK:
            - Use the **program-computed hash** (read from account or event) for status checks and PDAs; do not trust user-supplied hashes.
            - When building revert/fill TXs, derive addresses from the canonical hash only.
        - Add guards: creation must **revert** if a client tries to pass a mismatching hash (temporary compatibility path), and unit tests that “wrong hash” orders cannot be created.
- [ ]  **Delta-on-self, Transfer-to-Other (Permit2/Proxy misroute DoS)**
    - **Symptom:** Function computes `amountIn` via `delta = token.balanceOf(address(this)) - before`, but the permit/transfer helper sends funds to a different address (proxy/escrow). `delta` stays `0` → `amountIn == 0`/`<= fee` checks revert and the flow is effectively DoS’ed.
    - **Where to look (Solidity / Permit2 routers):**
        - Router entrypoints that have both “swap+create” and “create-only” paths (e.g., `swapAndCreateOrderPermit2` vs `createOrderPermit2`).
        - Permit2 helpers: calls to `IAllowanceTransfer.permit*/transferFrom` wrappers; any `_permitAnd*Transfer` utilities.
        - Balance-delta accounting sites: patterns like
            - `before = token.balanceOf(address(this)); … delta = token.balanceOf(address(this)) - before;`
            - fee math using `delta`.
        - Hardcoded destinations inside helpers: `to: address(PROXYCALL)` / escrow / adapter while caller later reads `balanceOf(address(this))`.
        - Adjacent modules receiving funds: `ProxyCall`, `Escrow`, `Aggregator`, `Settlement` contracts.
    - **Red flags:**
        - Helper constructs `AllowanceTransferDetails{ to: <not this> }` or `safeTransferFrom(owner, <not this>, amount)` and is reused by multiple flows.
        - Immediately after helper: computation of `delta` on the router (`this`) and usage of `delta` as `order.amountIn` or for fee checks.
        - Guards like `if (order.amountIn == 0 || order.amountIn <= order.fee) revert InvalidAmount();` firing on “create-only” path but not on “swap” path.
        - Comments/branches implying “send to proxy for swaps” reused unmodified for “create-only”.
        - Unit tests only covering the swap path; no test asserting `delta > 0` for create-only Permit2 flow.
        - Mixed native/erc20 flows where ETH goes to proxy but `balanceOf(this)` (WETH/erc20) is measured on the router.
        - **Questions to ask while reviewing:**
            - “Where do the tokens actually land for this code path?” (router vs proxy vs escrow)
            - “Is the destination address parameterized per path, or hardcoded in a shared helper?”
            - “Is `amountIn` derived from requested input, or from `delta` on the *same* address that received funds?”
            - “Do fee/slippage checks rely on `delta`? What happens if `delta == 0`?”
            - “Are there tests for create-only Permit2 path validating non-zero `delta` and correct fee math?”
    - **Fix pattern:**
        - **Make destination explicit:** Update the permit/transfer helper to accept a `destination` and pass `address(this)` for create-only paths; keep proxy for swap paths.
        - **Align measurement with receiver:** If funds must go to proxy/escrow, compute `delta` on that receiver (or have the receiver report back the received amount).
        - **Prefer pull-pattern:** Approve the router and `transferFrom` into the router for create-only flows; reserve proxy funding for swap execution.
        - **Harden with guards/tests:** Add `require(delta > 0, "ZeroDelta/Misrouted")` post-transfer; unit test both paths (with/without proxy) and deflationary tokens so fee/amount use `delta`, not user-provided amount.
- [ ]  **Seed/nonce mismatch between on-chain and off-chain (truncated-hash seeds)**
    - **Symptom:** Off-chain solver/orchestrator rejects otherwise valid orders with “Seed does not match” when `arbitraryCall` is present. On-chain contract derives `reconstructedSeed = keccak256(target, keccak256(data))` but only checks `bytes16(reconstructedSeed) == bytes16(order.seed)` (the other 16 bytes are a nonce), while off-chain code compares the **entire** `bytes32` value. Result: fills/reverts are DoS’d whenever the nonce is non-zero or seeds are intended to be reusable.
    - **Where to look (Solidity + off-chain TS/Go):**
        - On-chain: functions that validate seeds (e.g., `fillOrder`, `execute*`, or libs): look for `bytes16(reconstructed) != bytes16(order.seed)` patterns and how `order.seed` is constructed (`bytes16(hash) || bytes16(nonce)`).
        - Off-chain services/solver: seed reconstruction utilities like `calldataToSeed(Sync)` in files akin to `services/blockchain/vault/*-vault.ts` and checks in `actions/solver/core/*` (e.g., `SolverBase.ts`), especially code that does `calldataToSeed(...) !== order.seed`.
        - Cross-chain glue (EVM ↔ SVM): any place computing `orderHash/seed` twice—once in contracts and once in off-chain—ensure the same truncation/packing is used.
        - Test fixtures: contract tests that deliberately pack seeds as `bytes16(keccak256(...)) || bytes16(nonce)` vs off-chain tests comparing full `bytes32`.
    - **Red flags:**
        - Off-chain compares the full `bytes32` (`===`/`!=`) while on-chain masks to `bytes16`.
        - Off-chain ignores/omits nonce in the seed or lacks a mask (`bytes16`) when comparing.
        - Mixed encodings: on-chain `abi.encodePacked(target, keccak256(data))` vs off-chain different packing (e.g., `ethers.utils.solidityKeccak256(["address","bytes32"], ...)` but then comparing full 32 bytes to a seed that embeds a nonce).
        - Hex/string slicing bugs: comparisons on hex strings including the `0x` prefix or wrong substring lengths (e.g., not accounting for 2 chars/byte).
        - Inconsistent comments: contract docs state “first 16 bytes from hash, last 16 bytes nonce” but client code says “seed = keccak256(target|calldata)”.
        - Repeated orders with same calldata fail unless nonce is zero.
    - **Questions to ask next time:**
        - “Is `order.seed` a **composite** (hash prefix + nonce) or a full hash? Which bytes are validated on-chain?”
        - “Do off-chain builders **mask** to `bytes16` when checking, and do they append a nonce when generating seeds?”
        - “Are the hashing domains identical? (packed vs ABI-encode, address normalization, data pre-hashing).”
        - “Do tests cover (a) non-zero nonces, (b) repeat orders with identical calldata, and (c) cross-chain orders?”
        - “Are we comparing bytes or hex strings? If strings, are slice lengths correct (34 chars for 16 bytes with `0x`)? ”
    - **Fix pattern:**
        - **Align comparison semantics:** Off-chain should compare only the validated portion: `bytes16(reconstructedSeed) == bytes16(order.seed)` (masking in TS/Go by slicing first 16 bytes or applying a `0xffff...` mask).
        - **Generate seeds consistently:** Off-chain seed = `bytes16(keccak256(target, keccak256(data))) || bytes16(nonce)` to match contract packing.
        - **Unify hashing:** Use the same encoding as Solidity: `solidityPackedKeccak256(["address","bytes32"], [target, keccak256(data)])`.
        - **Test hardening:** Add e2e tests with non-zero nonce and multiple identical `arbitraryCall`s; assert fills succeed both on EVM and any SVM bridge.
        - **Defensive checks:** If full-seed equality is desired off-chain, explicitly strip/compare only the high 16 bytes and separately verify/track the nonce component.
- [ ]  **Swap recipient bypasses escrow/minOut checks (receiver set to user instead of orchestrator)**
    - **Symptom:** Off-chain solver builds a swap where `receiver` is the **user**, but the settlement flow expects the **orchestrator** to hold `tokenOut` and then transfer to the user under a `minAmountOut` guard. Result:
        1. swap sends `tokenOut` directly to the user;
        2. settlement tries to transfer from orchestrator’s ATA (balance = 0) → program reverts on min-out/zero-amount;
        3. order is DoS’d and the user may already have received tokens outside protocol checks.
    - **Where to look (Solana solver + program):**
        - Off-chain: `actions/solver/implementations/SolanaSolver.ts`
            - `fillOrderSvm` → `getSolanaSwapTx(...)`
        - Off-chain swap util: `utils/swap/fill-order-swap-quote-util.ts`
            - Parameters to aggregator/quote: `from`, `receiver`, `tokenIn`, `tokenOut`, `amountIn`
            - Watch for `receiver = order.receiver` instead of orchestrator/PDA
        - Solana pool/tx builder: `services/blockchain/vault/solana/svm-transaction-builder.ts`
            - Balance fetch for `orchestratorTokenOutATA` before building `FILL_ORDER_TOKEN_TRANSFER`
            - Any `previousBalance`/delta encoding in instruction data
        - On-chain Solana program: `instructions/fill_order_token_transfer.rs`
            - Assumes tokens are **in orchestrator ATA** and enforces `min_amount_out`
    - **Red flags:**
        - Quote/fetch call sets `receiver` to the **user** while the next step transfers from the **orchestrator** (mismatched custody).
        - Comments like `// TODO: set adequate slippage` paired with direct delivery to user (no program-side `minOut` guard on the swap leg).
        - Balance probe returns `0` for orchestrator’s tokenOut ATA immediately before building the transfer instruction.
        - Multiple “receiver/beneficiary/to” fields across SDKs (Jupiter/Orca/aggregator) inconsistently assigned.
        - Settlement logic relies on “increase from previousBalance” but assets never land in the orchestrator’s ATA.
    - **Questions to ask next time:**
        - “Who should custody `tokenOut` between swap and settlement—user, orchestrator, or a program-owned PDA? Where is `minOut` actually enforced?”
        - “Does the aggregator enforce a `minOut` compatible with our order’s `minAmountOut`? If yes, why do we run a second guarded transfer? If no, why do we deliver directly to the user?”
        - “Are ATAs for orchestrator/PDA pre-created for `tokenOut` so the swap can actually deposit there?”
        - “If the swap leg fails or under-delivers, what prevents partial success (tokens to user) with a reverted settlement?”
        - “Do we have tests that: (a) set `receiver` to user; (b) verify orchestrator ATA delta; (c) verify `minAmountOut` guard fires correctly?”
    - **Fix pattern:**
        - **Custody-first:** Set swap `receiver` to the orchestrator (or a program-owned PDA/escrow) so `tokenOut` lands under protocol control, then run `FILL_ORDER_TOKEN_TRANSFER` to the user with `minAmountOut` enforced.
        - **Single-responsibility alternative:** If aggregator must deliver to user, pass and enforce the **order’s `minAmountOut`** within the swap transaction and **skip** the subsequent program transfer step (or mark the order filled accordingly).
        - **Pre-assertions & tests:** Before requesting the quote/route, assert `receiver == orchestratorOrEscrow`; unit/e2e tests that fail when set to `order.receiver`.
        - **Balance checks:** After swap, read orchestrator’s ATA balance and require it increased by ≥ `minAmountOut` before building the transfer instruction; otherwise abort orchestration.
- [ ]  **Native-token sentinel not branched: ERC20 calls on ETH cause hard reverts**
    - **Symptom:** Functions that accept a `token` parameter may receive a native-token sentinel (e.g., `NATIVE_TOKEN`, `address(0)`, `0xEeeee...`). The code still executes `IERC20(token).balanceOf/transfer/approve`, which **reverts** for native ETH/MATIC/etc. Typical pattern: helper does `IERC20(token).balanceOf(address(this))` then `safeTransfer(...)`, followed by `target.call{value: msg.value}(data)`. When swaps output native coin and a subsequent “transfer-and-execute” step runs, the call consistently fails.
    - **Where to look (routers/proxies/solvers):**
        - Token transfer helpers: `transferTokenAndExecute`, `sweep/withdraw`, “proxy call” modules, multicall routers, order-fill paths, bridge adapters.
        - Unwrapping/wrapping utilities: any use of WETH/WMATIC vs. native coin (look for `deposit()`/`withdraw()` calls).
        - Parameters flowing from off-chain/aggregator: swap quote builders that can set `tokenOut` to native.
        - Code smells:
            - Unconditional `IERC20(token).balanceOf(address(this))`, `safeTransfer`, `approve` where `token` is user-controlled or comes from swap.
            - A defined `NATIVE_TOKEN`/sentinel constant that is **never** checked in that function.
            - Using `address(0)` as a pseudo-token in ERC20 calls.
            - Forwarding `msg.value` to `target.call` without aligning it to actual native balance or the intention of the step.
    - **Red flags:**
        - Helper signature `transferTokenAndExecute(address token, address target, bytes data)` with no `if (token == NATIVE_TOKEN)` branching.
        - Comments like “supports native token” but no explicit branch/wrap.
        - Mixed semantics: sometimes treating `WETH` as “native”, other times treating `NATIVE_TOKEN` as “native”.
        - Refund/sweep blocks that assume “no native left” yet perform ERC20 ops first, or vice-versa.
    - **Questions to ask next time:**
        - Can `tokenOut` ever be the chain’s native coin from our swap/bridge path? Who sets that?
        - What sentinel represents native in this codebase? Is there a single `isNative(token)` helper and is it used **everywhere**?
        - If the next step expects ETH, do we unwrap WETH first? If the next step expects ERC20, do we forbid native or auto-wrap?
        - Does `msg.value` equal the native amount we intend to pass? Are we accidentally forwarding `address(this).balance` broadly?
        - Are there unit/e2e tests where `tokenOut == NATIVE_TOKEN` through the entire flow (swap → transfer → external call)?
    - **Fix pattern:**
        - **Branch explicitly on native:**
            
            ```solidity
            if (token == NATIVE_TOKEN) {
                // do NOT call ERC20 methods
                // optionally: wrap to WETH or forward native via {value: amount}
            } else {
                uint256 bal = IERC20(token).balanceOf(address(this));
                if (bal != 0) IERC20(token).safeTransfer(target, bal);
            }
            
            ```
            
        - Normalize flows: prefer receiving **WETH** from aggregator and unwrap only when truly needed; or enforce native-only paths with no ERC20 ops.
        - Centralize `isNative(token)` and use it in **all** transfer/sweep/execute helpers.
        - Align value forwarding: only send `{value: x}` when native is intended and ensure `x` equals the native amount to use.
        - **Tests:** add cases where swap outputs native; assert the function no longer reverts and the external call succeeds with the right funds.
- [ ]  **Param-order mismatch between orchestrator SDK and Solidity ABI (target/data swapped)**
    - **Symptom:** Off-chain builder passes `(data, to)` where the contract expects `(target, data)` (or vice-versa) for calls like `rebalanceLiquidity(uint256 amountIn, uint256 dstChainId, address target, bytes data)`. Transactions revert at execution time (often with `abi.decode`/`calldata` length errors or custom errors) and entire flows (rebalancing/bridging/meta-calls) DoS.
    - **Where to look (Routers/Orchestrators/SDKs):**
        - TypeScript/JS “actions” layers: rebalancing executors, solver/orchestrator code, quote→tx mappers (e.g., `getEvmRebalancingTxns`, `prepRebalanceLiquidity`, `executeSolver`).
        - Places that forward aggregator payloads: `quote.evmExecutionPayload.transactionData.{to,data}` → wrapper methods that call `populateTransaction.<fn>(..., target, data)`.
        - ABI wrappers and encode sites: `Interface.encodeFunctionData`, `populateTransaction.<fn>`, raw `ethers.Contract(...).functions.<fn>`.
        - Contract side: function signatures that accept `(address target, bytes data)` or similar `(token, target, data)` triplets.
    - **Red flags:**
        - Variable names like `to`/`target` and `data` being passed in opposite order to the ABI.
        - Wrapper signatures that differ from contract ABI order (e.g., `prepRebalanceLiquidity(amount, chainId, data, to)`), then forwarded 1:1 into `rebalanceLiquidity(amount, chainId, target, data)`.
        - Copy-pasted call sites from other chains/paths (Solana/EVM) reusing `{to,data}` but different ABI order.
        - Lack of e2e tests that actually execute the produced calldata on a fork (tests only mock return values).
        - Reverts only in specific networks or only on EVM path while other paths succeed.
    - **Questions to ask:**
        - What is the exact Solidity signature (order and types) of the callee? Do all SDK helpers mirror that order?
        - Where do `to` and `data` originate (aggregator/quote)? Are we preserving their order consistently to the ABI?
        - Do we have a fork test that takes the built tx data and performs a dry-run `callStatic` against the live/forked contract?
        - Are TypeChain/ethers types enforced, or are we passing `any`/`unknown` blobs around?
        - Are bigint/hex string conversions altering the tuple positions (destructuring mistakes)?
    - **Fix pattern:**
        - Align wrapper function parameters to the ABI and enforce with strong typing (TypeChain):
        `prepRebalanceLiquidity(amountIn, dstChainId, target, data)` → forward as `rebalanceLiquidity(amountIn, dstChainId, target, data)`.
        - When mapping from quotes, **explicitly** pass `transactionData.to` into `target` and `transactionData.data` into `data`:
            
            ```tsx
            const { to, data } = quote.evmExecutionPayload.transactionData;
            const txnData = await vault.prepRebalanceLiquidity(amount, dstChainId, to, data);
            
            ```
            
        - Add a sanity check before send: decode the calldata and assert decoded args match intended `{target, data}`.
        - Introduce fork-based CI tests that build the tx via SDK, then `callStatic` the on-chain method to catch ordering bugs.
        - Centralize a helper `mapQuoteToTargetData(quote)` used everywhere to avoid one-off parameter order drift.
- [ ]  **Unbounded swap-path lets users explode work (DB/CPU) → node DoS**
    - **Symptom:** Requests with very long (or cyclic) `path` arrays make routers/quoters iterate `O(N)`–`O(2N)` times (simulate + execute), hammering `getReserves`/pair lookups and writing per-hop history. On chains without strict gas metering for queries (or where queries are free), nodes time out, spike CPU/IO, or crash; on EVM, txs just OOG but off-chain services (quoters/relayers) can stall.
    - **Where to look (DEX routers & quoters):**
        - Solidity AMM clones (UniswapV2/V3, Solidly, Camelot, Sushiswap)
            - Router: `swapExactTokensForTokens`, `swapTokensForExactTokens`, `swapExactETHForTokens`, etc.
            - Quoters/helpers: `getAmountsOut/In`, `quoteExactInput/Output`, any route-building function that loops `for (i = 0; i < path.length - 1; i++)`.
            - Off-chain services: REST/WebSocket quoters, path finders, and keepers that accept user-supplied `path` or hop lists.
        - Chromia/Substrate/custom L2 backends (DB-backed execution):
            - Path resolution: `from_assets_symbol_to_assets(path)` → per-symbol SELECT.
            - Reserve fetch & execution: `get_amounts_out/in`, `_swap` loops, and query endpoints like `query_get_amounts_out/in`, `calculate_price_impact` that accept `path`.
            - Files similar to `src/contracts/uniswap/operation.rell` and any query handlers that take `list<text>` token symbols.
    - **Red flags:**
        - No `require(path.length <= MAX_HOPS)` and no minimum of 2 tokens.
        - No checks against repeated/cyclic tokens (e.g., `A→B→A→B…`).
        - Separate loops for simulation and execution, each doing `(N-1)` pair lookups and DB writes.
        - Query/API endpoints that process arbitrary `path` with no auth/rate-limit/body-size caps (or rely only on large HTTP body limits).
        - Per-hop history INSERTs or logging inside the loop.
        - Path symbol → asset resolution done *inside* hot loops without caching.
    - **Questions to ask:**
        - What’s the *hard* max hops allowed by on-chain and by API? Is it enforced at all entry points (state-changing and view/query)?
        - Do we explicitly reject cycles or repeated adjacent tokens?
        - Are quote endpoints free to call? If yes, what throttling/backpressure and per-request complexity limits exist?
        - How many DB SELECT/INSERTs happen per hop? Are we deduping/caching `asset` and `pair` lookups across simulation and execution?
        - Can an attacker submit tiny-amount swaps with massive paths to pay negligible fees but force maximal work?
        - Is there centralized path validation (shared helper) or duplicated logic that might miss some endpoints?
    - **Fix pattern:**
        - **Bounded complexity:** `require(path.length >= 2 && path.length <= MAX_HOPS)` (e.g., 5–10). Enforce in *all* functions that accept `path` (routers, quoters, and queries).
        - **Cycle/dup guards:** Reject paths with immediate repeats (`path[i] == path[i+1]`) and long cycles (track a small seen-set up to `MAX_HOPS`).
        - **Early validation:** Pre-validate all `(path[i], path[i+1])` pairs exist; abort on first missing pair.
        - **Cache & batch:** Cache symbol→asset and pair→reserves within request; avoid repeated SELECTs. For DB-backed systems, batch reads or use precomputed indexes.
        - **Rate-limit & budget:** Add per-IP/user rate limits and per-request “work budget” on query endpoints; enforce smaller HTTP/body limits for `path`.
        - **Separate simulation from writes:** Keep heavy logging/history out of per-hop loops; aggregate and write once.
        - **Testing:** Add fuzz/property tests that generate large and cyclic paths to assert: (1) rejection by validation, (2) bounded runtime under max path, (3) no per-hop DB write explosion.
- [ ]  **MINIMUM\_LIQUIDITY not permanently locked → donation-based mint dilution/DoS**
    - **Symptom:** After the first liquidity add, a whale can mint a trivial LP amount (e.g., 1 wei) and then “donate” large token balances directly to the pair/treasury. Because totalSupply stays tiny (MINIMUM\_LIQUIDITY wasn’t locked into supply), subsequent `mint` calculations floor to 0 (`INSUFFICIENT_LIQUIDITY_MINTED`), blocking new LPs; if the whale later burns the tiny LP, swaps can underflow/zero-liquidity.
    - **Where to look (Uniswap V2 forks / AMMs):**
        - Pair contract `mint()` branch when `totalSupply == 0`.
            - Solidity: `UniswapV2Pair.mint`, or clones’ pair manager `mint` logic.
            - Non-EVM (e.g., Rell/Move/Rust): the first-mint path that computes `liquidity = sqrt(amount0*amount1) - MINIMUM_LIQUIDITY`.
        - Token accounting around MINIMUM\_LIQUIDITY:
            - Is `MINIMUM_LIQUIDITY` **minted to `address(0)` (dead) and left there** (totalSupply includes it), or minted then **burned** (reducing totalSupply back)?
            - Any custom “treasury”/“vault” address receiving and then **burning** MINIMUM\_LIQUIDITY?
        - Donation flow:
            - Ability to transfer tokens directly to the pair/treasury and call `sync()` without minting LP shares.
    - **Red flags:**
        - Code mints `MINIMUM_LIQUIDITY` and then immediately **burns** it (total supply unchanged).
        - `MINIMUM_LIQUIDITY` minted to a non-dead account (e.g., treasury) and then burned.
        - No comment/reference to “permanently lock” MINIMUM\_LIQUIDITY; or locking implemented as burn instead of dead-mint.
        - `mint()` uses `min(amount0 * totalSupply / reserve0, amount1 * totalSupply / reserve1)` with **very small `totalSupply`** possible post-donation (prone to rounding to 0).
        - Router allows first LP to be almost zero; no min-liquidity guard.
        - Presence of `sync()` that updates reserves without minting shares for donations.
    - **Questions to ask during review:**
        - On first mint, where do MINIMUM\_LIQUIDITY LP tokens end up? Are they **minted to `address(0)` and never burned?**
        - Does `totalSupply` after first mint include MINIMUM\_LIQUIDITY? (It should.)
        - Can anyone donate tokens to the pair/treasury without receiving LP, and does `sync()` accept that? What protects later LP mints from flooring to 0?
        - What minimum LP amount must the first provider mint? Any guard against tiny first LP (e.g., require `liquidity > X`)?
        - Are there tests for the scenario “tiny first LP + large donation → second LP mint”?
        - If the initial LP exits, can reserves be left >0 with `totalSupply == 0` causing swap/mint division-by-zero or zero-liquidity reverts?
    - **Fix pattern:**
        - **Lock** MINIMUM\_LIQUIDITY by minting it to the **dead address** (e.g., `address(0)` on EVM) and **do not burn** it:
            - `liquidity = sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;`
            - `_mint(address(0), MINIMUM_LIQUIDITY);` (no `_burn` afterwards)
            - Require `liquidity > 0` and mint remainder to `to`.
        - Optionally enforce a **minimum first-liquidity threshold** to avoid pathological tiny first LP.
        - Consider mint-on-donation or donation caps (advanced): prevent `sync()`only reserve growth without proportional LP minting.
        - Add tests: (1) tiny-first-LP then donation should still allow nonzero mint for second LP; (2) removing initial LP must not leave pool stuck or swaps reverting.
    - **Why this happens:** Uniswap V2 relies on permanently locked MINIMUM\_LIQUIDITY to keep `totalSupply` sufficiently large from genesis, so donation-driven reserve explosions don’t collapse new-mint calculations to zero via integer division/rounding. Burning the minimum undoes that invariant and enables a whale to grief later LPs.
- [ ]  **Emergency-withdraw path skips supply bookkeeping → “ghost shares” dilute rewards**
    - **Symptom:** After `emergencyWithdraw`, other stakers earn less than they should because the pool’s `totalSupply/lpSupply/shares` wasn’t decreased; `accRewardPerShare` uses an inflated denominator.
    - **Where to look (MasterChef/Staking Rewards patterns):**
        - Solidity MasterChef-like contracts:
            - Pool struct fields: `totalSupply`, `lpSupply`, `shares`, `accRewardPerShare`, `lastRewardTime/Block`.
            - Functions: `deposit`, `withdraw`, **`emergencyWithdraw` / `emergency_withdraw` / `withdrawEmergency`**, `updatePool`.
            - Any external Rewarder module reading `pool.lpSupply` or `stakingToken.balanceOf(address(this))`.
        - Non-EVM staking code (Rell/Rust/Move):
            - Reward denominator kept in pool state vs. derived from actual token balance.
            - “Panic/exit/emergency” branches that bypass reward debt updates.
    - **Red flags:**
        - `deposit` does `pool.totalSupply += amount` and `withdraw` does `pool.totalSupply -= amount`, but `emergencyWithdraw` **only**:
            - transfers tokens back,
            - sets `user.amount = 0` (and maybe `user.rewardDebt = 0`),
            - **does not** decrement `pool.totalSupply/lpSupply`.
        - `updatePool` computes `reward * PRECISION / lpSupply` where `lpSupply` comes from **stored** pool state, not `stakingToken.balanceOf(this)`—making it susceptible to ghost shares.
        - Mixed accounting sources: some paths use `pool.totalSupply`, others use `stakingToken.balanceOf(this)`.
        - Exit/cancel/migrate/slash/rescue functions that zero user position but don’t touch pool supply.
        - Comments like “no rewards on emergency” without explicit supply adjustment.
    - **Questions to ask next time:**
        - What is the **single source of truth** for the reward denominator—internal `lpSupply` or on-chain token balance? Is it used **consistently** across all code paths?
        - Do all position-changing functions (withdraw, emergency withdraw, migrate, slash, liquidate) **mirror** the `pool.totalSupply` delta?
        - Does `updatePool` handle `lpSupply == 0` correctly (advance `lastRewardTime` and return) to avoid division by zero and reward leaks?
        - Are there external Rewarders relying on the same denominator? Do their emergency/exit paths adjust supply?
        - Are there tests where User A emergency-withdraws, time passes, and User B’s rewards equal 100% of emissions during that window?
    - **Fix pattern:**
        - In `emergencyWithdraw`:
            - `uint256 amt = user.amount;`
            - `pool.totalSupply -= amt;` (or decrement `shares`/`lpSupply`—whichever the denominator uses)
            - `user.amount = 0; user.rewardDebt = 0;`
            - transfer tokens; emit `EmergencyWithdraw`.
        - Prefer **one** denominator everywhere. If you derive from token balance (safer against ghost shares), document/handle fee-on-transfer tokens and donations; otherwise, keep an internal `lpSupply` and update it on **every** position change.
        - Add invariant tests: (1) A deposits, B deposits, A emergency-withdraws → all subsequent rewards accrue fully to B; (2) fuzz the call order of deposit/withdraw/emergencyWithdraw vs `updatePool`.
        - Consider a guardrails test that fails if `sum(user.amount)` ≠ `pool.totalSupply`.
- [ ]  **First-depositor “donation” attack inflates share price before others join**
    - **Symptom:** Pool mints shares as `amount * totalShares / totalAssets` (or 1:1 on first deposit). A first depositor transfers (“donates”) extra underlying to the treasury without minting shares, inflating `totalAssets`. Subsequent depositors receive fewer shares than their fair value; attacker later redeems at a premium.
    - **Where to look (ERC4626 / SushiBar / staking vaults):**
        - Share-mint path: `deposit`, `mint`, `enter`, `stake` equivalents.
        - Math: `if totalShares==0 -> shares=amount` else `shares = amount * totalShares / totalAssets`.
        - Source of `totalAssets`: raw `underlying.balanceOf(treasury/this)` vs. internally tracked accounting.
        - Chromia/Rell or non-EVM: operations that read treasury balance directly and compute `amount_x_token = amount * total_x / total_in_pool`.
    - **Red flags:**
        - No MINIMUM\_LIQUIDITY / “dead shares” minted on first deposit.
        - `totalAssets` derived from **free** token balance that anyone can increase via plain transfer.
        - No guard on first-deposit size; rounding down benefits early minters.
        - No donation handling (e.g., `skim()`/`sync()` or accounting delta using balance-before/after).
        - Tests don’t include “donate underlying between deposits” scenarios.
    - **Questions to ask next time:**
        - Can anyone send the underlying directly to the vault/treasury without minting shares? If yes, how does math react?
        - What anchors the initial exchange rate? Are virtual shares/assets or dead shares used?
        - Do preview functions (`previewDeposit/previewMint`) and implementation both ignore unsolicited donations, or are they donation-sensitive?
        - Is there a minimum first deposit or guarded launch to prevent dust/rounding exploits?
        - Are decimals/rounding paths tested for small first deposits and large subsequent deposits?
    - **Fix pattern:**
        - Adopt **dead shares / MINIMUM\_LIQUIDITY**: on first deposit, mint a fixed `MIN_SHARES` to a burn address to pin initial exchange rate.
        - Use **virtual assets/shares** offsets: compute with `(totalAssets + VIRTUAL_ASSETS)` and `(totalShares + VIRTUAL_SHARES)` so donations can’t skew price at genesis.
        - Base share mint on **accounting delta**, not raw `balanceOf`: `amountReceived = balanceAfter - balanceBefore`; mint `amountReceived * totalShares / totalAssetsBefore`.
        - Provide `skim/sync`style admin that converts donations into proportional gains (does not reward just the first depositor).
        - Add tests: (1) attacker deposits then donates then victim deposits → victim receives expected shares; (2) fuzz first-deposit size & donation amount; (3) rounding invariants.
- [ ]  **“Admin” fee sweep callable by anyone (missing access control on reward/fee distribution)**
    - **Symptom:** Functions that move protocol-wide fees/yield (e.g., `admin_withdraw_all_swap_fee`, `sweepFees`, `distributeRewards`) are callable by any authenticated/external user instead of a privileged role. Attackers can stake/flash-stake, trigger the sweep to inflate exchange rate, then immediately unstake to capture other users’ yield.
    - **Where to look (Solidity / Chromia-Rell):**
        - Solidity:
            - Routers/Vaults/DEX modules: methods named `admin*`, `sweep*`, `withdraw*Fee*`, `skim*`, `distribute*`, `harvest*`, `sync*`.
            - Access layers: `onlyOwner`, `onlyRole`, `onlyOperator`, `AccessControl`, custom `requiresAuth`/`auth.isOperator(msg.sender)` wrappers.
            - Fee collectors/treasuries: contracts holding swap fees and forwarding to staking/xToken treasuries.
        - Chromia / Rell:
            - `src/contracts/common/auth.rell` and role hooks referenced by operations.
            - Operations like `admin_withdraw_all_swap_fee`, `withdraw_swap_fee`, `distribute_fee_to_staking`.
            - Any `operation` that transfers from DEX treasury → staking treasury and is documented as “admin” but lacks an `operator`/`admin` check.
    - **Red flags:**
        - Function name/comment says “admin” but code path only checks “authenticated user” or **no** auth at all.
        - Inconsistent protection: `admin_withdraw_swap_fee` guarded, but `admin_withdraw_all_swap_fee` unguarded.
        - Fee-to-staking transfer updates values used in share accounting (`accRewardPerShare`, exchange-rate of xToken) without time-weighting/lockups.
        - No tests asserting **non-admin** callers revert on fee sweep.
        - Event or doc mismatch (e.g., “requires permissions” in docs, none in code).
    - **Questions to ask next time:**
        - Which exact role is authorized to trigger fee distribution/sweeps? Where is it enforced in code (not just docs)?
        - Does the sweep affect exchange rate or `accRewardPerShare` immediately? Can a user stake right before and unstake right after?
        - Is there a cooldown/epoch boundary/lock-up that prevents opportunistic staking around sweeps?
        - Are *all* fee-moving functions consistently gated (single entrypoint) or are there multiple paths with inconsistent auth?
        - Are there unit tests for: (1) unauthorized caller revert, (2) authorized success, (3) staking-around-sweep profit neutrality?
    - **Fix pattern:**
        - **Enforce role checks** uniformly: e.g., `onlyRole(OPERATOR_ROLE)`/`requiresOperator()` on every fee sweep/distribution op; centralize sweep in a single, guarded function.
        - **Align naming & docs** with permissions (“admin\_” implies admin-only); add NatSpec/Rell comments and emit events.
        - **Mitigate opportunistic capture:** use epoch-based accounting (distribute to `rewardDebt` buckets), add short lockup/cooldown around distributions, or pro-rate rewards by time-weighted stake.
        - **Tests:** add negative tests for unauthorized callers; simulate stake → sweep → unstake to ensure no instant profit; fuzz roles/addresses.
- [ ]  **Governance-liveness dependency can indefinitely block exits when “pending slashes” gate withdrawals**
    - **Symptom:** Withdraw/unstake paths are conditioned on resolving `slashClaims` (or disputes), but only the owner/governor can call `finishSlash/cancelSlash`. Once ownership moves to a timelocked governor, any lack of proposals/votes/execution stalls resolution and freezes user exits for affected actors.
    - **Where to look (Solidity / modular staking):**
        - Staking/validators modules: `withdraw/unstake/exit` checks like `require(slashClaims[prover].length == 0)` or `!isSlashed(prover)`.
        - Slashing flows: functions `startSlash`, `finishSlash`, `cancelSlash`, arrays/maps like `slashClaims[prover]`, stored timestamps.
        - Ownership/governance wiring: `owner = Governor`, `onlyOwner` on cancel/finish; governor parameters `votingDelay()`, `votingPeriod()`, `timelock`.
        - Config/state: any `slashPeriod`, `challengePeriod`, or timestamps saved at claim creation that are **not** consumed to create a timeout.
    - **Red flags:**
        - Only privileged resolution (`onlyOwner`) with **no** permissionless timeout or “forced exit” after a deadline.
        - Comments like “governance will resolve” but no liveness guarantees; no linkage to `votingDelay + votingPeriod + timelock`.
        - `cancelSlash` callable only by owner forever; `finishSlash` required before exit; no backstop if governance abstains or proposals fail.
        - Unbounded `slashClaims` arrays; `withdraw` hard-reverts while any claim exists; no partial/escrowed exits.
        - Off-chain keeper assumptions (e.g., “ops will propose/execute”) without on-chain fallback.
    - **Questions to ask next time:**
        - What **maximum** time-to-resolution is guaranteed on-chain if governance is apathetic or adversarial?
        - Is there a **permissionless** path (after a computed deadline) to: (a) cancel stale claims, or (b) allow a partial/escrowed exit?
        - Is the deadline formula tied to **current** governor timings (`votingDelay + votingPeriod + timelock`) plus a buffer, and resilient to those being updated?
        - Who can create multiple claims against the same prover? Are claims deduplicated or can they grief by queuing new ones to extend freezes?
        - Are there events/metrics to monitor stuck claims and circuit-breakers to pause new slashes if backlog > threshold?
    - **Fix pattern:**
        - Add **permissionless timeout**: `cancelSlash(prover, idx)` callable by anyone **after** `claim.timestamp + slashPeriod + votingDelay + votingPeriod + timelock + buffer`.
        - Validate deadline at call time (read governor for live timings), not only in initializer; add a separate `slashCancelDelay`.
        - Consider **partial exits**: allow withdraw of non-challenged principal while escrowed portion remains subject to slash outcome.
        - Rate-limit/lock claims per prover, dedupe overlapping claims, and cap claim lifetime.
        - Tests: (1) simulate governance inactivity → exit succeeds via timeout; (2) governance resolves before timeout → no premature cancel; (3) governor parameter changes don’t break deadline calculus.
- [ ]  **Snapshotting shares (not exchange rate) lets exiting stakers capture post-request yield**
    - **Symptom:** Two-step unstake takes a snapshot in **share units** (e.g., `iPROVE`), then later redeems those shares after rewards were **deposited as underlying** into the ERC-4626 vault. Since rewards raise `assetsPerShare`, the user still receives more **underlying assets** during `finishUnstake`—despite logic that caps them to the share snapshot—violating “no rewards while unstaking”.
    - **Where to look (Solidity / DeFi staking + ERC-4626):**
        - Unstaking flows: `requestUnstake`, `finishUnstake`, `_unstake`, `redeem`, `previewRedeem/convertToAssets`.
        - Reward distribution: any `dispense`/“harvest” that **transfers underlying** (e.g., `asset`) into the vault instead of minting shares.
        - Storage on request: fields like `snapshotShares`, `_iPROVESnapshot`, `pendingUnstake[addr]` that **do not** store `assetsPerShare` or `assetsAtRequest`.
        - Multi-layer vaults (staked share of a share): e.g., `stPROVE → iPROVE → PROVE`. Check which layer the snapshot is taken in vs. which layer receives rewards.
    - **Red flags:**
        - Snapshot records **only share quantity** (e.g., `sharesAtRequest`) without also storing `exchangeRateAtRequest` (`assetsPerShare`, `convertToAssets(1e18)`).
        - Payout at `finishUnstake` uses `vault.redeem(snapshotShares, ...)` at **current** exchange rate or converts with `convertToAssets(snapshotShares)` at T2.
        - Rewards are injected via `asset.safeTransfer(vault, …)` (or `vault.asset().transfer(...)`) instead of `vault.deposit/mint` → raises exchange rate for **all**, including queued exits.
        - Comments promise “no rewards during unstake”, but comparisons cap only **shares** (e.g., `min(receivedShares, snapshotShares)`), not **assets**.
    - **Questions to ask next time:**
        - Do we snapshot **assets** (or exchange rate) at request and cap the **underlying payout** to that value?
        - Are rewards during the queue routed to a beneficiary that isn’t the queued exiter (e.g., credited to the prover/treasury), or neutralized by minting/burning shares?
        - In multi-layer designs, which layer receives rewards and which layer is snapshotted? Are units consistent end-to-end?
        - How are **slashes** applied—also in assets or shares—and do they symmetrically affect queued exits?
    - **Fix pattern:**
        - On `requestUnstake`, store `assetsAtRequest = convertToAssets(sharesToUnstake)` (or store `exchangeRateAtRequest`).
        On `finishUnstake`, enforce `assetsOut = min(currentAssetsFrom(sharesToUnstake), assetsAtRequest)`; send any excess to the designated beneficiary (e.g., prover).
        - Alternatively, route rewards as **share mints** (not raw asset transfers) so `assetsPerShare` stays constant, or divert rewards accrued during queue to a separate accounting bucket.
        - For multi-layer vaults, snapshot and settle at the **final payout unit** (ultimate underlying) to avoid leakage across layers.
        - Tests: (1) increase `assetsPerShare` between request/finish and prove pre-fix profit; (2) verify post-fix user gets ≤ `assetsAtRequest`; (3) mirror test for slashing decrease.
- [ ]  **ERC4626 share-as-votes + external inflows skew governance (donation/dispense inflation)**
    - **Symptom:** Governance power is tied to **receipt shares** (ERC4626) while `totalAssets()` = `asset.balanceOf(this)`. Any **dispense** or **direct donation** raises `assetsPerShare`, so **early holders keep the same shares (votes) but new depositors mint far fewer shares per underlying**, making proposals/thresholds disproportionately easier for first movers.
    - **Where to look (Solidity / DeFi):**
        - The vote token: is it the **ERC4626 share** (e.g., `iToken` + `ERC20Votes` or similar)? Check `Governor` configs: `proposalThreshold`, `quorum`, `getVotes`.
        - Vault accounting: `totalAssets()` implementation (common red flag is `return _asset.balanceOf(address(this));`).
        - Reward/fee flows: functions like `dispense`, `harvest`, `donate`, or any `asset.safeTransfer(vault, amount)` that **bypass deposit/mint**.
        - Ingress hardening: absence of hooks that block **direct ERC20 transfers** to the vault (no `receive`/`fallback` for native, no ERC777 hooks, and no documented guard).
        - Any comments implying “shares = votes” with **no adjustment** for donations/dispenses.
    - **Red flags:**
        - Governance token = vault **share token** (not underlying), e.g., governor constructed with `ERC20Votes(iVault)`.
        - `totalAssets()` mirrors **raw balance**; no internal “accounted assets” variable; no separation for **unaccounted donations**.
        - Reward module sends underlying **directly** to the vault (`asset.transfer(vault, …)`) instead of minting shares to a neutral account or using deposit/mint semantics.
        - Proposal thresholds/quorum calibrated in **share units**, not underlying; no migration/normalization on exchange-rate shifts.
    - **Questions to ask next time:**
        - “If someone sends the underlying **directly** to the vault, what happens to `assetsPerShare`, and who benefits? Do later depositors get fewer votes per unit of underlying?”
        - “Can rewards be routed in a way that **doesn’t** change `assetsPerShare` for queued/new voters (e.g., mint compensating shares to a sink/treasury)?”
        - “Why are **shares** used for votes instead of a metric derived from **underlying** (e.g., `previewRedeem(balance)`), and can votes be decoupled?”
        - “Are governance thresholds resilient to exchange-rate shocks (donations/slashing), or do they implicitly favor early entrants?”
    - **Fix pattern:**
        - **Decouple votes from shares:** override voting power to be based on **underlying amount**, e.g., wrap `getVotes(addr)` to compute votes from `previewRedeem(shareBalance)` (note: OZ `ERC20Votes` isn’t drop-in—may require a custom Votes module or a mirrored, non-transferable vote token).
        - **Block or neutralize donations:** track **accounted assets** internally (`accountedAssets` += deposits/harvest via controlled paths) and override `totalAssets()` to return `accountedAssets`; treat raw donations as **protocol income** by **minting compensating shares to a sink** so `assetsPerShare` stays constant.
        - **Reward routing:** when dispensing rewards to the vault, **use `deposit/mint` with a neutral recipient** (treasury/dead address) chosen to **preserve exchange rate**, or credit rewards outside the voting supply.
        - **Governance hardening:** set proposal thresholds/quorum in **underlying terms** (or dynamically scaled by exchange rate) and document/share-inflation edge cases in tests.
        - **Tests to add:** (1) direct `asset.transfer` to vault before/after first deposit → assert later depositor gets fewer votes per underlying; (2) dispense path preserves vote fairness post-fix; (3) thresholds behave invariantly across exchange-rate changes.
- [ ]  **No per-operator cap → single delegate/prover can capture governance**
    - **Symptom:** A single prover/operator can accumulate a majority of voting power (e.g., via its ERC4626 vault’s shares) and unilaterally meet proposal thresholds/quorum, potentially modifying governor params (quorum/period/delay) or bricking governance before “ownership handoff.”
    - **Where to look (Solidity / governance + staking):**
        - Staking entry points: `stake(_prover, _amount)` / `_stake`—look for **`minStakeAmount` present but no `maxStakeAmount`**, and **no per-prover cap** or %-of-supply limit.
        - Vote token source: whether **votes = ERC4626 shares (iToken)** and whether **prover contracts call `IGovernor.castVote`** on behalf of all stakers (e.g., `onlyOwner` on prover).
        - Governor wiring: when is **governor made owner** of staking? Are proposals **live before handoff**? Check `proposalThreshold`, `quorumNumerator`, `votingDelay/Period`, and any **guardian/timelock**.
        - Snapshotting: OZ Governor **checkpointing** (`getVotes` at `proposalSnapshot`)—temporary concentration can still pass proposals.
        - Multiplicity: can the **same operator run multiple provers** (collusion bypassing any single-contract cap)?
    - **Red flags:**
        - “Anyone can create a prover,” **no global per-prover stake cap** (absolute or % of total), and **no consolidation guard** across provers sharing the same owner.
        - Governor address set at deploy time; **proposals executable before decentralization**, relying only on “we’ll switch later.”
        - Prover’s `castVote` is `onlyOwner` and **aggregates all staker votes by construction** (delegation without opt-out).
        - No circuit breaker to pause proposals when **concentration metrics** (e.g., top prover share) exceed thresholds.
    - **Questions to ask next time:**
        - “Is there a **cap per operator/prover** (absolute or % of total voting supply)? If not, why is capture risk acceptable?”
        - “Are **governance actions disabled** until decentralization targets are met (supply size + Nakamoto coefficient)? Who enforces that on-chain?”
        - “Can a proposer **lower quorum/period** pre-handoff? Is there a **guardian/timelock** that can veto or delay params changes?”
        - “Do you **aggregate votes at the prover**? Can individual stakers override or undelegate per-proposal?”
        - “How do you mitigate **collusion** (same owner with many provers)?”
    - **Fix pattern:**
        - **Stake caps:** enforce `maxStakePerProver = min(absoluteCap, percentCap * totalSupply)`. Reject stake that would exceed cap; or soft-cap with **decaying voting weight** beyond a threshold (e.g., square-root or capped linear).
        - **Decentralization gates:** add an **on-chain gate** that disables `propose`/`execute` (or `castVote`) until **(i)** total votes ≥ target and **(ii)** concentration metrics (top N share / Herfindahl index) are under limits.
        - **Governance guardrails:** hard-min bounds on `quorumNumerator`, `votingDelay`, `votingPeriod`; protect them behind **timelock + guardian** during bootstrap; add an **emergency pause** for proposal execution when concentration spikes.
        - **Delegation controls:** allow **per-staker opt-out** (vote directly) or constrain prover-level aggregation (e.g., require staker signatures for `castVoteBySig`).
        - **Operator Sybil mitigation:** optional **KYC/registry** or “one-operator-one-prover” policy; at minimum, **monitor and alert** if multiple provers share keys/owners.
        - **Tests/invariants:** fuzz stake flows to push a prover over cap; assert that **proposal creation/execution reverts** under concentration; simulate param-change proposals pre-handoff and ensure **guardianship blocks** them.
- [ ]  **Ceiling-division in “tokens-per-second” dispenser drains future emissions**
    - **Symptom:** Reward “dispense”/“stream” uses `timeConsumed = (amount + rate - 1) / rate` (ceil) and advances `lastDispenseTimestamp += timeConsumed` while **transferring only `amount`**—discarding the remainder and letting dust calls (e.g., `amount = 1`) **consume a full period**. Repeated dust sends can deplete future rewards or underpay stakers.
    - **Where to look (Solidity / emissions & vesting):**
        - Streaming/dispensing functions: `dispense(amount)`, `claim(amount)`, `distribute(amount)`.
        - Rate + timestamp math: fields like `dispenseRate`, `tokensPerSecond`, `lastDispenseTimestamp`, `maxDispense() { (now - last) * rate }`.
        - Any **conversion between amount and time** using division; watch for **ceil emulation**: `(x + d - 1) / d`.
        - State carried across calls: is there a **remainder accumulator** (e.g., `carry`/`leftover`/`pending`) or is it missing?
        - Call permissions: is `dispense` **centralized** (a “dispenser” role) and can it be called with **arbitrarily small amounts**?
    - **Red flags:**
        - `timeConsumed = (amount + rate - 1) / rate; lastDispenseTimestamp += timeConsumed; IERC20.transfer(amount);` with **no handling of `amount % rate`**.
        - `maxDispense = (block.timestamp - lastDispenseTimestamp) * rate` and **no min amount** / **no exact-multiple check**.
        - `lastDispenseTimestamp` can move **forward by more seconds than the paid tokens justify**.
        - Unit tests missing cases for **`amount < rate`**, **`amount = k*rate ± 1`**, and repeated dust calls.
    - **Questions to ask next time:**
        - “Is `amount → time` conversion **floor or ceil**? Why?”
        - “What happens to the **remainder (`amount % rate`)**? Is it carried to next call or lost?”
        - “Can the caller pass **dust** amounts? Is there a **minimum dispense** or **exact multiple** requirement?”
        - “Can `lastDispenseTimestamp` **ever exceed `block.timestamp`** after many small calls?”
        - “Who can call `dispense` and how often—any **rate-limit** or **reentrancy** that could amplify rounding?”
    - **Fix pattern:**
        - **Token-bucket accumulator:**
            - Keep `acc += amount; time = acc / rate; last += time; acc -= time * rate;`
            - This floors consumption and **persists remainder** (`acc`) instead of losing it.
        - **Exact-multiple enforcement:**
            - `require(amount % rate == 0)` (or `amount % (k*rate) == 0`), then `timeConsumed = amount / rate;`.
        - **Min-amount guard (secondary):**
            - Set `minDispense = rate` (or higher) to reject dust; still **prefer remainder accounting** to avoid silent loss.
        - **Safety clamps & tests:**
            - Clamp `lastDispenseTimestamp = min(lastDispenseTimestamp + timeConsumed, block.timestamp)`.
            - Add tests: `amount = 1`, `rate-1`, `rate`, `rate+1`, repeated dust loops; assert **no over-consumption of time** vs tokens.
- [ ]  **Overlapping-unstake snapshot drift (shares not removed at request)**
    - **This is the same root cause as a heuristic I already generated: *“Rewards accrue to ‘unstaking’ shares that aren’t removed from the pool.”* Rather than duplicate it, here’s the variant-specific guidance for overlapping requests.**
    - **Symptom:** A staker who calls `requestUnstake` while earlier unstake requests are still pending gets a **too-low snapshot** (e.g., `iPROVESnapshot`). When the earlier requester later finishes, their “excess” is returned to the vault, not to the later requester who took the diluted snapshot — yielding **underpaid rewards / APY drag / MEV windows** for the later requester.
    - **Where to look (Solidity / staking + ERC4626 wrappers):**
        - Unstake flow split across `requestUnstake(...)` → `finishUnstake(...)`.
        - Snapshot code: fields like `iPROVESnapshot`, `exchangeRate`, `previewRedeem`, and when/what they read (pre/post rewards, include pending unstake shares?).
        - Finish logic: branches like “if received > snapshot then send **excess back to vault/prover**”.
        - Vault accounting during the gap: reward “dispense” / “donation” flows into the ERC4626 vault while pending requests still **hold shares**.
    - **Red flags:**
        - `requestUnstake` **does not burn/lock/escrow** the user’s shares (they still affect `totalSupply` / exchange rate).
        - Snapshot computed from **live** vault state (including pending requesters) without removing those shares from accounting.
        - `finishUnstake` returns `received - snapshot` to the **prover/vault**, not to contemporaneous requesters who were diluted by those pending shares.
        - No queue/escrow structure (no `pendingUnstakeShares`, `escrowedIProve`, or per-request bucket).
    - **Questions to ask next time:**
        - “At `requestUnstake`, do we **remove** the user’s shares from all future exchange-rate/reward math (burn/escrow/mark-pending)?”
        - “If rewards arrive between `request` and `finish`, **who should own them** and how is that enforced?”
        - “Can two requests overlap? What invariant ensures later snapshots aren’t diluted by earlier **still-counted** shares?”
        - “Does any ‘excess’ on finish go to the **right party** (e.g., the requester(s) impacted) or just back to the pool?”
        - “Are there tests for **multiple overlapping** requests with rewards landing between them?”
    - **Fix pattern (tested in the wild):**
        - **Escrow-at-request:** Immediately `redeem`/lock the user’s claim (convert/bucket into an escrow balance), record snapshot, and exclude it from live `totalSupply`/`totalAssets`. On finish, pay from escrow; apply slashing as a haircut to escrow, not pool math.
        - **Accounting variant:** Track `pendingUnstakeIProve` and subtract it from share accounting used for snapshots/rewards; or maintain a per-request queue that isolates entitlements.
        - **Distribution correctness:** If “excess” arises on finish, route it to the **affected cohort** or pro-rata across overlapping requesters (not to the general vault).
- [ ]  **ERC4626 “ghost assets”: rewards/donations sent to a vault with zero shares (assets > 0, totalSupply == 0)**
    - **Symptom:** Rewards/excess are transferred into an ERC4626 vault (the prover’s `PROVER-N`) **while no receipt shares exist**. Later deposits mint 0/revert or leave the reward **stranded**; subsequent stakers cannot claim it and accounting gets distorted.
    - **Where to look (ERC4626 + staking/unstake plumbing):**
        - ERC4626 math: `convertToShares`, `previewDeposit`, `deposit`, `mint`, and any codepaths that special-case `totalSupply() == 0`.
        - Reward/“dispense” flows and unstake finishers that move the **asset token** directly into the vault (e.g., `IERC20(iPROVE).transfer(prover)` or `iPROVE.deposit(..., prover)`).
        - “Excess on finishUnstake” branches that **return** surplus `iPROVE` to the prover vault after the **last share was burned**.
        - vApp fulfillment paths that fund a prover before anyone has staked to it (prover `totalSupply() == 0`).
    - **Red flags:**
        - Direct `ERC20.transfer(asset, vault)` donations with **no accompanying share mint**.
        - `totalAssets() = asset.balanceOf(this)` (vanilla ERC4626) + **no seed/dead shares** minted at vault creation.
        - `finishUnstake`: `if (received > snapshot) transfer(excess, prover)` (instead of paying the impacted staker cohort) when `prover.totalSupply()==0`.
        - Assumptions like “off-chain will ensure prover always has stake” without an on-chain guard.
    - **Questions to ask next time:**
        - “What happens if `totalSupply()==0` and we receive assets? Can anyone mint non-zero shares from a deposit afterward, or does it revert/mint 0?”
        - “Can rewards be routed to a prover that currently has **no** stakers? What invariant prevents assets>0, shares==0?”
        - “When the **last** unstaker finishes and there’s ‘excess’ (rewards during window), **who** gets it if the vault would become empty?”
        - “Do we ever send asset tokens directly to the vault (donate) rather than `deposit`/`mint` with an explicit recipient?”
        - “Is there a recovery/sweep path for ghost assets (admin mint to fee sink, escrow, or delayed deposit)?”
    - **Fix pattern:**
        - **Never allow assets without shares.**
            - Seed the vault with **dead/fee shares** at creation (≥ `decimalsOffset`) so future donations can’t strand value.
            - Or gate reward routing: **if `totalSupply()==0`** accumulate rewards in **staking escrow** and only `deposit` once stake exists.
        - **Unstake excess handling:** On `finishUnstake`, if `received > snapshot` and `prover.totalSupply()==0`, send the excess to a **designated escrow** (or distribute to affected stakers), *not* to the empty vault.
        - **Deposit-not-transfer:** Replace raw `ERC20(iPROVE).transfer(prover, amt)` with `IERC4626(prover).deposit(amt, beneficiary)` on a path that **mints shares** (or explicitly mints shares to a fee/escrow address when empty).
        - **Guard rails:** Add a check in the vApp/dispatcher: **do not fund** provers with rewards while `totalSupply()==0` (or auto-seed minimal shares).
        - **Tests to add:**
            - Last-unstaker finishes → rewards land → prover has `assets>0, shares==0`; new stake should still succeed and be able to **claim** those rewards (assert balances).
            - Rewards arrive before first stake; assert no ghosting or revert on first `deposit`.
- [ ]  **Slashing that uses live ERC4626 price (`previewRedeem`) instead of a snapshot can over-burn**
    - **Symptom:** The slash claim records **shares** (e.g., `iPROVE`) and on `finishSlash` the contract computes the burn in base tokens via **current** `previewRedeem`. If rewards/“dispense” increase `assetsPerShare` during the slash window, the same share amount maps to **more** base tokens → the protocol burns **more PROVE than intended** and effectively destroys other stakers’ rewards.
    - **Where to look (ERC4626 + slashing flows):**
        - Slash claim structs: what unit is stored (shares vs assets)? any stored exchange rate?
        e.g., `SlashClaim { iPROVE, timestamp }` with no snapshot of `assetsPerShare`.
        - `finishSlash`: uses `previewRedeem`/`redeem` at settlement time to determine tokens to burn.
        - Reward/dispense paths callable **between** `requestSlash` and `finishSlash` that change `totalAssets/totalSupply`.
        - Time delays/governance gates: `slashPeriod`, voting delay/period, owner/gov controlled execution windows.
    - **Red flags:**
        - No snapshot of `assetsPerShare` (or base token amount) captured at `requestSlash`.
        - Slashed **shares keep accruing rewards** while pending (no escrow/freeze), or “excess” rewards logic that credits the prover during the window.
        - Comments/assumptions that “off-chain accounts for it” while on-chain still calls `previewRedeem` at finish.
    - **Questions to ask next time:**
        - “Is the slash obligation recorded in **base token units** or **share units**? If shares, what prevents price drift before settlement?”
        - “Do pending-slash shares continue to earn rewards/dispenses? If yes, who should own those rewards?”
        - “Can anyone call `dispense()` (or donate) during `slashPeriod`? What’s the worst drift between request and finish?”
        - “If governance delays execution, does the slash amount grow with time? Is that intended?”
    - **Fix pattern:**
        - **Snapshot at request time:** store `snapshotAssetsPerShare` (or `proveToBurn = previewRedeem(iPROVE)` at request) and burn based on this snapshot, not live price.
        - **Freeze economics:** move the slashed shares into an **escrow** that **does not participate in rewards**, or siphon any accruals during the window to a neutral sink (fee vault/escrow) so `assetsPerShare` doesn’t inflate the obligation.
        - **Guard dispenses:** optionally block/queue `dispense()` for provers with active slash claims, or net dispensed rewards out of the slash when settling.
        - **Unit choice:** define slash amounts in **base tokens** with a recorded snapshot, and only use shares as a container (convert at snapshot rate).
    - **Tests to add:**
        - Create slash → perform `dispense()` (increase PPS) → `finishSlash`; assert burned base tokens equal the **snapshot** amount, not the inflated live `previewRedeem`.
        - Variant with multiple dispenses and with governance delay to bound worst-case drift.
- [ ]  **Governance proposals that reference array indices can be invalidated by earlier mutations**
    - **Symptom:** A proposal to act on an indexed item (e.g., `finishSlash(prover, idx)`) succeeds/executes first and **reindexes** the array (swap-and-pop or pop), causing other queued proposals to **revert** (out-of-bounds) or to **operate on the wrong item**.
    - **Where to look (Solidity / OZ Governor set-ups):**
        - Queued actions that take an **array index**: `cancelSlash(_prover, index)`, `finishSlash(_prover, index)`, `removeClaim(pool, i)`, etc.
        - Storage layout for mutable queues: `SlashClaim[]`, `Request[]`, `Proposal[]` with code that does `claims[i] = claims[last]; claims.pop();`.
        - Governor pipelines: places building calldata off-chain with `(target, calldata(index))` that may sit through `votingDelay`/`votingPeriod` while on-chain state shifts.
        - Any **batched/parallel** operations on the same array (multiple proposals, admin calls, keepers).
    - **Red flags:**
        - Public/external functions accept **index** instead of a stable identifier.
        - Comments like “// delete by swapping with last and popping” on arrays that are **governance-controlled** targets.
        - Events/logs don’t emit a **claim/request ID** (only index), making off-chain proposal builders index-based.
        - No invariant that only **one** outstanding item can exist; multiple concurrent requests are allowed.
    - **Questions to ask next time:**
        - “Can two proposals referencing different indices be active simultaneously? What happens if one mutates the array first?”
        - “Is there a **stable key** (id/hash) for each item that proposals can reference across time?”
        - “Do we ever **reorder** or **pop** items between proposal creation and execution?”
        - “If execution reverts due to index drift, is there an automatic recovery path, or does the proposal get stuck forever?”
    - **Fix pattern:**
        - **Use stable IDs**: store `struct { uint256 id; ... }` and expose `cancelSlash(prover, id)` / `finishSlash(prover, id)`. Resolve via **mapping** `mapping(address => mapping(uint256 => Claim))` with **O(1)** access; if enumeration is needed, keep a side list of IDs.
        - **Append-only + tombstone:** keep an array but never reorder/pop; mark `resolved=true` and maintain `activeCount` to cap scanning. Guard functions with `require(!resolved)`.
        - **Idempotent execution:** safe-check at execution (e.g., ID exists & matches expected fields) and **no-op** if already resolved, so proposals don’t brick.
        - **Proposal binding:** include the **ID** (and optionally important fields hash) in calldata and verify on-chain (`require(claim.id==id && keccak256(abi.encode(...))==expectedHash)`).
- [ ]  **No bad-debt absorption → residual borrows linger and solvency drifts**
    - **Symptom:** After seizing all collateral, the borrower’s remaining debt is **left outstanding** (no “absorb/socialize” step). `liquidateBorrow`/`liquidateBorrowFresh` only transfer collateral vs. `repayAmount` and never write any shortfall to reserves/insurance or haircut suppliers. Protocol can show healthy utilization while carrying silent underwater accounts.
    - **Where to look (Solidity lending/CToken-style):**
        - Liquidation path: `liquidateBorrow`, `liquidateBorrowFresh`, `seize`, `repayBorrowFresh` (does anything happen when `borrowBalance > collateralValue` after seize?).
        - Controller/risk: `liquidateCalculateSeizeTokens`, `closeFactorMantissa` (is full liquidation allowed when underwater?), `seizeAllowed`, `redeemAllowed` (are withdrawals gated during shortfall?).
        - Accounting: presence/absence of `totalBadDebt`, `shortfall`, `absorb`, `buyBadDebt`, `auction` functions; updates to `totalReserves`, `reserveFactorMantissa`, `protocolSeizeShareMantissa`, `totalBorrows`, `exchangeRateStored`.
        - Withdrawals: any checks that block/sanitize supplier redemptions when `totalAssets < totalLiabilities`.
    - **Red flags:**
        - No variable/event for bad debt (`totalBadDebt`, `Shortfall`, `AbsorbAccount`), and no function that zeroes borrower debt post-seize.
        - Fixed `closeFactor` (<100%) even for clearly insolvent accounts; no “override to 100% if underwater”.
        - Reserves never decrease to cover losses; no insurance/backstop module; no dutch/english debt auction.
        - Suppliers can redeem freely regardless of aggregate shortfall; `exchangeRate` is never adjusted down for losses.
        - Tests only cover healthy or over-collateralized liquidations; no scenario with `collateralValue << debt`.
    - **Questions to ask:**
        - “After max liquidation + full seize, **who eats the remainder** if debt > collateral? Where is it recorded on-chain?”
        - “Are reserves/insurance/backstop used before socializing to suppliers? What is the waterfall?”
        - “Can liquidators liquidate to **100%** when borrower is underwater? If not, how is the leftover cleared?”
        - “Are redemptions/borrows **paused or gated** while `totalBadDebt > 0`?”
        - “Is there a keeper-call (e.g., `absorb(account[])`) to crystallize and distribute bad debt, and does it update indices/exchange rate?”
    - **Fix pattern:**
        - **Add an absorption path:** after seizing all collateral, compute `shortfall = remainingDebt` and:
            1. reduce `totalReserves` (up to available),
            2. tap insurance/backstop (e.g., backstop vault or auction), then
            3. **socialize** remainder across suppliers by lowering `exchangeRate` / advancing a supply index with a negative delta (haircut pro-rata).
        - **Allow full liquidation when underwater:** override close factor to 100% if `liquidity < 0`, so liquidators can zero out debt; emit `LiquidateShortfall` for any residual that still remains.
        - **Gate risky operations:** prevent/red-flag redemptions/borrows while `totalBadDebt > threshold` or until `absorb()` has run; expose `totalBadDebt` in views and events.
        - **Backstop/auction option:** implement a `buyBadDebt` or auction module selling shortfall for protocol/backstop tokens, with incentives for keepers to clear insolvencies.
        - **Tests/invariants:** simulate price crashes, stale oracles, dust positions, and gas-bounded liquidations; assert accounting identity: `cash + presentValue(borrows) + reserves − badDebt == totalAssets backing suppliers`.
- [ ]  **Ramping/price-sync uses cached balances ⇒ supply/buffer drift**
    - **Symptom:** During parameter ramps (e.g., `A` changes) the pool syncs `totalSupply`/buffer using **stale** `balances` (and thus stale exchange rates), e.g., `_syncTotalSupply()` calls `_getD(balances, A)` without first refreshing `balances` with current `exchangeRate()`s. Supply may be increased/decreased against the wrong state, diluting LPs or burning excess.
    - **Where to look (StableSwap/pegged-asset pools, rebasing/interest-bearing tokens):**
        - Any **ramp** modifier/hook (e.g., `syncRamping`, `onAChange`) that calls a supply-sync like `_syncTotalSupply()`, `_rebase()`, `_collectYield()`, or “buffer” adjustments.
        - Functions computing invariant `D` or similar with a cached `balances` array/state var vs. recomputing from **actual token balances × current exchange rates & precisions**.
        - Code paths for `mint/swap/redeem`, donations, and fee/yield collection where **order** is: (update balances & rates) → (compute `D`) → (mutate supply/buffer).
    - **Red flags:**
        - `_getD(balances, A)` or invariant math fed by a **storage** `balances` without a preceding refresh like `getUpdatedBalancesAndD()`.
        - Comments like “balances updated elsewhere” or reliance on earlier calls, but the ramp path can trigger independently.
        - Exchange-rate providers present (`exchangeRateProviders[i].exchangeRate()`) yet not read immediately before supply math.
        - Buffer mutations (`addBuffer/removeTotalSupply/addTotalSupply`) keyed off `newD` computed from cached inputs.
        - Tests lack scenarios where **exchange rates change** (rebasing/price move) between last `balances` refresh and ramp sync.
    - **Questions to ask:**
        - “Before any supply/buffer mutation, where do we **refresh balances** (token balance × current exchangeRate × precision)?”
        - “Can `_syncTotalSupply` be reached without a prior `collectFeeOrYield`/refresh in the **same** call path?”
        - “What happens on **donations** or passive rebases between calls—does `balances` reflect on-chain token balances?”
        - “Do we ever compute `D` from storage state while rates moved since last update?”
    - **Fix pattern:**
        - **Single source of truth:** Always refresh inputs right before invariant/supply math. Replace direct `_getD(balances, A)` with `(balances, newD) = getUpdatedBalancesAndD();`.
        - **Ordering invariant:** `(refresh balances & rates) → compute newD → mutate supply/buffer → write new totalSupply`.
        - Guard against drift: assert monotonicity/expected deltas in tests when rates fall/rise; add property tests with mid-tx rate changes & donations.
        - If refresh is expensive, cache **block-scoped** computed balances/newD and reuse within the same transaction, never across txs.
- [ ]  **Ideal-balance fee + fee-to-supply revaluation enables sandwich fee capture**
    - **Symptom:** Fees are computed as a function of deviation from an “ideal” balance (often zero when user picks just-right amounts). Collected fees are then added to pool **totalSupply** (bumping share price). A searcher can mint with \~zero deviation (pay 0), let a victim pay fees (share price jumps), then redeem with \~zero deviation (pay 0) and extract the victim’s fee via the higher share price.
    - **Where to look (StableSwap/pegged-asset pools with LP shares):**
        - Deposit/withdraw paths that price against an **ideal balance**: e.g. `mint(...)`, `redeemMulti(...)`, `redeemSingle(...)`, `_getIdealAmounts/_calcFee`, `_mintFee/_collectFeeOrYield`.
        - Any place that credits fees by **increasing LP supply/virtual price** (`addTotalSupply`, `addBuffer`, `totalSupply += fee`) rather than paying a separate rewards pot with vesting.
        - Whether fee logic allows **exact-balance deposits/withdrawals** to be fee-free (difference == 0 or within epsilon).
        - Block/tx-level constraints: can users **mint then redeem in the same block/tx**? Is there a **min base fee** or **one-block lock**?
    - **Red flags:**
        - Fee = f(|actual − ideal|) with **no fee floor** and symmetry on both mint and redeem.
        - Fees immediately **reprice shares** (credited to supply/buffer) and are **immediately withdrawable** by anyone holding shares.
        - No anti-sandwich measures (no TWAP ideal, no cooldown/lock on new shares, no “withdrawal fee if exiting soon after entering”).
        - Tests only cover random/slippage paths, **not** MEV sequences (front-run mint zero-fee → victim pays fee → back-run redeem zero-fee).
    - **Questions to ask:**
        - “Can a user choose amounts so `fee == 0` on both enter and exit?”
        - “Where do fees go—directly to **LP totalSupply/virtual price** or a **separate accumulator** with vesting?”
        - “Is there **minFee bps** on mint **and** redeem? Any **cooldown** before new shares can redeem without penalty?”
        - “Is the **ideal balance** instantaneous (manipulable intra-block) or TWAP’d over time?”
        - “Do we prevent **same-block mint→redeem** (or multi-call atomic round-trip)?”
    - **Fix pattern:**
        - Add a **base fee floor** (e.g., a few bps) on **both** mint and redeem so zero-diff paths still pay something.
        - **Time-weight fees:** credit fees to a **vesting buffer** distributed pro-rata over N blocks/epochs; newly minted shares can’t immediately capture them.
        - **Cooldown/penalty:** stamp deposit block; apply an **early-exit fee** (burned or sent to long-term LPs) if redeemed within M blocks.
        - **TWAP the ideal** (or clamp per-tx delta) so attackers can’t engineer fee-free round-trips around a victim’s state change.
        - Split accounting: route fees to a **separate rewards pot** claimable over time instead of bumping `totalSupply` instantly.
- [ ]  **Integer math order causes silent value leak in rebase/redistribution**
    - **Symptom:** When converting raw token balances using price/exchange-rate and pool precision, code divides **before** multiplying by precision (e.g., `bal * rate / 10**decimals * precision`). The early division truncates, leaking value on every `rebase()`/`distributeLoss()` and underpaying LPs by \~bps–1% depending on decimals/rates.
    - **Where to look (StableSwap/omnipool/4626-style vaults):**
        - Balance normalization paths: `rebase()`, `distributeLoss()/distributionLoss`, `collectFeeOrYield`, `getUpdatedBalancesAndD`, or any helper that computes “normalized balances”/`D` with `exchangeRateProviders` and `precisions`.
        - Lines that look like:
        `normalized[i] = (balance * rate) / 10**rateDecimals * precision;`
        - Mixed-decimals tokens (6/8/18) and places that later call `_getD(normalized, A)` or adjust `totalSupply/buffer` using those normalized values.
    - **Red flags:**
        - Multiply–divide–multiply sequence: `a * b / c * d` instead of `mulDiv(a * b, d, c)` or `mulDiv(a, b * d, c)`.
        - Use of raw `/` (flooring) without comment on rounding direction; no unit tests asserting conservation before/after rebase/loss distribution.
        - Inconsistent ordering across functions (e.g., `collectFeeOrYield` updates balances with new ER first, but `_syncTotalSupply` / `rebase` don’t, or they order ops differently).
        - Tokens with low decimals (≤8) combined with large `precision` scalars—precision loss magnifies.
    - **Questions to ask:**
        - “Do all code paths that compute normalized balances use the **same rounding order** and exchange rate snapshot?”
        - “Have we measured delta between `expected` vs `applied` rebase using a PoC with 6-dec tokens and non-integer rates (e.g., 1.9999e18)?”
        - “Are we okay always rounding **down**? If not, where is the remainder tracked (carry/fee pot)?”
        - “Could `_syncTotalSupply` or buffer adjustments be using **stale** normalized balances (compounding the leak)?” *(Related but distinct heuristic; don’t duplicate if already tracked.)*
    - **Fix pattern:**
        - **Reorder math** to multiply first, divide last (or use **FullMath.mulDiv** / **PRBMath** to avoid overflow and pick rounding):
        `normalized = mulDiv(balance, rate * precision, 10**rateDecimals);`
        - Centralize normalization in one helper (e.g., `_normalize(balance, rate, rateDecimals, precision)`) and **reuse everywhere**.
        - Make rounding policy explicit (down/up/nearest) and, if needed, **accumulate remainders** into a fee/reward bucket instead of dropping them.
        - Add invariants/tests: conservation across `rebase()/distributeLoss()`, property tests over random rates/decimals; assert percent error < 1e-9.
    - **Not a duplicate of:** “Stale normalized balances during A-ramp sync” (uses **old exchange rates**); this heuristic targets **arithmetic ordering/rounding** causing deterministic truncation even with fresh rates.
- [ ]  **Zero-balance → div-by-zero in invariant `D` (DoS across pool ops)**
    - **Symptom:** `_getD()` divides by each normalized balance: `pD = (pD * D) / (_balances[j] * n)`. If **any** `_balances[j] == 0` (e.g., single-sided donate/mint arrays with zeros, near-drained asset + small ER dip, or rounding to 0 during normalization), this panics. Every path that calls `_getD` (mint/swap/redeem\*/donateD/rebase/distributeLoss/sync during A-ramp) can be bricked.
    - **Why it happens:** The pre-loop “zero guard” only sets a **local** `bal = 1` for the sum/Ann calc but **doesn’t update** `_balances[j]`; the inner loop still uses raw zeros, so the denominator becomes 0. Rounding from exchange-rate scaling can also push tiny balances to 0, making this easier.
    - **Where to look:**
        - Invariant code with a product term over balances (Newton iterations) that **multiplies then divides by `_balances[j]`**.
        - Any normalization that can floor to 0 (6/8-dec tokens, ER < 1, wrong multiply/divide order).
        - Callers that allow zero amounts per-token (single-sided flows) without preconditioning.
    - **Red flags:**
        - Patterns like `else bal = 1;` in a **separate** scope from the array actually used in the division.
        - No tests where one asset’s normalized balance is 0 just before `_getD` (including after an ER change).
    - **Fix patterns (pick one or combine):**
        - **Denominator guard (safe & localized):** In `_getD`, use a non-zero floor **only in the divisor**:
        `uint256 bal = _balances[j]; if (bal == 0) bal = 1; pD = (pD * D) / (bal * n);`
        (Keep `sum` based on true balances to avoid inflating D; the floor is a math sentinel, not economic value.)
        - **Normalization floor:** When computing normalized balances, ensure `normalized[i] = max(1, …)` so later divisions can’t hit 0 (document rounding bias).
        - **Input gating (optional):** For APIs that **assume** all tokens participate, `require(amounts[i] > 0)`; keep single-sided flows only where the invariant supports zeros safely.
        - **Consistent math order:** Multiply before divide (or use `mulDiv`) to avoid rounding to zero that creates the condition. *(This is a distinct root cause from our earlier “precision-leak via op order” heuristic.)*
    - **Tests to add:**
        - Single-sided donate/mint with other entries = 0 → should not revert and should not inflate `D`.
        - Near-zero balance, then small ER decrease → subsequent `_getD` calls still succeed.
        - Property test over ER in (0.8, 1.2), mixed decimals, random zeroed entries.
    - **Trade-offs:** Using `1` as a divisor sentinel slightly biases `D` downward when an asset is truly absent; confine the floor **to the divisor** (not the sum) to minimize drift, and consider tracking the tiny rounding residuals in a fee pot to keep accounting exact.
- [ ]  **Donations must settle buffer/insurance bad-debt before crediting surplus**
    - **Symptom:** External “donations” (e.g., `donate()`, `donateD()`, `topUp()`) increase the buffer/insurance pot (`bufferAmount`) but do **not** reduce tracked deficit (`bufferBadDebt` / `shortfall`), while profit/yield paths *do* net bad-debt first. This leaves the system in a de-facto deficit even after new value arrives.
    - **Where to look (Solidity):**
        - Reserve/insurance token or accounting module:
            - `SPAToken.sol` / `ReserveToken.sol` / `InsuranceToken.sol` (look for `addBuffer`, `addReserve`, `addInsurance`, `addTotalSupply`, `absorb`, `coverLoss`).
            - State: `bufferAmount`, `bufferBadDebt` (aka `shortfall`, `deficit`, `protocolLoss`, `insuranceDebt`).
        - Pool/vault entry points that accept donations:
            - `SelfPeggingAsset.sol` / `Vault.sol` / `Pool.sol` (`donateD`, `donate`, `topUpBuffer`, `recapitalize`, `sweepToBuffer`).
        - Profit/yield accrual paths:
            - Functions like `rebase()`, `collectFeeOrYield()`, `distributeFees()`, `addTotalSupply()` that *do* net bad-debt first—compare logic with donation path.
    - **Red flags:**
        - Donation path calls `addBuffer(amount)` (or similar) that **only** does `bufferAmount += amount` and never touches `bufferBadDebt`.
        - Profit path uses a different sink (e.g., `addTotalSupply`) that **first** applies `min(amount, bufferBadDebt)` to clear debt, then allocates remainder.
        - Post-donation state where `bufferBadDebt` stays constant while `bufferAmount` increases (`bufferBadDebt_after == bufferBadDebt_before && bufferAmount_after > bufferAmount_before`).
        - No event or accounting line item showing “bad-debt repaid” on donation.
        - Multiple value-ingress functions (fees, yield, donations) with inconsistent ordering of operations (credit buffer before netting debts).
    - **Questions to ask when reviewing:**
        - “What variable represents outstanding losses/deficit? In which *exact* functions is it decreased?”
        - “Walk me through a donation while `bufferBadDebt > 0`: what is the expected post-state? Which line decrements the debt?”
        - “Why do profit/yield paths net bad-debt but donation paths don’t? Is that intentional? Where is this invariant documented?”
        - “Are there unit tests that donate while in deficit and assert `bufferBadDebt` decreases by `min(donate, bufferBadDebt)`?”
        - “Do any emergency or admin ‘top-up’ functions bypass the centralized absorb/netting logic?”
    - **Fix pattern:**
        - Normalize value ingress through a single internal absorber, e.g.:
            - `_absorb(uint256 amount, Source src)`:
                1. `repaid = min(amount, bufferBadDebt); bufferBadDebt -= repaid; amount -= repaid;`
                2. For `Source.Profit`: split remainder per design (e.g., `bufferAmount += x`, mint/supply += y).
                3. For `Source.Donation`: `bufferAmount += amount` (no supply mint).
            - Make `addBuffer`, `addTotalSupply`, `donateD`, etc. call `_absorb`.
        - Emit explicit events (`BufferBadDebtRepaid(repaid)`, `BufferToppedUp(remaining)`).
        - Add invariants/tests:
            - If `bufferBadDebt_before > 0`, then after donation of `z`:
            `bufferBadDebt_after = bufferBadDebt_before − min(z, bufferBadDebt_before)`.
        - Documentation: state that *all* external value (profit, fees, donations) must extinguish bad-debt first to avoid overstating solvency.
- [ ]  **No slippage bounds on rate-based mint/redeem/liquidate lets users get worse-than-quoted results**
    - **Symptom:** Functions that compute shares/asset amounts from a live exchange rate (cash+borrows−reserves / totalSupply, or ERC-4626 previews) do not accept user-provided min/max bounds. Between quote and execution, interest accrual, reserve ops, or price updates change the rate so depositors mint fewer shares, redeemers get fewer assets, and liquidators seize less collateral than intended.
    - **Where to look (Solidity):**
        - Compound-style tokens: `exchangeRateStored()` / `exchangeRateStoredInternal()` vs `exchangeRateCurrent()`; call sites in `mint/deposit`, `redeem/withdraw`, `liquidateBorrow`.
        - ERC-4626 vaults/adapters: external `deposit/mint/withdraw/redeem` and whether they honor `preview*` quotes under the *same* rate snapshot.
        - Interest/reserve paths invoked inside user flows: `accrueInterest()` calls inside `mint`, `redeem`, `borrow`, `repay`; admin `addReserves/reduceReserves`.
        - Price/oracle usage in liquidation math: `liquidateCalculateSeizeTokens`, seize exchange rate, and whether the liquidation entrypoint takes `minSeize/maxRepay`.
        - Parallel mutators that can move the rate pre-state: other users’ deposits/withdraws, borrows/repays, fee skim, donations.
    - **Red flags:**
        - External functions have signatures like `deposit(uint assets, address to)` / `mint(uint shares, address to)` / `withdraw(uint assets, address to, address owner)` / `redeem(uint shares, address to, address owner)` **without** `minSharesOut`, `maxAssetsIn`, `minAssetsOut`, `maxSharesIn`, or `deadline`.
        - Liquidation entrypoints accept only `repayAmount` and **no** `minSeizeTokens` (or slippage bounds).
        - Flow calls `accrueInterest()` *inside* `mint/redeem`, so `preview*` underestimates/overestimates payment.
        - Exchange rate computed from mutable state (`getCash() + totalBorrows − totalReserves`) with no guard that it matches a quoted snapshot.
        - Tests/no tests: missing unit tests where rate/price changes just before execution; no mempool/frontrun simulations.
    - **Questions to ask when reviewing:**
        - “What guarantees does a caller have about minimum shares minted / assets redeemed? Where is that enforced?”
        - “Can any path (accrual, borrow/repay, reserve ops) change `exchangeRate` between quote and execution? If yes, how does the function cap slippage?”
        - “Do `preview*` functions use the *same* rate as execution? If not, how do we reconcile?”
        - “Does liquidation accept `minSeize` or `minCollateralOut`? What happens if oracle/ER moves one block later?”
        - “Are there deadlines/permit variants to reduce mempool exposure?”
    - **Fix pattern:**
        - Add bounded params to *all* user-facing flows:
            - `deposit(assets, to, uint256 minSharesOut, uint256 deadline)`
            - `mint(shares, to, uint256 maxAssetsIn, uint256 deadline)`
            - `withdraw(assets, to, owner, uint256 maxSharesIn, uint256 deadline)`
            - `redeem(shares, to, owner, uint256 minAssetsOut, uint256 deadline)`
            - `liquidateBorrow(borrower, repayAmount, collateral, uint256 minSeizeTokens, uint256 deadline)`
        - At execution, compute the live amount and `revert Slippage()` if it violates the bound. Optionally snapshot `exchangeRate` once per call to make `preview*` match execution.
        - Consider `permit`/`permit2` support to shorten quote-to-execute window.
        - Tests: inject `accrueInterest()`, reserve changes, oracle moves between preview and call; assert slippage reverts fire.
- [ ]  **Missing ramp-sync guard lets A-changes skew supply/buffer accounting**
    - **Symptom:** One or more external entrypoints (e.g., `rebase()`) skip the “ramp A” sync modifier, so when `A` has changed (via `getCurrentA()`), the pool updates `totalSupply`/buffer using a *different* path than `_syncTotalSupply()`. Result: divergent `totalSupply`, `bufferAmount`, or `bufferBadDebt` depending on call order (e.g., `rebase()` vs any `syncRamping`gated call).
    - **Where to look (Curve-like stableswap / basket AMMs):**
        - Pool contract:
            - Modifiers & gates: `syncRamping`, calls to `_syncTotalSupply()`, `getCurrentA()`, storage `A`.
            - Entry points that change accounting: `rebase()`, `mint/deposit`, `swap`, `redeemSingle`, `redeemMulti`, `redeemProportion`, `donateD`, `distributeLoss`, `collectFeeOrYield`.
            - Math helpers: `_getD(...)`, `getUpdatedBalancesAndD()`.
        - Token/accounting contract (SPA/LP token): `addBuffer`, `addTotalSupply`, `removeTotalSupply`, handling of `bufferBadDebt`.
        - Ramp controller: `RampAController.rampA/stopRampA`, and any time-based A schedule.
    - **Red flags:**
        - Some external functions have `syncRamping` but others (notably `rebase()`) don’t.
        - `_syncTotalSupply()` on “A increased” path uses `addBuffer(...)`, while `rebase()` on growth uses `addTotalSupply(...)` (or vice-versa)—i.e., inconsistent side-effects for the same ΔD/ΔA.
        - Functions compute `newD = _getD(balances, getCurrentA())` but don’t first normalize `balances` the same way other paths do.
        - Order-sensitivity in tests: calling `rebase()` after a completed ramp yields different `totalSupply`/`bufferAmount` than calling any `syncRamping`gated function first, then `rebase()`.
    - **Questions to ask:**
        - “Is there a *single* authoritative path that applies A-changes to `totalSupply`/buffer? Do *all* entrypoints reach it?”
        - “When A increases/decreases, should the delta hit `buffer` or `totalSupply`? Are all paths consistent with that rule?”
        - “Which functions can be front-run to force `_syncTotalSupply()` before/after `rebase()` and produce different state?”
        - “Do we have a unit test that asserts `rebase()` and a no-op call that triggers `syncRamping` produce identical `totalSupply` and `bufferAmount` after an A ramp?”
    - **Fix pattern:**
        - Add the ramp gate universally: `function rebase() external syncRamping returns (uint256) { ... }`.
        - Or refactor: route *every* A-change application through one internal function (e.g., `_applyARampAndSync()`) that (1) refreshes balances using the canonical helper, (2) computes `newD`, and (3) updates `totalSupply`/`buffer`/`bufferBadDebt` with uniform semantics.
        - Normalize math: ensure the same balance/exchange-rate preparation (`getUpdatedBalancesAndD()`) is used across `_syncTotalSupply()` and `rebase()`.
- [ ]  **Double-applied exchange-rate inflates dynamic fees (unit mismatch)**
    - **Symptom:** Dynamic fee inputs (`xs`, `ys`) are computed in “normalized” units twice (e.g., multiplying by the exchange rate after `balances/_balances` were already normalized), leading to overstated fees on `mint`/`redeem` while `swap` uses correct units.
    - **Where to look (stable/basket AMMs with ER providers):**
        - Pool math:
            - `getUpdatedBalancesAndD()` (or equivalent): does it already convert raw token balances to a common unit via `exchangeRate * precision / 10**decimals`?
            - All fee call sites: `mint`, `redeemSingle`, `redeemMulti`, `redeemProportion`, `donateD`, `swap`.
            - Dynamic fee helpers: `_dynamicFee(xs, ys, feeBps)`, `_volatilityFee(...)` — confirm expected **unit domain** for `xs, ys` (raw vs normalized).
        - Any place computing `xs`, `ys`:
            - Patterns like `((balances[i] + _balances[i]) * exchangeRate) / 10**rateDecimals` when `balances/_balances` are **already** exchange-rate adjusted.
            - Contrast with swap path (often correct): inputs averaged without re-applying ER, e.g. `(prevBalance + newBalance)/2`.
    - **Red flags:**
        - `balances` is assigned as `balance * exchangeRate * precision / 10**rateDecimals`, **and later** code does `something = (balances[...] + _balances[...]) * exchangeRate / 10**rateDecimals` before `_dynamicFee(...)`.
        - Inconsistent domains across routes: `swap` passes mid-balances without ER; `mint/redeem` re-applies ER → different fees for equivalent state changes.
        - Mixed scaling: some places multiply by `precision` before divide; others divide then multiply (precision loss) or repeat scaling.
        - Comments/docs for `_dynamicFee` don’t specify the expected unit (raw token vs normalized invariant space).
    - **Questions to ask:**
        - “Are `balances/_balances` already normalized to invariant units when we compute `xs/ys`?”
        - “What unit does `_dynamicFee(xs, ys, ...)` expect? Raw token, normalized, or D-space?”
        - “Do all fee paths (`mint`, all `redeem*`, `swap`) pass `xs/ys` in the same unit domain?”
        - “If I toggle exchange rate by 1%, do `mint` and `swap` routes quote proportional fees for equivalent net state change?”
        - “Is there a single helper to construct `xs/ys` to avoid duplication and drift?”
    - **Fix pattern:**
        - Make `xs/ys` unit-safe:
            - If `balances/_balances` are already normalized, **do not** multiply by exchange rate again: `xs = balances[i] + _balances[i];`
            - Or centralize normalization: create `toNormalized(balanceRaw, i)` and **use once**.
        - Standardize across paths: ensure `mint/redeem/swap` compute fee inputs via the same helper; add unit annotations in code comments.
- [ ]  **Mint-before-fee distribution rebates part of the fee to the payer (order-of-ops bug)**
    - **Symptom:** The depositor is minted LP shares **before** fees/yield are distributed (e.g., `mintShares(...)` precedes `collectFeeOrYield(true)` / `addTotalSupply(...)`). Because distribution rewards **current** holders, the just-minted user captures a pro-rata slice of the fee they paid.
    - **Where to look (AMMs, baskets, vaults, ERC4626 wrappers):**
        - Deposit/mint paths in pool/vault:
            - Functions like `mint`, `deposit`, `joinPool`, `addLiquidity`.
            - The sequence around: (i) transfer in assets → (ii) update virtual balances/invariant → (iii) **mint shares** → (iv) **collect fee / harvest / rebase / addTotalSupply**.
        - Fee/yield distribution hooks:
            - Helpers such as `collectFeeOrYield`, `collectFees`, `harvest`, `rebase`, `skim`, `accFeePerShare` updates, or token methods `addTotalSupply`, `rebaseToHolders`, `distributeFees`.
        - Token module:
            - LP/share token calls that “reward all current holders”, e.g., rebases or “mint to treasury then pro-rata reflect”.
        - Symmetry check on exit:
            - `redeem*` / `withdraw*`: ensure **burn happens before** fee distribution so redeemers don’t share in fees from their own exit.
    - **Red flags:**
        - Exact sequence:
        `... balances[i] = _balances[i]; transferFrom(...);totalSupply = oldD + mintAmount;mintShares(msg.sender, mintAmount);` **then** `collectFeeOrYield(true);`
        - Comments like “collect fees” or “rebase” **after** a user becomes a holder.
        - Distribution primitives that modify `totalSupply`/per-share accounting *after* membership changes.
        - Net-of-fee `mintAmount` is computed correctly, but a subsequent distribution still occurs before the depositor is excluded.
        - In tests, value of the new LP’s shares immediately after mint is **greater** than the value implied by `mintAmount` (i.e., implicit rebate).
    - **Questions to ask:**
        - “At what precise line do we add the new LP to the holder set vs. when we distribute fees/yield?”
        - “Does our distribution primitive reward **all current** holders? If yes, how are new minters excluded from fee they just paid?”
        - “Is the exit path symmetric (burn before distribution) so redeemers don’t share in fees tied to their own exit?”
        - “Do we have a unit/invariant test asserting the payer’s post-mint share value equals `mintAmount` (no instant gain)?”
        - “If fees are pushed into a ‘buffer’, does any subsequent ‘addTotalSupply/rebase’ still reward the current minter?”
    - **Fix pattern:**
        - **Reorder operations:** perform fee/yield realization **before** minting depositor shares. E.g., compute and call `collectFeeOrYield(true)` (or update `accFeePerShare`) first, then `mintShares(msg.sender, mintAmount)`.
        - Or **temporarily exclude** the depositor from the distribution: mint to a staging address, distribute, then transfer shares to the depositor.
        - Alternative: route fees to a non-pro-rata bucket (e.g., “buffer/treasury”) and only rebase to holders **prior** to adding the new holder.
- [ ]  **Proportional multi-asset redemption bypasses fees (difference=0 dynamic-fee path)**
    - **Symptom:** A “multi-asset” redemption path (e.g., `redeemMulti(amounts)`) computes fees from the **imbalance** `|idealBalance - newBalance|`. If a user requests amounts perfectly proportional to pool balances/`D` (so `ideal == new`), the **difference is zero → fee = 0**, bypassing the flat/percentage `redeemFee` that is charged on other paths (e.g., `redeemProportion(shares)`).
    - **Where to look (stableswap/basket AMMs, multi-redeem vaults):**
        - Redeem/withdraw functions:
            - Compare `redeemProportion/withdraw(shares)` vs `redeemMulti(amounts)` / `redeemExact` variants.
            - Any fee path that looks like:
            `difference = abs(idealBalance - _balances[i]);` → `fees[i] = difference * (dynamicFee + volFee) / DENOMINATOR;`
        - Invariant / accounting helpers:
            - Functions computing `oldD`, `newD`, `idealBalance = newD * balances[i] / oldD`, `getUpdatedBalancesAndD()`.
            - Fee helpers like `_dynamicFee(xs, ys, redeemFee)` / `_volatilityFee(...)`.
        - LP token side effects:
            - Where `totalSupply`, `buffer`, or `collectFeeOrYield(true)` is applied during exit; ensure alignment across all exit paths.
    - **Red flags:**
        - Two code paths for redemption with **different fee models**: one uses **flat/percentage-of-shares** (e.g., `redeemFee` on `redeemProportion`) and the other uses **imbalance-only** fees (on `redeemMulti`).
        - Fee computed **only** from `difference` with **no base/min exit fee**.
        - `idealBalance` derived from `oldD/newD` and current `balances` such that proportional amounts make `difference == 0`.
        - Tests or comments implying “users can withdraw proportionally” without acknowledging a fee.
        - Extremely small dust thresholds (`epsilon`) that allow near-proportional withdrawals with effectively zero fee due to rounding.
    - **Questions to ask:**
        - “Do all redemption methods charge at least the same **baseline exit fee**? If not, why is one path exempt?”
        - “Is the fee model meant to compensate **existing LPs** for any exit, or only for **unbalancing** exits?”
        - “What happens if a user requests `_amounts` exactly proportional to `balances` (or to `idealBalance` from `D`)? Is `fee == 0` by construction?”
        - “Do we enforce a **minimum fee** or a **tolerance band** so that `difference≈0` doesn’t zero out fees?”
        - “Are `redeemMulti` and `redeemProportion` sharing a **unified fee engine**, or are they drifted implementations?”
    - **Fix pattern:**
        - **Unify exit fees** across all redeem paths:
            - Charge a **base exit fee** (percentage of value/shares) on **every** redemption, then **add** imbalance/volatility surcharges when applicable.
            - Alternatively, compute fees from the **shares burned** (or from `ΔD`) rather than solely from per-token imbalance.
        - Introduce a **min fee floor** (e.g., a few bps) and an `epsilon` guard so near-proportional withdrawals still pay baseline fees; document and cap `epsilon`.
        - Reuse the same fee helper for `redeemProportion` and `redeemMulti`, or route both through a single internal function that:
            1. derives `oldD/newD`,
            2. computes baseline exit fee on redeemed value/shares,
            3. adds imbalance/volatility components,
            4. applies fees **before** finalizing state.
- [ ]  **Orphaned `borrowMembership` after `exitMarket` lets borrowers bypass collateral checks**
    - **Symptom:** A user exits a market (removed from `accountAssets[]` / `collateralMembership`), but their **`borrowMembership` remains `true`** (e.g., set via `borrow(0)`). Subsequent borrows in that market **don’t re-add** the market to `accountAssets[]`, so **liquidity/debt checks skip it**, allowing borrowing without collateral accounting.
    - **Where to look (Compound-style lending / Comptroller-like “RiskEngine”):**
        - Membership + assets bookkeeping:
            - Controller: `enterMarket/exitMarket`, `addToMarketBorrowInternal`, `markets[pToken].{collateralMembership,borrowMembership}`.
            - Per-account holdings: `accountAssets[account]` array vs membership flags.
        - Borrow gatekeeping:
            - `borrowAllowed(pToken, borrower, amount)` (does it push into `accountAssets[]` only when membership is false?).
            - Any path where `borrow(0)` is permitted and toggles `borrowMembership`.
        - Liquidity math:
            - `getHypotheticalAccountLiquidity(…)` / `…Internal`: Does it **iterate only `accountAssets[account]`** to compute collateral and debts?
    - **Red flags:**
        - `exitMarket` removes from `accountAssets[]` when `amountOwed == 0` but **does not clear `borrowMembership[account]`**.
        - `borrowAllowed` code like:
            
            ```solidity
            if (!markets[pToken].borrowMembership[borrower]) {
                addToMarketBorrowInternal(...); // adds to accountAssets
            }
            // if borrowMembership already true, skip adding to accountAssets
            
            ```
            
        - **Zero-amount borrow** allowed, flipping membership:
            
            ```solidity
            pToken.borrow(0); // sets borrowMembership = true
            
            ```
            
        - Liquidity functions only loop `for (p in accountAssets[account]) { … }` and **don’t union** with markets where `borrowMembership==true`.
        - No invariant/assert ensuring:
        `pToken ∈ accountAssets[acct] ⇔ (collateralMembership || borrowMembership)`.
    - **Questions to ask:**
        - “Does `exitMarket` clear **both** collateral and borrow membership when `amountOwed==0`?”
        - “Is borrowing `0` allowed anywhere? What flags does it mutate?”
        - “If `borrowMembership` is true but the market is missing from `accountAssets[]`, will `borrowAllowed` **reinsert** it **every time** (not only when membership is false)?”
        - “Do liquidity/debt calculations derive markets from the **union** of: `accountAssets[]`, `borrowMembership==true`, and **non-zero borrow balances**?”
        - “Is there a periodic invariant check / migration to reconcile `accountAssets[]` with membership flags?”
    - **Fix pattern:**
        - **State consistency:**
            - In `exitMarket`, when `amountOwed==0`, **set `borrowMembership[account]=false`** in addition to removing from `accountAssets[]`.
            - In `borrowAllowed`, ensure the market is present in `accountAssets[]` **even if** `borrowMembership[borrower] == true`:
                
                ```solidity
                if (!isInAccountAssets(borrower, pToken)) {
                    addToAccountAssets(borrower, pToken);
                }
                
                ```
                
        - **Input hardening:** **Disallow `borrow(0)`** (and any zero-effect ops) from mutating membership:
            
            ```solidity
            if (borrowAmount == 0) revert ZeroBorrow();
            
            ```
            
        - **Accounting robustness:** Make `getHypotheticalAccountLiquidity` compute over **all relevant markets**
        (`accountAssets ∪ {p | borrowBalance(p, acct) > 0} ∪ {p | borrowMembership[p][acct]}`) or **derive `accountAssets`** from those sets.
        - **Invariant & tests:**
            - Add an invariant: for all `acct`, `p`:
            `borrowMembership[p][acct] || collateralMembership[p][acct]` ⇒ `p ∈ accountAssets[acct]`.
            - Unit test the exploit flow: `borrow(0)` → `exitMarket` → attempt large borrow ⇒ **must revert** on liquidity shortfall and/or reinsertion into `accountAssets`.
            - Backfill/migration script to reconcile historical state (clear stray `borrowMembership` where `amountOwed==0` and not in `accountAssets`).
- [ ]  **Front-run repay shrinks `maxClose` and reverts liquidation (DoS via “repay 1 wei”)**
    - **Symptom:** Liquidations revert when the borrower front-runs with a tiny `repayBorrow` (e.g., 1 wei), reducing `borrowBalance` so `repayAmount > maxClose` (close-factor cap) and `liquidateBorrowAllowed` returns `TOO_MUCH_REPAY`.
    - **Where to look (Compound/Pike-style lending):**
        - Controller/Risk engine:
            - `liquidateBorrowAllowed(pTokenBorrowed, borrower, repayAmount)` and close-factor math:
                
                ```solidity
                uint256 maxClose = closeFactor * borrowBalance;
                if (repayAmount > maxClose) return TOO_MUCH_REPAY;
                
                ```
                
            - Any pre-check that uses **live** `borrowBalance{Stored/Current}` after accrual.
        - pToken / market:
            - `liquidateBorrow()` → calls `riskEngine.liquidateBorrowAllowed(...)`.
            - Repay paths the borrower can use to front-run: `repayBorrow`, `repayBorrowBehalf`, especially permitting `repayAmount == 0 || 1`.
        - Mempool sensitivity:
            - Off-chain helpers/SDKs that compute `maxClose` for liquidators using a **snapshot** then submit a tx with a **fixed** `repayAmount`.
    - **Red flags:**
        - Hard revert on `repayAmount > maxClose` instead of clamping to `maxClose`.
        - Borrower-accessible, gas-cheap micro-repay that mutates `borrowBalance` without fees/min amounts.
        - No liquidation **tolerance/min-out** parameters; liquidator must pass exactly computed `repayAmount`.
        - `maxClose` derived post-`accrueInterest()` with **no** adjustment path (e.g., `repayAmount = min(repayAmount, maxClose)`).
        - Tests only for “equals” case, not “greater-than by 1 wei”.
    - **Questions to ask:**
        - “If `repayAmount` is slightly above `maxClose` at execution time, do we **revert** or **cap** the repayment?”
        - “Can a borrower call `repayBorrow(1)` cheaply and change `borrowBalance` between quote and execution?”
        - “Do we require a **minimum repay amount** for borrowers (to avoid micro-dust front-runs)?”
        - “Does `liquidateBorrow()` accept **slippage/constraints** (e.g., min collateral seized) so we can safely **cap repay** without harming liquidators?”
        - “Are we recomputing `maxClose` after **all** accrual and balance updates in the same tx?”
    - **Fix pattern:**
        - **Clamp, don’t revert:** In `liquidateBorrowAllowed` or in the market method, set `repayAmount = min(repayAmount, maxClose)` and proceed.
            
            ```solidity
            uint256 maxClose = closeFactor.mul_ScalarTruncate(borrowBalance);
            if (repayAmount > maxClose) {
                repayAmount = maxClose; // instead of revert
            }
            
            ```
            
        - **Add liquidator protections:** Accept `minSeizeTokens` / `minCollateral` (slippage guard) so clamping won’t create MEV grief.
        - **Dust guard for borrower:** Enforce a reasonable `minRepay` (or fee) to disincentivize 1-wei front-runs.
        - **Recalc ordering:** Ensure `accrueInterest()` and **latest balances** are used before computing `maxClose`.
- [ ]  **Insolvent-liquidation frees more collateral than user has (per-account underflow → DoS)**
    - **Symptom:** During liquidation / debt burn, the contract computes a `toFree` amount and subtracts it from both a **global** lock and the **account’s** `rawLocked`. If the position is insolvent, `toFree > account.rawLocked`, causing a Solidity 0.8 underflow revert and **blocking liquidation** (bad debt persists).
    - **Where to look (Alchemix/Alchemist-style “lock & free” systems):**
        - Debt decrement paths called from liquidation/repay:
            - Functions like `_subDebt`, `_burnDebt`, `_onLiquidate`, `_repayInternal`.
            - Lines that update both **global** and **per-account** collateral locks, e.g.:
                
                ```solidity
                uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / 1e18;
                if (toFree > _totalLocked) toFree = _totalLocked;   // capped globally only
                _totalLocked -= toFree;
                account.rawLocked -= toFree; // ⚠ may underflow when insolvent
                
                ```
                
        - Converters & risk params:
            - `convertDebtTokensToYield`, price/oracle adapters, and `minimumCollateralization` multipliers.
        - Storage layout:
            - Per-account fields like `rawLocked`, `freeCollateral`, and globals like `_totalLocked`, `totalDebt`.
        - Liquidation entry points:
            - `liquidate(tokenId)` → calls into the above accounting.
    - **Red flags:**
        - Capping `toFree` only by a **global** variable (`_totalLocked`) but not by `account.rawLocked`.
        - No explicit insolvency branch/guard (assumes “liquidations only when above min LTV”).
        - Direct subtraction on per-account lock without `min(toFree, account.rawLocked)` or saturating math.
        - Comments that mention “cases above minimum LTV” with no “else” handling.
        - Reliance on current price conversions that can exceed per-account locks.
    - **Questions to ask:**
        - “What happens if an account is **insolvent** (negative equity)? Do we clamp `toFree` to `account.rawLocked`?”
        - “Is there a **bad-debt** path when locked collateral can’t cover the computed release?”
        - “Are **global** and **per-account** totals kept consistent under all branches (repay, liquidate, write-off)?”
        - “Does `convertDebtTokensToYield` use prices that can overshoot per-account locks, and is that handled?”
    - **Fix pattern:**
        - **Clamp per-account:** `uint256 freeable = toFree; if (freeable > account.rawLocked) freeable = account.rawLocked;`
        Then update `_totalLocked = _totalLocked > freeable ? _totalLocked - freeable : 0;` and `account.rawLocked -= freeable;`
        - **Insolvency branch:** If `toFree > account.rawLocked`, treat the excess as **unrecoverable** for that account; record/write off bad debt or charge protocol reserves instead of reverting.
        - **Safety:** Use explicit `min()` on both global and per-account locks; avoid relying solely on Solidity 0.8 checked math; ensure order of updates can’t transiently underflow.
- [ ]  **Cross-module “repeg” loop broken → bad-debt ratio never heals**
    - **Symptom:** Redeemers are *always* haircut (e.g., `claimAmount = claimAmount * 1e18 / badDebtRatio`) because `badDebtRatio` (BDR) stays > 1.0. Funds redeemed/collected sit in an auxiliary module (transmuter/escrow) and never return to the core vault that the BDR reads from, so BDR never improves across claims.
    - **Where to look (Alchemix/Alchemist-style “transmuter + core vault” setups):**
        - Auxiliary redemption/escrow modules: `Transmuter.sol` / `Redemption.sol` functions `createRedemption`, `claimRedemption`, `settle`, `harvest`, `sweep`, fee transfers.
        - Core accounting contract (vault/CDP): `AlchemistV3.sol` or equivalent:
            - BDR inputs: `totalSyntheticsIssued`, `getTotalUnderlyingValue`, `badDebt`, `convertDebtTokensToYield`.
            - Inbound settlement hooks: `onRedeemSettle`, `repayBadDebt`, `depositFromTransmuter` (or missing).
        - Transfer paths inside `claimRedemption`: look for `safeTransfer(yieldToken, msg.sender, …)` / fee receiver transfers **without** a matching transfer back to core or a state update that reduces BDR.
        - Any price-based metric that only reads `balanceOf(core)` and ignores balances held by helper modules.
    - **Red flags:**
        - BDR/solvency computed solely from **core** contract holdings (`getTotalUnderlyingValue()` uses `IERC20(yield).balanceOf(address(core))`) while redeemed funds accumulate in a different contract.
        - `claimRedemption` scales down outputs by BDR, yet does **not** call back into core to credit assets or repay bad debt.
        - No periodic `settle()`/`sweep()` that moves assets from transmuter → core; or it exists but is permissioned/off-chain-triggered and never called.
        - Comments/TODOs like “transfer to core later” or fee-only transfers to `protocolFeeReceiver`.
        - Unit tests only assert per-tx haircuts, never assert **BDR decreases over time** after redemptions.
    - **Questions to ask:**
        - “Exactly which balances feed `badDebtRatio`? Do they include assets held by the transmuter/AMO/escrow, or only the core vault?”
        - “After a claim, which contract *owns* the underlying? Where is the write that reduces `badDebt` or increases `getTotalUnderlyingValue()`?”
        - “Is there a public/keeper `settle()`? Who can call it? What happens if it’s skipped—does BDR stagnate?”
        - “What invariant ensures `BDR_{t+1} ≤ BDR_t` when net assets increased? Is it tested end-to-end?”
        - “If claims/fees are paid out, from which pot? Are we accidentally paying users while *not* repairing core solvency?”
    - **Fix pattern:**
        - **Repatriate on claim:** In `claimRedemption`, transfer the redeemed underlying (and/or a computed “repeg amount”) **to the core** (e.g., `alchemist.repayBadDebt(amount)` or `alchemist.depositFromTransmuter(amount)`) before or along with user payouts, and update core accounting.
        - **Consolidated accounting:** Compute BDR over **aggregate** assets (core + transmuter receivables), or persist a “due to core” liability that `getTotalUnderlyingValue()` includes.
        - **Sync hook:** Add a callable `settle()` that sweeps module balances to core and updates bad-debt state; wire it into claim flows or cron/keeper.
        - **Tests/invariants:** Add tests that simulate bad debt, run multiple claims, and assert BDR monotonically decreases to ≤ 1 when net assets suffice. Add invariant: “if module balance ≥ deficit, BDR returns to 1 after settle/claim.”
- [ ]  **Redemption sync misses free-collateral credit → funds get stranded**
    - **Symptom:** After redemptions repay a user’s debt, `_sync()` (or equivalent reconciliation) reduces `rawLocked` / `collateralBalance` but **doesn’t credit `freeCollateral`**, so users can’t withdraw or re-mint even though their debt is gone. Balance appears “stuck.”
    - **Where to look (Alchemix/Alchemist-style CDP + transmuter systems):**
        - Core vault/CDP contract (e.g., `AlchemistV3.sol`):
            - Reconciliation paths: `_sync()`, `_subDebt()`, redemption settlement, earmark accruals, and any “claim”/“harvest” handlers.
            - State fields and gates: `collateralBalance`, `rawLocked`, `freeCollateral`, `earmarked`, `debt`, and `_Weight` accumulators.
            - Withdraw/mint guards that check `freeCollateral` or `collateralBalance - rawLocked`.
        - Redemption/Transmuter module:
            - `claimRedemption()` → look for core callbacks/updates that eventually adjust `freeCollateral` on the CDP.
        - Any central “account struct” writes around redemptions and debt changes.
    - **Red flags:**
        - Code blocks that **decrease** `rawLocked` or **reduce** `debt`/`earmarked` **without** a symmetric **increase** to `freeCollateral`.
        - Invariants not enforced anywhere: `freeCollateral + rawLocked == collateralBalance` (or ≤ with fees).
        - Withdraw/mint reverts after full debt payoff via redemption; tests only cover normal repay but **not** “debt cleared via redemption → withdraw all.”
        - No “sweep/unblock” admin path; no periodic recomputation of `freeCollateral` from first principles.
        - Comments like `// redeem user collateral` paired with `rawLocked -= ...` but **no** `freeCollateral += ...`.
    - **Questions to ask next time:**
        - “What hard invariant ties `freeCollateral`, `rawLocked`, and `collateralBalance`? Where is it enforced after every redemption/debt update?”
        - “When debt decreases via **redemption** (not repay), which line credits collateral back to the user’s **withdrawable** bucket?”
        - “Do withdraw/mint checks read a **derived** value (e.g., `collateralBalance - rawLocked`) or a **stored** `freeCollateral` that can drift?”
        - “Is there a single source of truth for withdrawability, or multiple fields that must move in lockstep?”
    - **How to hunt it quickly:**
        - Grep for writes: `rawLocked -=`, `collateralBalance -=`, `debt -=`, `earmarked =`, and inspect the **same block** for `freeCollateral +=`.
        - Trace redemption flow: `claimRedemption()` → core `_sync()` → account struct mutations; ensure credit path exists.
        - Add an invariant test: after any call that changes debt/locked values, assert `freeCollateral + rawLocked == collateralBalance` (± fee dust).
    - **Fix pattern:**
        - In the reconciliation that removes locked collateral (e.g., `collateralToRemove`), **also** do `freeCollateral += collateralToRemove` (bounded by `collateralBalance`), or refactor withdraw guard to compute `available = collateralBalance - rawLocked` so it can’t drift.
        - Add an internal `_recomputeFreeCollateral(account)` that derives from balances and call it after redemptions/slashes/liquidations.
        - Add invariants/Events: emit before/after values; unit tests where redemption fully clears debt then **withdraw all** succeeds.
        - Consider consolidating to a single authoritative metric (derive-on-read) to avoid multi-field divergence.
- [ ]  **Hot-swap of external module (Transmuter) without state migration corrupts accounting**
    - **Symptom:** After switching the transmuter, global/accounting totals drift: `totalDebt` and `totalSyntheticsIssued` stay inflated, `cumulativeEarmarked` and weight variables (`_redemptionWeight`, `_collateralWeight`) don’t reflect what was actually redeemed, pending earmarks get stuck, and liquidation math can falsely treat the system as under-collateralized (e.g., full liquidations triggered by a phantom high global debt).
    - **Where to look (Alchemix-like architectures / modular DeFi):**
        - Admin setters that swap critical downstream components: `setTransmuter`, `setController`, `setVault`, `setRouter`, etc.
        - The “redemption/repayment” path that normally updates invariants: functions like `redeem()`, `_earmark()`, weight updates, and global totals (`totalDebt`, `cumulativeEarmarked`, `totalSyntheticsIssued`, `_totalLocked`).
        - The migration pre-check in the setter (e.g., `require(convertYieldTokensToDebt(balanceOf(oldTransmuter)) >= oldTransmuter.totalLocked())`) and any docs/comments telling ops to “send tokens directly” before switching.
        - `onlyTransmuter` (or similar) guards: any logic that *only* updates state when called by the old module—direct token transfers will bypass it.
    - **Red flags:**
        - Setter merely reassigns the module address (e.g., `transmuter = value`) after a balance check; no call to a migration routine, no state reconciliation, no pause window.
        - Guidance to top up the old module with direct ERC20 transfers to satisfy a coverage check (bypasses `redeem()`/book-keeping).
        - No adjustments to `cumulativeEarmarked`, `totalDebt`, `totalSyntheticsIssued`, or weights at the switch boundary.
        - Pending redemptions/earmarks exist at the time of switch (non-zero `totalLocked` / `cumulativeEarmarked`) and there is no “drain & settle” step.
        - Accounting depends on *internal* counters but the pre-switch fund move is done via *external* token transfers.
    - **Questions to ask next time:**
        - What invariants are guaranteed to change *only* via guarded functions (e.g., `onlyTransmuter`)? Can ops ever move funds in ways that bypass these guards?
        - Is there a defined two-step migration (pause → settle/redemption finalization → state reconciliation → switch) with tests for mid-redeem swaps?
        - How are `cumulativeEarmarked`, `totalDebt`, weights, and `_totalLocked` reconciled at handover? Is there a one-shot “reconcile(oldTransmuter)” call?
        - Does the new module inherit or pull state from the old one, or does the Alchemist reconcile its own counters to on-chain balances?
        - Are direct donations to the transmuter/Alchemist normalized or explicitly rejected during migration windows?
    - **Fix pattern:**
        - Implement a **stateful migration**: add `beginTransmuterMigration(old, new)` that (a) pauses redemptions/minting, (b) calls a *settle* routine to `redeem`/apply earmarks through the **old** transmuter so all internal counters (`cumulativeEarmarked`, `totalDebt`, weights) update, (c) optionally pulls residual funds back to the Alchemist, then (d) `finalizeTransmuterMigration(new)`.
        - If full settlement isn’t possible, add a **reconciliation function** that snapshots external balances and **atomically adjusts** internal counters to match (e.g., decrement `totalDebt`/`cumulativeEarmarked`, bump weights) before flipping the address.
        - Prohibit or ignore **direct token top-ups** as a way to pass the coverage check; enforce settlement via contract calls, or treat donations as accounting events.
        - Add a timelock + pause around the swap and tests that switching with non-zero pending redemptions preserves invariants.
- [ ]  **Signed-range miscalculation in Fenwick/“staking graph” enables overflow & state corruption**
    - **Symptom:** Graph limits use `2**BITS` as if values were unsigned; with signed ints the true range is `[-2**(BITS-1), 2**(BITS-1)-1]`. Large (or cumulative) deltas/products can exceed the real max, corrupting the tree and downstream accounting.
    - **Where to look (Fenwick/weight graphs in staking/redemption math):**
        - Files named like `StakingGraph.sol`, `Fenwick.sol`, `WeightGraph`, `PositionDecay`, “Graph”, or “Index”.
        - Constant definitions: `DELTA_BITS`, `PRODUCT_BITS`, `_MAX/*_MIN` and any `int256(2**BITS)` math; uses of `BLOCK_SCALING_FACTOR`, `timeToTransmute`, or similar time/value scalers.
        - Update paths that compute graph deltas: `_updateStakingGraph(...)`, `_add/_update`, `recordStake/unstake`, `earmark`, `redeem`, `accumulateWeight`.
        - Casts to signed: `.toInt256()`, `int256(x)` right before insertion into the graph.
        - Query/accumulator functions: `queryGraph`, `prefixSum`, `lowerBound` that assume no overflow in stored nodes.
    - **Red flags:**
        - Constants like:
            
            ```solidity
            int256 constant DELTA_MAX   = int256(2**DELTA_BITS) - 1;   // ❌
            int256 constant DELTA_MIN   = -int256(2**DELTA_BITS);      // ❌
            
            ```
            
            (should be `1 << (BITS-1)` based).
            
        - Mixed fixed-point scaling: `amount * 1e8 / timeToTransmute` (or similar) fed into a limited-width signed slot.
        - “Cumulative” semantics (Fenwick nodes store sums) without guarding total range; comments like “stores cumulated value”.
        - Minimum time windows (e.g., 1 week in blocks) + large deposit units (1e18) + extra scale factors (1e8) → huge deltas.
        - No explicit `require`/assert that `delta` and intermediate products fit within `[*_MIN, *_MAX]` before write.
        - Packed storage of signed subfields (e.g., int112/int144) without range checks.
    - **Questions to ask next time:**
        - What are the true signed ranges of the graph subfields? Are max/min constants derived as `type(intN).max/min` or via `1 << (N-1)`?
        - Are deltas cumulative in the structure? What’s the worst-case cumulative delta under min `timeToTransmute` and max deposit scale?
        - Do we cap deposit size, enforce min time, or down-scale `BLOCK_SCALING_FACTOR` to ensure deltas fit?
        - Any migration/reset that can shrink accumulated values, or do nodes grow unbounded?
        - Are there fuzz/property tests that push deltas near limits and assert no overflow/corruption?
    - **Fix pattern:**
        - Define ranges via shifts (or native types):
            
            ```solidity
            int256 constant DELTA_MAX   = (int256(1) << (DELTA_BITS - 1)) - 1;
            int256 constant DELTA_MIN   = - (int256(1) << (DELTA_BITS - 1));
            int256 constant PRODUCT_MAX = (int256(1) << (PRODUCT_BITS - 1)) - 1;
            int256 constant PRODUCT_MIN = - (int256(1) << (PRODUCT_BITS - 1));
            // or store in intN and use type(intN).max/min directly
            
            ```
            
        - Add pre-insertion guards:
            
            ```solidity
            int256 delta = syntheticDepositAmount.toInt256()
                * int256(BLOCK_SCALING_FACTOR) / timeToTransmute.toInt256();
            if (delta < DELTA_MIN || delta > DELTA_MAX) revert DeltaOutOfRange();
            
            ```
            
        - Revisit scaling: reduce `BLOCK_SCALING_FACTOR`, increase `timeToTransmute` floor, or widen storage (e.g., int128/ int192) for safety margin.
        - Prefer widening intermediates to `int256` and only store after checked casts to the target width.
        - Add invariant tests: cumulative sequence of max-size updates must not exceed bounds; fuzz min time × large deposits; assert graph queries remain consistent near limits.
- [ ]  **Burn gating uses debt instead of outstanding synthetic supply → valid burns revert**
    - **Symptom:** `burn()` rejects repayment with checks like `credit > totalDebt - totalLocked`, even though the redeemable synthetic supply hasn’t changed. After `repay()` decreases `totalDebt` (but doesn’t burn synths), subsequent `burn()` calls can revert unnecessarily.
    - **Where to look (stablecoin/synthetic debt + transmuter/redemption systems):**
        - Core accounting vars: `totalDebt`, `totalSyntheticsIssued` (or `totalSupply` of the synth), `totalLocked/cumulativeEarmarked`, `totalUnderlyingValue/balances`.
        - Debt lifecycle functions: `mint()`, `burn()`, `repay()`, `redeem()/claimRedemption()`, `earmark()`; any gateway that caps burns using a “capacity” formula.
        - Guard clauses around burn/repay: patterns like
        `if (amount > totalDebt - transmuter.totalLocked()) revert;`
        - Places where **repayment** changes only `totalDebt` but **does not** burn/balance down the synthetic token supply.
        - Transmuter/AMM modules that compute solvency against **redeemable supply** (outstanding synths) rather than `totalDebt`.
    - **Red flags:**
        - Invariant mismatch: `repay()` → `totalDebt--` while `totalSyntheticsIssued` is unchanged.
        - Burn capacity computed from `totalDebt - totalLocked` instead of `(outstanding synth supply) - totalLocked`.
        - Mixed terminology (“debt” vs “issued”) used interchangeably in guards and accounting.
        - Negative/underflow-prone headroom if `totalDebt < totalLocked`.
        - Event/accounting asymmetry: `repay()` affects transmuter solvency checks indirectly, but redemption logic is based on synth supply.
    - **Questions to ask next time:**
        - Which variable represents **what users can redeem** right now—`totalDebt` or `totalSyntheticsIssued`? What invariant ties them?
        - Does `repay()` burn synths, escrow them, or just credit an internal balance? If not burned, how is transmuter capacity protected?
        - Is the burn headroom calculated from **outstanding supply** (e.g., `totalSyntheticsIssued - totalLocked`) or from `totalDebt`?
        - Could a sequence `repay(); createRedemption(); burn();` revert despite solvency? Any tests for that ordering?
        - Do redemption/claim paths update the same metric that the burn guard uses?
    - **Fix pattern:**
        - Gate burns against **outstanding redeemable supply**, not raw debt:
            
            ```solidity
            uint256 headroom = totalSyntheticsIssued > totalLocked
                ? totalSyntheticsIssued - totalLocked
                : 0;
            if (credit > headroom) revert BurnLimitExceeded(credit, headroom);
            
            ```
            
        - Alternatively, **align accounting**: make `repay()` burn synthetic tokens (or decrement `totalSyntheticsIssued`) and/or move repaid synths into an explicit escrow bucket that is excluded from “outstanding”.
        - Add invariants/tests: after `repay()` reduces `totalDebt`, a subsequent `burn()` of ≤ outstanding–locked must succeed; fuzz ordering of `repay/createRedemption/burn`.
- [ ]  **Liquidation debits user state in one unit but transfers more in another**
    - **Symptom:** In liquidation, the contract reduces the borrower’s `collateralBalance` by an amount in **yield-token units** (e.g., `adjustedLiquidationAmount`) but transfers **`adjustedDebtToBurn`** (also yield-token units) to the transmuter without capping it to the user’s remaining collateral. If `convertDebtTokensToYield(debtToBurn) > account.collateralBalance`, the extra comes from the pool (other users) or the tx reverts.
    - **Where to look (CDP + redemption/transmuter architectures):**
        - Liquidation path: functions like `_liquidate/finishLiquidation/calculateLiquidation`, especially around
            - conversions: `convertDebtTokensToYield`, `convertYieldTokensToDebt`
            - per-account debits: `account.collateralBalance`, `rawLocked`, `freeCollateral`
            - global transfers to a transmuter/AMM: `safeTransfer(yieldToken, transmuter, ...)`
        - Any pricing route where the **yield token exchange rate can move** (ERC4626 `pricePerShare`, oracle rate) between computing `debtToBurn` and seizing collateral.
        - Rounding/ceil-vs-floor decisions in conversions and fee application.
        - Invariants tying **per-account** collateral debits to **global** token outflows.
    - **Red flags:**
        - Compute both:
            
            ```solidity
            uint256 adjustedLiquidationAmount = convertDebtTokensToYield(liquidationAmount);
            uint256 adjustedDebtToBurn      = convertDebtTokensToYield(debtToBurn);
            
            ```
            
            then do:
            
            ```solidity
            account.collateralBalance = account.collateralBalance > adjustedLiquidationAmount
                  ? account.collateralBalance - adjustedLiquidationAmount : 0;
            TokenUtils.safeTransfer(yieldToken, transmuter, adjustedDebtToBurn); // no cap
            
            ```
            
        - Any case where **transfer amount** is not `min(user-collateral, needed-in-yield)` or not recomputed **after** collateral seizure.
        - Mixed units in the same block (debt tokens vs. yield tokens) without a reconciliation step.
        - Liquidation logic assumes the pool can front collateral shortfalls (“global pot” leakage).
    - **Questions to ask next time:**
        - Is the **amount transferred out** during liquidation **strictly bounded** by the borrower’s seized collateral (after fees/rounding)?
        - If `convertDebtTokensToYield(debtToBurn)` exceeds available user collateral due to price moves/rounding, do we **recompute** `debtToBurn` downward, **cap** the transfer, or **revert**?
        - Are conversions consistently floored/ceiled so `seizedCollateralInYield >= transferredYield` holds?
        - Do we update user state **before** token transfer and then derive the transfer from the **actual** seized amount?
        - Are there property tests asserting:
        `Δ(poolBalance) + ΣΔ(user.collateralBalance) == 0` for the liquidated user only?
    - **Fix pattern:**
        - **Cap by seized collateral** and burn proportional debt:
            
            ```solidity
            uint256 seizeYield = adjustedLiquidationAmount;
            uint256 maxPayable = account.collateralBalance >= seizeYield ? seizeYield : account.collateralBalance;
            
            account.collateralBalance -= maxPayable;
            
            // Burn only the debt that corresponds to what we actually seized
            uint256 actualDebtToBurn = convertYieldTokensToDebt(maxPayable);
            _subDebt(accountId, actualDebtToBurn);
            
            TokenUtils.safeTransfer(yieldToken, transmuter, maxPayable);
            
            ```
            
        - Alternatively, **seize first** in yield units, then **derive** burn amounts from the seized quantity (single source of truth).
        - Add invariant tests: after liquidation, the pool should never be short-changed by paying more yield than removed from the borrower; other users’ balances must be unaffected.
- [ ]  **Ceil-rounded graph queries overflow earmark budget and brick state transitions**
    - **Symptom:** A time-weighted staking/redemption graph (`queryGraph`) **rounds up** the returned amount (e.g., `+1` after division by `BLOCK_SCALING_FACTOR`). Over time the sum of earmarked amounts exceeds `totalDebt - cumulativeEarmarked`, tripping `require(increment <= total)` inside `PositionDecay.WeightIncrement` and causing all calls that `_earmark()` to **revert** (repay/mint/burn/redeem/withdraw), effectively locking the system.
    - **Where to look (Alchemist/Transmuter-style systems):**
        - Transmuter graph accessors: `Transmuter::queryGraph(startBlock, endBlock)`; any Fenwick/segment-tree accumulator (`StakingGraph`, `Fenwick`, `queryStake`) that returns scaled sums.
        - Scaling constants & conversions: `BLOCK_SCALING_FACTOR`, shifts (`<< 128`), and any `+1` / ceil-division used to “fix” rounding.
        - Earmarking path in the core vault: `_earmark()` and its callers (often runs before **every** state-changing op). Look for:
            - `amount = ITransmuter(transmuter).queryGraph(...)`
            - `_earmarkWeight += PositionDecay.WeightIncrement(amount, totalDebt - cumulativeEarmarked);`
            - `cumulativeEarmarked += amount;`
        - The weight math library: `PositionDecay.WeightIncrement(increment, total)` (notably `require(increment <= total)` and the log/ratio math).
    - **Red flags:**
        - `return (queried / BLOCK_SCALING_FACTOR).toUint256() + 1;`
        or any unconditional **ceil**/“+1 for rounding error” on graph queries.
        - Multiple `_earmark()` calls over time while `queryGraph` is **already** floor-rounded internally in its storage layer—then re-ceiling at API boundary.
        - Off-by-one accumulation patterns: first sample is tiny but rounded up, last sample then fails `increment <= total` when `total` has been depleted by prior over-rounding.
        - Invariants that depend on **exact** conservation between “budget” (`totalDebt - cumulativeEarmarked`) and “spend” (`amount`) without clamping.
    - **Questions to ask next time:**
        - Is `queryGraph` **flooring** or **ceiling** the scaled sum? Why? Could a per-call upward bias accumulate to exceed the total earmark budget?
        - Do we **clamp** `amount` to `min(amount, totalDebt - cumulativeEarmarked)` before passing to `WeightIncrement`?
        - Are there property tests asserting `Σ(earmarked_i) ≤ totalDebt_initial` across a full transmutation window?
        - Does rounding happen **once** at final settlement, or at each incremental query (risking drift)?
        - Can `_earmark()` be externally triggered in tight loops or multiple times per block, amplifying rounding drift?
    - **Fix pattern:**
        - **Remove the upward bias**: replace `return (queried / BLOCK_SCALING_FACTOR) + 1;` with **flooring**:
            
            ```solidity
            return (queried / BLOCK_SCALING_FACTOR).toUint256();
            // or: (queried + BLOCK_SCALING_FACTOR - 1) / BLOCK_SCALING_FACTOR  // only if you truly need ceil, but then…
            
            ```
            
- [ ]  **Overwritten redemption snapshots desync per-account state after multiple redemptions**
    - **Symptom:** If `redeem()` is executed more than once between a user’s `_sync()`, the contract overwrites the prior redemption snapshot (e.g., `_redemptions[block.number] = RedemptionInfo(...)` + `lastRedemptionBlock = block.number`). During `_sync()`, the account uses only the **latest** snapshot to rebuild state, so `account.debt` and `account.earmarked` drift from globals (sum of accounts < `totalDebt`, sum of earmarked < `cumulativeEarmarked`).
    - **Where to look (Alchemix-style earmark/weight systems):**
        - Core vault: `_sync()`, `_earmark()`, `redeem()`, and data like `lastRedemptionBlock`, `_earmarkWeight`, `_redemptionWeight`, `cumulativeEarmarked`, `totalDebt`.
        - Snapshot storage: mappings keyed by `block.number` such as `_redemptions[blockNumber] = RedemptionInfo(...)`.
        - Weight math: calls like
        `debtToEarmark = PositionDecay.ScaleByWeightDelta(..., previousRedemption.earmarkWeight - account.lastAccruedEarmarkWeight)`.
        - Call graph: ensure **redeem** doesn’t assume only one redemption occurs between a given account’s `_sync()`; check whether `_sync()` is called for accounts during redemption flows (often it isn’t).
    - **Red flags:**
        - Writing a **single** snapshot per block (`_redemptions[block.number] = ...`) with no append/nonces; later calls in the same block (or before a user pokes) overwrite prior data.
        - `_sync()` reads only `_redemptions[lastRedemptionBlock]` instead of iterating/accumulating over **all** redemptions since the account’s last accrual.
        - State invariants not asserted: e.g., `Σ account.debt == totalDebt` and `Σ account.earmarked == cumulativeEarmarked` after arbitrary sequences of `redeem()` calls.
        - Comments/assumptions like “only one redemption per block” or tests that never simulate multiple redemptions before user `_sync()`.
    - **Questions to ask next time:**
        - Can `redeem()` be called multiple times in the **same** block (or many times) before users interact/poke? What guarantees prevent that?
        - Does the account update path reconstruct state from **cumulative** weights (monotonic counters) or from the **last snapshot only**?
        - If redemptions are frequent/partial, can a non-interacting account fall behind and later compute from an **overwritten** snapshot?
        - Are there unit/property tests that fuzz sequences of: `redeem()`×N → `_sync()` across blocks and assert global vs per-account conservation?
    - **Fix pattern:**
        - Replace per-block overwrite with **cumulative snapshots**:
            - Maintain monotonic `cumRedemptionWeight`, `cumEarmarkWeight` and store only these scalars during `redeem()`.
            - In `_sync()`, compute deltas using `cum* - account.last*` (no need to read a per-block map).
        - If historical data must be stored, **append** with an ever-increasing nonce (or ring buffer) and iterate/apply all events since the account’s last index; avoid `mapping[block.number]` overwrites.
        - As a guard, clamp/recompute using global invariants post-sync and add tests asserting conservation after arbitrary redemption sequences.
        - Optionally, call `_sync()` on affected accounts during `redeem()` (or require `poke`)—but prefer the cumulative-counter design so correctness does not depend on side effects.
- [ ]  **Earmarked-first repayment skews redemptions and can strand CDPs**
    - **Symptom:** `repay()` (or `_forceRepay()`) decrements `account.earmarked` **before** general debt, but later `redeem()` / `_sync()` still removes collateral via global weights. Result: collateral is reduced with **no matching debt reduction** for the repaid user, pushing `rawLocked` down (even below zero) and making the CDP uncloseable/unliquidatable.
    - **Where to look (Alchemix-style earmark/redemption models):**
        - Vault core: `repay()`, `_forceRepay()`, `_subDebt()`, `_sync()`, `redeem()`.
        - State buckets: `account.debt`, `account.earmarked`, `account.rawLocked`, `account.collateralBalance`.
        - Global weights/counters: `_earmarkWeight`, `_redemptionWeight`, `_collateralWeight`, `cumulativeEarmarked`, `totalDebt`, `totalSyntheticsIssued`.
        - Math helpers: `PositionDecay.ScaleByWeightDelta/WeightIncrement`, and conversions `convertYieldTokensToDebt/convertDebtTokensToYield`.
        - Update order: does `repay()` update globals/weights (e.g., reduce user’s share of `cumulativeEarmarked`) or only local `account.earmarked`?
    - **Red flags:**
        - Lines like `account.earmarked -= min(credit, account.earmarked);` in `repay()` with **no** corresponding adjustment to `cumulativeEarmarked` / redemption weights.
        - `_sync()` computing `collateralToRemove = ScaleByWeightDelta(account.rawLocked, _collateralWeight - account.lastCollateralWeight)` that **ignores** that the user already repaid the earmarked share.
        - No clamp against underflow on `rawLocked -= collateralToRemove` or on liquidation paths that assume `rawLocked` ≥ required collateral for remaining debt.
        - Lack of invariants/tests asserting:
        `collateral removed by redemption ≈ proportional to (remaining debt share)` and “post-repay + redeem, position remains closable/liquidatable”.
    - **Questions to ask next time:**
        - When a user repays **earmarked** debt, how is their pending redemption exposure reduced globally (weights and `cumulativeEarmarked`), so a later `redeem()` doesn’t still pull their collateral?
        - Do redemptions remove collateral proportional to **current** (post-repay) earmarked/debt mix per account, or only to historical global weights?
        - Can a user sequence: earmark → partial repay (earmarked only) → redemption → end up with collateral cut but unchanged debt? Any guardrails/timelocks?
        - Do liquidation/close flows ever revert on arithmetic underflow after such sequences? Are there property tests for `repay → redeem → liquidate/close`?
    - **Fix pattern:**
        - **Pro-rata repay:** Split `credit` across `(earmarked, non-earmarked)` by the account’s current composition, or
        - **Unwind exposure on repay:** When reducing `account.earmarked`, **also** reduce the user’s share from `cumulativeEarmarked` / redemption weights (or record a per-account offset) so future `redeem()` won’t pull their collateral.
        - **Apply redemption to remaining debt only:** In `_sync()`, compute redemption collateral removal using the account’s **effective earmarked** after repay (e.g., `min(account.earmarked, account.debt)`), and **clamp** collateral removal so `rawLocked` cannot underflow.
        - Prefer **cumulative counter** designs (monotonic `cumEarmarkWeight` per account) so adjustments from repay naturally change future redemption deltas.
- [ ]  **Double conversion rounding leaves `earmarked > debt` → `_sync()` underflow**
    - **Symptom:** Liquidation calls `_forceRepay(accountId, convertDebtTokensToYield(account.earmarked))`; inside `_forceRepay` the amount is converted **back** with `convertYieldTokensToDebt(amount)`. Truncation/rounding in the two opposite conversions produces `yieldToDebt < original earmarked` by 1 wei. After liquidation burns the remaining debt to 0, the account can end as `debt == 0 && earmarked > 0`. Next `_sync()` computes `account.debt - account.earmarked` and underflows, bricking actions that call `_sync()`.
    - **Where to look (Alchemix-style earmark/repay/liquidate flows):**
        - Core paths: `_liquidate()`, `_forceRepay()`, `repay()`, `_sync()`, `_subDebt()`.
        - Unit conversions: `convertDebtTokensToYield`, `convertYieldTokensToDebt` (direction and rounding).
        - Call sites that do **convert(X(convert(Y(...))))** or convert the same quantity twice with opposite functions.
        - State coupling: `account.earmarked`, `account.debt`, `account.rawLocked`, and how `_sync()` uses `account.debt - account.earmarked`.
        - Math helpers using deltas/weights: `PositionDecay.ScaleByWeightDelta`, `WeightIncrement` (any assumption that inputs are non-negative).
    - **Red flags:**
        - `_forceRepay(id, convertDebtTokensToYield(account.earmarked))` followed by `yieldToDebt = convertYieldTokensToDebt(amount)` and then `account.earmarked -= min(credit, account.earmarked)` — credit measured in **debt units** but sourced from a **round-trip** conversion.
        - Conversions that **truncate down** in both directions; lack of consistent “round-toward-safety” policy (e.g., round up when converting from yield→debt during repay).
        - No invariant checks like `if (account.debt == 0) account.earmarked = 0;`.
        - `_sync()` uses unsigned subtraction: `account.debt - account.earmarked` with no clamp/saturation.
        - Decimals mismatch between debt/yield tokens and use of integer division without guards.
    - **Questions to ask next time:**
        - Are `convertDebtTokensToYield` and `convertYieldTokensToDebt` true inverses across all decimals? What are the rounding rules? (down, up, banker’s?)
        - Do we ever pass values through **both** conversions before applying them to state? If so, why not operate in a **single canonical unit**?
        - Can `debt == 0 && earmarked > 0` happen by construction (repay, forced-repay, liquidation)? What invariant enforces `earmarked ≤ debt` post-action?
        - Does `_sync()` protect against negative intermediates (use of `account.debt - account.earmarked`)?
        - Are there tests for liquidation → forced-repay → `_sync()` sequences across “edge” decimal setups (e.g., 6↔18)?
    - **Fix pattern:**
        - **Canonicalize units:** Handle `_forceRepay` purely in **debt units**; pass `earmarked` directly, compute `credit` in debt units, convert **once** to yield at transfer time.
        - **Rounding discipline:** When repaying debt, choose rounding that does **not** leave a positive `earmarked` residue (e.g., round **up** yield→debt or manually bump 1 wei of debt where safe).
        - **Invariant clamps:** After any repay/liquidation path: `if (account.debt == 0) account.earmarked = 0;` and always `account.earmarked = min(account.earmarked, account.debt)`.
        - **Saturating math in `_sync()`:** Use a safe helper `sub0(a,b)` for `max(a-b,0)` before passing to `ScaleByWeightDelta`.
- [ ]  **Saturated redemption weight (>LOG2NEGFRAC\_1) breaks earmark/debt math**
    - **Symptom:** When a redemption sets `ratio==0` (`increment==total`), `WeightIncrement` returns `LOG2NEGFRAC_1+1`. Later, `ScaleByWeightDelta` treats any `weightDelta > LOG2NEGFRAC_1` as full application (`Exp2NegFrac` returns 0), so additional redemptions made before a user sync are effectively ignored. Final `debt/earmarked` depends on whether `poke()` (sync) was called between redemptions—i.e., timing-dependent accounting.
    - **Where to look (Alchemix-style decay/weight systems):**
        - `libraries/PositionDecay.sol`: `LOG2NEGFRAC_1`, `WeightIncrement`, `ScaleByWeightDelta`, `Exp2NegFrac`.
        - State that accrues weights: `_redemptionWeight`, `_earmarkWeight`, `lastAccruedRedemptionWeight`, `lastAccruedEarmarkWeight`.
        - Callers updating or consuming weights: `redeem()`, `_sync()`, `poke()`, and the per-block snapshot `_redemptions[block.number]`.
        - Boundary math: sites returning `LOG2NEGFRAC_1+1` and comparisons like `if (x > LOG2NEGFRAC_1)`.
    - **Red flags:**
        - `if (ratio == 0) return LOG2NEGFRAC_1 + 1;` in `WeightIncrement`.
        - Boundary mismatch: `Exp2NegFrac` uses `if (x > LOG2NEGFRAC_1) return 0;` (strict `>`), while the producer can emit `LOG2NEGFRAC_1+1`, creating an off-by-one saturation regime.
        - No clamping on `_redemptionWeight` (can exceed the maximum meaningful delta).
        - Accounting outcomes differ between sequences `(redeem; redeem; sync)` vs `(redeem; sync; redeem; sync)`.
        - Tests missing for “poke vs no-poke” equivalence and multiple redemptions between `_sync()`.
    - **Questions to ask:**
        - Is `_redemptionWeight` allowed to exceed `LOG2NEGFRAC_1`? What are the intended semantics at and beyond the boundary?
        - Should `ScaleByWeightDelta` treat `x >= LOG2NEGFRAC_1` as full application (0) and are all producers consistent with that?
        - Do two sequences—(A then B, then sync) vs (A, sync, then B)—produce identical `debt/earmarked` for the same totals?
        - Are per-block snapshots (`_redemptions[block.number]`) robust when multiple redeems occur before any user `_sync()`?
    - **Fix pattern:**
        - **Unify boundary semantics:** Either (a) cap producer: `WeightIncrement` returns **at most** `LOG2NEGFRAC_1` (remove `+1`), or (b) cap consumer: make `Exp2NegFrac` use `if (x >= LOG2NEGFRAC_1) return 0;`. Do not let “> vs >=” create off-by-one behavior.
        - **Clamp accumulated weights:** `_redemptionWeight = min(_redemptionWeight, LOG2NEGFRAC_1)` (and same for earmark) to avoid meaningless “extra” weight.
        - **Determinism tests:** Add property tests asserting that final `debt/earmarked` are invariant to insertion/removal of intermediate `poke()` calls and to batching multiple `redeem()` ops into one block.
        - **Audit snapshots:** If multiple `redeem()`s can share a block, ensure `_redemptions[block.number]` aggregates correctly (or use unique indices) so `_sync()` composes all deltas instead of overwriting.
- [ ]  **Naive `balanceOf` TVL lets deposits/donations skew BDR & liquidation math**
    - **Symptom:** Protocol uses `IERC20(yieldToken).balanceOf(address(this))` inside `_getTotalUnderlyingValue()` for “system health” (bad-debt ratio and global collateralization). Anyone can temporarily deposit or *donate* yield tokens (or hold large free collateral) to inflate TVL, which (a) reduces/avoids redemptions’ bad-debt haircut in `claimRedemption()` and (b) flips full → partial liquidation in `_liquidate()` / `calculateLiquidation(...)`.
    - **Where to look (Alchemist-style vaults/CDPs):**
        - TVL helpers: `_getTotalUnderlyingValue()`, `convertYieldTokensToUnderlying`, `normalizeUnderlyingTokensToDebt`.
        - Consumers: `Transmuter.claimRedemption()` bad-debt ratio calc, `_liquidate()` → `calculateLiquidation(...)` (global collateralization branch).
        - Balance sources: `deposit`, `withdraw`, any path that increases the contract’s token balance without updating *locked* accounting (e.g., direct `transfer`/donation).
        - State that tracks *locked* vs *free*: `account.rawLocked`, `account.freeCollateral`, `_totalLocked`, reserves.
    - **Red flags:**
        - Health metrics or haircut math use `IERC20(token).balanceOf(this)` directly.
        - No separation of **locked collateral** vs **free collateral** vs **unaccounted donations** in TVL.
        - `claimRedemption()`/liquidation decisions read TVL that can change mid-tx with an external deposit.
        - No snapshotting (e.g., prior-block TVL) or anti-sandwich window around `claimRedemption()`.
        - Comments like “treat donations as TVL” or lack of handling for `onERC4626Deposit`/adapter inflows.
    - **Questions to ask:**
        - Which units should BDR/global-collateralization reflect: only *locked* value (`sum(rawLocked)`), or also free collateral and idle reserves?
        - Can a user deposit/withdraw in the same block as `claimRedemption()` or a liquidation check? Are those ops sequenced/snapshotted?
        - How are direct token transfers handled (no hook)? Are “donations” excluded from TVL used for haircuts and liquidation mode?
        - Do liquidations rely on a *global* TVL that can be griefed, or per-account ratios only?
    - **Fix pattern:**
        - **Use accounted TVL, not wallet balance.** Introduce `_getLockedUnderlyingValue()` computed from internal accounting (e.g., `sum(account.rawLocked)` or `_totalLocked`), and use that for:
            - Bad-debt ratio in `claimRedemption()`: `BDR = totalSyntheticsIssued / normalizeUnderlyingTokensToDebt(_getLockedUnderlyingValue())`.
            - Global collateralization in `_liquidate()` → `calculateLiquidation(...)`.
        - **Exclude free & donated funds** from these calculations (or classify them explicitly as reserves with separate rules).
        - **Snapshot before action:** For `claimRedemption()`, read a cached/snapshotted locked-TVL from the previous block (or start-of-tx) so deposit-then-claim cannot change the denominator.
- [ ]  **ID-based deposits into existing positions lack ownership/approval (reorg ID drift)**
    - **Symptom:** A follow-up deposit uses a stale `positionId/recipientId` and credits another account after a reorg or race. Collateral lands in the wrong position; unintended owner can withdraw.
    - **Where to look (Solidity DeFi / NFT-positioned CDPs & LPs):**
        - Functions that accept an existing position ID: `deposit`, `addCollateral`, `increaseLiquidity`, `mintOrIncrease`, `join`, etc.
        - Code paths like:
            - `if (id == 0) { mint(...) } else { _checkForValidAccountId(id); /* no owner check */ }`
            - Any use of `recipient` alongside a non-zero `id`.
        - Position registry contracts (ERC-721 position NFTs): calls to `ownerOf`, `getApproved`, `isApprovedForAll`.
        - Frontend/backend that derive `id` from “expected sequence” instead of reading emitted `tokenId` from events.
    - **Red flags:**
        - Only existence checks (e.g., `_checkForValidAccountId(id)`) without `owner/approval` validation.
        - Accepting a `recipient` for `id != 0` and writing to that account instead of the actual owner.
        - Sequential, auto-incremented `tokenId`/position IDs with no authority gate on “increase” operations.
        - Funds (`safeTransferFrom`) moved **before** access control on the provided `id`.
        - No tests simulating swapped ordering of two initial mints.
    - **Questions to ask next time:**
        - “When `id != 0`, where do we assert `msg.sender` is `ownerOf(id)` or approved (`getApproved`/`isApprovedForAll`)? Is it before moving funds?”
        - “If two first-time mints swap order (reorg), can a follow-up deposit target the wrong `id` and succeed?”
        - “Do we ignore the `recipient` parameter when increasing an existing position?”
        - “How do clients obtain `tokenId`—from emitted events or guessed sequencing? Do we document finality requirements?”
        - “Are there unit tests that reorder two mints and ensure the follow-up deposit reverts without ownership?”
    - **Fix pattern:**
        - For `id != 0`, **require** `msg.sender` is owner or approved (`ownerOf(id) == msg.sender || getApproved(id) == msg.sender || isApprovedForAll(owner,msg.sender)`); **ignore** external `recipient` and set it to `owner`.
        - Apply an `onlyOwnerOrApproved(id)` modifier to *all* position-mutating entrypoints (deposit/increase/withdraw/repay/rollover).
        - Perform access checks **before** token transfers and state writes.
- [ ]  **Fee plumbing mismatch between “calc” and “execute” paths (base vs. surplus fee inversion)**
    - **Symptom:** A pure/pricing helper (e.g., `calculateLiquidation`) returns a fee computed on *surplus* collateral, but the caller labels it as “base fee” (or vice-versa) and pays it from the wrong source and/or in the wrong token (e.g., pays surplus fee from fee vault and pays base/guaranteed fee from principal collateral). Liquidators may still net the right total, but accounting, vault balances, and token types are wrong.
    - **Where to look (Solidity lending/CDP/liquidations):**
        - Pricing helpers: `calculateLiquidation`, `getSeizeAmounts`, `computeFees`, `previewLiquidation`, `previewRedeem`.
        - Execution sites: `_liquidate`, `seize`, `closePosition`, `redeem`, `repayAndSeize`.
        - Fee sources & sinks:
            - Protocol fee vault withdrawals (underlying).
            - Transfers from market contract balances (yield/pToken/collateral).
            - Conversions: `convertDebtTokensToYield/Underlying` around fees.
        - Event fields & names: `baseFee`, `liquidatorFee`, `surplusFee`, `protocolCut` → do they match actual transfers?
    - **Red flags:**
        - Helper returns a *single* `fee` without clarifying whether it’s on `surplus` or on `debtToBurn`.
        - Caller renames return values (`fee` → `baseFee`) and independently recomputes another fee (e.g., `feeBonus = debtToBurn * feeBps / BPS`) — two fees with ambiguous semantics.
        - Comments that say “fee taken from surplus” but code uses `debtToBurn` (or the reverse).
        - Paying one fee from fee vault (underlying) and the other from contract balance (yield) with no spec stating which should come from where.
        - Unit tests assert *amounts* paid but not *sources/tokens* of those payments.
    - **Questions to ask next time:**
        - “Is the returned `fee` applied on `surplus = max(collateral - debt, 0)` or on `debtToBurn`? Where is that documented?”
        - “Which fee is ‘guaranteed’ (base liquidator incentive) vs. ‘surplus/profit share’? Which token and source should fund each?”
        - “If surplus = 0, which fee(s) should still be paid? From which balance?”
        - “Do events and variable names (`baseFee`, `surplusFee`) match the actual arithmetic and transfer sources?”
        - “Do conversions (`convertDebtTokensToYield/Underlying`) around fees change token type inadvertently?”
    - **Fix pattern:**
        - **Make semantics explicit:** Return a struct from the helper:
        `struct LiquidationQuote { uint256 seizeCollateral; uint256 debtToBurn; uint256 baseFeeOnDebt; uint256 surplusFeeOnSurplus; }`
- [ ]  **Untracked netting during redemption inflates internal “issued” counters**
    - **Symptom:** Redemptions “net” against yield held by the transmuter (or a vault), but the system burns the full `amountTransmuted` of synths while only decrementing `totalSyntheticsIssued` by the *net* `amountToRedeem`. Over time, `totalSyntheticsIssued` > actual synth totalSupply, skewing metrics like `badDebtRatio` and reducing claimant payouts.
    - **Where to look (Solidity lending/redeemer modules):**
        - Redeemer flow (e.g., `Transmuter.claimRedemption` / `claim`):
            - Pattern:
                
                ```solidity
                uint256 ytBal = IERC20(yieldToken).balanceOf(this);
                uint256 debtValue = alchemist.convertYieldTokensToDebt(ytBal);
                uint256 amountToRedeem = amountTransmuted > debtValue ? amountTransmuted - debtValue : 0;
                if (amountToRedeem > 0) alchemist.redeem(amountToRedeem); // updates totals
                TokenUtils.safeBurn(syntheticToken, amountTransmuted);   // burns FULL amount
                
                ```
                
        - Alchemist/accounting side (`redeem`, `repay`, internal counters): look for updates to `totalSyntheticsIssued`, `totalDebt` vs. ERC20 `totalSupply()` of the synth.
        - Inbound funding paths into the redeemer that pre-fill yield: liquidations and repayments that `transfer` yield to the transmuter (or similar module).
        - Any place computing `badDebtRatio` or solvency using `totalSyntheticsIssued`/TVL.
    - **Red flags:**
        - Burn amount ≠ internal counter decrement (e.g., burn `amountTransmuted` but decrement `totalSyntheticsIssued` by `amountToRedeem`).
        - “Netting” logic that reduces the *Alchemist-side* `redeem` call because the redeemer already holds yield, but still burns the full synthetic amount.
        - `totalSyntheticsIssued` used as a source of truth for debt/bad debt while it can diverge from synth `totalSupply()`.
        - Comments like “reduce redeem by transmuter balance” followed by an unconditional full burn of synths.
    - **Questions to ask next time:**
        - “What invariant ties `totalSyntheticsIssued` to `syntheticToken.totalSupply()` after any path (redeem, repay, liquidation)?”
        - “When we net with pre-held yield, which component decrements ‘issued’? Is there a second call to account for the netted portion?”
        - “What is the single source of truth for circulating synthetics? How do we prevent drift if redemptions can be partially satisfied from module balances?”
        - “If `badDebtRatio` depends on `totalSyntheticsIssued`, what protects it from drift caused by netting and rounding?”
    - **Fix pattern:**
        - **Single-point accounting:** Make the *same* module/function that burns synths also update `totalSyntheticsIssued` by the **same burn amount**. Avoid split responsibility across contracts.
        - **Two-amount quote:** Have the redeemer call an alchemist hook like `redeemWithPrepaidYield({burnAmount, prepaidDebtValue})` so the alchemist:
            - Decrements `totalSyntheticsIssued` by `burnAmount`.
            - Decrements `totalDebt` by `burnAmount`.
            - Internally nets `prepaidDebtValue` against its books (no external transfer).
        - **Or align burns to netting:** If you insist on netting, then **burn only `amountToRedeem`** and separately “apply”/record the `debtValue` portion via an explicit accounting call (no-op transfer) that also reduces `totalSyntheticsIssued`.
- [ ]  **Integer truncation in linear-vest/redemption math overpays early claimers**
    - **Symptom:** `amountTransmuted = amount - floor(amount * blocksLeft / transmutationTime)` rounds **up** the matured amount. Early claims get slightly extra, cumulatively exceeding earmarks; later `redeem()` can revert on `require(increment <= total)` (e.g., `PositionDecay.WeightIncrement`) or leave the system under-earmarked.
    - **Where to look (Solidity vesting/redemption flows):**
        - Redemption/vesting claimers: `Transmuter.claimRedemption`, `claim/withdrawUnlocked`, bond/stake unlockers using `blocksLeft` / `timeLeft`.
        - Any proportional unlock math `x * remaining / total` done with **integer division**.
        - The accounting join with earmarks/weights: functions that compute `amountToRedeem` then call core `redeem()` or update `cumulativeEarmarked`/`_earmarkWeight`.
        - Fixed-point helpers/decay libs (e.g., `PositionDecay.WeightIncrement/ScaleByWeightDelta`) that enforce `increment <= total`.
    - **Red flags:**
        - Pattern:
            
            ```solidity
            uint256 amountNot = pos.amount * blocksLeft / transmutationTime; // truncates down
            uint256 amountTrans = pos.amount - amountNot;                   // rounds up
            
            ```
            
        - No cap like `amountTrans = min(amountTrans, earmarkedAvailable)`.
        - Repeated small claims per position (amplifies truncation bias).
        - `redeem(amountTrans)` immediately after computing without reconciling against `totalDebt - cumulativeEarmarked`.
        - Mixing block-count schedules with integer math instead of fixed-point (no 1e18 scaling; no remainder carry).
    - **Questions to ask next time:**
        - What’s the explicit **rounding policy** (favor protocol vs user) for partial maturation?
        - Do claims **cap** to available earmarked/global headroom (`totalDebt - cumulativeEarmarked`)?
        - Is fractional dust **carried forward** per position (e.g., stored remainder) to neutralize bias?
        - Can a user **front-run** with many micro-claims to accumulate rounding gains?
    - **Fix pattern:**
        - Compute **unmatured** with ceil and mature with floor:
            
            ```solidity
            // ceilDiv to avoid rounding matured up
            uint256 amountNot = blocksLeft == 0 ? 0 : ceilDiv(pos.amount * blocksLeft, transmutationTime);
            uint256 amountTrans = pos.amount - amountNot; // rounds matured DOWN
            
            ```
            
        - Or switch to **fixed-point** rate: `rate = pos.amount * 1e18 / transmutationTime; matured += (elapsed * rate)/1e18;` store and carry per-position **remainders**.
        - Always enforce:
        `amountTrans = Math.min(amountTrans, earmarkedAvailable, totalDebt - cumulativeEarmarked);`
- [ ]  **Liquidation fees mis-sourced (treasury/fee-vault pays most of the bounty)**
    - **Symptom:** Liquidators are paid primarily from the protocol’s fee-vault/treasury instead of from the seized collateral. “Base” vs “bonus/excess” fee labels don’t match sources; most fees come from vault even when collateral is near the threshold, shifting costs from debtors to protocol.
    - **Where to look (Solidity lending/CDP/liquidation flows):**
        - Liquidation entrypoints: `liquidate`, `_liquidate`, `batchLiquidate`, `seize`, `close`.
        - Fee math helpers: `calculateLiquidation`, `getSeizeAmounts`, `liquidationIncentiveBps`, `liquidatorFee`, `surplus = max(collateral - debt, 0)`.
        - Payout paths:
            - Transfers from **user collateral** (e.g., `account.collateralBalance -= adjustedLiquidationAmount` and `safeTransfer(yieldToken, liquidator, feeInYield)`).
            - Transfers from **treasury/fee vault/insurance fund** (e.g., `feeVault.withdraw(liquidator, feeInUnderlying)`).
        - Token domain boundaries: conversions between debt/yield/underlying (`convertDebtTokensToYield/Underlying`), and any “1:1” assumptions.
        - Branches for bad-debt / low global collateralization (full vs partial liquidation paths).
    - **Red flags:**
        - Fee components named “base”/“excess” but sourced oppositely (e.g., “base fee” paid from vault, “bonus” paid from collateral).
        - Fee computed as a % of **debt** and paid by vault; fee computed as a % of **surplus** paid by user → vault almost always covers most of the bounty.
        - No cap like `feeFromCollateral = min(availableCollateralAfterDebt, computedFee)`.
        - Vault payment guarded only by `if (vaultBalance > 0)` rather than “only top-up when collateral can’t cover fee”.
        - Mixed denominations (fee in underlying from vault; fee in yield from borrower) without consistent conversion or ordering.
        - Comments say “excess from treasury” but code always (or typically) pulls from treasury.
        - Lack of unit tests around near-threshold collateralization (e.g., 1.02x–1.10x) showing fee source ordering.
    - **Questions to ask next time:**
        - Who is economically intended to fund the liquidation incentive **first**—the debtor’s seized collateral or the protocol? Under what conditions does the treasury top-up?
        - Are fees **capped** by available collateral after covering debt? Is there an ordering: pay debt → take fee from remaining collateral → only then top-up from vault?
        - Are fee amounts consistently denominated? If the vault pays in underlying but collateral/fees are in yield, is conversion loss or mismatch considered?
        - In bad-debt scenarios, is there a formal policy for treasury spend (limits, rate, budget), or can liquidators drain the vault?
        - Do tests demonstrate that with typical liq fees (1–3%) and positions at \~1.05× collateralization, the **majority** of fee still comes from the borrower, not the vault?
    - **Fix pattern:**
        - **Source ordering & caps:**
            1. Compute `debtToBurn`.
            2. Compute `fee = debtToBurn * feeBps / BPS` (or your chosen basis).
            3. Determine `available = max(0, collateralInYield - convertDebtToYield(debtToBurn))`.
            4. `feeFromCollateral = min(feeInYield, available)`.
            5. `feeFromVault = feeInYield - feeFromCollateral` (converted if vault pays in another token).
            6. Transfer debt burn to target, send `feeFromCollateral` from seized collateral, **only** then top-up `feeFromVault` from treasury.
        - **Rename for clarity:** Use `feeFromCollateral` / `feeFromVault` instead of “base/excess”; ensure comments match sources.
        - **Denomination consistency:** Centralize conversion helpers; avoid 1:1 assumptions; test both directions (yield↔underlying↔debt).
- [ ]  **Forced-repay before liquidation subtracts collateral without an availability clamp**
    - **Symptom:** Liquidation reverts/DoS when “earmarked/queued” debt is auto-repaid from the account but `collateralBalance` can’t cover it; subtraction underflows (or reverts in ≥0.8.x), leaving the position unliquidatable.
    - **Where to look (Solidity lending/CDP engines):**
        - Liquidation paths: `liquidate/_liquidate`, `close`, `seize`.
        - Pre-liquidation “force repay” helpers that clear earmarked/redemption debt: `_forceRepay`, `_repayEarmarked`, “settle before seize”.
        - Balance updates: `account.collateralBalance -= X` where `X` is derived from `convertDebtToCollateral/Yield`.
        - Any earmark/redemption queue logic and unit conversions between debt/yield/underlying.
    - **Red flags:**
        - No `min()` clamp: deducting `creditToYield` (or equivalent) from collateral without `min(creditToYield, collateralBalance)`.
        - Assuming full repayment is always possible before liquidation proceeds.
        - Mixing units (debt vs collateral token) without checking available balance; off-by-one from rounding.
        - Lack of fallback path (partial repay) or ordering that lets liquidation continue if full forced-repay isn’t possible.
    - **Questions to ask:**
        - What if collateral < amount needed for forced-repay? Do we clamp, partially repay, or block liquidation?
        - Are denominations consistent and rounded conservatively before subtraction?
        - Does earmarked debt get reduced proportionally to what was actually repaid?
    - **Fix pattern:** Clamp deductions (`repayFromCollateral = min(requiredInCollateral, collateralBalance)`), reduce earmarked/debt by the repaid portion only, then proceed with (partial/full) liquidation; validate unit conversions and add guards so liquidation cannot be permanently blocked by insufficient collateral.
- [ ]  **Front-run repay zeros pre-liquidation “force-repay” via unit conversion (DoS)**
    - **Symptom:** Liquidation reverts because a pre-step “force-repay(earmarked)” is called with `amount==0` after flooring conversions (debt→underlying/yield). Borrower front-runs with a tiny `repay()` to push `convert(debt)` below the conversion factor threshold, tripping `require(amount > 0)` or similar.
    - **Where to look (generic CDP/liquidation code):**
        - Liquidation paths that first settle/repay “earmarked/queued” debt: `_liquidate`, `_forceRepay`, `_settleEarmarked`.
        - Unit conversion helpers: `convertDebtToYield`, `normalizeDebtToUnderlying`, decimals math between debt/underlying/yield.
        - Earmark/decay accounting updated by `repay()` before liquidation.
    - **Red flags:**
        - `require(amount > 0)` (or non-zero asserts) inside internal `forceRepay` when `amount` comes from a conversion that floors via division.
        - Large decimals deltas (e.g., ÷1e12) + integer division → easy zeroing.
        - Borrower-callable `repay()` that reduces `earmarked` *before* liquidation, with no snapshot or slippage/threshold checks.
        - Relying on converted amounts for control flow (revert on zero) instead of continuing with liquidation.
    - **Questions to ask:**
        - What if `convertDebtToYield(earmarked)` floors to zero—do we revert or no-op?
        - Can a borrower reduce `earmarked` right before liquidation (same block/front-run)?
        - Are conversions consistently done in a single canonical unit to avoid double-rounding?
        - Is there a minimum effective unit or clamp to prevent zeroing by dust repays?
    - **Fix pattern:** Make liquidation resilient to zero conversions: compute in a single unit (debt), clamp with `min`/ceil where appropriate, treat zero as a no-op (early-return) instead of revert, or snapshot `earmarked` at liquidation start and block pre-emptive reductions; add dust thresholds that round *against* the manipulator.
- [ ]  **Earmark math ignores external buffers → `increment > total` invariant & global DoS**
    - **Symptom:** Accrual/earmark step computes a weight increment from “matured claims since last checkpoint,” but fails to subtract assets already sitting in a redeem/settlement buffer (e.g., transmuter/queue contract). Invariants like `require(increment <= total)` or log/exp math preconditions are hit, bricking `_earmark()` and all core actions that call it.
    - **Where to look (generic CDP/redemption systems):**
        - Earmark/accrual paths: `_earmark`, `_accrue`, `_sync`, `redeem`, `claim`, `liquidate` that call a decay/weight library (e.g., `WeightIncrement`, `Log2/Exp2NegFrac`).
        - State totals: `totalDebt`, `cumulativeEarmarked`, `lastEarmarkedBlock/Weight`, `lastRedemptionBlock/Weight`.
        - External buffers used to fulfill claims *before* pulling from vaults (transmuter, redemption queue, fee/buffer vault) and any logic that sends repayments/yield to that buffer **without** burning the synthetic.
        - Math libs and guards: checks like `require(increment <= total)`, handling of `ratio==0`, saturation branches.
    - **Red flags:**
        - Earmark uses “matured since last” from a graph/queue (`queryGraph`) and plugs it as `increment` against `total = totalDebt - cumulativeEarmarked` **without** subtracting `bufferBalance (in debt units)`.
        - Repay/forceRepay transfers underlying/yield to buffer but doesn’t reduce the synthetic supply or adjust earmark baselines.
        - Ability to deposit yield directly into the buffer contract (donations) not reflected in earmark math.
        - Long intervals between earmarks; monotonic matured amounts; rounding to 0/∞ branches in decay math.
    - **Questions to ask:**
        - When computing earmark increment, is it `maturedClaims - bufferAssetsInDebtUnits`, capped at outstanding (`min(increment, totalOutstanding)`)?
        - Do repays move value to the buffer without burning? If yes, where is that netted out?
        - Does every claim path ping the core to update its stored view of the buffer balance?
        - What happens if `increment > total`—revert or clamp? Does that halt all core functions?
    - **Fix pattern:** Track and subtract buffer deltas from matured claims (store `lastBufferDebt` and use net change); cap increment via `min(maturedNet, totalOutstanding)`; unify units before decay math; ensure repays either burn synths or adjust `totalSyntheticsIssued/cumulativeEarmarked`; add saturation guards in weight math to avoid protocol-wide reverts.
- [ ]  **Off-ledger burns/mints desync protocol debt counters**
    - **Symptom:** External redemption/settlement modules (queues, “transmuters”, bridges, fee vaults) burn or mint the synthetic/debt token without updating the protocol’s canonical debt counter (e.g., `totalDebt`, `totalSyntheticsIssued`). Ratios that use that counter (bad-debt ratios, collateralization, fee splits) become inflated/deflated, reducing user payouts and skewing fees.
    - **Where to look (any CDP/stable/savings system):**
        - External flows: `claimRedemption`, `settle/claim`, `bridgeIn/Out`, `repayToVault`, `forceRepay`, liquidation payout paths, fee collectors.
        - Token interactions: direct `ERC20(_debt).burn(...)`/`_mint(...)` in modules other than the core; conditional calls to core like `if (shortfall) core.redeem(x)`.
        - Core counters & ratios: variables named `totalDebt`, `totalIssued`, `globalDebt`, `syntheticsIssued` used in formulas like `badDebtRatio = totalIssued / totalUnderlyingValue`.
    - **Red flags:**
        - Conditional redemption: *only* call into core to pull assets when a buffer is insufficient; otherwise burn debt locally (no core callback) → core counter never decremented.
        - Multiple “sources of truth” (ERC20 totalSupply vs. core `totalIssued`) not reconciled every path.
        - Comments like “burn remaining synths” in non-core modules; or burns after an early-return branch skipped the core update.
        - Buffers that can be prefunded (donations, liquidations, repay) and satisfy claims without notifying the core.
    - **Questions to ask next time:**
        - When and where is the canonical debt counter decremented/incremented? Is there *any* path that burns/mints debt without touching it?
        - If a buffer can fully pay a claim, does the core still get notified to reduce `totalIssued`?
        - Is there an invariant enforced or tested: `core.totalIssued == debtToken.totalSupply()` (or the intended relation) after *every* state-changing flow?
        - What happens if redemptions are partially from buffer and partially from core—are both sides’ counters adjusted exactly once?
    - **Fix pattern:**
        - Single source of truth: make the core the only place that mutates the debt counter and expose `burnFromCore/ mintFromCore` APIs; require external modules to route through it *always*, regardless of buffer levels.
        - If burns must happen off-core, emit a mandatory callback/hook (or pull pattern) that the core processes atomically; otherwise clamp ratios off `debtToken.totalSupply()` directly.
- [ ]  **Module-switch sufficiency checks using spot FX/price cause stranded claims**
    - **Symptom:** After an admin swaps a settlement/redemption module (e.g., “transmuter”, bridge, AMM router, auction house), some users can no longer claim or withdraw, or surplus assets become stuck in the old module. The pre-switch “funds are enough” check used a *current* exchange rate (yield↔underlying, share↔asset, rebasing index, oracle price), which later moved—breaking redemptions because the old module can’t call the core anymore.
    - **Where to look (general Solidity DeFi):**
        - Admin setters: `setTransmuter/setBridge/setRouter/setAuction/setLendingModule`, address upgraders, registry swaps.
        - Guard checks around the swap: comparisons like `convert(X.balanceOf(oldModule)) >= oldModule.totalLocked()` or similar.
        - Core entrypoints gated by `onlyModule`/`onlyTransmuter` that old module needs to call for top-ups (`redeem()`, `pullLiquidity()`, `settle()`) after the swap.
        - Conversion layers: token adapters (`convertYieldToDebt`, `convertSharesToAssets`), oracle reads, rebasing/share price math.
    - **Red flags:**
        - Sufficiency check compares *values via conversion* (spot) instead of *like-for-like units* (same token).
        - No state/asset migration step; outstanding claims remain in old module but old module loses permission to pull from core post-swap.
        - Yield-bearing/rebasing tokens whose exchange rate can change after the swap (ERC-4626 `convertToAssets`, cToken exchangeRate, aToken index).
        - One-shot assertion at upgrade time with no escrow/grace period, no dual-allowlist, no snapshot of rate used to satisfy existing claims.
    - **Questions to ask next time:**
        - After switching modules, **who** is authorized to finalize/redress existing claims in the *old* module? For how long?
        - Are pending claims denominated in **units** of a specific token or in **value** via a rate? Is that rate **snapshotted**?
        - If the conversion rate shifts after the swap, is there a mechanism (escrow buffer, migration, dual-permission window) to cover the delta?
        - Does the pre-swap check compare **token amounts** (same decimals) rather than **converted values** that can drift?
    - **Fix pattern:**
        - Snapshot & settle: Freeze new entries, **snapshot the conversion rate**, and either (a) settle all pending claims using that snapshot before flipping `onlyModule`, or (b) store the snapshot in old module to price all remaining claims deterministically.
        - Dual authorization window: Temporarily allow **both** old and new modules to call required core functions until `oldModule.totalLocked()==0`.
        - Unit-based checks: Validate `oldYieldBalance >= oldModule.totalLockedInYieldUnits` (same token/decimals), avoiding FX exposure in the guard.
        - Explicit migration: Add `migrateState(old→new)` to move locked amounts/assets atomically; block `setModule` unless migration succeeds.
        - Safety rails: Timelock + pause redemptions during switch; require a buffer margin (e.g., 102–105%) rather than exact equality; add invariant tests for post-upgrade claimability.
- [ ]  **Missing `minOut`/deadline on redemptions with dynamic haircuts**
    - **Symptom:** Users claim/redeem/withdraw and receive materially fewer output tokens because a state-dependent ratio (e.g., “bad debt” factor, `pricePerShare`, oracle rate, TVL ratio) moved between request and execution or was manipulable; no way to bound slippage or front-run risk.
    - **Where to look (general DeFi):**
        - User exit paths: `claimRedemption/claim/redeem/withdraw/exit/unstake`.
        - Any scaling step like `out = nominalOut * X / Y` using: total debt / TVL, `convertToAssets/convertToShares`, oracle rates, or contract `balanceOf(this)` as TVL.
        - Functions that **burn** user IOUs then compute `out` and `transfer` without taking a user-provided `minAmountOut` and `deadline`.
    - **Red flags:**
        - No `minOut`/`deadline` params or checks (`require(out >= minOut)`).
        - Ratios derived from **spot** TVL using `IERC20.balanceOf(address(this))` (manipulable via donations/flash-moves).
        - Rounding-down on divisor (`/`) that systematically reduces `out` without user guardrails.
        - Comments that mention “bad debt”/“health factor” scaling but no slippage protection.
    - **Questions to ask:**
        - How can a user cap worst-case `out` when redemptions are haircut by solvency ratios?
        - Can TVL or exchange rates move within a single tx/MEV window? Are they TWAP’d or snapshotted?
        - Do we burn or lock user IOUs **before** verifying `out >= minOut`?
    - **Fix pattern:**
        - Add `minAmountOut` and `deadline` to all exit/claim functions; validate pre-transfer.
        - Snapshot the ratio at function entry or use TWAP; exclude extraneous balances from TVL (track locked vs free).
- [ ]  **Post-transfer TVL haircuts penalize claimants**
    - **Symptom:** A solvency/“bad debt” ratio (e.g., liabilities ÷ TVL) is computed **after** pulling assets out of the vault (or moving them into an intermediate contract), so the denominator shrinks first. The haircut uses this worse ratio, users get less than intended, and excess assets pile up idly in the intermediate contract.
    - **Where to look (general DeFi):**
        - Redemption/claim/withdraw flows that: (1) `redeem/pull` funds from a vault, then (2) compute `haircut = f(liabilities, TVL)`, then (3) scale the user payout.
        - Any ratio using spot TVL via `IERC20(token).balanceOf(vault)` or similar right **after** an internal transfer.
        - Bridges/transmuters/routers that first fetch full liquidity from source, then “downscale” the user amount.
    - **Red flags:**
        - Order of ops: `transfer/redeem()` → compute `ratio` → `amountOut = nominal * 1e18 / ratio`.
        - Full redemption of `nominal` followed by payout of `scaled` amount (“downscale after send”); growing idle balances in the middle contract.
        - Ratios derived from balances that exclude freshly moved funds or include only a subset of system assets.
    - **Questions to ask:**
        - Is the solvency/haircut ratio snapshotted **before any transfers** that affect TVL?
        - Do we redeem/pull **only the scaled amount**, or do we redeem full then downscale?
        - Is TVL definition consistent (includes assets held in intermediaries, pending queues, and protocol vaults) within the same tx?
        - Can MEV/donations/flash moves shift `balanceOf(vault)` between redemption and payout?
    - **Fix pattern:**
        - Snapshot all bases (TVL, liabilities, exchange rates) **prior to side-effects**; compute haircut from the snapshot.
        - Redeem/pull **exactly** `scaledAmount`, not the full nominal.
        - Treat intermediary balances consistently in TVL (or exclude them and adjust the logic).
- [ ]  **Queued-redemption counters can exceed outstanding debt (global DoS)**
    - **Symptom:** Any action that “earmarks” or weights redemptions computes with `available = totalLiability - cumulativeQueued`, and math assumes `available ≥ 0`. A big repay/burn (or liability drop) that doesn’t reduce `cumulativeQueued` makes `available` negative, tripping guards like `require(increment <= total)` or underflow in decay/weight math; thereafter, mint/repay/withdraw/liquidate all revert.
    - **Where to look (general DeFi):**
        - Global redemption/queue/earmark systems: functions like `_earmark`, `_sync`, `updateWeight`, “decay/VRGDA/Log2/Exp” helpers.
        - Liability updates: `repay`, `burn`, `liquidate`, bad-debt write-downs, oracle price drops that reduce `totalDebt/outstanding`.
        - Cross-module flows where repayments send assets to a separate contract (router/transmuter) without reconciling the queue.
    - **Red flags:**
        - Formulas like `WeightIncrement(increment, totalDebt - cumulativeEarmarked)` or `queued += min(increment, totalDebt - queued)` **without clamping**.
        - `repay()` / `burn()` reduces `totalDebt` but **never** decreases `cumulativeEarmarked` (or an equivalent queued counter).
        - Monotonic `cumulativeEarmarked` that only grows; comments or checks like `require(increment <= total)`.
        - Order: `_earmark(); _sync();` then mutate liabilities, or vice-versa, with no reconciliation.
    - **Questions to ask next time:**
        - Can any path **reduce liabilities** (repay, liquidation, write-down) without proportionally reducing the queued/earmarked amount?
        - Is `increment` for new earmarks **capped at** `max(0, totalLiability - cumulativeQueued)`?
        - What happens if matured claims > (current outstanding)? Do weight/decay helpers saturate or revert?
        - Is there a **saturating min** relation guaranteed on-chain: `cumulativeQueued ≤ totalLiability`, enforced after each liability change?
    - **Fix pattern:**
        - Clamp everywhere: `available = totalLiability > cumulativeQueued ? totalLiability - cumulativeQueued : 0`; `increment = min(increment, available)`.
        - On liability decreases (`repay/_subDebt/writeDown`), do `cumulativeQueued = min(cumulativeQueued, totalLiability)` or decrement queue first.
        - Make weight/decay libs tolerant: early-return 0 on `x > domain`, avoid unchecked underflow, and add invariant tests for `queued ≤ total`.
        - Reconcile external balances (e.g., tokens already at the transmuter) when computing what still needs earmarking.
- [ ]  **Cap checks use raw token balances → griefable deposit DoS**
    - **Symptom:** Deposits revert with “cap exceeded” even when no one has actually deposited that much; an attacker can pin `depositCap` by (a) directly `transfer`ing tokens to the contract, or (b) cycling deposit/withdraw (no fee/cooldown) to keep the on-chain `balanceOf(this)` near the cap.
    - **Where to look (general DeFi):**
        - Any cap/TVL/limit logic in `deposit`, `mint`, `stake`, `bond`, `addLiquidity`.
        - Code that computes capacity via `IERC20(token).balanceOf(address(this))` (or `totalAssets()` that proxies to it).
        - Places receiving tokens from other flows (liquidations, fee modules, rewards) that also land in the same balance.
    - **Red flags:**
        - `require(token.balanceOf(this) + amount ≤ depositCap)` (or variants).
        - No internal ledger like `totalDeposited/totalShares` distinct from raw token balance.
        - No sweep/reconciliation for stray tokens; no access control on who can send tokens in.
        - Zero withdrawal fee/cooldown → cheap cap pinning via rapid deposit/withdraw.
        - Rebasing tokens or inflows from other modules counted toward the cap.
    - **Questions to ask next time:**
        - Does the cap measure **intentional deposits** or **all tokens held**? What prevents arbitrary transfers from counting?
        - If other modules send tokens here, are those excluded from cap math or reconciled out?
        - Is there a cooldown/fee that makes cycling costly?
        - How are rebases handled—could they silently hit the cap?
        - Is there a sweep/rescue path that adjusts accounting after stray inflows?
    - **Fix pattern:**
        - Enforce caps against an **internal accumulator** (e.g., `totalDeposited` or share supply), not `balanceOf(this)`.
        - Compute headroom as `cap - totalDeposited` (saturating at zero). Ignore unsolicited balance for admission control.
        - Add reconciliation: track `externallyReceived = balanceOf(this) - accountedAssets` and either sweep it or exclude from cap.
        - Consider a small withdrawal fee or short cooldown to deter cap-pinning cycles.
        - If ERC4626, cap on **shares minted** (`totalSupply`) or on accounted assets, not raw token balance.
- [ ]  **Division-by-zero in cross-module “health” ratios (claims/redemptions)**
    - **Symptom:** Claim/redemption paths revert with panic 0x12 (division by zero) when computing ratios like `issued * 10^d / underlyingValue`. A module drains/holds the payout asset so the producer’s `underlyingValue` becomes 0 while obligations still exist, DoSing claims.
    - **Where to look (general DeFi):**
        - Any function that scales payouts/fees by “bad-debt ratio”, “coverage”, “utilization”, “HF”, or `A * SCALAR / B`.
        - Spots that read `getTotalUnderlyingValue()`, `totalAssets()`, or `token.balanceOf(this)` from one module while another module (fee vault, transmuter, rewards sink) can also hold the same asset.
        - Flows that **move funds before** computing the ratio (post-transfer denominator).
        - Upgradeable pointers (e.g., swapping transmuter/treasury addresses) that strand assets across modules.
    - **Red flags:**
        - No guard for `B == 0` or near-zero in `A/B`.
        - Denominator uses **raw balances** instead of accounted/aggregated assets across modules.
        - Burns of debt/synth in module X without decrementing `totalIssued` in module Y.
        - Commented logic like “if bad debt, scale claim by ratio” with no branch for zero TVL.
    - **Questions to ask next time:**
        - Can the denominator reach zero because repayments/liquidations/fees send the asset to a different contract?
        - Is the ratio computed **before or after** transferring out funds for the same operation?
        - When another module pays from its own balance, who updates `totalIssued`/debt counters?
        - If both `issued == 0` and `underlying == 0`, do claims still exist? What does the code do?
        - Are multi-module holdings aggregated (view that sums vault + transmuter + fee vault), or is it a single `balanceOf(this)`?
    - **Fix pattern:**
        - Compute ratios against **accounted assets** (internal ledger or aggregator view across all holders), not just `balanceOf(this)`.
        - Take a **pre-transfer snapshot** for ratio math; don’t shrink the denominator mid-calc.
        - Add explicit guards: if `denominator == 0` → (a) skip scaling when `issued == 0`, (b) revert with clear “InsufficientUnderlying” error, not a panic, or (c) clamp denominator to a minimum and gate payouts.
        - Keep `totalIssued` in sync wherever burns happen (redeem paths that don’t call the issuer must still decrement).
        - Optionally add user-side min-out/slippage to avoid silent haircut when ratios change.
- [ ]  **Bad-debt/health ratio ignores off-vault assets (inflated haircuts)**
    - **Symptom:** After liquidations or yield-based repayments, the protocol’s “badDebtRatio/coverage” spikes even when positions aren’t creating bad debt; redeemers get hair-cut because the denominator only counts assets still in the main vault.
    - **Where to look (any DeFi with redemptions/liquidations):**
        - Functions that compute global ratios: `badDebtRatio`, `coverage`, `utilization`, `health = issued / totalUnderlyingValue`.
        - Denominator sources: `getTotalUnderlyingValue()`, `totalAssets()`, `balanceOf(address(this))`, or adapter reads.
        - Flows that **move backing assets to another module** (transmuter/bridge/fee-vault/auction house) on repay/liquidation, while **not** burning or reducing system debt at the same time.
    - **Red flags:**
        - Denominator uses only `balanceOf(this)` (or single vault) and excludes sibling modules that can temporarily/partially hold the same backing asset.
        - Repay by yield token: debt **not** burned; assets sent to another contract; ratio still computed against vault-only balance.
        - Ratio computed **after** transferring assets out in the same call path (post-transfer denominator).
    - **Questions to ask:**
        - Which modules can hold the backing asset at any time? Is the ratio’s denominator aggregating **all** of them?
        - When debt is repaid by yield, who decrements `totalIssued` (if anyone)?
        - During liquidation, do assets leave the vault without reducing debt immediately?
        - Is the ratio snapshotted **before** movements that fulfill the same obligation?
    - **Fix pattern:**
        - Base ratios on **accounted TVL**: sum vault + transmuter + fee-vault + pending buffers (or maintain an internal ledger) instead of raw `balanceOf(this)`.
        - Snapshot denominator **pre-transfer** for the current operation’s math.
        - Keep `totalIssued` in sync when obligations are satisfied out of sibling module balances (burn or explicit decrement).
- [ ]  **Same-block “lock” inflation DoS on cross-module sanity checks**
    - **Symptom:** Calls guarded by `require(X >= external.totalLocked())` (or similar “queued obligations” checks) revert when an attacker briefly inflates `totalLocked` (create/lock) and then deflates (claim/cancel) within the same block, sandwiching the target txn.
    - **Where to look (any redemption/queue-based systems):**
        - Admin ops or burns that read another contract’s **pending/locked** counters: e.g., `setX()`/migrations, `burn()`/`redeem()` caps, safety checks comparing vault balance (or converted balance) to `other.totalLocked()`.
        - External reads to modules like *transmuters, auctions, bridges, queues* used in `require(...)` fences.
    - **Red flags:**
        - Ability to **create and claim/cancel in the same block**; no per-block nonce, timelock, or cooldown on queue mutations.
        - Sanity checks use **current** `totalLocked()` from another contract without smoothing/snapshotting.
        - Checks compare **converted balances** to an **instantaneous** queue size (price/decimals rounding amplifies).
    - **Questions to ask:**
        - Can users both open and settle a queue position in one block? Any cooldown or min age?
        - Are “locked” amounts **time-weighted** or **block-latched**, or purely instantaneous?
        - Do guarded functions read **multiple external values** that a user can move in the same block?
        - What happens if `totalLocked()` spikes between mempool and inclusion—does it hard revert?
    - **Fix pattern:**
        - **Block-latch** queue deltas (e.g., disallow create+claim in the same block; require `maturationBlock > block.number`).
        - Snapshot `lockedAtLastEarmark` / use **prior-block** values for gates, not live ones.
        - Introduce **cooldowns / minimum age** for claims; rate-limit queue growth/settlement.
        - Replace hard `require(X >= Y_live)` with **buffered** thresholds (margin) or staged two-step ops.
- [ ]  **Time-variant lock/unlock math strands collateral**
    - **Symptom:** After full debt repayment, user can’t withdraw all collateral; `rawLocked > 0` or `freeCollateral < collateralBalance` even when `debt == 0`. Unlock uses a worse ratio than the original lock (price rose or LTV changed).
    - **Where to look (lending/CDP, vaults with “free vs locked” collateral):**
        - Borrow path: functions like `mint/borrow/_addDebt` that compute `toLock = f(debt) * minCollat / price` (or via an adapter `convertDebt↔collateral`).
        - Repay/burn path: functions like `repay/burn/_subDebt` that compute `toFree` with the **current** price/index and **current** min-collateral/LTV.
        - Any adapter/index math (`exchangeRate()/pricePerShare()/index()`) used on both lock and unlock.
        - Admin-set risk params (`minimumCollateralization`, LTV, HF thresholds) referenced in both paths.
    - **Red flags:**
        - Lock and unlock both call the same **spot conversion** (e.g., `convertDebtTokensToCollateral(amount)`), with **no snapshot** or per-position record of the rate/ratio at lock time.
        - `rawLocked += toLock` / `rawLocked -= toFree` without ensuring `toFree` is computed from the **same basis** as `toLock`.
        - Unlock uses **current** `minimumCollateralization` instead of the value at borrow time.
        - Yield-bearing or rebasing collateral (Aave aTokens, Yearn shares, cTokens) but no “lock index” captured.
        - Missing invariant checks like: when `debt == 0` ⇒ `rawLocked == 0` and `freeCollateral == collateralBalance` (± rounding).
    - **Questions to ask:**
        - Is there a **snapshot** (per-position) of price/index and min-collateral used when locking?
        - What happens if the **price/index increases** between borrow and repay? If the **LTV/min-collateral** is changed by governance mid-loan?
        - Are lock amounts tracked in **debt units** or **collateral units**? Are decimals/rounding symmetric both ways?
        - Do tests assert that a position with `debt==0` can withdraw **100%** of its deposit (minus explicit fees)?
    - **Fix pattern:**
        - **Snapshot-based unlocking:** store `lockIndex` and `lockMCR` per lock tranche (or per position) and compute `toFree` using the **same** index/MCR used for `toLock`.
        - **Debt-unit accounting:** record `lockedDebtUnits = debtAmount * MCR` and free based on these units, converting once using the original basis; avoid spot conversions on repay.
- [ ]  **Uninitialized Fenwick/BIT ranges yield negative stake → DoS**
    - **Symptom:** Range queries over a Fenwick (Binary Indexed) tree can return negative stakes (due to expirations/deltas) when the query end crosses into nodes added by a recent resize but **not yet initialized**, causing signed→unsigned casts to revert and halting flows that depend on the query.
    - **Where to look (staking/decay graphs, rewards, TWABs):**
        - Any “Fenwick/BIT/segment tree” lib storing cumulative deltas + expirations and queried via prefix sums then `end − start`.
        - Resize logic (e.g., when graph/window grows to next power of two) that writes default values to new nodes.
        - Call sites that query immediately after growth: earmark/poke, redemption accounting, reward accrual, liquidation guards.
        - Signed → unsigned conversions (`toUint256`, `uint256(intX)`) on query results.
    - **Red flags:**
        - Growth path sets `size = newSize` but **defers** propagation/initialization of new level-1 nodes.
        - Queries allowed with `end > initializedSize` (even if `end ≤ size`).
        - Negative deltas at `start` (e.g., expiry subtraction) combined with **zero/uninitialized** `end` branch.
        - Invariants like `require(increment <= total)` relying on query monotonicity, without guarding uninitialized ranges.
    - **Questions to ask:**
        - When the tree grows, are all new nodes **fully initialized before any query** can touch them?
        - Do queries **clamp** `[start, end)` to the **initialized** range, or short-circuit if `end` isn’t ready?
        - How are negative intermediate results handled? Is there a safe floor/validation before casting to `uint`?
        - Are there tests at **power-of-two boundaries** and immediately after resizes?
    - **Fix pattern:**
        - **Initialize-on-resize**: fully propagate baseline values to all new nodes **atomically** before exposing `size`.
        - **Query guards**: clamp `end` to `initializedSize`, or revert with a clear error until initialization completes.
- [ ]  **Liquidation cheaper than honest repayment (fee asymmetry / treasury-subsidized liqs)**
    - **Symptom:** A borrower pays more to voluntarily repay than to be liquidated because liquidation fees are applied only to *surplus collateral* (or paid by a treasury/fee vault), while normal repay/burn pays a protocol fee on the *entire debt*. Rational users prefer liquidation; protocol loses fees and subsidizes liquidators.
    - **Where to look (CDP/lending/stablecoin systems):**
        - Liquidation path: `liquidate`, `_liquidate`, `seize`, `close`, `auction`, `kick/take/settle` (Maker-style), plus helpers that compute `surplus`, `debtToBurn`, `feeInYield/Underlying`.
        - Repay/burn path: `repay`, `burn`, `_subDebt`, protocol-fee hooks.
        - Treasury flows: `FeeVault`, `Treasury`, `Rewarder`, `reserve()`; look for `vault.withdraw(...)` or `transferFrom(treasury, liquidator, ...)`.
        - Fee config: `protocolFee`, `liquidationFee`, `liquidatorFeeBps`, `bonusBps`, and any “base/excess” split.
        - Accounting units: conversions debt↔underlying↔yield (adapters), places where fees are computed on `surplus = max(collateral - debt, 0)`.
    - **Red flags:**
        - Repay path: `fee = repayAmount * protocolFeeBps / BPS` (charged on *debt*).
        - Liquidation path:
            - `fee = surplus * liquidationFeeBps / BPS` (fee only on leftover collateral).
            - Liquidator “bonus” paid from `FeeVault/Treasury` (`feeVault.withdraw(liquidator, ...)`) instead of borrower collateral.
            - **No** protocol fee applied during liquidation on the extinguished debt (`debtToBurn`).
        - Comments like “excess fee from vault” / “base fee in yield, bonus from vault”.
        - Conditional ordering that repays debt, then computes fee from remaining collateral only.
        - Unit/decimals asymmetry: repay fee in debt units vs liquidation fee in surplus-underlying units.
    - **Questions to ask the codebase owners (and yourself while reviewing):**
        - Is the **protocol fee also charged on liquidation** (on `debtToBurn`), or only on user-initiated repay?
        - **Who funds the liquidator reward**—borrower’s collateral first, or the treasury/fee vault?
        - Are any liquidation fees computed on **surplus only**? What happens when `collateral ≈ debt`?
        - In a typical liquidatable state (e.g., collateral = 1.05–1.10 × debt), is `Cost(liquidation)` < `Cost(repay)`?
        - Are fees consistently applied in the same **unit/base** (debt vs underlying vs yield), with correct rounding?
        - Are there **caps** or limits on treasury subsidies to prevent users from gaming liquidation?
        - Is there a stated **invariant** or test asserting `Cost_liquidation ≥ Cost_repay` for all positions?
    - **Fix pattern:**
        - **Fee parity:** On liquidation, charge at least the **same protocol fee** as a normal repay: `protocolFeeOnLiq = debtToBurn * protocolFeeBps / BPS`, taken from borrower collateral *before* any liquidator bonus.
        - **Borrower-funded rewards first:** Pay liquidator fees/bonus from seized collateral; draw from treasury **only** if strictly necessary and within a capped subsidy.
        - **Consistent base:** Compute both repay and liquidation fees off the **debt extinguished** (`debtToBurn`) or enforce a penalty so `Cost(liq) ≥ Cost(repay)`.
- [ ]  **Mutable LTV/`minimumCollateralization` breaks lock accounting (underflow & DoS)**
    - **Symptom:** After governance raises `minimumCollateralization` (or lowers max LTV), later debt reductions/freeing of collateral subtract more from a global “locked” counter than was ever added at mint-time. Global `_totalLocked` (or equivalent) drifts down and eventually underflows on `redeem/claimRedemption`, DoSing redemptions.
    - **Where to look (lending/stablecoin/CDP with redemptions):**
        - Debt add/remove: `_addDebt/_subDebt`, `mint/repay/burn/_forceRepay/_liquidate`.
        - Redemption path invoked by a separate module: `redeem/claimRedemption/settle`, guarded by `onlyTransmuter/onlyModule`.
        - Global lock tracking: `_totalLocked`, `lockedCollateral`, `rawLocked`, `freeCollateral`.
        - Governance setters: `setMinimumCollateralization`, `setLTV`, `setLiquidationThreshold`, any param that changes lock ratios.
        - Unit conversions used in lock math: `convertDebtTokensToYield/Underlying`, scaling constants.
    - **Red flags:**
        - Lock/free computed with **current** parameter:
        `toLock = convert(debt) * minimumCollateralization / 1e18toFree = convert(amount) * minimumCollateralization / 1e18` (uses updated value, not historical).
        - Global update without saturation: `_totalLocked = _totalLocked - totalOut;` (no `min` with `_totalLocked`).
        - No clamp vs per-account ledger: missing `toFree = min(toFree, account.rawLocked)` and/or `min(toFree, _totalLocked)`.
        - Parameter changes **do not** recompute per-position locked snapshots or deltas.
        - Mixed bases/rounding differences between lock and free (debt vs underlying vs yield).
    - **Questions to ask next time:**
        - Does changing LTV/min collateral retroactively affect **existing** positions’ unlock math?
        - Is there a **per-position** record (e.g., `rawLocked` checkpoints) ensuring freeing never exceeds what was locked?
        - Are all global decrements **saturated** (`min(subtrahend, _totalLocked)`) and aligned to the same unit as increments?
        - Can redemptions free collateral based on protocol fee or conversion deltas that weren’t part of the original lock?
        - Are there tests asserting **conservation**: sum of per-account locked equals global locked; raising the parameter can’t cause underflow/DoS?
    - **Fix pattern:**
        - **Snapshot-based freeing:** Track `rawLocked` per position at mint-time and free using that ledger (`toFree = min(request, account.rawLocked)`), not today’s parameter.
        - **Saturating math:** `_totalLocked = _totalLocked - min(_totalLocked, toFreeOrTotalOut);` and clamp per-account first.
- [ ]  **Misrouted settlement transfer (tokens sent to `address(this)` instead of the settlement module)**
    - **Symptom:** Debt/earmark counters go down but the *receiver module’s* balance (e.g., transmuter/amm/redemption vault) doesn’t increase; redemptions underfunded while the core contract silently accumulates tokens.
    - **Where to look (any lending/stablecoin with external settlement modules):**
        - Debt-settlement helpers: `_forceRepay/_repay/_liquidate/_settle/_redeemInternal`.
        - Transfer sinks: variables like `transmuter`, `redeemer`, `feeCollector`, `vault`, `treasury`.
        - Post-debt paths that move value: right after `_subDebt` / earmark updates.
    - **Red flags:**
        - Comments say “send to module” but code does `safeTransfer(token, address(this), amount)`.
        - Transfers to `address(this)` or `msg.sender` in settlement flows.
        - Inconsistent sinks across paths (e.g., `repay()` sends to module, `_forceRepay()` sends to self).
        - Module balance used in invariants (`totalLocked`, redemption math) diverges from on-chain balance.
    - **Questions to ask:**
        - What is the **single source of truth** for where repaid collateral should live? Is every code path sending there?
        - Do all settlement paths (user repay, forced repay, liquidation, redemption) route tokens to the **same** sink?
        - Are there invariants/tests asserting `moduleBalance += creditedAmount` whenever debt is reduced?
        - Could a zero/unset module address or shadowed storage variable cause a fallback to `address(this)`?
    - **Fix pattern:** Route via a dedicated helper `__sendToSettlementSink(token, amount)` that always transfers to the configured module (`safeTransfer(token, settlementModule, amount)`), guard `settlementModule != address(0)`, and add an invariant/test that settlement-module balance increases by the credited amount on every debt-reduction path.
- [ ]  **Over-broad “recovery mode” guards block solvency-improving closes**
    - **Symptom:** When system TCR < CCR (recovery mode), user close/repay paths are blanket-reverted even if removing that vault/position (with MCR < ICR < CCR) would *increase* TCR. Results in unnecessary DoS of voluntary closures and slower exit from recovery.
    - **Where to look (any CDP/borrowing system with global health gates):**
        - Global health checks: `isRecoveryMode/_checkRecoveryMode`, `require(!recoveryMode)` wrappers.
        - Close/repay flows: `closePosition/repayAll/burnAndClose`, and their internal helpers executed under recovery mode.
        - System metric getters: `getTCR/getSystemCollateralRatio`, `getICR`, CCR/MCR constants.
        - Ordering of updates: where TCR is checked vs. when per-position debt/collateral are removed.
    - **Red flags:**
        - Unconditional `if (isRecoveryMode) revert` around *all* closes/repays.
        - Gates compare only `TCR vs CCR` without considering the closing position’s `ICR`.
        - No “delta-TCR” simulation (i.e., “TCR after removing this position”).
        - Comments claiming “protect solvency” while forbidding actions that unequivocally improve TCR.
    - **Questions to ask:**
        - Does closing a position with `MCR < ICR < CCR` strictly increase (or not decrease) TCR when removed from totals?
        - Are there fees/penalties paid from system pools that could make a close *reduce* TCR?
        - Is TCR computed **before** or **after** subtracting the position? (Any TOCTOU between check and state update?)
        - Which actions are allowed in recovery mode today (partial repay, full close, top-up)? Why are closes treated differently?
        - What boundary should be allowed: `ICR ≥ TCR`, `ICR ≥ MCR`, or “newTCR ≥ oldTCR”?
    - **Fix pattern:** Replace blanket recovery-mode bans with monotonic guards. Permit closes/repays if either (a) `ICR ≥ TCR` (conservative) or (b) `simulateClose()` yields `newTCR ≥ oldTCR` and `ICR ≥ MCR`. Implement a pure helper `canCloseInRecovery(position, totals) → bool` and unit tests at edges (`ICR = MCR`, `ICR = CCR`, tiny dust).
- [ ]  **Redemption scan assumes sort-key ⇒ health; per-trove health not rechecked**
    - **Symptom:** Redemption walks a sorted list (e.g., by NICR = collateral/principal) and redeems against predecessors without revalidating health; troves with high accrued interest (entireDebt > principal) get redeemed even if `ICR < MCR`.
    - **Where to look (any CDP/trove system with sorted lists + interest):**
        - Sorted structures: `SortedTroves/SortedVaults`, `getPrev/getNext`, hint-based searches.
        - Redemption paths: `redeemCollateral/redeem`, loops that traverse from a “first healthy” trove.
        - Debt fields: `principal`, `accruedInterest/fees`, `entireDebt`, and how/when the sort key is updated.
    - **Red flags:**
        - Sort key uses **principal** (NICR) while health uses **entire debt** (`ICR = collateral*price / entireDebt`).
        - Heterogeneous interest/fees per trove or time-varying rates.
        - Only the **first** trove’s `ICR>=MCR` is checked; subsequent troves in the loop are not.
        - Sorted list not rebalanced when interest accrues (order can go stale).
    - **Questions to ask:**
        - Is the sort key provably **monotone** with the health metric under all interest/fee regimes?
        - Do we re-check `ICR>=MCR` **for each trove** before redeeming/partial redeem?
        - When/what events trigger re-insertion/re-sorting as interest accrues?
        - Can two equal-NICR troves have different `ICR` due to different interest? What path handles that?
    - **Fix pattern:** Make the sort key reflect health (`collateral / entireDebt`) **or** keep existing key but **validate `ICR>=MCR` per trove during redemption** and skip unhealthy ones; ensure list reordering on interest accrual/fee changes and add tests asserting no redemption from `ICR<MCR` positions.
- [ ]  **Volatile fee “modes” let users pay more than quoted at execution**
    - **Symptom:** A tx preview shows 0/discounted borrow fee (e.g., “recovery mode”), but the global mode flips before/mined during execution and a non-zero fee is charged, minting more debt than the user expected.
    - **Where to look (any lending/CDP/AMM with mode-gated fees):**
        - Position entry paths: `openTrove/openPosition/borrow/mint`.
        - Fee gates tied to global state: `isRecoveryMode`, `isEmergency`, utilization/health thresholds (TCR/CCR, utilization %, oracle price guards).
        - Mode check helpers vs charge sites: `_checkRecoveryMode(...)` vs `_triggerBorrowingFee(...)` (or equivalents).
        - Oracle/price reads that affect mode and fee in the same tx.
    - **Red flags:**
        - `if (!isRecoveryMode) fee = ...` (or similar) with **no user-supplied max fee / max net debt cap**.
        - Mode derived from volatile inputs (prices, utilization) without stickiness, grace period, or snapshotting.
        - UI/preview route returns a quote but on-chain path **doesn’t assert** the quote (no slippage guard).
        - Mode checked once, then state used later without re-validation or bounds.
    - **Questions to ask:**
        - Can the mode change within the same block due to other txs or oracle updates?
        - Do users pass `maxFeeBps` / `maxNetDebt` and does the function `require(actual <= userCap)`?
        - Is there a `previewOpen/quoteBorrow` and does the stateful function bind to that quote?
        - Are mode changes sticky (e.g., epochized) or instantaneous?
    - **Fix pattern:** Add slippage-style guards to state-changing calls (e.g., `maxFeeBps`, `maxDebtWithFee`, `require(isMode==expected || fee<=maxFeeBps)`). Prefer quote→execute flows where `execute` enforces the quoted bounds. Consider making mode transitions sticky for N blocks or snapshotting inputs at call start and re-requiring them before charging fees.
- [ ]  **Under-scaled cumulative product (P) can hit zero → liquidation/claim DOS**
    - **Symptom:** Repeated partial-depletion updates multiply `P` by tiny factors (`newProductFactor = 1e18 - lossPerUnit`) and apply only a **single** `SCALE_FACTOR` (e.g., `1e9`) bump; over several cycles `P` underflows to `0`, tripping `assert(newP > 0)` and blocking liquidations/claims.
    - **Where to look (Stability/Rewards pools with epoch/scale math):**
        - Product/sum trackers: `_updateRewardSumAndProduct`, `P`, `S`, `currentScale`, `currentEpoch`.
        - Fixed-point helpers/constants: `DECIMAL_PRECISION` (1e18), `SCALE_FACTOR` (often 1e9).
        - Branches around scaling: checks like `if (PBeforeScaleChanges < SCALE_FACTOR) { P *= SCALE_FACTOR; scale++ }`.
        - Any `assert(newP > 0)` / division-by-precision hotspots.
    - **Red flags:**
        - Single-step scaling (at most +1/+2 scale) regardless of how small `PBeforeScaleChanges` is.
        - Equality edge cases like `if (PBeforeScaleChanges == 1) { scale += 2 }` with no general fallback.
        - Using `uint256(newP) = currentP * factor / 1e18` with truncation and **no** `mulDiv` (512-bit) or rounding-up.
        - `SCALE_FACTOR` set too low (e.g., `1e9`) vs. worst-case `newProductFactor` sequences near zero.
        - Reliance on `assert` (panic) rather than guarded recovery.
    - **Questions to ask:**
        - What is the **minimum** `newProductFactor` per event and can adversaries trigger many such events back-to-back?
        - Do tests simulate long runs of high-loss liquidations to the brink of precision?
        - Is there a **looped** scaling (apply `SCALE_FACTOR` repeatedly) or mantissa/exponent representation to keep `P` bounded away from 0?
        - What happens when `PBeforeScaleChanges == 0` after truncation—do we clamp or revert?
        - Can `S`/reward math tolerate additional scaling steps without drift?
    - **Fix pattern:**
        - Use mantissa/exponent (floating) `P` (store `scale`/`epoch`; keep mantissa in \[1e17,1e18)) **or** loop scaling:
        `while (PBefore < SCALE_BOUND) { P *= SCALE_FACTOR; scale++; PBefore *= SCALE_FACTOR; }`
        - Replace raw `a*b/1e18` with `mulDiv(a,b,1e18)` (512-bit) and consider **rounding up** when shrinking `P`.
        - Add a hard floor/clamp: `if (newP == 0) newP = 1;` plus an event + defensive mode, instead of `assert`.
        - Size `SCALE_FACTOR` to cover worst-case sequences (or make it adaptive), and add invariants/unit tests covering many sequential high-loss updates.
- [ ]  **Refinance/borrow paths skip “apply pending rewards” → stale debt/collateral in fees & ICR**
    - **Symptom:** Refinancing (or similar debt-changing ops) compute fees/ICR on **pre-reward** state: fee undercharged, ICR check may pass when unsafe, max borrow computed too high. After liquidations/redistribution, `pendingRewards` aren’t merged before reading `getTroveDebt/getTroveColl`.
    - **Where to look (Liquity-style troves / CDP engines):**
        - User ops in `BorrowerOperations` (or equivalent): `_refinance`, `refinance`, `borrow`, `repay`, `adjustTrove`, `reopen`, `rebalance`.
        - Reward/interest sync points: `applyPendingRewards`, `updateSystemAndTroveInterest`, `sync/poke/settle/checkpointAccount`.
        - Getters used for math right after interest update: `getTroveDebt`, `getTroveColl`, `getTrovePrincipal`, `computeCR/_computeCR`, `maxBorrowingCapacity`.
        - Liquidation redistribution flow in `TroveManager`: where `pendingDebt/pendingColl` accumulate and how/when they’re applied.
    - **Red flags:**
        - Fee/ICR math like:
        `oldDebt = getTroveDebt(borrower); fee = pct * oldDebt; newICR = CR(getTroveColl, getTroveDebt)`**without** a prior `applyPendingRewards(borrower)`.
        - Functions that call `updateSystemAndTroveInterest` then immediately read debt/collateral but never call `applyPendingRewards`.
        - Inconsistency: `adjustTrove()` applies rewards, but `_refinance()`/`borrow()` doesn’t.
        - Tests never create non-zero `pendingRewards` before invoking refinance/borrow.
        - Getters returning **principal** vs **actual** debt mixed interchangeably in fee/ICR math.
    - **Questions to ask:**
        - Do **all** state-changing trove paths call `applyPendingRewards(borrower)` (or equivalent) before any reads and fee/ICR calculations?
        - Are fee bases intentionally principal-based, or should they include accrued/pending debt?
        - Do getters return post-redistribution values, or must callers sync first?
        - What happens if `pendingRewards` are large (after recent liquidations)? Can an unsafe ICR slip through?
        - Are there unit tests that: (1) create pending rewards, (2) run refinance/borrow, (3) compare fee/ICR vs a synced baseline?
    - **Fix pattern:**
        - Introduce a **standard preflight** at the top of every trove-mutating function:
        `updateSystemAndTroveInterest(borrower); applyPendingRewards(borrower);`
        - Compute fees & ICR **after** syncing; use a single helper like `getSyncedTrove(borrower)` to fetch debt/collateral for calculations.
        - Align fee base (principal vs actual debt) with spec and document it; add invariants asserting fee/ICR equivalence to a fully-synced path.
        - Add tests where pending rewards exist before refinance/borrow to prevent regressions.
- [ ]  **Rounding drift traps the last LP (Σ claims > pool balance)**
    - **Symptom:** Final depositor’s withdraw reverts (`ERC20InsufficientBalance`) because `getCompoundedDeposit(user)` (or share → asset math) exceeds the contract’s actual token balance after many scale/epoch updates and floor division.
    - **Where to look (Liquity-style StabilityPools / staking vaults / ERC-4626 clones):**
        - Pool exits: `withdrawFromSP`, `withdraw`, `redeem`, `_send*ToDepositor`.
        - Global vs per-user accounting: `totalDeposits/totalAssets/totalShares` vs `getCompoundedDeposit`, price-per-share math, epoch/scale `P/S` products.
        - Math helpers and precisions: `LiquityMath._min`, `DECIMAL_PRECISION`, rounding on `shares*assets/totalShares`.
    - **Red flags:**
        - Transfer uses user-computed amount **without** `min(amount, token.balanceOf(this))` (or `min(amount, totalDeposits)`).
        - `totalDeposits` updated **after** transfer or drifts from `balanceOf(this)`.
        - Multiple liquidations/rebases/fee-on-transfer tokens involved; floor division everywhere; no dust sweeping.
        - No invariant/test that `Σ user claims ≤ pool balance` and that “last user withdraw” succeeds.
    - **Questions to ask:**
        - Do we cap payout by on-chain balance / pool accounting before transferring?
        - Is rounding direction specified (favor user vs protocol) and consistent across deposit/withdraw?
        - Are pending gains/losses applied (scale/epoch sync) **before** computing the user’s claim?
        - Is there a dust-sweep or `withdrawAll()` path that drains the pool for the last LP?
    - **Fix pattern:** Before sending, compute `payout = min(requested, min(totalDeposits, token.balanceOf(address(this))))`; apply pending updates first; decrement global accounting by `payout`; optionally sweep residual dust to the last withdrawer; add tests for final-LP withdraw after many loss/gain and rounding events.
- [ ]  **NICR ignores accrued interest → withdrawals allowed while troves are under-collateralized**
    - **Symptom:** Stability Pool `withdraw` guard checks only the last trove by **NICR** ordering; since NICR excludes accrued interest, older troves with high accrued interest can sit **under MCR** while the last trove looks healthy—allowing SP withdrawals that dodge pending bad debt.
    - **Where to look (Liquity forks / NICR-sorted Trove systems):**
        - Sorting & reinsertion: `sortedTroves.reInsert(...)` callers, `LiquityMath._computeNominalCR(coll, principal)` vs `getCurrentICR(coll, debtWithInterest)`.
        - Interest & rewards flow: `updateSystemAndTroveInterest`, `applyPendingRewards`, `getTroveDebt/Principal/Collateral`.
        - SP exit gates: `StabilityPool.withdrawFromSP` → `_requireNoUnderCollateralizedTroves()` (does it only check `sortedTroves.getLast()`?).
    - **Red flags:**
        - NICR computed from **principal** (and maybe pending principal) but not accrued interest.
        - Reinsert uses NICR, while liveness checks use ICR of just the boundary trove.
        - Variable/heterogeneous interest rates per trove; no resorting when interest accrues.
        - No system-wide “any under MCR?” view; guards read a single address from the list.
    - **Questions to ask:**
        - Is the sorting key monotonic with **true ICR** when interest accrues? What events resort?
        - Are `applyPendingRewards()` and interest accrual applied **before** fee/ICR checks and before reinserting?
        - Can a non-last trove drop below MCR without moving to the end of the list?
        - Do SP withdrawals/large exits force a global sync or at least scan until the last **under-MCR** trove is skipped?
    - **Fix pattern:** Include accrued interest in the sorting key **or** force sync+reorder before critical checks; change SP guard to iterate from the tail until `ICR(last) ≥ MCR` (or consult a maintained “min ICR” tracker); add invariant tests: (1) interest accrual can’t make `exists(trove: ICR<MCR)` while SP withdrawal still passes, (2) `applyPendingRewards` is invoked before NICR/ICR reads in refinance/adjust/withdraw paths.
- [ ]  **Pre-rewards debt deltas drift ActivePool vs Trove**
    - **Symptom:** In `repay/_adjustTrove`, debt/fee deltas are computed **before** `applyPendingRewards()`. After rewards are applied, Trove debt changes differ from the precomputed deltas used to move funds, causing **Trove ↔ ActivePool mismatch** (principal/interest don’t net out), later underflows, and blocked withdrawals.
    - **Where to look (Liquity-style trove systems):**
        - `BorrowerOperations._adjustTrove / repay / refinance` flow:
            - Order of calls: `updateSystemAndTroveInterest()` → `calculateDebtAdjustment(...)` → `applyPendingRewards(...)` → `_updateTroveDebt(...)` → `_moveTokensAndCollateralFromAdjustment(...)`.
        - Debt sources of truth:
            - `TroveManager.getTrove{Principal,Interest,Debt}`, `getPendingDebt`, `applyPendingRewards`.
        - Pool updates:
            - `ActivePool.{increase,decrease}{Principal,Interest}` or helper that transfers MUSD and updates pool accounting.
    - **Red flags:**
        - Any `calculateDebtAdjustment` (principal/interest split) or fee math executed **before** `applyPendingRewards`.
        - `_moveTokensAndCollateralFromAdjustment(...)` uses stale `{principalAdjustment,interestAdjustment}` computed pre-rewards.
        - Mixed reads: some from `{principal,interest}` while others from `getTroveDebt()` without first syncing rewards.
        - No invariant tests asserting ∑Trove principal/interest == ActivePool principal/interest after adjustments.
    - **Questions to ask:**
        - Do we always call `applyPendingRewards(_borrower)` **before** computing fees/deltas and **before** any pool mutation?
        - Are principal/interest splits **recomputed** after rewards application (or derived on-chain from updated Trove state) rather than passed through?
        - Does the movement to `ActivePool` exactly match the Trove delta post-rewards? (Add assertion or event cross-check.)
        - Do liquidation-then-repay paths include a test with non-zero pending principal **and** pending interest?
    - **Fix pattern:** Normalize state first, then compute deltas.
        - Call `applyPendingRewards(_borrower)` at the **start** of adjust/repay/refinance flows.
        - Compute `principalAdjustment/interestAdjustment` from the **post-rewards** Trove state (or compute a single `payment` then deterministically split using the updated interest-first rule).
        - Use those deltas for both `_updateTroveDebt` and pool mutations, or derive pool mutations directly from the updated Trove delta.
        - Add invariants/tests: after any adjust/repay, `ΔActivePool == −ΔTrove` for principal and for interest, across scenarios with pending rewards.
- [ ]  **Double-counted interest in borrow-capacity checks blocks valid borrows**
    - **Symptom:** Debt-increase/withdraw actions revert “exceeds maxBorrowingCapacity” even when the user should fit, because the check adds `interestOwed` on top of `getTroveDebt()` that already includes accrued interest.
    - **Where to look (Liquity-style troves / CDPs / lending controllers):**
        - Borrow flows: `BorrowerOperations._adjustTrove / withdrawMUSD / borrow()`.
        - Capacity guards: `_requireHasBorrowingCapacity`, `getUserAvailableBorrow()/getTroveMaxBorrowingCapacity()`.
        - Debt accessors: `TroveManager.getTroveDebt()` (often = principal + interestOwed + accruedInterest) vs. separate fields `principal`, `interestOwed`, `accrued`.
    - **Red flags:**
        - Guard like `_vars.maxBorrowingCapacity >= _vars.netDebtChange + _vars.debt + _vars.interestOwed` where `_vars.debt = getTroveDebt()`.
        - Any check that mixes “total debt” with per-component adds (principal + interest) in the same sum.
        - Two sources of truth for interest (e.g., `getTroveDebt()` plus `interestOwed` passed alongside).
        - Comments say “includes interest” but code still adds an interest term.
    - **Questions to ask:**
        - Does `getTroveDebt()` already include accrued interest and/or pending interest? If yes, where else is interest added again?
        - Is `netDebtChange` already fee-inclusive? Are we also adding protocol/borrow fees separately in the guard?
        - Do we normalize/apply pending rewards/interest **before** reading debt and capacity?
    - **Fix pattern:**
        - Use a **single source of truth** for current debt in capacity checks (either `getTroveDebt()` **or** explicit `principal + interest`), not both.
        - Change guard to `_vars.maxBorrowingCapacity >= _vars.netDebtChange + _vars.debt` (drop the extra interest term), or recompute components after syncing interest and use components exactly once.
        - Add unit tests at the edge (near max capacity) after time elapses so interest accrues, asserting borrow succeeds once and fails when expected.
- [ ]  **Refinance fee mistakenly charged on gas compensation (and sometimes accrued interest)**
    - **Symptom:** `_refinance()` computes fee from **total trove debt** (principal + accrued interest + **gas compensation**), unlike open/adjust where fees apply to principal (or net debt increase) only.
    - **Where to look (Liquity-style troves / CDPs):**
        - `BorrowerOperations._refinance` (fee base and call to `_triggerBorrowingFee`).
        - `TroveManager.getTroveDebt()` vs. helpers like `_getCompositeDebt()` / `getCompositeDebt()` (often include gas comp).
        - Gas-comp constants/derivations: `GAS_COMPENSATION`, `MIN_NET_DEBT`, `getBorrowingFeeBase()` if present.
        - Open/adjust paths: `_openTrove`, `_adjustTrove` to compare fee base semantics (usually principal or net increase).
    - **Red flags:**
        - `oldDebt = troveManager.getTroveDebt(borrower)` then `amount = refinancingFeePercentage * oldDebt / 100`.
        - Any fee function that takes **total debt** or **composite debt** as input during refinance.
        - No explicit subtraction of gas comp: `amountBase = oldDebt - GAS_COMPENSATION`.
        - Inconsistent treatment across flows (open/adjust vs. refinance).
    - **Questions to ask:**
        - Should refinancing fees apply only to **principal** (parity with borrow/adjust), or also to accrued interest? What does the spec/UX promise?
        - Does `getTroveDebt()` include gas comp on this protocol? If yes, where is it excluded for other fees?
        - Is there a helper to fetch **feeable principal** (principal without gas comp), or do we need to derive it?
        - Are pending rewards/interest applied before fee calculation, and does that silently change the base?
    - **Fix pattern:**
        - Define fee base explicitly: `feeBase = currentPrincipal` (or `netDebtChange`) and **exclude gas compensation**: `feeBase = max(currentPrincipal - GAS_COMPENSATION, 0)`.
        - If policy excludes interest from fees: `feeBase = principalOnly`.
        - Add unit tests: (1) refinance with nonzero gas comp → fee excludes it; (2) parity test: open/adjust vs. refinance charge same fee for equivalent principal.
- [ ]  **Upgrade-mode gate blocks redemptions at low TCR**
    - **Symptom:** `redeemCollateral` reverts with “TCR < MCR” during upgrades when minting is disabled (old `BorrowerOperations` removed from `mintList`), preventing peg-arb redemptions and prolonging depeg.
    - **Where to look (Liquity-style CDP systems):**
        - `TroveManager.redeemCollateral` → `_requireTCRoverMCR` / `_getTCR`.
        - Token governance gates: `musdToken.isInMintList(borrowerOps)` / pause flags / upgrade switches.
        - Upgrade paths that relax other checks (close trove in RM, etc.) but **not** the redemption TCR check.
    - **Red flags:**
        - Unconditional `require(TCR >= MCR)` in redemption path.
        - No conditional allowing redemption when minting is paused/disabled or during upgrade windows.
        - Inconsistent safety model across modules (e.g., open/adjust checks bypassed in upgrade, redemption still strictly gated).
        - Known deadlocks: last/undercollateralized troves cannot be liquidated, and redemption is blocked by TCR.
    - **Questions to ask:**
        - During upgrades, is minting paused/disabled? If so, what is the **intended** peg-restoration path?
        - Should redemptions be allowed under `TCR < MCR` when minting is off (no new risk introduced)?
        - Are there emergency/unwind modes where SP or keeper redemptions should bypass TCR?
        - Do unit tests cover “upgrade + low TCR + redemption” scenarios?
    - **Fix pattern:** Gate the TCR check by upgrade/minting state:
        - Example:
            
            ```solidity
            if (token.isInMintList(address(borrowerOps))) {
                require(_getTCR(price) >= MCR, "Cannot redeem when TCR < MCR");
            }
            
            ```
            
        - Alternatively introduce an explicit `upgradeMode`/`mintingPaused` flag that permits redemptions (maybe with adjusted fees) while other risky actions remain disabled; add tests asserting redemptions work when minting is disabled and TCR < MCR.
- [ ]  **“Don’t-liquidate-last-trove” guard strands positions during upgrades**
    - **Symptom:** When upgrading (minting disabled / old `BorrowerOperations` removed from mintList), the unconditional “skip if last trove” rule blocks liquidation of the final trove, leaving a zombie position and trapped collateral/debt.
    - **Where to look (Liquity-style CDP systems):**
        - `TroveManager.liquidate()` / `_liquidate()` last-trove checks (`TroveOwners.length <= 1`).
        - Token/mint gating: `musdToken.isInMintList(borrowerOperations)` (or pause/upgrade flags).
        - Upgrade flows/migrations that move users off the old manager.
    - **Red flags:**
        - Hardcoded early return like `if (TroveOwners.length <= 1) { … return; }` without considering upgrade state.
        - No alternative close path for the final trove in legacy/retired deployments.
        - Tests cover “normal” ops but not “upgrade + one trove left”.
    - **Questions to ask:**
        - During upgrades, can the last trove be closed via redemption or admin-assisted unwind?
        - Is minting paused/removed from mintList during upgrade? If so, what’s the intended exit for remaining troves?
        - Do we have an emergency liquidation/settlement path that bypasses the last-trove rule?
    - **Fix pattern:** Condition the last-trove guard on active minting/normal mode; allow liquidation when upgrading.
        
        ```solidity
        bool normalMode = musdToken.isInMintList(address(borrowerOperations));
        if (normalMode && TroveOwners.length <= 1) {
            return singleLiquidation; // keep protection only in normal mode
        }
        
        ```
        
        - Or add an `upgradeMode` flag enabling liquidation/redemption of the final trove; include unit tests for “upgrade + single trove” ensuring it can be closed.
- [ ]  **Mismatched “who-pays” checks in meta-tx paths cause DOS**
    - **Symptom:** Signature/relayed flows validate the **borrower’s** balance but burn/transfer from the **caller/payer**, or vice-versa. Valid meta-ops revert (e.g., close/adjust with signature) because the precondition is checked on the wrong address.
    - **Where to look (Lending/CDP + EIP-712 meta paths):**
        - Gateways like `BorrowerOperationsSignatures` / `WithSignature` / `restricted*` that pass both `_borrower` and `_caller` (or `_payer`).
        - Core ops invoked by those gateways: `restrictedCloseTrove`, `restrictedAdjustTrove`, `repay*`, `burn`, `liquidate*`, `redeem*`.
        - Balance/allowance guards and burns/transfers:
            - Guards: `_requireSufficient{Token}Balance(addr, amount)`, allowance checks.
            - Effects: `token.burn(from, …)`, `token.transferFrom(from, …)`, `pool.repayFrom(from, …)`.
    - **Red flags:**
        - Preconditions use `_borrower` while effects use `_caller`/`msg.sender` (or the reverse).
        - Comments say “payer is caller/relayer” but code checks borrower’s balance.
        - Meta-tx verification (`_verifySignature`) proves borrower intent, but funds are still expected from borrower without `transferFrom`.
        - In adjust/close flows: guard uses `vars.debt` of borrower, then `musd.burn(_caller, …)`; or repayment guard `_requireValidMUSDRepayment` on borrower with later `burnFrom(caller)`.
    - **Questions to ask:**
        - In signature flows, **who is intended to pay** (borrower vs caller vs third-party relayer)?
        - Do we ever burn/transfer from an address that we did **not** validate for balance/allowance?
        - If borrower funds should be used while caller executes, do we have `permit/transferFrom` in place and are we checking **borrower’s allowance to the contract**?
        - Are there unit tests where borrower has 0 balance but caller funds the operation (and vice versa)?
    - **Fix pattern:** Make the subject of checks match the subject of effects.
        
        ```solidity
        // decide the payer once
        address payer = _caller; // or `_borrower` if design requires borrower funds
        
        _requireSufficientMUSDBalance(payer, amount);
        _requireSufficientAllowance(payer, address(this), amount); // if using transferFrom
        
        // effects must use the same `payer`
        musdToken.burn(payer, amount);
        // or: musdToken.transferFrom(payer, address(this), amount); musdToken.burn(address(this), amount);
        
        ```
        
        - In mixed modes, separate entry points or an explicit `_payer` param; enforce `require(payer==_caller || payer==_borrower)` as per design.
        - Add tests: (1) borrower≠caller with caller paying; (2) caller≠borrower with borrower paying via `permit`+`transferFrom`; (3) negative tests ensuring mismatch reverts for the **right** reason.
- [ ]  **Genesis timelocks freeze interest at 0% (and “zero-rate troves” persist)**
    - **Symptom:** After launch, interest stays at 0 for ≥ governance delay (e.g., 7 days) because `propose → approve` is timelocked and no rate is set in `initialize`. Positions opened before activation keep the 0% snapshot unless they refinance.
    - **Where to look (rate/governance modules):**
        - Interest controller: `InterestRateManager` / `RateModel` functions like `proposeInterestRate`, `approveInterestRate`, `getCurrentRate`, `effectiveFrom`.
        - Governance/timelock: modifiers `onlyGovernance`, queues (`start…/finalize…`) and delays; role wiring (council/owner) in `PCV`, `TimelockController`, etc.
        - Initialization: proxy `initialize()` / constructors of `BorrowerOperations`, `TroveManager`, and the token; check if an initial non-zero rate is set.
        - Per-position rate snapshot paths: `openTrove`/`openPosition`, `updateSystemAndTroveInterest`, `refinance`, `applyPendingRewards`—does a new global rate auto-apply or require user action?
    - **Red flags:**
        - No parameter seed in `initialize()`; comments like “first rate must be proposed” with `minDelay > 0`.
        - Rate gating that requires council to be set via a separate delayed governance flow before rate approval.
        - Accrual uses “rate at open time” stored on the trove and only updates on explicit `refinance/adjust`, not on global rate change.
        - Ability to open positions while `currentRate == 0` or `rateActive == false`.
    - **Questions to ask:**
        - Can the initial rate (and governance roles) be set **atomically at deployment/initialization**?
        - Is there a **no-delay first-set** or “bootstrap window” for critical params?
        - Do positions **auto-migrate** to the new rate on next accrual tick, or only on `refinance`?
        - Is opening new debt **blocked** until a valid rate is active?
        - What happens if governance delay on role setup + rate approval overlaps launch—does the system accrue 0% meanwhile?
    - **Fix pattern:**
        - Seed non-zero rate (and council/roles) in `initialize()` or via a one-time, no-delay bootstrap setter guarded to only work when `rate==0` and before launch flag flips.
        - Alternatively, queue the first rate pre-launch and **activate before enabling opens**.
        - Make accrual **pull new global rate automatically** (e.g., rate with `effectiveFrom` timestamp applied during `updateInterest`), so pre-activation troves don’t remain 0% forever.
        - Gate user actions: `require(rateActive)` for opening/borrowing until the first rate is live.
        - Add tests: (1) launch day accrual > 0, (2) trove opened pre-activation accrues > 0 after activation **without** refinance, (3) governance delays don’t block initial activation.
- [ ]  **Pre-reward deltas used to move pools (stale debt snapshot)**
    - **Symptom:** Repay/adjust paths compute `principalAdjustment/interestAdjustment` **before** `applyPendingRewards()`. After rewards are applied, Trove debt changes again, but ActivePool is updated with the **old** deltas → pool principal/interest diverge from sum over troves.
    - **Where to look (Liquity-like CDP systems):**
        - `BorrowerOperations._adjustTrove / repayMUSD`: order of `updateSystemAndTroveInterest()`, `applyPendingRewards()`, delta calculation, and pool transfers.
        - `TroveManager.decreaseTroveDebt()/increase…`: does it recompute deltas post-reward and is that consistent with what pools are debited/credited by BO?
        - Pool movers: `_moveTokensAndCollateralFromAdjustment`, `ActivePool.{increase, decrease}{Principal,Interest}`.
    - **Red flags:**
        - Calculating `principalAdjustment, interestAdjustment` **prior** to `applyPendingRewards()` or any redistribution/interest sync.
        - Using cached `vars.*` deltas for pool transfers **after** calling functions that mutate trove debt/collateral (e.g., `applyPendingRewards`, `_updateTroveDebt`).
        - Two sources of truth (TroveManager vs ActivePool) without a single return value that dictates pool movements.
        - No invariant checks like: `sum(trove principal) == ActivePool.principal` after operations.
    - **Questions to ask:**
        - Do we always call `applyPendingRewards()` (and interest accrual) **before** computing any repay/burn deltas?
        - From which values are pool movements derived—the **same** deltas used to update the trove, or earlier snapshots?
        - Can a liquidation/redistribution occur in the same tx path between accrual and repayment logic?
        - Are we enforcing “interest-first, then principal” consistently in both trove math and pool updates?
    - **Fix pattern:** Apply rewards/interest first → compute deltas **once** → use those exact deltas for **both** trove updates and pool transfers (or have TroveManager return the applied principal/interest changes that the caller must mirror). Add post-op invariant/assert/tests that pool balances equal aggregation over troves.
- [ ]  **User-chosen recipients can funnel collateral into protocol pools (inflated TCR / bad accounting)**
    - **Symptom:** Close/withdraw paths allow an arbitrary `_recipient`; pools accept transfers from `ActivePool`, so users can send their trove collateral **into** `DefaultPool/CollSurplusPool/StabilityPool`, inflating on-chain balances used by `getEntireSystemColl()` while funds aren’t actually earmarked/usable.
    - **Where to look (Liquity-style systems):**
        - `BorrowerOperations._closeTrove/_withdrawCollateral/claimSurplus`: lines that call `ActivePool.sendCollateral(_recipient, coll)`.
        - Signature/meta-tx entrypoints (e.g., `closeTroveWithSignature`) that pass caller-supplied `_recipient`.
        - Pool contracts’ deposit guards: `receive()/fallback`, `sendCollateral` targets, and `onlyX` modifiers that only check **caller** (ActivePool) but not **recipient**.
        - TCR sources: `getEntireSystemColl()`—does it read **raw balances** on pools or tracked accounting variables?
    - **Red flags:**
        - No `require(!_isSystemAddress(_recipient))` before sending collateral.
        - Pools count collateral via `address(this).balance`/`token.balanceOf(this)` instead of internal “accounted” totals.
        - Pools allow inbound collateral from `ActivePool` without asserting a matching accounting update.
        - Upgrade/ops registries exist but are not consulted for “forbidden recipients”.
    - **Questions to ask next time:**
        - Can a user direct collateral to **any** address during close/withdraw/surplus claims? Is there a denylist of protocol contracts?
        - Do pool contracts differentiate between **accounted collateral** and **incidental transfers**?
        - Does TCR/health math sum internal variables or raw balances? What happens if someone dusts pools?
        - Is there a rescue path for mistakenly sent funds that doesn’t corrupt accounting?
    - **Fix pattern:**
        - Add `_requireNonSystemRecipient(_recipient)` using a central `AddressRegistry` (ActivePool/DefaultPool/CollSurplusPool/StabilityPool/TroveManager/MUSD/PCV/etc.).
        - In pools, reject unsolicited deposits or require a structured call (`depositFromActivePool(amount, reason)`) that updates internal accounting; make `receive()` revert.
        - Compute TCR from **tracked totals** (e.g., `accountedColl`) rather than raw balances; assert invariants after close/withdraw.
- [ ]  **JIT Stability Pool deposits capture liquidation rewards (sandwichable SP)**
    - **Symptom:** A user deposits right before a liquidation, gets full pro-rata collateral gains, then withdraws next block—profiting with near-zero time at risk.
    - **Where to look (Liquity-style SP / offsetting liquidations):**
        - `StabilityPool.provideToSP` / `withdrawFromSP` (or `deposit/withdraw`): when does a new stake become eligible for rewards?
        - Reward math: snapshot variables like `P`, `S`, `epoch/scale`, `depositorSnapshots` and the update path in `_updateRewardSumAndProduct` / `offset`.
        - Liquidation path: `TroveManager.liquidate` → `StabilityPool.offset` (does it use `totalDeposits` that already includes the just-in deposit?).
        - Any cooldown/maturity: search for `minDepositTime`, `activationBlock`, `cooldown`, `timelock`, or per-block gating.
        - Flash-loan vectors: is the debt token mintable/borrowable in one tx and depositable to SP?
    - **Red flags:**
        - Deposits immediately affect `totalDeposits` and snapshot on the same block as liquidation.
        - No deposit activation delay (e.g., `activationBlock = block.number + k`) or time-weighted stake.
        - `withdrawFromSP` allowed immediately after liquidation with no cooldown/penalty.
        - Rewards purely proportional to current stake (no aging/weight), and snapshots taken **after** including the new depositor.
        - No commit-reveal or batching; liquidations are public and callable by anyone.
    - **Questions to ask:**
        - When does a new SP deposit start earning liquidation rewards—immediately or next block/epoch?
        - Is there a cooldown or minimum holding period before withdrawal or before eligibility?
        - Are rewards time-weighted or age-weighted, or purely balance-weighted at liquidation instant?
        - Can deposits/withdrawals occur in the same block as `offset`/liquidation? In what order are state updates executed?
        - Can attackers source the stake capital atomically (mint/flash-loan) then unwind?
    - **Fix pattern:**
        - Add an **activation delay** (e.g., stake eligible at `block.number + k`) or **epoch gating** (new stakes join `nextEpoch`).
        - Introduce **time-weighted rewards** or a **cooldown/exit penalty** for withdrawals within N blocks of last liquidation.
        - Batch liquidations and snapshot **before** accepting new deposits for that batch.
        - Optionally require a **commit→reveal** deposit, or disallow deposit/withdraw in blocks that contain `offset`.
        - Document intended economics and add tests for deposit-right-before-liquidation and withdraw-right-after scenarios.
- [ ]  **Instant trove closure dodges bad-debt redistribution (pre-liquidation exit)**
    - **Symptom:** A borrower closes their trove right before a liquidation, so they’re excluded from the redistribution set. The entire uncovered debt is pushed onto remaining troves; attacker re-opens after.
    - **Where to look (Liquity-style CDP with SP + redistribution):**
        - `BorrowerOperations.closeTrove / restrictedCloseTrove` (and signature helpers): is close allowed any time?
        - `TroveManager.liquidate` → `_redistributeDebtAndColl` (or similar): when is the active set snapshotted?
        - Ordering around: `removeStake`, `sortedTroves.remove`, `applyPendingRewards`, and SP path `StabilityPool.offset` when `totalMUSDDeposits` is insufficient.
        - Guards: checks like `isRecoveryMode`, `_requireNoUnderCollateralizedTroves`, `_requireValidTroveAdjustment`—do they block closes while liquidatables exist?
    - **Red flags:**
        - No **cooldown/min lifetime** before a trove can be closed or re-opened.
        - Close is permitted **even while** some troves have `ICR < MCR` (or during Recovery Mode).
        - Redistribution set is computed **after** removing the closing trove (same block) with no epoch snapshot.
        - Re-open is allowed immediately with only a small/zero borrow fee (cheap sandwich cost).
    - **Questions to ask:**
        - Can a trove be closed in a block where another trove is liquidated? Is sequencing deterministic?
        - Do we snapshot the redistribution participants at **liquidation start** (epoch/round), or read live `TroveOwners`?
        - Are closes disabled when `existsLiquidatable == true` or when `TCR < CCR`?
        - Is there any **exit penalty** or **cooldown** that makes churn unprofitable?
    - **Fix pattern:**
        - Snapshot the **redistribution membership** at liquidation start (epoch-based accounting) and use that snapshot for debt/coll rebalancing.
        - Add a **close cooldown / min lifetime** (and/or re-open cooldown) and optionally an **exit penalty** that approximates expected bad-debt share.
        - Block `closeTrove` while there are liquidatable troves or during Recovery Mode; alternatively, process pending liquidations before allowing closes for the block.
        - Add tests: (1) close-then-liquidate same block, (2) liquidate-then-close ordering, (3) re-open churn profitability under fees/cooldowns.
- [ ]  **Batch liquidation defers redistribution → subsequent ICRs use stale debt**
    - **Symptom:** In `batchLiquidateTroves`, bad debt/collateral from earlier liquidations isn’t redistributed until the end, so later troves in the same batch are liquidated using pre-redistribution debt/ICR, skewing who absorbs bad debt and who gets collateral.
    - **Where to look (Liquity-style CDP with SP + redistribution):**
        - `TroveManager.batchLiquidateTroves` (or equivalent): look for a single final call to `_redistributeDebtAndColl` after looping all targets.
        - Helpers used by batch path: `_getTotalsFromBatchLiquidate`, `finalizeLiquidation`, accumulators like `totalPrincipalToRedistribute/…ToOffset/…Coll*`.
        - Per-trove liquidation path: single `liquidate()`—compare sequencing of `offset()` (SP) → `_redistributeDebtAndColl()` vs. batch path.
        - “Pending rewards” mechanism: per-share accumulators (`L_debt`, `L_coll`) and when they are advanced inside the batch loop.
        - Stability Pool interaction: `stabilityPool.offset(principal, interest, coll)` and how partial coverage is handled mid-batch.
    - **Red flags:**
        - Batch loop aggregates totals (`+=`) and calls `_redistributeDebtAndColl` **once** after the loop.
        - ICR checks for later troves computed from `Troves[addr]` without applying the redistribution produced by earlier batch items (no intermediate `applyPendingRewards` or accumulator bump).
        - Comments noting “gas efficiency” or “do at end” for redistribution/snapshots.
        - Snapshot/system updates (`_updateSystemSnapshots…`) only run once post-loop.
    - **Questions to ask:**
        - After liquidating item `i`, do later items `i+1..n` see the increased system debt via updated per-share accumulators, or are they using stale values?
        - How does partial SP coverage mid-batch propagate—are redistributions reflected before the next item’s ICR/debt math?
        - Are rounding/precision carries different between single vs batch paths, and are they **bounded**?
        - Could the order of addresses in the batch materially change outcomes due to deferred redistribution?
    - **Fix pattern:**
        - **Redistribute-as-you-go:** inside the batch loop, for each trove: compute SP offset; immediately call `_redistributeDebtAndColl` for the residual; advance per-share accumulators; (optionally) `applyPendingRewards` to the *next* victim before its ICR check.
        - Or maintain **virtual state** in the loop (temporary per-share deltas) and ensure all ICR/debt reads use base + virtual deltas so later items observe prior redistributions.
        - Add tests that compare single-liquidation sequence vs batch over identical sets (full/partial SP coverage, varying order), asserting close/bounded deltas only from rounding.
- [ ]  **Zero-cost, same-block borrows grief pool-level risk caps**
    - **Symptom:** Global safety checks (e.g., `MAX_BORROW_PERCENTAGE`) inside liquidity actions (`burn/withdraw/borrow`) revert after an attacker briefly inflates `…BORROW_L` with a fee-free borrow, then repays immediately (no instant fee), DoS’ing other users’ ops.
    - **Where to look (AMM/lending pair with pool-wide limits):**
        - Pair: `AmmalgamPair.burn/withdraw/borrow` pre-checks that call validation.
        - Risk module: `Validation.verifyMaxBorrowL/verifyMinLiquidity` and any `%` caps computed from `totalAssets[...]` / `rawTotalAssets(BORROW_*)`.
        - Interest model: borrow/repay paths—confirm whether **upfront** interest/fee is charged (or only time-based accrual).
        - “Lock”/reentrancy guards and block-scoped state: can borrow→victim op→repay happen across the same/next block without cost?
    - **Red flags:**
        - `borrow()` updates `BORROW_L` and `repay()` can fully unwind **same block** with **zero fee/penalty**.
        - Validation compares *instantaneous* `totalBorrowedLAssets` vs a cap (e.g., `if (depL * MAX_BORROW_PERCENTAGE < borrowedL * MAG2) revert`) with no hysteresis/TWAP.
        - Pool-level caps enforced in `burn/withdraw` but **not** in `borrow/repay` (asymmetric enforcement surface).
        - No dedicated `flashBorrow()` route; users can synthesize a free flash-loan via `borrow()+repay()`.
    - **Questions to ask:**
        - Is there a **minimum borrow duration** or **activation fee** that’s not refunded on immediate repay?
        - Are max-borrow/min-liquidity checks based on **block-start snapshots / TWAP**, or on mutable live totals?
        - Do validations consider **post-state** of an atomic sequence (borrow→use→repay), or each call in isolation?
        - Can a third party front-run a victim’s `burn/withdraw` with `borrow()`, and back-run with `repay()` at zero cost?
    - **Fix pattern:**
        - Charge a **non-refundable upfront fee** (or minimum interest for 1 block/epoch) on `borrow()`; waive only for a separate **`flashBorrow()`** that settles net-flat **within the same tx** and is capped/whitelisted.
        - Add **cooldown** (e.g., cannot fully repay to zero within N seconds/blocks without paying a penalty).
        - Compute caps against a **smoothed metric** (prev-block snapshot/TWAP of borrowed L) or add **hysteresis bands** to avoid edge-toggle griefing.
        - For multi-step atomic flows, validate against **pro forma post-state** (simulate with `newBorrowedLAssets` including matched repay) to prevent harmless sequences from tripping caps while still blocking external grief.
- [ ]  **Dust transfers reset saturation and erase penalties**
    - **Symptom:** An over-saturated borrower can zero-out their accrued penalty by transferring a tiny amount of **borrow** tokens to a solvent address. The transfer path accrues penalties for the *recipient* (`validate`) but then **updates and deletes** the sender’s saturation/account data (`update`) without minting the sender’s penalty first—wiping their penalty state.
    - **Where to look (pairs with saturation/penalty trees):**
        - Borrow/Deposit token hooks: ERC20 `_update(from, to, amount)` overrides that call into the pair.
            - Branching on token kind: `tokenType < FIRST_DEBT_TOKEN ? (validate=from, update=to) : (validate=to, update=from; isBorrow=true)`.
            - Guard flags like `transferPenaltyFromPairToBorrower`.
        - Pair: `validateOnUpdate(validate, update, isBorrow)`:
            - Calls `mintPenalties(validate, deltaPenaltyTimestamp)` and `validateSolvency(validate, isBorrow)`, then
            - `getInputParams(update, /*updatePrice=*/true)` → `saturationAndGeometricTWAPState.update(inputParams, update)`.
        - Saturation module:
            - `Saturation.update(...)` → `updateTreeGivenAccountTrancheAndSat(...)`.
            - `removeSatFromTranche(...)` → `removeSatFromTrancheStateUpdates(...)` → **`delete tree.accountData[account];`**
            (penalty fields like `penaltyInBorrowLShares` / `treePenaltyAtOnset…` live here).
        - Penalty accrual elsewhere: any `accrueSaturationPenaltiesAndInterest()` invoked by `mint/burn/deposit/borrow` but **not** on the transfer path.
    - **Red flags:**
        - `validateOnUpdate` **only** calls `mintPenalties(validate, …)`; there’s **no** `mintPenalties(update, …)` before mutating `update`’s saturation.
        - Penalty storage is embedded in `tree.accountData[account]` and that mapping is **deleted** during update/reinsert.
        - Solvency check runs on `validate` (recipient) rather than the penalty holder (`update`) on borrow-token transfers.
        - No minimum transfer / dust floor; 1 wei borrow-token transfer can trigger the path.
        - Comments or tests imply transfers should “accrue” but code path skips `accruePenalties()` for `update`.
    - **Questions to ask:**
        - On **borrow-token** transfers, which address is penalized vs. which is updated? Should both be accrued?
        - Can any code path **mutate** an account’s saturation or delete `accountData` **without** first minting its penalties?
        - Is penalty accounting stored only under `accountData[account]` (and thus vulnerable to deletion/reset)?
        - Do we require **minimum transfer amounts** or a restriction on transferring while `penaltyInBorrowLShares > 0`?
        - Should `validateSolvency` run on **both** parties or specifically the penalty holder when `isBorrow=true`?
    - **Fix pattern:**
        - **Accrue before mutate:** In `validateOnUpdate`, call `mintPenalties(update, deltaPenaltyTimestamp)` **before** any saturation update/reinsert for `update`.
            - Optionally, call a unified `accrueSaturationPenaltiesAndInterest(validate)` **and** `…(update)` up front.
        - **Order & atomicity:** Reorder so penalty accrual and state snapshots occur **prior** to `removeSatFromTranche` / any `delete accountData`.
        - **Stronger guards:**
            - Block transfers when `penaltyInBorrowLShares > 0` (or auto-settle penalties from transfered amount).
            - Add a **dust floor** on borrow-token transfers that trigger validation, or rate-limit saturation updates.
            - Run `validateSolvency` on the **penalty holder** (`update`) when `isBorrow=true`.
        - **State layout hardening:** Keep penalty counters in a **separate mapping** that is **not** cleared by saturation reinserts, or explicitly migrate them across updates.
        - **Tests to add:** “Dust-transfer penalty wipe” regression test: give account pending penalties, transfer 1 wei borrow token to a solvent address, assert penalties remain **unchanged** after transfer.
- [ ]  **Reserves updated but “missing/shortfall” counters aren’t (interest → price skew)**
    - **Symptom:** After interest accrual, reserves are incremented (and protocol fees minted) but “missingAssets/shortfall” aren’t updated in the same tick. Pricing/swap math that relies on both becomes inconsistent, enabling pre/post-update sandwiches that extract LP value.
    - **Where to look (AMM-lending hybrids / pair controllers):**
        - Interest accrual entrypoints: `accrueInterest*`, `sync()`, controller `update*()` (e.g., `updateTokenController`).
        - Reserve writers vs. debt/shortfall writers: `updateReserves(...)` vs. `updateMissingAssets()` / `updateShortfall()`.
        - State bundles/arrays: `allAssets`, `totalBorrow*`, `totalDeposit*`, `borrowAssets`, `reserves`, `missingAssets`.
        - Swap paths consuming state: `swap()`, price quotes, and any “depleted/liquidation” branch that references `missingAssets`.
        - Fee paths: `mintProtocolFees(...)` and any place interest for LPs is added to reserves.
    - **Red flags:**
        - Accrual sequence: `accrueInterest → updateReserves → mint fees` **without** a subsequent `updateMissingAssets()` in the same function.
        - `missingAssets` only touched during `repay()`/`borrow()` but not during interest accrual or `sync()`.
        - Pricing formulas that use `reserves ± missingAssets` while those variables can be updated in different transactions/blocks.
        - Ability to `sync()` (or any accrual updater) and then immediately `swap()` before shortfall is recomputed.
        - Comments/branches for “depleted state” but no atomic state refresh covering **both** reserves and shortfall.
    - **Questions to ask:**
        - Which storage variables change on interest accrual, and are **all** dependent quantities (reserves, borrows, shortfall/missing) updated atomically?
        - Can a user observe an accrual step (via `sync()`/oracle tick) and sandwich swaps before `missingAssets` is refreshed?
        - Do protocol fee mints or “interest for LP” credits increase tradable reserves without adjusting shortfall?
        - Does `swap()`/quote call an internal “ensure up-to-date” hook (accrue + recompute shortfall) or rely on stale state?
        - What invariant ties `reserves`, `totalBorrow*`, `totalDeposit*`, and `missingAssets`? Is it asserted after accrual?
    - **Fix pattern:**
        - Make accrual **atomic**: after `accrueInterestAndUpdateReserves*`, immediately `updateMissingAssets()` **before** any external effects or swaps; ideally encapsulate in one internal `_accrueAndReconcile()` used by `sync()`, `swap()`, `repay()`, `borrow()`.
        - Reorder so derived values (`missingAssets/shortfall`) are recomputed **after** reserves/borrows/fees are updated in the same transaction.
        - Harden pricing to read from a single source of truth (e.g., recompute from totals) or gate `swap()` behind an accrual+reconcile step.
        - Add invariants/tests: post-accrual `missingAssets == f(totalBorrows, totalDeposits, reserves, fees)`; fuzz around depleted states; test sandwich (swap → accrue → swap) and ensure no free gain.
- [ ]  **Non-swap state changes can flip “depleted mode” (bonus credits without a trade)**
    - **Symptom:** Interest accrual / protocol-fee minting moves the pool into a “depleted” branch (e.g., `reserves * 95 < missing * 100`) even though no swap occurred, letting one-sided deposits mint extra credits at LPs’ expense.
    - **Where to look (AMM-lending hybrids / controllers):**
        - Accrual/housekeeping paths: `updateTokenController`, `sync()`, `accrueInterest*`, `repay()`, fee minting (`mintProtocolFees`).
        - Balance writers: `updateReserves(...)` vs. `updateMissingAssets()`/`missing`/`shortfall` updates.
        - Depleted/imbalanced checks used by `deposit()`/`swap()` curves (the inequality that triggers bonus crediting).
    - **Red flags:**
        - Fees/interest added to **both** `reserves` and `missing` in the same tick (or only to one) such that `(reserves + fee)*α < (missing + fee)*β` can newly hold.
        - Depleted check is evaluated on post-accrual state, and deposit/credit logic doesn’t distinguish *how* depletion was reached (swap vs. accounting event).
        - Bonus credit path in `deposit()` is callable right after `sync()/accrue` without a swap; no guard that depletion must be swap-induced.
        - `missingAssets` updated only in `repay()/borrow()` while reserves/protocol fees move elsewhere.
    - **Questions to ask:**
        - Can any **non-swap** function change `reserves` or `missing` so the depleted predicate flips?
        - When depletion is caused by **fees/interest**, who funds the extra credits—LPs or borrowers? Is that intended?
        - Does `deposit()` verify depletion was triggered by a price-forming trade (or apply a grace window) before awarding bonuses?
        - Are accrual and missing/shortfall reconciled **atomically** in the same transaction?
    - **Fix pattern:**
        - Decouple accounting from pricing: exclude protocol-fee/interest deltas from the depleted predicate or compute it on trade-effective balances.
        - Reconcile state atomically: after accrual, recompute `missing` and tag depletion-cause; gate bonus credits to swap-induced depletion only.
        - Alternatively, stop adding fees symmetrically to variables used by the depleted check, or route fees to a separate bucket not counted in `missing`.
        - Add invariants/tests: accrual-only transitions must not enable bonus credits; fuzz `sync/repay → deposit` for free-gain exploits.
- [ ]  **Borrow-capacity converts gross singles to LP, ignoring deposited singles**
    - **Symptom:** Borrows/burns of liquidity revert with “max borrow reached” despite ample liquidity (e.g., `AmmalgamMaxBorrowReached()`), especially when single-sided deposits offset opposite-asset borrowing.
    - **Where to look (AMM-lending hybrids):**
        - Max-borrow guards like `Validation.verifyMaxBorrowL(...)`.
        - Converters that translate X/Y shortfall to L via reserves: `Convert.mulDiv(...)`.
        - State used in the check: `totalAssets[DEPOSIT_*]`, `totalAssets[BORROW_*]`, `newActiveLiquidityAssets = DEPOSIT_L - BORROW_L`.
    - **Red flags:**
        - Using **gross** `BORROW_X`/`BORROW_Y` in L-conversion when the condition is `BORROW_* > DEPOSIT_*` instead of the **net** shortfall `BORROW_* - DEPOSIT_*`.
        - No reference to `missingAssets`/shortfall in capacity math.
        - Inequality of the form `DEPOSIT_L * MAX_BORROW_PERCENTAGE < totalBorrowedLAssets * MAG2` where `totalBorrowedLAssets` is inflated by gross cross-asset borrow.
        - `unchecked` arithmetic around these conversions, risking silent over/underflow of the guard inputs.
    - **Questions to ask:**
        - Are we converting **net imbalance** (`max(0, borrow - deposit)`) or **gross borrowed** amounts when mapping X/Y pressure to L consumption?
        - Does the guard double-count the **new** L being requested (pre/post state mix)?
        - Are `reserves` taken pre-update and representative for the conversion denominator?
        - How do single-sided deposits reduce effective shortfall in the opposite asset within the capacity formula?
    - **Fix pattern:** Compute shortfalls as `shortX = borrowX.saturatingSub(depositX)` and `shortY = borrowY.saturatingSub(depositY)`; convert **only** these to L: `… += Convert.mulDiv(shortX, newActiveLiquidityAssets, reserveX, …)` (and similarly for `shortY`). Prefer using canonical `missingAssets` if present. Add tests where `borrow ≈ deposit` to ensure capacity is not prematurely exhausted.
- [ ]  **Inverse-liquidity growth underflow blows up fee growth (Q128 − 1/G)**
    - **Symptom:** Interest/fee accrual reverts or returns gigantic numbers after a tiny deposit/sync; subsequent txs brick. Pattern: `oneOverGQ128 = (activeL * Q128) / adjustedActiveLiquidity` exceeds `Q128`, so `Q128 - oneOverGQ128` underflows, exploding `swapFeeGrowth`.
    - **Where to look (AMM-lending with time/fee growth):**
        - Interest math: `Interest.accrueInterestWithAssets` around `oneOverGQ128` and `swapFeeGrowth` (e.g., L236).
        - Adjusted liquidity derivation: `adjustedActiveLiquidity = lastActiveL * currentReserveLiq / lastReserveLiq` (controller) and any rounding-mode flags.
        - Reserve/reference updates that can shrink reference liquidity: pair `deposit()` / `sync()` paths (e.g., `updateReservesAndReference(_rx, _ry, _rx - Δx, _ry - Δy)`).
    - **Red flags:**
        - Subtractions like `Q128 - oneOverGQ128` inside `unchecked` blocks or before explicit range clamps.
        - Using **rounded-down** division when computing `adjustedActiveLiquidity`, allowing `adjustedActiveLiquidity < activeLiquidity`.
        - No invariant/require that `adjustedActiveLiquidity ≥ activeLiquidity` or that `oneOverGQ128 ≤ Q128`.
        - Fee growth formula of the form `mulDiv(borrowL, Q128 - oneOverGQ128, Q128, ...)` without saturating subtraction.
    - **Questions to ask:**
        - Can any reserve/reference update or additional-credit path reduce `adjustedActiveLiquidity` below the live `activeLiquidity` (especially due to rounding down)?
        - Are growth factors (G or 1/G) explicitly bounded to `[0, Q128]` before use?
        - Are rounding modes chosen to preserve monotonicity of liquidity references (should this div round up)?
        - Do we have fuzz tests where `adjustedActiveLiquidity ≈ activeLiquidity` (off-by-one) and where tiny deposits flip the inequality?
    - **Fix pattern:**
        - **Clamp before subtract:** `oneOverGQ128 = min(oneOverGQ128, Q128); diff = Q128 - oneOverGQ128;` use `saturatingSub`.
        - **Enforce invariant:** `require(adjustedActiveLiquidity >= activeLiquidity)` or compute `adjustedActiveLiquidity = max(adjustedActiveLiquidity, activeLiquidity)`.
        - **Rounding:** compute `adjustedActiveLiquidity` with **round-up** (or 512-bit mulDiv) to avoid underestimation.
        - **Type/arith safety:** avoid `unchecked`; use 512-bit mulDiv or signed intermediates if computing deltas; add revert on `adjustedActiveLiquidity == 0`.
        - **Tests:** add adversarial tests where a minimal deposit/sync or reference-liquidity drop would previously make `oneOverGQ128 > Q128`; assert accrual does not revert and fee growth stays bounded.
- [ ]  **Block-gated accrual lets first burner skim same-block swap fees**
    - **Symptom:** First `burn()`/`mint()` in a block gets outsized outputs because fee/penalty rebasing runs only when `deltaLendingTimestamp > 0`; later callers in the **same block** see worse terms for identical shares.
    - **Where to look (AMM-lending hybrids):**
        - Pair: `accrueSaturationPenaltiesAndInterest` guard `if (deltaLendingTimestamp > 0) updateTokenController(...)`.
        - Controller: `updateTokenController(...)` paths that rebase deposit/borrow L against reserve sqrt-liquidity.
        - Interest math: `accrueInterestWithAssets` fee growth (`G`, `oneOverGQ128`, `swapFeeGrowth`) that should apply on **every** state change, not only time ticks.
        - Entry points that change reserves without time advancing: `swap`, `mint/deposit`, `burn/withdraw`, `sync`.
    - **Red flags:**
        - Comments or intent like “should run every call” behind a timestamp gate.
        - `deltaUpdateTimestamp` computed but ignored when `deltaLendingTimestamp == 0`.
        - Two identical burns in one block return different amounts; first caller advantage in tests.
        - Accrual only in some code paths (e.g., before `burn` but not before `mint`).
    - **Questions to ask:**
        - Do swaps/penalties increase reserves within a block without triggering a rebase of `depositL/borrowL`?
        - Is there a single “applyAccrual()” pre-hook executed at the top of **all** mutating functions?
        - Are fee growth indices tied solely to time, or also to reserve changes (additional credit, penalties)?
        - Is there a test asserting equal output for equal shares burned back-to-back in a quiet block?
    - **Fix pattern:**
        - Split accrual into two branches and invoke **on every state change**:
            - **Time-based interest** (gated by `deltaLendingTimestamp > 0`).
            - **State-based fee/penalty rebase** (runs unconditionally; can use `deltaUpdateTimestamp` or simply current reserves).
        - Centralize in a pre-hook called by `swap/mint/burn/deposit/withdraw/sync` before any share or amount math.
        - Add invariants/tests: two burns of the same share in the same block (no intervening swaps) must output the same; a burn after a same-block swap should not depend on caller order.
- [ ]  **Stale adjusted-liquidity misallocates lending fees to the first burner**
    - **Symptom:** The first `burn()` after interest accrues pulls **extra lending fee** compared to identical burns that follow. Minters are short-changed; early burners skim yield even with equal shares.
    - **Where to look (AMM + lending hybrids):**
        - Interest math: `Interest.accrueInterestWithAssets(...)`
            - `oneOverGQ128 = (activeLiquidity₀ / adjustedActiveLiquidity₁)` using `params.adjustedActiveLiquidity`.
            - `swapFeeGrowth` and the LP interest portions (`interestXPortionForLP`, `interestYPortionForLP`) calculation and **application order**.
        - Controller: `updateTokenController(...)`—how `adjustedActiveLiquidity` (ALA₁) is derived from last active liquidity and **current** reserve liquidity.
        - Pair orchestrator: `accrueSaturationPenaltiesAndInterest(...)` pre-hooks before `burn/mint/swap`, and whether accrual runs **before** share math.
    - **Red flags:**
        - `adjustedActiveLiquidity` computed **before** adding the current-period LP interest to reserves, then reused to compute `swapFeeGrowth`.
        - Reserve/ALA updates applied **after** growth calc in the same call.
        - Two back-to-back equal burns (no intervening swaps) return different amounts; first one larger.
        - Comments like “avoid extra payment by borrowers” without a mirrored, atomic update of fee growth indices.
    - **Questions to ask:**
        - Does `ALA₁` reflect **post-fee** reserves for the *same* accrual step, or is it stale (pre-fee)?
        - In what exact order are: (1) borrower interest split → (2) reserves updated → (3) `G`/growth computed?
        - Are growth indices and reserve updates performed **atomically** inside one accrual, or split across calls/blocks?
        - Do tests assert equal outputs for identical burns when only time (not swaps) advances?
    - **Fix pattern:**
        - Make accrual **single-sourced and ordered**:
            1. Compute borrower interest & LP portions in memory.
            2. **Apply** those portions to provisional reserves.
            3. **Recompute `adjustedActiveLiquidity`** (ALA₁) from the provisional, post-fee reserves.
            4. Compute `G`, `oneOverGQ128`, and `swapFeeGrowth` using this updated ALA₁.
            5. Commit state, then perform `burn/mint` math.
        - Alternatively, compute `swapFeeGrowth` from an “effective ALA₁” that already includes LP interest for the period.
        - Add invariants/tests:
            - Two equal burns in the same state (no intervening swaps) must return equal outputs within 1 wei.
            - A burn immediately before vs. after accrual reordering must not change outputs for the same share.
- [ ]  **Boundary-flip “additional credit” mints a windfall on tiny deposits**
    - **Symptom:** When an asset is “depleted”, depositing a **dust** amount flips the pool to “undepleted” and grants the depositor **full gap credit** (`reserveAssets - _missingAssets`) instead of credit proportional to what they added—yielding instant profit (e.g., +3e18 X with a 0.01 wei Y deposit).
    - **Where to look (AMM + lending hybrids):**
        - `AmmalgamPair.sol`:
            - `depletionReserveAdjustmentWhenAssetIsAdded(amountAssets, reserveAssets, _missingAssets)` — additional-credit math.
            - Deposit flow: `deposit()` → calls the above, then mints deposit credits.
            - Threshold logic for “depleted”: checks like `reserveAssets * BUFFER < _missingAssets * MAG2`.
            - Any helpers using `calculateBalanceAfterFees` that change slope/scale between depleted/undepleted.
        - Constants: `BUFFER`, `MAG2`, `INVERSE_BUFFER` (the %s used in depleted/undepleted comparisons).
        - State toggles: paths that test depletion **before** vs **after** applying `amountAssets`.
    - **Red flags:**
        - Branch that does `adjustReserves_ = reserveAssets - _missingAssets` once a **post-deposit** inequality would become non-depleted, without clamping by `amountAssets`.
        - Depletion check computed **only on pre-state**, but credit uses **post-state** effect (or vice-versa), creating discontinuity at the boundary.
        - Additional credit not continuous (C⁰) at the threshold; a 1-wei deposit yields huge `adjustReserves_`.
        - Ability to force depleted state with a swap, then flip back with a dust deposit within the same tx.
    - **Questions to ask next time:**
        - Is the *extra* credit capped by `amountAssets` (and never exceeds the precise boundary delta needed to reach “undepleted”)?
        - Are we computing the boundary condition using **post-deposit** values (i.e., with `amountAssets` applied) to avoid pre/post mismatch?
        - Do the formulas normalize both sides to the same scale (e.g., divide by `MAG2`/`BUFFER` consistently) to prevent slope jumps?
        - Are there invariant tests near the threshold (±1 wei around `reserve*BUFFER == missing*MAG2`) proving no depositor can extract more than fees?
    - **Fix pattern:**
        - Replace the “grant the whole gap” logic with **boundary-delta, clamped by deposit**:
            - Derive the minimal `Δ` needed to reach the equality boundary of undepleted (solve the inequality `reserve' * BUFFER >= (missing' ) * MAG2` using **post-deposit** terms), then:
                - `adjustReserves_ = min(amountAssets, max(0, Δ))`
            - In your codebase’s terms, this aligns with using a boundary delta like:
            `Δ ≈ (_missingAssets * MAG2 / INVERSE_BUFFER) - (reserveAssets * BUFFER / INVERSE_BUFFER)` (scaled to your constants), rather than `reserveAssets - _missingAssets`.
        - Make the function **continuous and monotone** at the boundary:
            - When `amountAssets` is tiny, `adjustReserves_` must also be tiny (no step function).
            - Never mint “replenishment” credit exceeding the **actual** replenishment contributed.
- [ ]  **Rate-vs-amount confusion doubles penalties/interest**
    - **Symptom:** A helper that already returns an **amount of interest/penalty** is treated as if it returned a **rate/factor**, and the caller multiplies by the base again (e.g., `penalty = base * computeInterest(...) / WAD`). Results: penalties/interest are overcharged (sometimes by \~base factor), especially when using “compounded rate per second” naming.
    - **Where to look (Solidity DeFi codebases):**
        - Penalty/fee accrual paths: `accruePenalties()`, `accrueFees()`, `accrueInterest()`, “saturation” or “utilization” modules.
        - Math libs with names like `computeInterest*`, `compounded*`, `wTaylor*`, `expWad*`, `ratePerSecond*`, `perSecondInterest*`.
        - Call sites that pass both a **base amount** (e.g., `borrowedAssets`, `totalSat...`) into the helper **and** then multiply that same base again afterward.
        - Unit-space conversions: `WAD/Q128/Q72` math immediately before or after interest helpers; look for `Convert.mulDiv(base, helperResult, WAD)` right after the helper already used `base`.
    - **Red flags:**
        - Variables named like `compoundedPenaltyRatePerSecond` hold **asset amounts** (not rates).
        - Helpers accept `(duration, baseAmount, ... , rate)` and return a number proportional to `baseAmount`, but the caller still does `baseAmount * helper / WAD`.
        - Comments say “rate” while code passes/returns “assets”.
        - The same quantity (e.g., `totalSatLAssetsInPenalty` or `borrowedAssets`) appears both **inside** the helper call **and** in a subsequent multiplication.
        - Duplicate application of `duration` (inside helper and again outside).
    - **Questions to ask next time:**
        - Does this helper return a **factor/rate** (unitless, WAD-scaled) or an **amount in assets**? Where is that documented?
        - Is the **base amount** (debt/penalized assets) multiplied by the helper **inside** the helper already?
        - Are variable names (`...Rate`, `...Amount`) consistent with what the function actually returns?
        - Do units line up? After the helper returns, should the next operation be **addition** to state, or another **multiplication**?
        - Is `duration` applied exactly once along the path from rate → amount?
    - **Fix pattern:**
        - If the helper returns **amount**, **use it directly** as the penalty/interest delta; do **not** multiply by the base again.
- [ ]  **Wrong-pool share→asset conversion dilutes positions**
    - **Symptom:** Penalties (or fees) computed in **Borrow L shares** are converted to **assets** using the **Deposit L** totals/price-per-share (PPS). Then the protocol mints `Borrow L` shares **and** credits `Deposit L` assets, skewing both PPS and user proportions. Normal (non-penalized) accounts see their position diluted or inflated.
    - **Where to look (multi-bucket share/asset systems):**
        - Controllers that touch multiple ledgers/buckets (e.g., `TokenController::mintPenalties`, fee/penalty mints, liquidations).
        - Any `Convert.toAssets(shares, totalAssets, totalShares, …)` / `toShares(...)` helpers—verify **which bucket’s** `totalAssets/totalShares` are passed.
        - Mint/burn paths like `mintId(bucket, ..., assets, shares)` where `assets` and `shares` must belong to the **same bucket**.
        - Arrays or enums like `DEPOSIT_L` / `BORROW_L` (or `A`, `B`, `C`) used as indices into `allAssets[]`, `allShares[]`—scan for crossed indices.
    - **Red flags:**
        - Converting `totalPenaltyInBorrowLShares` with `(allAssets[DEPOSIT_L], allShares[DEPOSIT_L])` instead of `(allAssets[BORROW_L], allShares[BORROW_L])`.
        - Subsequent call like `mintId(BORROW_L, ..., totalPenaltyInDepositLAssets, totalPenaltyInBorrowLShares)`—**assets from Deposit L, shares from Borrow L**.
        - State updates that credit `allAssets[DEPOSIT_L] += ...` for a penalty that conceptually belongs to the borrow side.
        - PPS drift for non-targeted users after a penalty/fee event; unchanged users’ required repayment or share fraction changes.
    - **Questions to ask next time:**
        - In which **domain** (bucket) is the penalty/fee denominated—Borrow L or Deposit L? Are both **amount** and **shares** kept in that same domain?
        - Do the `toAssets/toShares` calls use the **matching bucket’s** `totalAssets/totalShares`?
        - When we mint `(assets, shares)`, are both values from the **same** bucket? If not, why is the cross-bucket transfer economically sound?
        - After a penalty, do invariant checks show unaffected users’ PPS and share ratios unchanged (unless intended)?
    - **Fix pattern:**
        - Convert using the **correct bucket totals**:
        `assetsForPenalty = Convert.toAssets(penaltyBorrowLShares, allAssets[BORROW_L], allShares[BORROW_L], ROUNDING_...)`.
        - Mint/burn within a **single** bucket per event, or explicitly model a cross-bucket transfer (and update both sides symmetrically with clear economics).
- [ ]  **TWAP checkpointing without interval guard yields zero/ stale tick**
    - **Symptom:** Functions that compute an average tick as
    `avg = (currentCumulative - previousCumulative) / elapsed` may return **0** (or a stale default) when the **cumulative sum hasn’t advanced** because the *TWAP buffer interval* wasn’t met—yet the code still “updates” lending/interest state. Downstream math (interest accrual, reserve updates, fee splits) then runs on an incorrect tick.
    - **Where to look (AMMs/oracles/TWAP buffers):**
        - Observation rings and mid/long-term TWAP modules (e.g., `Observations`, `recordObservation`, `get*StateTick*`, `calculateTickAverage`).
        - Call sites that **read and checkpoint** in one go (e.g., `getLendingStateTickAndCheckpoint()`style helpers used inside controller updates).
        - Any place that compares `timeElapsedSinceUpdate` vs. `intervalConfig` and then computes averages for **another** elapsed window (e.g., *mid-term window* vs. *lending window*)—mismatched windows are the root cause.
    - **Red flags:**
        - `timeElapsedSinceLendingUpdate > 0` but `timeElapsedSinceUpdate < midTermIntervalConfig` ⇒ `currentCumulative == previousCumulative` and **avg becomes 0**.
        - Code paths that set `tickAvailable=false` or pass `newTick=0`, yet still produce an “average”.
        - Returning `lastLendingStateTick` **only when elapsed == 0**, not when elapsed `< interval`.
        - Low block-time chains (e.g., \~2s) with a larger mid-term interval (e.g., 8s): frequent sub-interval reads.
        - Integer division truncation on `(curr - prev) / elapsed` producing zero for small numerators.
    - **Questions to ask next time:**
        - Does the cumulative buffer **advance** before we compute `(curr - prev)`? What guarantees that `curr != prev`?
        - Are we **mixing windows** (mid-term buffer interval vs. lending elapsed)? If so, what if one advanced and the other didn’t?
        - What do we return when elapsed `< interval`? Do we **carry forward** the last valid tick, or compute a bogus average?
        - Are there tests on fast block chains asserting: (a) no zero/garbage average; (b) state unchanged if TWAP not advanced?
        - If we must produce an estimate sub-interval, do we fall back to the **last tick** or a **high-res cumulative**?
    - **Fix pattern:**
        - **Guard** average computation behind the **same interval** that advances the cumulative: if `elapsed < intervalConfig`, **reuse `lastLendingStateTick`** (don’t recompute).
- [ ]  **Piecewise swap-fee cliff enables zero-fee “overshoot” trades**
    - **Symptom:** When fees are computed from a *reference reserve* vs *current reserve* with a piecewise rule (min-fee while moving toward reference; quadratic fee once past it), a trade that lands exactly at—or 1 wei past—the reference reserve can be charged **0 bps**. If params are accidentally reversed (current ↔ reference), traders can pay zero fees while increasing imbalance.
    - **Where to look (AMMs with reference-vs-current, quadratic/min fee blends):**
        - Fee math libs: `QuadraticSwapFees.calculateSwapFee*`, especially branches like
        “if input moves past reference, charge quadratic on (input - deltaToReference)”.
        - Swap paths that pass reserves into the fee lib: helpers like `computeExpectedSwapOutAmount(...)`, `swap(...)` → `calculateSwapFeeBipsQ64(input, referenceReserve, currentReserve)`.
        - State maintaining both **reference** and **current** reserves (and any “depleted” logic) and call sites that may **swap parameter order**.
        - Points where min-fee and quadratic-fee are **mutually exclusive** instead of **blended** across segments.
    - **Red flags:**
        - Branch like:
        `if (input + currentReserve <= referenceReserve) fee = MIN_BIPS; else fee = f_quadratic(pastBy);`
        (no continuity term or floor at the boundary).
        - No **fee lower bound** (`max(fee, MIN_FEE)`), or the bound checked **before** the piecewise substitution in a way that can be bypassed.
        - Using `pastBy = input + currentReserve - referenceReserve` without guarding ±1 wei “just-past” cases; integer math making `pastBy == 0` produce `fee == 0`.
        - Callers that pass `(current, reference)` sometimes and `(reference, current)` elsewhere.
        - Tests only for large inputs; missing **boundary tests** at `input == reference - current` and `±1 wei`.
    - **Questions to ask next time:**
        - At the **exact crossing** and **±1 wei** around it, do we still charge at least the min fee? Are left/right fees **continuous and ≥ MIN\_FEE**?
        - If a trade **spans two regions** (up to reference, then beyond), do we **sum** fees for both segments or only apply the post-crossing rule?
        - Are **parameter orders** for `(referenceReserve, currentReserve)` consistent across all callers? Any helper that silently flips them?
        - Do tests assert **monotonicity** of fee vs. imbalance and **no zero-fee window** near the reference?
    - **Fix pattern:**
        - **Blend segment fees:**
        Let `toRef = max(0, reference - current)`. For `input`:
            - Charge `fee = MIN_BIPS` on `min(input, toRef)`, **plus**
            - Charge `fee += f_quadratic(max(0, input - toRef))`.
            Apply `fee = max(fee, MIN_BIPS)` *per trade* (or per-segment min if design requires).
        - Add a **global floor**: `feeBips = max(feeBips, MIN_FEE_BIPS_Q64)` right before return.
        - Normalize and **validate param order** at entry (e.g., assert/reference tagged struct).
- [ ]  **Floor-rounding in %/bips checks lets attackers take “just-under” premiums**
    - **Symptom:** Premium/fee caps are enforced by comparing a **bips** value computed with integer division (floor) against a `maxPremiumBips`. Attackers choose amounts so the floored `premiumInBips` stays **zero or under the cap**, yet they still seize value (e.g., 1 bip − 1 wei), repeatable to drain users.
    - **Where to look (liquidations/auctions/penalties):**
        - Liquidation libraries and ops that compute “requested premium” or “seized share” as a **ratio in bips/percent**, e.g. `calcSoftPremiumBips(...)`.
        - Math helpers like `mulDiv(x, y, z, /*rounding=*/false)` or `10_000 / denom` used before `if (maxPremiumBips < premiumBips) revert`.
        - Any place that **sums** per-asset bips (L/X/Y) after individually flooring each piece.
        - Branches that gate “soft”/partial liquidations or partial seizures by a bips limit.
    - **Red flags:**
        - Floor division for bips: `premium = amountToTake * BIPS / userDeposit` (or `mulDiv(..., false)`).
        - Inequality that permits equality or a floored zero: `if (maxPremiumBips < premiumBips) revert` (instead of checking the **amounts**).
        - Per-asset premiums are **floored individually** and then **added**, compounding underestimation.
        - Lack of “round-up” or “ceiling” option in math utils; comments like `// round down`.
        - Tests missing edge cases: **1 bip**, **1 bip − 1 wei**, and **boundary equality** scenarios.
    - **Questions to ask next time:**
        - Do we enforce the cap on the **actual seized amounts** (with exact arithmetic) or on a **floored bips approximation**?
        - At 1 bip and (1 bip − 1 wei), does the operation still pass? Should it?
        - Are we summing floored per-asset bips (bias ↓) rather than computing one ratio over the **total**?
        - Do we ever compare `premiumBips` with `<` where `<=` (or a ceiling on the left) is required to be conservative?
        - Can the attacker **iterate** the “just-under” premium many times (gas-bounded per tx but unbounded over time)?
    - **Fix pattern:**
        - **Ceil the ratio** when converting to bips for safety-critical checks:
        `premiumBips = mulDiv(amountToSeize, BIPS, deposit, /*roundUp=*/true);`
        - Or better, **avoid bips rounding entirely** in the guard: enforce on amounts:
        `require(amountToSeize * BIPS <= deposit * maxPremiumBips);` (use 512-bit `mulDiv` to prevent overflow).
        - If summing across assets, compute premium on the **aggregate** (or **ceil each term** before summing).
        - Make the revert condition conservative: `require(premiumBips <= maxPremiumBips)` only if `premiumBips` is a **ceiling**; otherwise compare amounts.
- [ ]  **Confused-deputy: debt/penalties keyed to `to` while solvency is checked on `msg.sender`**
    - **Symptom:** Borrow/borrow-liquidity paths mint debt and accrue penalties to the **recipient** (`to`) but run solvency/limits against the **caller** (`msg.sender`). A solvent caller can push debt onto an unsuspecting recipient (or vice-versa), griefing or liquidating them.
    - **Where to look (loans/flash-borrows/liquidity borrows):**
        - Entry points with a **recipient parameter**: `borrow(address to, ...)`, `borrowLiquidity(address to, ...)`, `flashBorrow(address to, ...)`, `redeem(to, ...)`.
        - Helpers called inside borrow paths: accrual/penalty updaters (`accrue...Interest/Penalties(...)`), share mints (`_helper(...)` → `mintId`/`_mintShares`), solvency/health checks (`validateSolvency`, `checkBorrowCapacity`), and callbacks (`ammalgamBorrowCallV1`) that use `to`.
        - Token controller / accounting that maps balances or debt **by address**; verify the same principal address is threaded end-to-end.
    - **Red flags:**
        - Any call that passes `to` (or `recipient`) into accounting/penalty/interest functions, while checks use `msg.sender` (or vice-versa).
        - Mixed usage of identities within one function: e.g., `accrue...(..., to)` → `mint debt to to` → `validateSolvency(msg.sender)`.
        - Minting debt shares to `to`, transferring assets to `to`, and then invoking a **callback on `to`**, enabling reentrancy before solvency is validated on the true debtor.
        - Lack of explicit “on-behalf-of” authorization (no approval/permit/delegate for assigning debt to another account).
    - **Questions to ask next time:**
        - Who is the **debtor of record** for this borrow? Is that same address used for: interest accrual, penalties, debt share minting, health checks, and limits?
        - If debt is allowed **on behalf of** another address, where is the **explicit consent** (permit/delegate) enforced?
        - Could a caller make a victim **recipient** of debt or penalties by choosing their address as `to`?
        - Do any callbacks or external calls happen **before** solvency checks finalize, letting state change mid-check?
    - **Fix pattern:**
        - **Unify the actor.** Introduce a `_borrower` variable and pass it consistently to accrual, penalty, share-mint, and solvency functions; use `to` **only for asset transfer**.
        - If “on-behalf-of” is a feature: require an explicit delegate approval/permit (`onBehalfOf`) and run **all** checks and mints against that approved borrower; otherwise enforce `require(to == msg.sender)`.
        - Move external callbacks **after** accounting & solvency updates, or perform a second post-callback solvency check; apply reentrancy guards.
- [ ]  **Dust-ratio rounding → zero price/tick → repay/liquidate DoS**
    - **Symptom:** Partial repayments or soft/hard liquidations intermittently revert (e.g., `PriceOutOfBounds`) when a math helper converts a very small **netDebt** vs. very large **active liquidity** into a fixed-point price/tick. A `mulDiv(..., roundDown=false)` chain rounds the intermediate to **0** (e.g., `inputQ64=0`), later squared or fed into `TickMath.getTickAtPrice/Ratio`, bricking the call until state shifts.
    - **Where to look (fixed-point + tranche/penalty math):**
        - Tranche/threshold calculators used by repay/liquidate paths: e.g., `calcTrancheAtStartOfLiquidation(...)`, saturation/penalty helpers.
        - Any **Q64/Q96/Q128** conversions using `mulDiv` with **round down**, especially when numerator is `netDebt * buffer * constants` and denominator is `activeLiquidity * threshold`.
        - Calls into Uniswap-style tick/price helpers: `TickMath.getTickAtPrice`, `getTickAtSqrtRatio`, or custom log/price mapping that **requires strictly positive** input.
        - Repay/liquidate entrypoints that:
            - Use **shares** (`balanceOf`) instead of **assets** (`previewWithdraw/maxWithdraw`) to size the repay—creating tiny residuals (“dust”) that trigger the zeroing.
            - Allow partial repay/liquidation and route through tranche math before a “full repay” escape hatch.
        - Places that **square** a fixed-point value after rounding (e.g., `(inputQ64)**2`) or multiply by large constants (`Q32`, `Q72`) post-division.
    - **Red flags:**
        - `Convert.mulDiv(..., /* roundUp? */ false)` on a path where the result must be **> 0**.
        - Comments like “round down” or `unchecked` around price/tick conversions.
        - Ratios with potentially tiny numerators: `netDebtXorYAssets << activeLiquidityAssets`.
        - Retry succeeds only with **full repay**, not with partial; or succeeds “after some time” as debt accrues.
        - Market actions (swaps) that increase active liquidity make the revert **more** likely.
    - **Questions to ask next time:**
        - Can any ratio we compute for price/tick be **zero** due to rounding when debt is small? What’s the **minimum positive** representable value in this Q format?
        - Do we ever **square** or log-map a value that could be zero? Do we clamp or guard it?
        - Do repay/liquidate flows use **assets** (ERC-4626 preview/max) or **shares** to size inputs? What happens if dust remains?
        - Is there a **dust floor** or **round-up** policy where positivity is required? Are rounding directions **consistent** across compute vs. validation?
        - Could third-party state changes (swaps increasing liquidity) turn a previously valid partial repay into a revert?
    - **Fix pattern:**
        - **Enforce positivity** before tick/price mapping: compute with **round-up** (`mulDiv(..., true)`) where a strictly positive result is required; or `value = max(value, MIN_Q)` with a documented `MIN_Q`.
- [ ]  **Reference snapshot before interest accrual → fee mispricing**
    - **Symptom:** Swap fees that depend on deviation from a *reference reserve/tick* are too low/high because the protocol (per-block) resets the reference using **stale reserves**, then **accrues interest** (or other balance changes) that shift the live reserves **after** the snapshot. The fee path compares `currentReserve` vs `referenceReserve` that were computed from different states, pushing trades into the wrong fee tier (min/quadratic/linear) or even to a near-zero fee edge.
    - **Where to look (AMM + lending hybrids / fee-by-deviation designs):**
        - Pair core: the “pre-action” hook that does **(a)** penalties/rewards, **(b)** **TWAP/observation update & reference reset**, **(c)** **interest accrual / reserve updates**. The order of (b) vs (c) is the bug magnet.
        - Observation/TWAP module: functions like `recordObservation(...)`, `updateReferenceReserve(...)`, `getObserved...Tick()`.
        - Interest/Accounting module: functions that **mint interest to reserves** or move balances: `accrueInterest*`, `updateReserves(...)`, protocol-fee minting, donations, or “depleted reserve” adjustments that change X/Y asymmetrically.
        - Fee library: functions like `calculateSwapFee...` that branch on `currentReserve < referenceReserve` or use `(input + current) > k * reference`.
        - Any caching of reserves (`(_reserveX, _reserveY) = getReserves()`) passed into observation before mutations are applied.
    - **Red flags:**
        - Sequence “**updateObservation/reference**” → **then** “**accrueInterest/updateReserves**”.
        - Comments like “reset each block” near reference update while interest accrual uses **elapsed time since last update**.
        - Reference tick/reserve derived from **cached** reserves; interest accrual later uses **storage** reserves (mismatch).
        - Interest accrual is **per-asset** (X and Y change by different amounts), so price/tick changes materially even without swaps.
        - Extremely high possible APRs or long inactivity windows (minutes/hours) that can move price by ≥ 1 tick between steps.
    - **Questions to ask next time:**
        - In the pre-swap/pre-liquidity hook, **which exact state** are reference reserves/tick derived from—before or after interest/fees/penalties?
        - Could a user trigger an action that *accrues interest but skips reference update*, then immediately swap and enjoy distorted fees?
        - Are reserves/ticks computed from **the same snapshot** for both reference and current? Any **cached-vs-storage** drift?
        - What’s the **max single-step interest delta** versus tick size? Can one block’s accrual move more than one tick?
        - Does the fee function have discontinuity “cliffs” at `current vs reference` boundaries that become exploitable if reference lags?
    - **Fix pattern:**
        - **Reorder**: Accrue interest/penalties/fees and update the actual reserves **before** updating observations/reference tick.
        - Or **derive the reference** from the **post-accrual reserves** (single source of truth), not from pre-accrual caches.
        - Add a **versioned snapshot** (block-scoped) so fee logic uses `reference` and `current` from the **same version**.
- [ ]  **Rounding-to-zero when minting protocol fees (assets→shares)**
    - **Symptom:** Protocol fee amounts (computed in **assets**) are converted to **shares** with **roundDown**, so adversaries can make `toShares(feeAssets)` return **0**, eliminating fees. They do this by (a) forcing **tiny, frequent accruals** (per-block sync), and/or (b) **skewing price-per-share** (e.g., donating assets so `assets >> shares`). Over time, substantial fee assets translate to **zero fee shares minted**.
    - **Where to look (ERC4626-like vaults / AMM+Lending hybrids):**
        - Fee minting path: functions like `mintProtocolFees`, `distributeFees`, `takeFees`, or controllers that do `toShares(feeAssets, totalAssets, totalShares, /*roundUp?*/ false)` and then `mint`.
        - Interest/Revenue accrual: `accrueInterest*`, “protocol cut = X% of interest” → later **converted to shares**.
        - Share math helpers: `toShares`, `previewMint/Deposit`, `mulDiv(..., rounding=false)`, or custom rounding flags.
        - State that can drift **assets vs. shares**: donations, external transfers to the pool, interest credited directly to **assets** without minting shares, “dead shares” constants that keep `totalShares` small, rebases.
        - Any hook that can be **called very frequently** (every block/tx) to slice feeAssets into dust (e.g., `sync()`, `updateTokenController()`, “accrue on any action”).
    - **Red flags:**
        - `toShares(feeAssets, ..., /* roundUp */ false)` or equivalent **flooring** when converting protocol-fee *assets* → *shares*.
        - No **fee remainder accumulator** (fractional-share carry) kept in assets or high-precision fixed point.
        - Ability to **donate assets** (increase `totalAssets` without increasing `totalShares`) or otherwise pump **assets/share**.
        - Fee mint uses **current** `totalAssets/totalShares` *after* externalizable changes (donations, interest credit), not a protected snapshot.
        - Very fast block times (L2s) + fee accrual triggers on **every** block/action.
    - **Questions to ask next time:**
        - When protocol fees are minted, is the **assets→shares** conversion **rounded up**? If not, why not?
        - Do we **accumulate** fee assets until they convert to at least 1 share, or do we mint per-tx?
        - Can anyone **donate assets** or otherwise change `assets/share` before fee conversion? How do we neutralize that?
        - Is there **carry-over of fractional shares** (e.g., store `feeRemainderAssets` and add it to the next round)?
        - Are there **minimum per-accrual thresholds** or rate limits, or can an attacker force many tiny accruals?
    - **Fix pattern:**
        - Convert protocol fee **assets→shares with rounding up** (ceiling) for the protocol’s benefit; or
        - Keep a **feeAssets accumulator** per token (high precision). Only convert/mint when it would produce ≥1 share; retain the remainder.
        - Alternatively, compute fees **directly in shares** using the same conversion basis as lender interest (avoid a separate conversion step).
        - Guard against **donation skew**: on external asset deltas, either (1) mint corresponding shares to a neutral sink (e.g., the pool) to keep price/share stable, (2) or **exclude** donated assets from `totalAssets` used for fee conversion until normalized via a `skim()`.
- [ ]  **Swaps bypass interest accrual → “sync-arbitrage” steals LP yield**
    - **Symptom:** `swap()` quotes against **stale reserves** because interest isn’t accrued on the swap path. An attacker can: (1) swap before accrual, (2) trigger accrual (e.g., `sync()`/state-updating call), then (3) swap back—capturing the price shift created when interest is credited to reserves. Net result: part of the **interest meant for LPs** is extracted by the swapper.
    - **Where to look (AMMs with built-in lending/interest or fee reinvest):**
        - Entry points: `swap()` vs “everything else” (mint/burn/deposit/borrow/repay/liquidate). Verify whether **only non-swap paths** call the interest accrual routine.
        - Accrual driver: functions like `accrueSaturationPenaltiesAndInterest`, `updateTokenController`, `accrueInterest*`. Check if `swap()` calls them (directly or via a shared prehook).
        - Accrual crediting: interest credited **to reserves** (changes price) vs **to deposit/receipt tokens** (price-neutral). Lines that add `interestXForLP/interestYForLP` to reserves are a tell.
        - Time gates/conditions: `if (deltaLendingTimestamp > 0)` or similar guards that may **skip** accrual in fast blocks (L2s).
        - Pricing/fees: amountOut math that uses current/reference reserves and fee curves (e.g., quadratic/linear regions). If reserves are updated **after** reference reset or accrual, price/FEE discontinuities appear.
    - **Red flags:**
        - Comments or code paths like “accrue on all actions **except** swaps”.
        - Accrual splits per-asset (X vs Y) with different utilizations, then **adds to reserves** → immediate **price ratio change**.
        - A cheap, public “tickle” (e.g., `sync()`/any write) that anyone can call to force accrual **after** a swap.
        - Accrual uses **averages/TWAP** but **still** mutates reserves (average smooths parameters, not the price jump).
    - **Questions to ask:**
        - Does `swap()` ensure reserves include **pending interest** (either by accruing or by using **virtual reserves**)? If not, what’s the worst-case staleness?
        - Is interest credited in a **price-changing** way (to reserves) or **price-neutral** way (to deposit tokens/receipts)?
        - Can an external party cheaply force an accrual **between two swaps** in the same block/epoch?
        - Are swap fees guaranteed to exceed the maximum interest-induced price delta per accrual interval? What if the attacker is also a large LP (recaptures fees)?
    - **Fix pattern:**
        - **Pre-swap accrual:** run the same interest accrual prehook used by other state-changing functions before computing `amountOut`.
        - **Virtual reserves lens:** compute “as-if accrued” reserves for quoting/fee calculation (no state write), then settle real accrual opportunistically.
        - **Price-neutral accrual:** credit LP interest to **deposit tokens** (or an internal accumulator) instead of raw reserves; only convert to reserves when it won’t shift price used for quoting.
        - **Accrual thresholds:** require accrual if `deltaTime ≥ T_min` or if utilization/tick drift crosses a bound; otherwise block or reprice swaps.
- [ ]  **Out-of-range utilization math bricks accrual paths**
    - **Symptom:** Utilization adjustment code (e.g., saturation/penalty shapers) assumes `utilization ≤ MAX_UTILIZATION`. If utilization creeps above the cap, expressions like `MAX_UTILIZATION - utilization` underflow, so accrual/penalty pipelines revert. Because most state-changing methods prehook into accrual, the pair becomes **perma-reverting** (except paths that skip accrual, e.g., `swap()`), blocking repay/liquidate/borrow/mint/burn.
    - **Where to look (lending AMMs / over-collateralized DEXs / interest modules):**
        - Utilization shaping helpers: `mutateUtilization*`, `getUtilizationsInWads`, penalty/“saturation” functions that compute `adjustedUtilization = f(MAX_UTILIZATION - utilization, …)`.
        - Accrual entrypoints: top-level prehooks like `accrue*()` called by every method **except** `swap()`. Trace: pair.accrue → tokenController.update → interest.accrue → utilization mutation.
        - Constants & guards: `MAX_UTILIZATION_PERCENT`, `MAX_SATURATION_PERCENT`, `PENALTY_*`, `_BUFFER_IN_WAD`. Check for missing **clamp** before `mulDiv` / subtraction.
        - Arithmetic helpers: `Convert.mulDiv(..., false)` with unchecked math; any place computing `reference - current` without guarding sign.
    - **Red flags:**
        - “Cap after compute” patterns: calculate `adjusted = … (MAX - utilization) …` then later `min(adjusted, MAX)`—the **cap never runs** if you already reverted on underflow.
        - Comments that assume “utilization can’t exceed X due to incentives,” but no runtime `min()`/early return.
        - Underflow-prone lines: `MAX_UTILIZATION - utilization`, `MAX_SAT - sat`, or denominators that can become 0 when “at cap”.
        - Global accrual prehooks on most functions; only one public function (often `swap`) bypasses accrual → easy to enter bad range and impossible to exit.
    - **Questions to ask:**
        - Can utilization exceed MAX due to **time-based interest**, **penalty accrual**, or **rounding drift** without any user action?
        - Which public methods **must** call accrual, and do they all share the same out-of-range math? What’s the **recovery path** if utilization > MAX (admin, circuit breaker, or special function)?
        - Are all `(A - B)` terms protected by a clamp (`A >= B ? A - B : 0`) or early return when inputs are out of domain?
        - Do tests cover edge cases at/over caps (MAX±1 wei) and long idle periods on fast-block chains (L2s) where accrual deltas spike?
    - **Fix pattern:**
        - **Clamp inputs** before math: `if (utilization >= MAX_UTILIZATION) return MAX_UTILIZATION;`.
        - Use **saturating arithmetic** or explicit `uint256 subFloor = (utilization < MAX_UTILIZATION) ? (MAX_UTILIZATION - utilization) : 0;`.
- [ ]  **Swap-time reference refresh makes quotes uncomputable**
    - **Symptom:** `swap()` updates observations/TWAP and recomputes **reference reserves** right before enforcing K/fees. Off-chain quoters that read `referenceReserves()` **pre-call** can’t reproduce the exact in-swap reference used, so `amountOut` diverges from estimates and can be MEV-sandwiched via state timing.
    - **Where to look (TWAP’d AMMs / dynamic-fee pairs):**
        - `swap()` path: calls like `updateObservation(...) → recordObservation(...) → updateReferenceReserve(...)` just before invariant/fee checks.
        - Reference state readers: `referenceReserves()`, `getReserves()`, `lastUpdateTimestamp`, tick buffers (mid/long-term), and any `getObserved*Tick(...)`.
        - Fee/invariant code that consumes **reference vs current** reserves (e.g., `calculateBalanceAfterFees`, quadratic/price-impact fee modules).
        - Any quoter/helper: ensure view functions use the **same “would-be updated” reference** as `swap()`, not the stored one.
    - **Red flags:**
        - `swap()` mutates observation/reference state using `block.timestamp` deltas, then reads it for fees/K.
        - `referenceReserves()` returns **stored** values that differ from what `swap()` will recompute.
        - No public **pure/view** “predictReferenceReserves”/“quoteExact\*” that mirrors the in-swap update logic deterministically.
        - Observation update depends on mid-term/long-term tick state or rounding (ceil/floor toward new tick) that off-chain cannot infer without duplicating internals.
    - **Questions to ask next time:**
        - Does a read-only quoter exist that **applies the same observation/reference refresh** (without persisting) the swap will do?
        - Are all inputs required to recompute the in-swap reference **publicly readable** (ticks, last indices, last timestamps, config intervals, rounding rules)?
        - What is the **ordering** of (1) observation refresh, (2) interest/penalty accrual, (3) fee/K checks inside `swap()`? Could reordering remove nondeterminism?
        - Can `swap()` skip reference refresh if `deltaUpdateTimestamp==0` and still be correct, or should refresh be **moved earlier** (global accrual hook) for predictability?
    - **Fix pattern:**
        - Expose a **view quoter** that replicates `swap()`’s exact pre-check pipeline (observation refresh → reference prediction → fees/K) using `staticcall`safe code paths and no state writes.
        - Or make `swap()` compute reference reserves from a **pure function of public state** (reserves, last ticks/timestamps) so off-chain can mirror byte-for-byte.
        - Consider **pre-refreshing** observation/reference in a shared accrual hook invoked by both `swap()` and off-chain estimators (with identical rounding), or gate refresh to block boundaries to reduce timing variance.
- [ ]  **Admin “external liquidity” knobs instantly tighten LTV and nuke healthy loans**
    - **Symptom:** An admin-writeable liquidity buffer (e.g., `externalLiquidity`) is added to `activeLiquidityAssets` in solvency checks. Decreasing it takes effect **immediately**, shrinking health factors across the book and making previously-healthy positions liquidatable in the same tx/block (no grace).
    - **Where to look (lending AMMs / risk engines):**
        - Parameter writes: `updateExternalLiquidity(...)`, `setBackstop/Buffer(...)`, guardian/owner “risk knob” setters.
        - Health-factor path: `getInputParams()` → `activeLiquidityAssets` composition → `checkLtv()` / `validateOnUpdate()` / `validateSolvency()`.
        - Slippage/penalty helpers folded into debt: functions like `increaseForSlippage(...)`, “saturation” or “utilization” adjustments that scale debts by available liquidity.
        - Read paths used by **all** actions (borrow, repay, deposit, withdraw, liquidate) that call a common accrual/validation hook before proceeding.
    - **Red flags:**
        - Admin setter has **no delay**, no epoching, no bounds, no monotonicity (can jump down arbitrarily).
        - `externalLiquidity` (or “insurance”, “backstop”, “oracle buffer”, “safety module liquidity”) is **directly added** to `activeLiquidityAssets` and used **immediately** in LTV without staging.
        - Solvency checks computed off **current** buffer only; no per-position snapshotting / grandfathering.
        - Liquidation eligibility depends on `activeLiquidityAssets` rather than only user collateral / debt.
    - **Questions to ask next time:**
        - Does changing risk knobs (external/backstop liquidity, LTV caps, liquidation premiums) apply **atomically** to all open loans? Is there a **grace period** or **rate limiter** on decreases?
        - Are borrowers **grandfathered** to the previous buffer until an expiry timestamp/epoch? Is there a migration path?
        - Are there **lower/upper bounds** and **monotone ramps** (e.g., decreases limited to X% per hour/day)?
        - Which actions read the buffer first (e.g., `validateOnUpdate`) and can a single admin tx both **reduce** the buffer and **liquidate** in the same block?
        - Is the buffer used in **slippage/penalty amplifiers** (e.g., saturation math) that can spike effective debt?
    - **Fix pattern:**
        - **Two-step / timelocked decreases:** Stage a `pendingExternalLiquidity` with `effectiveAt` (block/time). Use current value for open positions until the timestamp; only **increases** may be immediate.
        - **Ratcheting / rate-limited changes:** Enforce max downward delta per interval; disallow same-block use for liquidations when a decrease occurs.
        - **Grandfathering:** Compute solvency using `min(position.snapshotExternalLiquidity, currentExternalLiquidity)` or pin a per-loan snapshot that decays to current over a grace window.
        - **Action gating:** Block liquidations/health checks that rely on the new lower buffer for N seconds/blocks after an admin decrease; or require a separate “finalize” tx.
- [ ]  **Off-reference pivot miscomputes → quadratic fees keep growing past 4,000 bips**
    - **Symptom:** The branch that should switch from quadratic growth to the slower post-cliff formula compares `input + currentReserve` against `referenceReserve * LINEAR_START_REFERENCE_SCALER`. When `currentReserve ≠ referenceReserve`, the pivot is shifted, so swaps that should have crossed the 4,000 bips threshold remain in the quadratic regime (or flip at the wrong point). Around the pivot you can even see **fee drop when input increases** (non-monotonic).
    - **Where to look (AMMs with reference-price–based dynamic fees):**
        - Fee modules like `QuadraticSwapFees.calculateSwapFee…` (or equivalents): branches for `currentReserve < referenceReserve` vs `>`, and the “pivot” condition that compares to `LINEAR_START_REFERENCE_SCALER`.
        - Constants: `MAX_QUADRATIC_FEE_PERCENT_BIPS`, `LINEAR_START_REFERENCE_SCALER`, `MIN_FEE_Q64/BIPS_Q64`, and the `N`/slope terms.
        - Any logic that mixes “**distance traveled**” (`input`) with “**distance from reference**” (`currentReserve − referenceReserve`) but only guards one of them.
        - Places where reference reserves are updated separately from current reserves (TWAP/observation updates) — look for assumptions that reference==current at branch time.
    - **Red flags:**
        - Pivot check of the form `if (input + currentReserve > referenceReserve * scaler)` (or the mirror case) without explicitly adding/subtracting the **offset** `currentReserve − referenceReserve`.
        - Asymmetric formulas between the “toward reference” and “past reference” branches (e.g., one uses `input + 2*(current−reference)` while the guard uses `input + current`).
        - Fee curves that **aren’t monotonic** in `input` for fixed `(current, reference)` or show a **discontinuity** at the pivot when plotting.
        - Relying on absolute reserves in guards instead of normalized deltas like `pastBy`, `toPivot`, or `% of reference`.
    - **Questions to ask to surface it next time:**
        - Is the “switch to slower growth” condition expressed in the **same dimension** as the fee formulas (i.e., both in “distance from pivot” space)?
        - When `currentReserve` is **off reference**, does the pivot still occur when the **marginal** distance past the reference reaches the target percentage, or is it using an absolute `input`?
        - Do the fee functions guarantee **continuity at the pivot** and **monotonicity in input** for any `(current, reference)`?
        - Are the “toward reference” and “past reference” branches **symmetric** across the two directions of imbalance?
    - **Fix pattern:**
        - Compute explicit deltas:
        `toPivot = max(0, referenceReserve − currentReserve)` and `pastBy = max(0, input − toPivot)`.
        Use `toPivot` for the quadratic segment and `pastBy` for post-cliff, independent of raw `currentReserve`.
        - Guard with the correct pivot test:
        `if (toPivot == 0 ? input : pastBy) > referenceReserve * LINEAR_START_REFERENCE_SCALER` (or equivalent normalized check).
        - Ensure **continuity** at the pivot by evaluating both formulas at the boundary and matching them; clamp within `[MIN_FEE, MAX_QUADRATIC_FEE]`.
        - Mirror the logic for both `current < reference` and `current > reference` directions; avoid mixing “absolute reserve” terms inside guards.
- [ ]  **Bypassable collateral accounting lets borrowers hide assets from bad-debt seizure**
    - **Symptom:** After a bad-debt liquidation, the liquidator seizes only the **accounted** collateral (e.g., `registry.collateral[holding]`), while extra tokens remain in the user’s holding because they were **invested via a non-accounting path (“donation route”)**. Strategy losses from that unaccounted investment can *decrease* the accounted collateral (reducing what the liquidator gets), but the principal stays unseizable and is later withdrawable by the user.
    - **Where to look (vaults/strategy frameworks):**
        - Holding/Registry layer: `SharesRegistry.collateral(holding)` mapping, and any place it’s read during liquidation (e.g., `liquidateBadDebt`, `_retrieveCollateral`).
        - Strategy entry points: strategy `deposit()` that does `IHolding(_recipient).transfer(...)` (pulls straight from the holding), `StrategyManager` routes, and any path that **doesn’t** call the canonical deposit/registry update (`HoldingManager`).
        - Strategy accounting: `recipients[_recipient].investedAmount/totalShares`, `claimInvestment()` profit/loss flows and how they mutate `collateral[holding]`.
        - Liquidation flow: amount passed to retrieval (e.g., `_retrieveCollateral(_amount = totalCollateral)`), and the final transfer to liquidator (does it send **only** `totalCollateral` or the **entire token balance** in the holding after retrieval?).
        - Withdraw guards: checks that block withdrawals only when `collateral[holding] > 0`, enabling “ghost” funds to be pulled out post-liquidation.
    - **Red flags:**
        - Strategy `deposit()` pulls from the holding (`IHolding.transfer`) **without** a preceding registry increment of `collateral[holding]`.
        - Multiple deposit paths (e.g., `HoldingManager` vs `StrategyManager`/direct) with inconsistent accounting; “donation” transfers succeed and are investable.
        - Liquidation computes seizable assets from **registry values** (`totalCollateral`) and **does not sweep the actual token balance** left in the holding after strategy withdrawals.
        - Profit/loss from unaccounted strategy positions **mutates** `collateral[holding]` (e.g., losses reduce it) even though the principal was never accounted—mismatch between “what counts” and “what can be seized.”
        - Post-liquidation, `collateral[holding] == 0` but `IERC20(token).balanceOf(holding) > 0`, and `withdraw()` is allowed.
    - **Fix pattern:**
        - **Single entry point & invariant:** Enforce all investments to go through a manager that first updates `registry.collateral` (or an equivalent accounting structure) **before** funds can leave the holding; block strategy pulls that don’t present an “accounted amount” proof.
        - **Atomic accounting on strategy deposit:** In strategy `deposit()`, require a manager-only call that (a) tags funds as collateralized or non-collateralized and (b) updates registry counters in the same tx; reject raw “donation” funding for invest.
        - **Seizure uses balances, not just registry:** During bad-debt liquidation, withdraw **max** from strategies, then **sweep the holding’s entire token balance** for the collateral token to the liquidator (or, compute `seizable = accounted + unaccountedInHolding`, and transfer all).
        - **Consistent PnL application:** Ensure PnL from positions that were never accounted **cannot** mutate `collateral[holding]`; alternatively, force all invested principal to be accounted so PnL and seizure align.
        - **Withdrawal guard:** Disallow withdrawing any token from a holding that has ever been used as collateral unless the registry declares the holding fully solvent and no bad-debt flow is active; add an invariant check (`collateral + withdrawable == onchain balances + strategy NAV`).
    - **Questions to ask next time:**
        - Can strategies pull tokens from a holding **without** first increasing the holding’s `collateral` in the registry?
        - In liquidation, do you seize **only** `registry.collateral` or do you also sweep **all token balances** left in the holding after strategy unwinds?
        - Can PnL from unaccounted strategy positions change `collateral[holding]`?
        - Are there **multiple** deposit/invest paths (manager vs direct) with divergent accounting semantics?
        - After liquidation sets `collateral[holding]` to zero, can the user still withdraw leftover tokens from the holding? If yes, why weren’t those swept to the liquidator?
- [ ]  **Collateral offboarding flag freezes exits (deactivate ≠ retire)**
    - **Symptom:** Flipping a collateral/registry to **inactive** blocks **withdrawals, repayments, strategy unwinds, and liquidations**, trapping user funds and preventing bad-debt cleanup. Admins can’t safely retire a volatile/compromised asset because the same “active” gate guards both **entry** (new deposits/borrows) and **exit** (repay/withdraw/liquidate).
    - **Where to look (vaults/lend/strategy suites):**
        - Registry/asset allowlists: checks like `require(shareRegistryInfo[_token].active, "...")` in **withdraw**, **repay**, **liquidateBadDebt / liquidate**, and **strategy claim/withdraw** paths.
        - Liquidation flows: functions that compute seizable collateral then call `_retrieveCollateral(...)` and finally transfer only if `registry.active`—verify exits aren’t gated by “active.”
        - Strategy manager: `claimInvestment()/withdraw()` requiring active registry before pulling funds back to the holding.
        - Pausing / kill-switch logic: global `whenNotPaused` vs per-asset “active” flags—ensure deactivation is asset-scoped and **exit-safe**.
        - Token controller / validation: LTV checks (`checkLtv`, `validateOnUpdate`) that read `isRegistryActive` and revert for otherwise healthy **exit** operations.
    - **Red flags:**
        - One boolean (`active`) used for **both** admission control and **exit** operations.
        - `removeCollateral()` / `unregisterCollateral()` guarded by `active == true`.
        - Liquidation relies on registry active to withdraw or settle, so deactivating a depegged asset also blocks liquidations/repayments.
        - No “grace period” or “retirement mode” state; only {active, paused}.
    - **Fix pattern:**
        - **Split states:** Introduce `admitActive` (controls new deposits/borrows/strategy invests) and `exitEnabled` (controls withdrawals/repays/liquidations/strategy unwinds). Retirement sets `admitActive=false`, keeps `exitEnabled=true`.
        - **Allow-listed exits:** Gate **entry** paths with `require(admitActive)`, but allow exits regardless (or `require(exitEnabled)` which remains true during offboarding).
        - **Forced unwind on deactivate:** On switching to retirement mode, enable strategy-level force-withdraw; liquidation and repayment must bypass `admitActive` checks.
- [ ]  **Double-fee on strategy rewards (performance + withdrawal)**
    - **Symptom:** Rewards claimed from strategies pay a **performance fee on claim** and then get hit again by a **withdrawal fee** when the user pulls those reward tokens out of the system. This applies collateral-exit fees to non-collateral rewards, silently shaving yield.
    - **Where to look (vaults/strategies/holdings):**
        - Strategy reward flows: `StrategyManager.claimRewards()` → each strategy’s `claimRewards(...)` (e.g., `AaveV3StrategyV2.claimRewards`) for performance-fee deduction and recipient routing (does it force rewards to the holding?).
        - Holding withdrawals: `HoldingManager._withdraw(...)` fee branch (`manager.withdrawalFee()`) and pre-checks (`isTokenWithdrawable`, registry lookups) to see if **all** tokens—collateral or rewards—are charged the same exit fee.
        - Manager config: fee caps like `MAX_PERFORMANCE_FEE`, `MAX_WITHDRAWAL_FEE`, and any per-token/per-category fee switches (often missing).
        - Token classification: anywhere a token is tagged as **collateral vs reward/airdrop** (or lack thereof). Look for absence of `isRewardToken()` / “non-collateral” paths.
    - **Red flags:**
        - Rewards are **always** claimed to the user’s holding (no alternate recipient), then **always** withdrawn via the same fee’d path as collateral.
        - Single global `withdrawalFee` applied regardless of whether the token reduces registered collateral or is “extra” yield.
        - No separation in accounting (e.g., `collateral[holding]` vs `rewards[holding]`); withdrawal fee computed purely on `_amount`/`_token`.
    - **Fix pattern:**
        - **Classify rewards** and **exempt them** from withdrawal fees: add a reward/airdrop flag (per token or per transfer source) and branch in `_withdraw` to skip `withdrawalFee` when `_token` is reward-only or when the withdrawal does **not** reduce registered collateral.
        - **Direct-to-user reward routing:** allow `claimRewards(to)` to send net rewards directly to the user (after performance fee), bypassing fee’d holding withdrawal.
        - **Per-token fee policy:** maintain a registry mapping `{token → feePolicy}` so collateral tokens apply exit fees, while reward-only tokens (not used as collateral) do not.
        - **Accounting split:** track `collateral` vs `rewards` balances in holdings; only charge `withdrawalFee` when decreasing `collateral` (or when the token is whitelisted as collateral and the withdrawal reduces `collateral(holding)` in the shares registry).
- [ ]  **Slippage floor ignores token decimals (unit-mismatch on minOut)**
    - **Symptom:** `getAllowedAmountOutMin()` computes the min acceptable output using a WAD (1e18) rate but **doesn’t rescale for token decimals** (e.g., USDT 6 ↔ deUSD 18). The returned minOut is off by `10**|decIn-decOut|`, causing needless swap reverts or under-protected slippage.
    - **Where to look (routers/strategies/oracles):**
        - Slippage helpers: `getAllowedAmountOutMin`, `_applySlippage`, any `amountOutMinimum` checks in swap paths (e.g., `_swapExactInput*`).
        - Rate sources: `oracle.peek(...)` return scale (WAD? 1e8? raw?) and any comments about decimals.
        - Token metadata: places using `IERC20Metadata.decimals()` (or not), and conversions where amounts cross tokens with different precisions.
        - Math utils: `mulDiv` sites with hardcoded `1e18`; look for symmetric formulas for both directions that **don’t** include `decIn/decOut`.
    - **Red flags:**
        - Hardcoded `1e18` in FX math without `decIn/decOut` rescaling.
        - Slippage applied **before** converting to the output token’s precision.
        - Comparing `amountOutMinimum` (caller units) against an internally computed min that’s in **the wrong token scale**.
        - Reused formula for `FromTokenIn` and `ToTokenIn` that only inverts `rate` but never adjusts decimals.
    - **Fix pattern:**
        - Normalize to a common precision then convert to the **output token’s** decimals:
            - `W = _amountIn * 10**(18 - decIn)` → `expectedWad = W * rate / 1e18` → `expectedOut = expectedWad * 10**(decOut - 18)`.
            - For the reverse direction, invert the rate **and** include `decIn/decOut` in the numerator/denominator.
        - Apply slippage **after** you’re in `decOut` units; choose rounding that matches protection semantics (minOut typically rounds **down** after slippage).
        - Centralize a helper `scaleAmount(amount, decFrom, decTo)` and a `quoteOut(amountIn, rate, decIn, decOut)` to prevent recurrence.
        - Assert at runtime (or via invariants) that `getAllowedAmountOutMin(...)` and the swap’s `amountOutMinimum` are compared in the **same token and decimals**.
    - **Questions to ask next time:**
        - What scale does the oracle rate use, and how is it documented/enforced?
        - In which token’s units is `amountOutMinimum` expressed, and where is that guaranteed?
        - Where do we convert between `decIn` and `decOut`, and is slippage applied pre- or post-conversion?
        - Do both swap directions include explicit `decIn/decOut` terms (not just `rate` inversion)?
- [ ]  **Single-path strategy withdrawals ignore cooldown toggles (ERC4626 vs `unstake`)**
    - **Symptom:** Withdrawals “succeed” but return **0 assets** when the underlying staking wrapper disables cooldowns (e.g., `cooldownDuration == 0`). Strategy only calls `unstake()` (cooldown path) and never switches to **ERC-4626 `withdraw/redeem`** path required when cooldown is off.
    - **Where to look (Solidity adapters/strategies):**
        - Strategy adapter functions that **exit/withdraw/claim** from wrappers with dual modes (cooldown vs. ERC-4626), e.g. `stdeUSD`, LST wrappers, staking vaults with `cooldownShares/unstake` *and* `withdraw/redeem`.
        - Calls made via generic dispatcher patterns (e.g., `_genericCall`, `delegatecall`) to the staking token contract: look for hard-coded `unstake()` without checking mode.
        - Interfaces & guards on the wrapper token:
            - Modifiers like `ensureCooldownOn` / `ensureCooldownOff`.
            - Admin knobs like `setCooldownDuration(uint24)` that flip behavior at runtime.
        - Any use of per-user cooldown storage (`cooldowns[msg.sender].underlyingAmount`, `cooldownEnd`) that the strategy **relies on** being populated before `unstake()`.
    - **Red flags:**
        - Strategy only implements:
            
            ```solidity
            IStakedToken.unstake(receiver);
            
            ```
            
            with **no** `IERC4626.withdraw(...)` / `IERC4626.redeem(...)` fallback.
            
        - No read of `cooldownDuration()` (or equivalent) before deciding which method to call.
        - No assert on **non-zero** assets after a withdrawal call (silent success paths).
        - Tests cover only the cooldown-ON path; **no unit test** toggling `cooldownDuration` to `0` and attempting a withdraw.
        - Adapter tracks shares but never uses them in `redeem(...)` when cooldown is off.
    - **Questions to ask the codebase/maintainers next time:**
        1. *Can the underlying staking token flip between cooldown and ERC-4626 modes at runtime? Where is that knob?*
        (Search for `setCooldownDuration`, `ensureCooldownOn/Off`.)
        2. *What’s our withdrawal decision tree?* Do we branch on `cooldownDuration()` or attempt `redeem` first then fallback to `unstake`?
        3. *Do we verify non-zero asset out on withdraw paths?* What’s the behavior if `unstake()` returns 0?
        4. *Do we persist pending cooldown state (shares/amount, timestamps) per account and surface a claim function?*
        5. *Are there tests for ON→OFF and OFF→ON flips mid-lifecycle, including mid-cooldown and immediate claim when toggled off?*
        6. *If a generic call pattern is used, can it accidentally call `unstake` in OFF mode without reverting?*
    - **Fix pattern:**
        - **Branch by mode** or **optimistic ERC-4626 first**:
            
            ```solidity
            if (cooldownDuration() == 0) {
                IERC4626Like(token).redeem(shares, receiver, owner); // or withdraw(assets,...)
                require(assetsOut > 0, "zero-out");
            } else {
                token.cooldownShares(shares); // later: token.unstake(receiver)
            }
            
            ```
            
        - Add a claim path that calls `unstake(receiver)`; it should succeed **either** after cooldown expiry **or** immediately if admin flips cooldown to `0`.
        - Guard against silent zero returns (revert or emit error event).
        - Add unit tests covering: cooldown ON, OFF, mid-cooldown flip, partial withdrawals, and zero-asset regression.
- [ ]  **Cooldown-gated staking lets borrowers block liquidation (partial-cooldown griefing)**
    - **Symptom:** Positions using a staked asset with a **two-step cooldown → unstake** flow become practically un-liquidatable. Borrowers can (a) never trigger cooldown, or (b) keep resetting the cooldown by cooling down **1 wei of shares**, so liquidations/claimInvestment revert or return 0 until the timer elapses—creating bad-debt risk.
    - **Where to look (strategies/liquidations/integrations):**
        - Strategy wrappers for staking derivatives: functions like `cooldownShares/cooldownAssets`, `unstake`, `withdraw`, `redeem`, `claimInvestment`, `retrieveCollateral`.
        - Liquidation paths: `liquidate()/liquidateBadDebt()`, `StrategyManager.claimInvestment/_retrieveCollateral`, per-strategy adapters called during liquidation.
        - External token APIs: Cooldown storage (`cooldowns[account]`), `cooldownDuration`, semantics when `duration==0` (ERC4626 path vs cooldown path), whether `unstake` reads `msg.sender` vs `_recipient`.
        - Access control: who may initiate cooldown on behalf of the user (`owner()`, manager, **anyone**?), and whether partial cooldown is allowed.
    - **Red flags:**
        - Cooldown setter: `cooldowns[user].cooldownEnd = block.timestamp + cooldownDuration;` with **no minimum share amount** and **resets on every call**.
        - Liquidation implementation assumes **immediate redemption** (calls only `unstake`) and doesn’t start cooldown automatically when needed.
        - Partial cooldowns allowed; no requirement to cooldown **all** shares (or a threshold %) while position has debt.
        - `unstake` pulls from `cooldowns[msg.sender]` (not `_recipient`), but strategy calls it on the holding, leading to zero assets if cooldown wasn’t set for that exact address.
        - No “force-cooldown” hook (permissionless or keeper) for liquidatable accounts; no reward to incentivize it; no LTV haircut while not in cooldown.
    - **Fix pattern:**
        - **Enforce full cooldown for collateralized shares:** when borrowing or when becoming liquidatable, strategy/manager must initiate `cooldownShares(maxRedeem(holding))` (or ≥ X% of shares). Disallow partial cooldown while debt exists.
        - **Cooldown awareness in liquidation:** if asset is in cooldown mode and not fully cooled, (a) anyone-can-cooldown liquidatable accounts (with a small incentive), or (b) block borrowing/raising LTV until coverage ≥ threshold.
        - **Griefing mitigation:** don’t reset the timer for dust—require a **minimum shares** amount or only allow cooldown extension if **added shares ≥ threshold**; alternatively track coverage and **do not update `cooldownEnd`** unless the covered fraction increases.
        - **Dual-path integration:** support both cooldown (`cooldownShares`→`unstake`) and no-cooldown (`redeem/withdraw`) modes; branch on `cooldownDuration`.
        - **Risk controls:** apply LTV haircut (or disallow as collateral) when `cooldownDuration>0` unless protocol has a robust forced-cooldown flow; add SLA/expiry handling to avoid permanent bricks.
    - **Questions to ask next time:**
        - Who can initiate cooldown for a debtor—only the user, or can the manager/keeper/anyone do it? Is there an incentive to do so?
        - Can users **reset** the cooldown with tiny amounts? Is there a **minimum shares** or **coverage ratio** for (re)starting cooldown?
        - Does liquidation code **require** assets to be in cooldown before proceeding? What happens if only **part** of shares are cooled?
        - If the staking token toggles cooldown off (`cooldownDuration==0`), does the adapter **switch to ERC4626 `redeem/withdraw`** automatically?
        - Are `unstake`/cooldown mappings keyed by `msg.sender` or the **holding address** used by the strategy? Can mismatched callers yield zero withdrawals?
- [ ]  **Partial-withdraw mismatch: cooled amount ≠ requested shares (principal booked as “yield”)**
    - **Symptom:** In cooldown-based staking adapters, `withdraw/claimInvestment` uses **user-supplied `_shares`** to compute a `shareRatio` (principal portion), but then calls `unstake()` that pays out the **entire cooled underlying** (`cooldowns[user].underlyingAmount`), ignoring the requested shares. With “cool down 100%, withdraw 1 wei” the adapter receives (almost) all assets yet attributes only a dust amount as principal—counting the rest as “yield,” inflating collateral/PNL.
    - **Where to look (staking adapters / strategy managers):**
        - Adapter withdraw path: functions that compute `shareRatio = f(requestedShares/totalShares)` and then call `unstake()` or equivalent.
        - Cooldown state: external token’s `cooldowns[user].underlyingAmount/cooldownEnd`, `cooldownShares/assets`, and whether `unstake()` takes an **amount** or always sweeps cooled funds.
        - Accounting fields: `recipients[_recipient].investedAmount/totalShares`, post-withdraw `yield = withdrawnAmount - investment`, and any `_burn(receipt, shares)` calls.
        - Call context: `unstake(receiver)` reads `cooldowns[msg.sender]`; ensure the caller matches the account that cooled shares (holding vs adapter).
    - **Red flags:**
        - `unstake()` has **no amount parameter** and always withdraws **all** cooled assets.
        - Adapter computes principal via `shareRatio(requestedShares, totalShares)` but doesn’t bind that to the **cooledShares/underlying**.
        - No mapping from **cooledShares per holding**; no check that requested shares ≤ cooled shares.
        - Partial withdrawals allowed while cooldown exists; no split-claim or escrow for remainder.
        - Yield computed as `withdrawnAmount - investment` with investment derived from **requestedShares**, not from **cooled amount** actually redeemed.
    - **Fix pattern:**
        - **Bind principal to cooled amount:** record `cooledShares` (or `cooledUnderlying`) per holding at `cooldown*` time; at withdraw use `shareRatio = cooledShares/totalShares` (or min of requested & cooled) to compute principal.
        - **Align flows:**
            - If the staking token’s `unstake()` **sweeps all**, then require withdrawals to **request all cooledShares** (disallow partials while cooled > 0), or
            - Implement a two-phase: after sweeping, **allocate principal = min(cooledUnderlying, pro-rata principal)** and treat any excess as **return of principal** (not yield); keep leftover as cooled carry or re-deposit if supported.
        - **Parameterize or gate:** if the external token supports `redeem/withdraw(amount)`, prefer those for partials; else **block partial withdraws** until cooldown is empty or supports partial claim.
        - **Invariant/tests:** assert `accounted_principal_after + accounted_yield_after == accounted_principal_before ± fees` and that `sum(principal) ≤ investedAmount` across all claims.
    - **Questions to ask next time:**
        - Does the staking token’s claim/unstake accept an **amount**, or does it always **sweep all** cooled funds?
        - Where do we **track cooledShares/underlying per holding**, and do we enforce `requestedShares ≤ cooledShares`?
        - If the user cooled 100% but requests 1 wei, how do we prevent **principal→yield misclassification**?
        - Is `msg.sender` during `unstake()` the **same address** that set the cooldown entry?
        - Do we **disallow partials** while a cooldown exists, or do we implement **correct partial allocation** with leftover handling?
- [ ]  **Single-sided Pendle LP flows push pool into MarketProportionTooHigh (unliquidatable debt)**
    - **Symptom:** Strategies use Pendle `addLiquiditySingleToken` / `removeLiquiditySingleToken` (SY-only) which **buys/sells PT against the pool** and, on unwind, needs `swapExactPtForSy`. Large single-sided positions (esp. near expiry as rateScalar steepens) trigger Pendle safety revert `MarketProportionTooHigh`, blocking strategy withdrawals and downstream liquidations.
    - **Where to look (Pendle integrations / liquidation paths):**
        - Strategy deposit/withdraw: calls to `IPActionAddRemoveLiqV3.addLiquiditySingleToken` and `removeLiquiditySingleToken`, `IPMarket.swapExactPtForSy`, `TokenInput/TokenOutput`, `ApproxParams`.
        - Liquidation flows that unwind strategy positions: `_claimInvestment`, `claimInvestment`, `liquidate/ liquidateBadDebt` → any reliance on **single-token** removal.
        - Pool share & guardrails: max % of pool per position, checks on `SY:PT` market ratio, time-to-maturity gating, slippage/limit order configs.
        - Error handling: catch/propagate `MarketProportionTooHigh` vs. silent minOut failures.
    - **Red flags:**
        - Only single-sided add/remove is implemented (no path for **balanced** LP add/remove or two-leg unwind).
        - No cap on user/strategy **pool ownership** (e.g., >30–50%).
        - Unwind requires `swapExactPtForSy` immediately after `removeLiquiditySingleToken` with no mitigation if PT leg is too large.
        - No awareness of **expiry proximity** (rateScalar increases), or of SY/PT proportion checks before acting.
        - Assumes “just raise slippage” fixes reverts (it won’t for Pendle’s safety brakes).
    - **Fix pattern:**
        - **Prefer balanced LP flows:** use dual-leg add/remove (provide/withdraw PT+SY proportionally) or split path: remove to **PT+SY**, then route PT via aggregator/liquidity sources (e.g., 1inch/ParaSwap/other Pendle markets) instead of forcing `swapExactPtForSy` back into the same market.
        - **Position limits:** enforce per-strategy and per-user max % of market TVL; throttle single-sided adds; reject invests if post-trade SY/PT ratio or share exceeds thresholds.
        - **Maturity-aware policy:** taper/disable invests as expiry nears; mandate pre-unwind rebalancing (add temporary SY via flash-loan/add-LP) before partial removal; add a rescue path for liquidations that can **temporarily add LP** to neutralize proportions before withdrawing.
        - **Pre-checks & fallbacks:** simulate `swapExactPtForSy` and block if `MarketProportionTooHigh` is likely; implement alternative unwind routes (multi-hop, cross-market PT disposal) and partial claims that don’t exceed safe proportions.
        - **Monitoring:** alert on rising pool share, deteriorating SY/PT ratio, and time-to-maturity thresholds; gate liquidations with automated LP-assist if needed.
    - **Questions to ask next time:**
        - Do we ever hold >X% of a Pendle market? What is X, and is it enforced on-chain?
        - Does unwind **require** `swapExactPtForSy` into the same market, or do we have aggregator/cross-market routes for PT?
        - How do we behave as expiry approaches? Are invests/unwinds disabled or adjusted as rateScalar increases?
        - Do we check SY/PT proportions (pre- and post-action) and handle `MarketProportionTooHigh` with a fallback?
        - Can the liquidation bot **atomically** add LP (flash-loan SY), perform partial unwind, and repay—so liquidations can proceed even when the user holds a large, single-sided position?
- [ ]  **Front-run dust repayment breaks exact-amount liquidation (strict debt check DoS)**
    - **Symptom:** Liquidation requires an **exact** `_jUsdAmount <= borrowed(holding)` match. A borrower can **frontrun** a liquidator and repay **1 wei** (or dust) so that the liquidator’s prebuilt tx (repaying the old amount) now violates the check and **reverts**, DoSing liquidation at negligible cost.
    - **Where to look (CDPs/liquidations/repay paths):**
        - Liquidation entrypoints: `liquidate(...)`, `liquidateBadDebt(...)` — look for `require(_jUsdAmount <= borrowed(...))` or equality checks tied to a **caller-supplied exact amount**.
        - Debt reads & accrual: any `borrowed(holding)`/`debtOf(account, collateral)` fetched **before** transfers/accrual and not re-evaluated.
        - Repay functions: borrower-callable `repay(...)` that allow **tiny amounts** (no min-repay, no dust floor) at any time.
        - Multi-collateral paths: per-collateral borrowed tracking and liquidation functions that try to “repay full debt for this collateral” using a fixed amount.
        - Mempool assumptions: code that assumes debt won’t change between quote/signing and execution; no clamping to on-chain debt at execution.
    - **Red flags:**
        - `require(_jUsdAmount <= borrowedNow)` (or `==`) with `_jUsdAmount` coming from the **liquidator**, not computed internally from current state.
        - No `min()` clamp like `pay = min(_jUsdAmount, borrowedNow)`.
        - No partial-liquidation path (only “repay full debt” branch).
        - Repay accepts **1 wei** and has no dust threshold or “when liquidatable, min repay ≥ X” rule.
        - Interest accrual or per-block fees that can change `borrowedNow` between simulation and inclusion.
    - **Fix pattern:**
        - **Clamp at execution:** `uint256 debtNow = borrowed(holding); uint256 pay = Math.min(_maxRepay, debtNow);` Use `_maxRepay` param supplied by liquidator; never require exact equality.
        - **Support partial liquidation by default:** seize collateral **pro-rata** to `pay/debtNow`. Add `minCollateralOut` to protect liquidators.
        - **Dust rules:** enforce a **minimum repay unit** (per-token decimals) OR ignore/truncate residual dust (treat ≤ dust as zero) to prevent griefing with 1 wei.
        - **Re-read after side effects:** accrue interest/update accounting first, then read `debtNow`, then compute `pay`.
        - **MEV-hardening (optional):** allow liquidators to specify only bounds (`maxRepay`, `minSeize`) and compute exact amounts on-chain; encourage private tx where applicable.
    - **Questions to ask next time:**
        - Does liquidation require an **exact** amount from the caller, or does the contract compute `pay = min(maxOffered, currentDebt)`?
        - Can a borrower call `repay` for **arbitrary small** amounts at any time? Is there a dust floor or cooldown when liquidatable?
        - Are **partial liquidations** supported and safe across multi-collateral positions?
        - Is debt **accrued/re-read** inside the same tx before deciding `pay`, or is it taken from a stale precheck?
        - What protects liquidations from **mempool races** (front-run repay, interest tick) changing `borrowedNow` between simulation and inclusion?
- [ ]  **Fee-bypass via alternate exit path (liquidation instead of withdraw)**
    - **Symptom:** Users avoid protocol **withdrawal fees** by calling a liquidation route (e.g., `liquidate` / self-liquidate) that transfers collateral without charging the platform’s withdraw fee; only liquidator bonus (often waivable for self-liquidation) applies.
    - **Where to look (lending/CDP/liquidation managers):**
        - Withdrawal path: `HoldingManager.withdraw/_withdraw`, `manager.withdrawalFee()`, transfers to `feeAddress`.
        - Liquidation path(s): `LiquidationManager.liquidate(...)`, `liquidateBadDebt(...)`, and any “self-liquidate/close” helpers. Check how `collateralUsed` and `liquidatorBonus` are computed and whether **protocol fees** are applied.
        - Solvency gates: guards like `isLiquidatable()` or LTV checks—can liquidation be called while solvent?
        - Fee config: single global `withdrawalFee` vs separate `liquidationProtocolFee`/`liquidatorBonus`. Look for places where “fee = 0 if msg.sender == user”.
    - **Red flags:**
        - Liquidation functions that **transfer collateral directly** to user/liquidator without passing through the same fee hook used in `withdraw`.
        - Comments like “bonus waived for self-liquidation” with **no alternate protocol fee**.
        - Liquidation callable even when the position is solvent (or barely liquidatable), enabling an “exit” path.
        - Fee logic tied only to the **withdraw** codepath; liquidation path mentions only `liquidatorBonus` and debt math.
    - **Fix pattern:**
        - Unify exit economics: either
            - (A) **Disallow liquidation unless liquidatable** (strict insolvency gate) so solvent exits must use `withdraw`, **or**
            - (B) Apply a **protocol exit fee** on liquidation flows: `protocolFeeBps` taken from collateral transferred out (net of liquidator bonus); send to `feeAddress`.
        - Add explicit `liquidationProtocolFeeBps` to config; charge it for both third-party and self-liquidations (optionally different from `withdrawalFee`).
        - Route all collateral outflows through a shared `_collectProtocolFee(_token, _amount)` helper to prevent path-dependent fee gaps.
        - Unit tests: assert that for the same pre-state, **withdraw** and **self-liquidate** cost ≥ the configured fee; and that third-party liquidations net protocol fee + liquidator bonus.
    - **Questions to ask next time:**
        - Can a solvent user call `liquidate`/`selfLiquidate` to close and withdraw? If yes, what fee applies?
        - Is `liquidatorBonus` the **only** deduction on liquidation? Where is the **protocol** compensated?
        - Are bad-debt liquidations fee-free by design? If so, can users trigger that path while still solvent?
        - Do we have tests comparing effective fee paid via `withdraw` vs via (self-)liquidation across edge LTVs?
- [ ]  **Stale liquidatability check runs *before* collateral sync (healthy accounts can be liquidated)**
    - **Symptom:** `liquidate()` calls `isLiquidatable()` **before** pulling/settling invested collateral (e.g., `_retrieveCollateral` / strategy claims). After funds are retrieved, the position would be solvent, but liquidation proceeds because the check used **stale** collateral.
    - **Where to look (lending/CDP + strategy flows):**
        - Liquidation entrypoints: `LiquidationManager.liquidate(...)`, `liquidateBadDebt(...)`.
        - Solvency gates: `StablesManager.isLiquidatable(...)`, `isSolvent(...)`, `checkLtv(...)`, and where they read `SharesRegistry.collateral(holding)`.
        - Collateral sync path **inside liquidation**: calls like `_retrieveCollateral(...)`, `StrategyManager.claimInvestment(...)`, strategy `withdraw/unstake/redeem`, and any flags like `useHoldingBalance`.
        - Docs/comments listing “accepted risk: collateral changes not tracked while invested” — ensure liquidation **does** force a sync **before** the solvency decision.
    - **Red flags:**
        - `require(isLiquidatable(...))` at the **top** of `liquidate()`, followed later by `_retrieveCollateral(...)`.
        - No **second** solvency check after strategy funds are pulled/settled.
        - `isLiquidatable()` computed solely from on-ledger collateral that excludes invested balances or pending PnL.
        - Liquidation size/fees computed from pre-sync values (e.g., reading `collateralUsed` from stale reserves).
    - **Fix pattern:**
        - Move solvency check **after** collateral sync:
            1. Pull/settle strategy funds (or at least the minimal required amount), update accounting;
            2. Recompute `isLiquidatable()` and `borrowed/collateral` using **post-sync** state;
            3. Proceed only if still liquidatable.
        - Alternatively, have `isLiquidatable()` accept a **context** that includes in-flight retrieved amounts or a pre-liquidation “sync” routine that always runs first.
        - Add invariant tests: starting from invested state with positive yield, simulate liquidation -> assert that liquidation **reverts** post-sync if the account becomes solvent.
    - **Questions to ask next time:**
        - Does `isLiquidatable()` include invested assets or only on-holding balances?
        - In `liquidate()`, what changes state **between** the check and the collateral transfer? Is there any accounting sync in between?
        - Do strategies’ realized PnL apply to `collateral(holding)` **before** the solvency decision?
        - Are there scenarios (cooldowns, failed withdrawals) where partial syncs occur? Do we re-check solvency after *each* partial retrieval?
- [ ]  **Negative-yield accounting bricks liquidation (solvency check during collateral sync)**
    - **Symptom:** During liquidation, pulling funds from strategies realizes **negative yield** which triggers `removeCollateral(...)`. That call enforces `isSolvent(...)` mid-routine and **reverts**, leaving an under-collateralized position **unliquidatable** (bad debt persists).
    - **Where to look (lending + strategy stacks):**
        - Liquidation flows: `LiquidationManager.liquidate(...)`, `liquidateBadDebt(...)` → any `_retrieveCollateral` / `claimInvestment` step.
        - Strategy exit path: `StrategyManager.claimInvestment(...)` or strategy `withdraw(...)` returning `(withdrawn, invested, yield, fee)`; how **negative** `yield` is applied.
        - Collateral accounting: `StablesManager.removeCollateral(...)` / `addCollateral(...)`, `SharesRegistry.unregisterCollateral(...)`, and guards like `require(isSolvent(...), "…")`.
        - “Force” variants: presence/absence of `forceRemoveCollateral(...)` or a liquidation-only bypass for solvency checks.
    - **Red flags:**
        - `if (yield < 0) StablesManager.removeCollateral(…, abs(yield));` inside `claimInvestment` **even when caller is liquidation**.
        - `removeCollateral` performs `unregisterCollateral` **then** `require(isSolvent(...))`.
        - No context flag to distinguish **normal user actions** vs **admin/liquidation context**.
        - Strategies with withdrawal fees, slippage, cooldown penalties, or rounding that routinely produce small negative yields.
    - **Fix pattern:**
        - In liquidation context, **bypass solvency guard** for realized losses:
            - Use a `forceRemoveCollateral(...)` that updates shares **without** `isSolvent` (restricted to LiquidationManager), or
            - Apply negative yield as an **adjustment to liquidation math** (reduce distributable collateral / increase needed debt payment) and **defer** collateral accounting assertions until after debt is repaid, or
            - Temporarily “pad” collateral (admin-only add) before claim, run claim(s), then remove the pad; immediately re-check solvency post-liquidation.
        - Add **unit tests**: simulate invested collateral with −1% yield → ensure both `liquidate` and `liquidateBadDebt` succeed and bad debt decreases.
        - Audit all callsites that can realize negative PnL to ensure they don’t hard-require `isSolvent` mid-procedure.
    - **Questions to ask next time:**
        - Can any strategy exit path realize **negative yield** (fees, early-exit penalties, price impact)? How often?
        - Does the liquidation path **change accounting context** so that mid-procedure solvency checks are relaxed?
        - Is there a **single place** where `yield<0` is applied, or can strategies call `removeCollateral` themselves?
        - What is the order of operations: **(realize PnL) → (adjust collateral) → (solvency check) → (transfer collateral/debt)**? Where can this revert?
        - Do we have a **force/bypass** API restricted to LiquidationManager for exceptional flows (bad debt, negative PnL, partial retrievals)?
- [ ]  **Paused state blocks repayments but not post-unpause liquidations (unfair instant-liq window)**
    - **Symptom:** `repay()` is guarded by `whenNotPaused`, so users can’t cure risk while paused, but after unpause, `liquidate()` becomes callable immediately—letting liquidators nuke positions that drifted under water during the pause before borrowers can react.
    - **Where to look (protocol pause/guardian design):**
        - Gatekeeping modifiers on **repay/borrow/withdraw/liquidate** in `HoldingManager`, `LiquidationManager`, routers, and strategy managers (common pattern: OpenZeppelin `Pausable`).
        - Unpause paths: admin functions that flip `paused=false` on multiple modules in one tx; any **lack of grace period** or sequencing (e.g., unpausing LiquidationManager before HoldingManager).
        - Oracle & interest accrual during pauses (does utilization/interest accrue while users can’t repay? are oracle updates still live?).
        - Eventual consistency: mempool race where unpause + liquidation bundle beats user repayment txs.
    - **Red flags:**
        - `repay(...) whenNotPaused` + `liquidate(...) whenNotPaused` with **no post-unpause delay** or borrower cure window.
        - Separate pausers per module (HM vs LM) with **no coordinated unpause order** or atomic policy.
        - Pauses stop **user-beneficial** ops (repay, add collateral, self-liquidation) but allow **state drift** (oracle updates, interest accrual).
        - No “emergency-only” mode (e.g., disallow new borrows/withdraws but allow repay/add-collateral).
    - **Fix pattern:**
        - **Invert pause semantics**: in paused state, **allow health-improving ops** (repay, addCollateral, self-liquidate) and block risk-increasing ops (borrow, withdraw, strategy invest).
        - Add an **unpause thaw**: e.g., `LIQUIDATION_THROTTLE_BLOCKS` or `unpauseAt = block.timestamp; require(now >= unpauseAt + GRACE)` before any liquidation.
        - Orchestrate **unpause ordering**: unpause `HoldingManager` first, then `LiquidationManager` after `GRACE` (enforced on-chain).
        - Provide an **EmergencyRepay**/`whenCureAllowed` modifier that bypasses `whenNotPaused` strictly for `repay/addCollateral/selfLiquidate` (reentrancy-guarded).
    - **Questions to ask next time:**
        - During a pause, which operations remain available to **reduce risk**? Is there a one-tx path for users to cure?
        - What happens **the first block** after unpause? Can a liquidator atomically front-run a user’s repay?
        - Do oracles and interest continue updating while users can’t act? Is there a compensating mechanism?
        - Is there a **grace/cooldown** or **sequenced unpause** encoded on-chain—not just an ops runbook?
        - Can self-liquidation be permitted during pause to convert collateral → debt repayment without third-party liquidators?
- [ ]  **Missing pre-withdraw balance snapshot inflates yield (counts existing funds as “withdrawn”)**
    - **Symptom:** Strategy computes `withdrawnAmount` as `IERC20(tokenIn).balanceOf(_recipient) - params.balanceBefore`, but `balanceBefore` is **unset/zero** (or taken from the wrong account). Any tokens already sitting in the recipient/holding are miscounted as newly withdrawn “yield,” inflating collateral and enabling under-collateralized borrowing.
    - **Where to look (strategies/claim flows/registries):**
        - Strategy `withdraw/claimInvestment` paths (e.g., `ElixirStrategy.withdraw`, `StrategyManager.claimInvestment`) that:
            - Track `balanceBefore/balanceAfter` in a **local struct** (defaults to 0 if not explicitly set).
            - Credit yield to collateral via `StablesManager.addCollateral(...)` using the computed `withdrawnAmount`.
        - Token flow targets: is swap/router output sent **directly to the holding** or to the **strategy contract** first? (Delta should match the actual receiver.)
        - Any fee logic using `withdrawnAmount` (performance/withdraw fees) that would amplify the miscount.
        - Tests missing a case where the holding starts with a **non-zero** `tokenIn` balance.
    - **Red flags:**
        - `params.balanceBefore` (or similar) **never assigned** before swaps/unstake; struct declared in memory with default zeros.
        - Using `_recipient` for balance deltas when swaps actually pay the **strategy** (or vice-versa).
        - Yield credited as `withdrawnAmount - principal` without guarding against **pre-existing balances** or reentrancy/top-ups.
        - Collateral registry increases even when the net token delta to the system is \~0.
    - **Fix pattern:**
        - **Snapshot first**: set `balanceBefore = IERC20(tokenIn).balanceOf(actualReceiver)` **before** any external calls; compute `withdrawnAmount` against the **same address** that receives the tokens.
        - Prefer **router-returned `amountOut`** over balance-delta when possible (single-hop known pool), and route outputs to the strategy, then **explicitly transfer** the known amount to the holding.
        - Clamp yield: `yield = max(int(withdrawnAmount) - int(principalPortion), 0)` (no negative-to-positive wrap); assert `withdrawnAmount >= minExpected`.
        - Add unit/invariant tests: withdrawing with a **pre-funded holding** must not change collateral except by true net inflow.
    - **Questions to ask next time:**
        - Which address actually receives swap/redemption proceeds? Are deltas measured on that exact address?
        - Is `balanceBefore` always set before **any** external call? Could a memory struct default to 0?
        - Can users top-up the same token in the middle of the flow (directly or via reentrancy), and would that be miscounted?
        - Does the registry credit collateral strictly on **net** inflow, or on raw `withdrawnAmount`?
        - Are there tests with: (a) pre-existing holding balance; (b) partial withdraw; (c) multiple sequential withdraws?
- [ ]  **Rewards-claim path bypasses per-holding fee override (uses default strategy fee)**
    - **Symptom:** `claimRewards()` deducts fees using `StrategyManager.strategyInfo(address(this)).performanceFee` and **ignores** the `FeeManager.getHoldingFee(holding,strategy)` override. Result: custom per-holding fee schedules don’t apply on reward claims (AaveV3StrategyV2, PendleStrategyV2), while `withdraw()` correctly calls `_takePerformanceFee(...)`.
    - **Where to look (yield strategies with rewards):**
        - Strategy V2 contracts’ `claimRewards()` (e.g., `AaveV3StrategyV2`, `PendleStrategyV2`): look for `strategyInfo(...).performanceFee` reads.
        - Base fee helpers (`StrategyBaseUpgradeableV2::_takePerformanceFee`) and `FeeManager.getHoldingFee(...)` logic; ensure both **claims** and **withdrawals** route through the same helper.
        - Manager wiring: `feeManager` presence in initializers / upgrade steps and events `FeeTaken(...)`.
        - Any other reward flows: auto-claim on deposit/withdraw, compounding, third-party `redeemRewards()` wrappers.
    - **Red flags:**
        - `claimRewards()` computes `fee = getFeeAbsolute(claimed, performanceFee)` where `performanceFee` comes from **strategyInfo** instead of `feeManager.getHoldingFee(holding,strategy)`.
        - Inconsistent paths: `withdraw()` uses `_takePerformanceFee` but `claimRewards()` re-implements fee logic.
        - Multiple reward tokens loop where fee is applied per token without consulting `feeManager` or recipient is hardcoded.
        - Fee tests only cover default fee, none with **custom holding fee** set.
    - **Fix pattern:**
        - Replace in `claimRewards()`:
            - ❌ `(uint256 performanceFee,,) = _getStrategyManager().strategyInfo(address(this));`
            - ✅ `uint256 feeBps = feeManager.getHoldingFee(_recipient, address(this));`
            - Apply fee via the **same** helper as withdrawals: `fee = _takePerformanceFee(rewardToken, _recipient, claimedAmount)` (or inline the same logic), subtract from `claimedAmounts[i]`.
        - Unit tests:
            - Set `strategyInfo.performanceFee = 0`, set `feeManager.setHoldingCustomFee(holding,strategy, X)`; assert fee taken equals **X**, not default.
            - Multiple reward tokens: verify each applies the override.
            - No custom fee set ⇒ falls back to default strategy fee.
            - Regression: ensure no **double-fee** (claim + withdraw) on the same yield; fee should be on net reward credited.
    - **Questions to ask next time:**
        - Do **all** yield entry points (claim, auto-compound, withdraw) use a **single** fee application helper?
        - What’s the precedence: per-holding override vs default? Is it tested?
        - Are fees on **rewards** and fees on **principal/yield** during withdraw applied consistently (same basis, same rounding)?
        - Are there strategies with **non-standard reward flows** (Aave claim via RewardsController, Pendle `redeemRewards`, etc.) that skipped the shared helper?
        - Should fee be taken **in-kind** per reward token or converted first (and where is that defined)?
- [ ]  **Self-liquidation fee is based on manipulable swap input (path/price gaming)**
    - **Symptom:** `selfLiquidate()` computes the fee from **collateral used in the swap** (e.g., `finalFee = amountIn * selfLiquidationFeeBps / 1e3`). Because the user controls the swap path/price (e.g., via sandwiching or malicious LPs), they can momentarily inflate the collateral’s price so that `amountIn` shrinks, drastically reducing the fee—while still burning the same jUSD debt.
    - **Where to look (liquidations/swap modules):**
        - Liquidation entrypoints (e.g., `LiquidationManager.selfLiquidate`): locate where `amountIn/amountOut` are read from swaps and where the fee is computed.
        - Swap helpers/routers used by self-liquidation (e.g., `SwapManager.swapExactOutput*`, `validPool`/path validators): check if the user provides `_swapPath`, whether pools/tokens are whitelisted, whether the **last hop is the collateral**, and if TWAP or oracle checks exist.
        - Fee plumbing: `selfLiquidationFee`, precision math, and whether fees are taken on **swap input** vs **jUSD repaid** vs **oracle-valued notional**.
        - Mempool/MEV exposure: any reliance on spot pool state pre-/post-swap; lack of guard against sandwiches.
    - **Red flags:**
        - Fee formula uses `collateralUsedForSwap`/`amountIn` from a user-controlled path.
        - Path validator only checks first hop (e.g., jUSD) or “has liquidity” but not that **tokenIn == collateral** or pool is whitelisted.
        - Multihop allowed; no pool allowlist; no TWAP/oracle bound; slippage set by user.
        - No minimum fee floor (e.g., bps of jUSD burned) and no comparison vs oracle-priced value.
        - Comments/docs say fee is “8% of collateral used” rather than “% of debt repaid” or “% of oracle notional”.
    - **Fix pattern:**
        - **Base fee on invariant not under user price control:**
            - Charge on **jUSD amount burned** (e.g., `fee = jUSDBurned * feeBps / 1e4`), or
            - Charge on **oracle-valued notional**: `fee = (jUSDBurned * 1e18 / price(collateral)) * feeBps / 1e4`, or on `min(oracleQuote, amountInSpot)`.
        - **Constrain price/path manipulation:**
            - Require `tokenIn == collateral`, single hop, and **whitelisted** pool(s) with TWAP bounds or priced-oracle checks.
            - Alternatively, route via a protocol-owned/curated router that enforces **trusted pools** and TWAP checks.
        - **MEV hardening:** compute fee from pre-swap **debt amount**; apply fee before swap or as separate transfer independent of path; optional min absolute fee.
        - **Tests:** simulate self-sandwich and malicious LP path; assert fee equals bps of jUSD repaid even if `amountIn` deviates; fuzz multihop paths and ensure fee stability.
    - **Questions to ask next time:**
        - Is the self-liquidation fee derived from **amountIn** or from **debt repaid (jUSD burned)**? If amountIn, why is that robust against price manipulation?
        - Can the caller pick an arbitrary `_swapPath`? Are pools/tokens **allowlisted** and does validation ensure the **final token is the collateral**?
        - Are there TWAP/oracle clamps tying the swap result to an external price, or are we trusting spot?
        - Do we have a **minimum fee** (bps of jUSD repaid) to avoid dust-in tricks?
        - Should `repay()+withdraw` vs `selfLiquidate()` incur **economically equivalent** total fees? If not, what prevents fee arbitrage between the two paths?
- [ ]  **Partial-liquidation bonus lets liquidators strip collateral and strand bad debt**
    - **Symptom:** `liquidate()` allows any `_jUsdAmount <= borrowed` and pays a liquidation bonus while removing collateral via `forceRemoveCollateral` (no solvency check). A liquidator can repeatedly/partially liquidate **only up to the victim’s current collateral**, collect the bonus, and leave the account with **residual (bad) debt and zero collateral**.
    - **Where to look (liquidations/accounting):**
        - Liquidation entrypoint(s): `LiquidationManager.liquidate`, `liquidateBadDebt`. Trace: validate → pull from strategies → compute `collateralUsed` → pay bonus → `forceRemoveCollateral`.
        - Solvency gates: whether post-liquidation `isSolvent()` (or health factor) is enforced; use of `forceRemoveCollateral` vs `removeCollateral`.
        - Amount bounds: checks like `require(_jUsdAmount <= borrowed)` and absence of “must restore solvency” or “must zero debt/collateral” constraints.
        - Bonus math and source: how `liquidatorBonus` is applied (from collateral) even if debt remains after the operation.
    - **Red flags:**
        - No **post-state** solvency/health check after collateral removal and debt burn.
        - Liquidation bonus applied unconditionally (including when the account remains insolvent).
        - No minimum liquidation amount tied to restoring solvency/target HF; liquidator chooses `_jUsdAmount`.
        - Use of `forceRemoveCollateral` in normal liquidation paths (skips solvency).
        - Multiple-collateral setups where liquidator targets the richest collateral only.
    - **Fix pattern:**
        - Enforce **post-liquidation** condition: require `!isLiquidatable(...)` (or HF ≥ threshold) after applying `_jUsdAmount`, collateral removal, and bonus; otherwise revert.
        - Or require `_jUsdAmount` ≥ amount needed to reach target HF (compute from oracle prices) for the selected collateral set.
        - Disable bonus when the operation **does not** clear insolvency; or cap bonus so the account cannot end with zero collateral + residual debt.
        - Prefer `removeCollateral` (with solvency check) except inside an explicit **bad-debt** flow where bonus is zeroed and accounting routes to `liquidateBadDebt`.
        - Add integration tests: partial liquidation on an under-collateralized account must either (a) restore solvency or (b) revert; test multi-collateral edge cases.
    - **Questions to ask next time:**
        - After liquidation, do we **check** that the account is solvent / HF ≥ 1? Where?
        - Is the liquidation bonus suppressed or adjusted if the account remains insolvent?
        - Why is `forceRemoveCollateral` used in standard liquidations? Can we confine it to admin/bad-debt paths?
        - What’s the formula for “minimum repay to restore solvency”, and is `_jUsdAmount` constrained to it?
        - In multi-collateral scenarios, can a liquidator cherry-pick one asset to drain collateral while leaving net debt outstanding? How is that prevented?
- [ ]  **Bad-debt sweeper stops at registry principal, leaving strategy yield behind**
    - **Symptom:** In `liquidateBadDebt`, collateral retrieval targets `SharesRegistry.collateral(holding)` (principal only). The loop breaks once the holding’s on-chain balance ≥ that target, so **later strategies aren’t harvested** and any **unrealized positive yield** remains stranded in strategies. Liquidator/admin recovers less than the user actually has invested.
    - **Where to look (liquidations/strategy pulls/registry):**
        - `LiquidationManager.liquidateBadDebt` → `_retrieveCollateral` call: check `_amount` parameter and `useHoldingBalance` early-exit.
        - `_retrieveCollateral` loop: condition like `if (useHoldingBalance && IERC20(_token).balanceOf(_holding) >= _amount) break;`.
        - `SharesRegistry.collateral` semantics: does it exclude strategy-side accrued yield until `claimInvestment` runs?
        - Strategy withdraw path: `StrategyManager.claimInvestment` return tuple, and how positive `yield` is booked only **after** withdrawal.
        - Multi-strategy holdings: iteration order and whether all `shares` are redeemed.
    - **Red flags:**
        - `_amount` set to `registry.collateral(holding)` for bad-debt recovery instead of “withdraw all”.
        - Early `break` based on **holding balance** rather than **exhausting strategy shares**.
        - No assertion that all strategy `shares` for the holding became zero after the loop.
        - Assumption that registry value == realizable balance even while funds are invested.
    - **Fix pattern:**
        - In bad-debt flows, **harvest all strategies unconditionally**: pass `_amount = type(uint256).max` or ignore `_amount` and redeem **all shares** per strategy (query `recipients(holding)` / strategy share balance).
        - Remove/disable the `useHoldingBalance` early-exit for admin bad-debt paths; only stop when **all provided strategies are drained**.
        - Post-condition checks: assert remaining shares == 0 for each strategy; optionally assert `holdingBalance + unrealizedResidual == 0`.
        - Tests: (1) two strategies with profit; ensure both are harvested; (2) large profit in last strategy—verify no early exit; (3) gas-bound loops still complete across reasonable strategy counts.
    - **Questions to ask next time:**
        - Does your registry track **accrued yield while invested**, or only principal? If only principal, why is it used as the retrieval target?
        - Can `_retrieveCollateral` exit before all strategy shares are redeemed? Under what condition?
        - After bad-debt liquidation, how do you verify no strategy positions remain for the account?
        - Is there a dedicated “withdraw-all” code path or sentinel for admin liquidations?
        - How is iteration order over strategies defined—can ordering cause systematic under-recovery?
- [ ]  **Borrow mints stablecoin using its own market price (self-referential peg)**
    - **Symptom:** `borrow()` computes `jUsdMintAmount = collateralUSD * 1e18 / getJUsdExchangeRate()`. If jUSD trades **below \$1** (e.g., 0.90), users mint **> \$1** of jUSD per \$1 collateral (e.g., \$1,000 → 1,111 jUSD), amplifying the depeg and risking a death spiral. TWAP lag can also mint at stale, favorable prices then trigger instant liquidation.
    - **Where to look (stablecoin/lending core):**
        - Lending entrypoints: `HoldingManager.borrow`, `depositAndBorrow`, any “mint jUSD” paths.
        - Peg plumbing: `Manager.getJUsdExchangeRate()` and its oracle; look for Uniswap V3 TWAP oracles or anything returning jUSD/USDC.
        - Math sites like: `jUsdMintAmount = amountUSD * PRECISION / jUsdRate` (or equivalent `mulDiv` forms).
        - Config/feature flags: toggles that swap between a fixed `1e18` oracle and a market-based oracle.
        - Liquidation checks right after borrow (stale oracle + borrow at discount → immediate liquidatable).
    - **Red flags:**
        - Any borrow/mint path that divides by **jUSD price** instead of treating \$1 as constant.
        - Comments like “convert USD value to jUSD using oracle rate”.
        - jUSD oracle wired to DEX TWAP; presence of update windows and `peek()` returning ≠ `1e18`.
        - Tests missing a case where `jUsdRate < 1e18` leads to **more** jUSD minted than USD collateral.
    - **Fix pattern:**
        - **Decouple minting from jUSD market price.** Mint exactly `collateralUSD` (scaled) regardless of jUSD/USD oracle; i.e., set `jUsdMintAmount = collateralUSD`.
        - If you must consult a rate, **floor** it at `1e18` for minting (never mint > USD collateral). Optionally **cap** upside too to avoid under-minting on >\$1 spikes.
        - Add **circuit breakers**: halt borrowing when `jUsdRate < threshold` or oracle not fresh.
        - Peg management lives in **stabilizers** (redemptions, AMO/PSM, buybacks), **not** the borrower mint formula.
        - Tests: set jUSD oracle to `0.9e18` and assert minted == collateralUSD (post-fix) and that borrow doesn’t become instantly liquidatable due to stale TWAP.
    - **Questions to ask next time:**
        - Does any borrow/mint path ever use **jUSD price**? Why not a constant \$1?
        - What’s the behavior if the jUSD oracle < \$1 or stale—do we **block** borrow or clamp?
        - Are there stabilizers (PSM, keeper arbitrage) separated from **mint logic**?
        - Could TWAP lag let users mint then become liquidatable immediately? How is this prevented?
        - Are there invariants/tests asserting `mintedJUSD ≤ collateralUSD` for all oracle states?
- [ ]  **sUSDS staleness threshold = heartbeat ⇒ spuriously “stale”, branch perma-shutdown**
    - **Symptom:** In deployment, the sUSDS Chainlink feed’s **stalenessThreshold equals the heartbeat** (24h). Any routine delay (e.g., 20–30s mining lag after the 24h mark) makes `block.timestamp - answer.timestamp ≥ threshold`, tripping the **stale** path in `MainnetPriceFeedBase` and invoking shutdown logic. Because this branch is immutable/non-reenable, an attacker can repeatedly call the shutdown entrypoint shortly after each heartbeat tick to **permanently disable the sUSDS branch** and extract \~2% via urgent redemptions.
    - **Where to look (deploy/oracles/price-guards):**
        - Deploy scripts: `DeployUSDaf.s.sol` sUSDS case (`_stalenessThreshold` set to 24h despite name `_1_HOUR`), and any env var overrides.
        - Oracle impl & docs: `SusdsOracle.sol` (intended **48h** threshold / grace vs heartbeat), Chainlink aggregator addresses & `latestRoundData()`.
        - Price feed base: `MainnetPriceFeedBase.sol` staleness check and `_shutDownAndSwitchToLastGoodPrice()` callsites (who can call, when).
        - Branch lifecycle: code that prevents reenabling once shutdown; urgent redemption paths that pay \~2% on shutdown.
    - **Red flags:**
        - **threshold == heartbeat** (no grace buffer); naming mismatches like `_1_HOUR = 24 hours`.
        - Single-source oracle with **no fallback**; shutdown callable by public/keepers **without hysteresis**.
        - No “two-strike” policy (e.g., one stale observation immediately shuts the branch).
        - Tests only cover “way past stale” (hours) but not **edge window** (seconds past heartbeat).
    - **Fix pattern:**
        - **Set threshold > heartbeat** (e.g., `threshold = 2 * heartbeat` or **≥ heartbeat + safety margin (e.g., +5–10 min)**). If `SusdsOracle.sol` specifies **48h**, align deploy script to 48h.
        - Add **hysteresis**: require **two consecutive stale reads** (separated by ≥ X minutes) before shutdown; or enforce **min stale duration** (e.g., `now - ts ≥ heartbeat + 2 minutes`).
        - Gate shutdown: restrict to guardian and/or **rate-limit** shutdown calls; log reason with feed timestamps for post-mortem.
        - Prefer **fallback sources** (e.g., secondary Chainlink feed or TWAP) for liveness checks; only hard-shutdown when **all** are stale.
        - Tests: simulate just-over-heartbeat latencies (±60s), network congestion, and verify **no shutdown** within the grace window; simulate true outages to confirm shutdown still triggers.
        - If branch is immutable once down, consider a **new deployment** with corrected threshold and a migration/allowlist plan for sUSDS users.
    - **Questions to ask next time:**
        - What’s the **feed heartbeat** and typical **post-heartbeat publish latency** for this market? What buffer do we apply?
        - Can anyone call the shutdown path? Do we **require repeated confirmation** or multi-sig approval?
        - Do we have a **fallback oracle** or “last good price” **without** irreversible shutdown?
        - Is there a tested **reenable**/migration path if a branch gets shut down by a transient delay?
        - Are CI tests covering the **edge case** where `now - ts ∈ [heartbeat, heartbeat + 60s]` and proving we don’t nuke the branch?
- [ ]  **Cross-asset rate clamp misprices redemptions (bond market rate applied to leverage token)**
    - **Symptom:** Redemption for a leverage/share token is **capped by a market rate meant for the bond/debt leg**, so `min(redeemRate, marketRate)` always selects the (lower) **bond** quote → leverage holders get underpaid (e.g., 1.0 ETH expected vs ~0.16 ETH paid).
    - **Where to look (split redemptions / dual-token vaults):**
        - Redemption math helpers: `Pool.getRedeemAmount`, `simulateRedeem`, `redeem()/withdraw()` paths that compute `redeemRate` or `pricePerShare`.
        - Any **oracle adapter** returning a single `marketRate` (e.g., `BondOracleAdapter`, AMM TWAP) consumed by **both** token types (Bond/Lev, Senior/Junior, Stable/Vol).
        - Call sites passing `marketRate` into pricing functions for **all** token types.
        - Enum/branch handling: `if (tokenType == BOND) {...} else /* LEVERAGE */ {...}` followed by a **shared** post-processing clamp like:
            
            ```solidity
            if (marketRate != 0 && marketRate < redeemRate) redeemRate = marketRate;
            
            ```
            
    - **Red flags:**
        - A **single** external `marketRate` variable/arg applied **unconditionally** to multiple assets with different payoff profiles.
        - Comments/ABI naming implying **Bond** rate, but code uses it for **Leverage** too.
        - Lack of **type tagging** on rates (no struct like `{rate, assetType}`) and no assertion that the rate’s asset matches `tokenType`.
        - Mismatched scales: bond target price near par (~1e8) while leverage formula produces materially higher rates (e.g., ~6.25e8); code still clamps to the lower one.
        - Tests only cover bond redemption with market cap; **no test** for leverage redemption when `marketRate` is non-zero.
    - **Fix pattern:**
        - **Scope the cap to the right asset:**
            
            ```solidity
            if (tokenType == TokenType.BOND && marketRate != 0 && marketRate < redeemRate) {
                redeemRate = marketRate;
            }
            
            ```
            
        - Alternatively, **pass `marketRate = 0`** for leverage paths, or fetch a **separate Lev market rate** (if one exists).
        - Type-safe inputs: wrap rates in a struct `{uint256 rate; TokenType quotedFor;}` and `require(quotedFor == tokenType)`.
        - Add invariants/tests: Lev redemption with `marketRate!=0` remains ≥ formulaic value; Bond redemption respects market cap; decimals/precision aligned.
    - **Questions to ask next time:**
        - “Is this `marketRate` explicitly a **bond** price? If so, why is it applied to **other token types**?”
        - “Do we have **separate market quotes** per asset (Bond vs Lev/Senior vs Junior)? If not, should non-bond legs skip the clamp?”
        - “Are `redeemRate` and `marketRate` in the **same units/precision**? What prevents always picking the lower one?”
- [ ]  **Post-trade gating lets sellers force “par” redemption (use-after-mutation in pricing)**
    - **Symptom:** Redemption logic decides tiers (e.g., par vs formula) using a **forward-looking collateral metric** that already subtracts the user’s `depositAmount`/burn (`(tvl - amount*price) / (bondSupply - amount)`), so a seller can push the system **into the favorable tier** and redeem at the **max price** (e.g., $100) even if the **pre-trade** state was below the threshold. Net result: buy below par, redeem at par → extract reserve.
    - **Where to look (tiered AMMs / split-token pools):**
        - Pricing helpers for redemptions/creates: `getRedeemAmount/previewRedeem`, `simulateRedeem`, `create/previewCreate`.
        - Collateral‐level / health‐ratio gates: formulas like
        `collateralLevel = (tvl * PRECISION) / (bondSupply * targetPrice)` **vs**`collateralLevel = ((tvl - depositAmount*targetPrice) * PRECISION) / ((bondSupply - depositAmount)*targetPrice)`.
        - Hard tier switches: `if (collateralLevel <= THRESHOLD) { formulaRate } else { PAR_RATE }`.
        - Any **marketRate clamp** applied **after** tiering and shared across token types.
    - **Red flags:**
        - Gate predicate uses **post-trade** state (`depositAmount`, `shares`, `amount*price`) to select tier.
        - Create path gates on **pre-state**, but redeem path gates on **post-state** (asymmetry).
        - All-or-nothing tiering (no piecewise across the threshold) with a **flat par** rate when above threshold.
        - Profitability in a two-step: buy to move near threshold → redeem entire position at par.
    - **Fix pattern:**
        - **Gate on pre-trade state** (snapshot before applying the order) for tier selection; or
        - Implement **piecewise pricing**: split the redemption across tiers (portion priced with sub-threshold formula until threshold is reached, remainder at par).
        - Disallow a single call to both **flip the tier and benefit** from the higher tier for the full amount (cap the par-priced portion).
        - Keep clamps asset-specific; apply `marketRate` sanity checks **per token** and after piecewise calculation.
        - Add tests: (1) boundary redemption just crossing the threshold must not realize full par; (2) round-trip buy→redeem profit ≤ fees; (3) invariants under multiple sequential orders.
    - **Questions to ask next time:**
        - “Is the **tier gate** computed from **pre-trade** state, or can the trade itself push the system into a better tier?”
        - “Do we price **piecewise** across thresholds, or do we apply a single tier to the whole amount?”
        - “Are `create` and `redeem` gates **symmetric** in their state snapshot?”
        - “Does a shared `marketRate` clamp accidentally legitimize par pricing for assets it’s not designed for?”
- [ ]  **Epoch/period drift: write under N, read under N+1 → auth check reverts**
    - **Symptom:** Objects (auctions/rounds/tranches) are stored in a mapping keyed by `currentPeriod`, but the code **increments the period** after creation. Later flows (e.g., `transferReserveToAuction/finalize/distribute`) look up `auctions[currentPeriod]` again and perform `require(msg.sender == auctions[currentPeriod])`, which now points to **empty/next slot**, causing a revert and bricking the flow.
    - **Where to look (epoched systems / AMMs with rounds / tokenized tranches):**
        - Factories/creation paths that do both:
            - `auctions[currentPeriod] = new Auction(...);`
            - followed by a period advance like `bondToken.increaseIndexedAssetPeriod(...)`, `advanceEpoch()`, `rollover()`.
        - Read paths that authorize by **re-reading** period from an external contract:
            - `(_period, ) = bondToken.globalPool();`
            - `require(msg.sender == auctions[_period], "CallerIsNotAuction");`
        - Inconsistent helpers: one function uses `period-1` (e.g., `distribute()`), another uses `period` directly.
    - **Red flags:**
        - Use of **mutable external period** as the sole key, with no snapshot stored alongside the created object.
        - Creation code mutates period **after** writing the mapping.
        - Authorization based on `msg.sender == mapping[currentPeriod]` instead of `msg.sender == storedAddress`.
        - Comments like “increase period” right after storing `auctions[currentPeriod]`.
        - Lack of tests that call finalize/transfer **after** a period increment.
    - **Fix pattern:**
        - **Snapshot the period at creation** (`uint256 periodAtCreation = currentPeriod;`) and:
            - Store it in the auction and/or use `auctions[periodAtCreation]` for all future lookups; or
            - Defer period increment until after **all** references are wired to the new index; or
            - Derive the correct key on read (e.g., `previousPeriod = currentPeriod - 1`) if design demands increment-after-create.
        - Prefer **address-based auth** (store the auction address and compare directly) instead of recomputing the key.
        - Add invariants/tests: (1) start → advancePeriod → finalize succeeds; (2) mixed `distribute/transfer` functions use the **same** keying rule.
    - **Questions to ask next time:**
        - “Which index is canonical for an object: the **pre-increment** or **post-increment** period?”
        - “Do any read paths recompute the key from an **external** source that may have advanced?”
        - “Why does one function use `period-1` while another uses `period`? Is that intentional and tested across boundaries?”
        - “Can anyone advance the period between create and finalize? What if the call is delayed to the next block/epoch?”
- [ ]  **Pool-token order & decimals mismatch in TWAP adapter returns mis-scaled/inverted price → marketRate underprices redemptions**
    - **Symptom:** Oracle adapter pulls a Uniswap-style pool and **returns the raw price orientation** (tokenA/tokenB) while **hardcoding output decimals to the asset’s decimals**. If the asset is not the quoted side in the pool, the adapter returns a value in the **wrong units/scale**, e.g., “reserve per bond” with **bond decimals**, yielding a market rate that’s far too low. Any consumer logic like `if (marketRate != 0 && marketRate < redeemRate) redeemRate = marketRate;` will then **clip to the bad price**, causing large user losses on redeem.
    - **Where to look (oracle adapters / pricing gates / AMM integrations):**
        - Oracle adapter init & pool wiring:
            - `initialize(...)` → how the pool address is fetched (e.g., `factory.getPool[tokenA][tokenB][...]`) and whether **token order is canonicalized** (sorted) or an **invert flag** is stored.
            - Factories that populate `getPool[a][b] = pool; getPool[b][a] = pool` (both directions) without signaling orientation.
        - Pricing path:
            - `getSqrtTwapX96` → `getPriceX96FromSqrtPriceX96` → **is price inverted if asset != pool.quote**?
            - Any decimal handling: adapter’s `decimals()` return; conversion from X96/X128; adjustments for token decimals differences.
        - Downstream consumers:
            - Redemption/cash-out functions that **prefer market rate** when it’s lower than a model rate (e.g., `Pool.getRedeemAmount` with `marketRate < redeemRate` override).
            - Any “min(price, model)” or “clamp to oracle” logic that can propagate a mis-scaled oracle into payouts.
    - **Red flags:**
        - Adapter sets `decimals = asset.decimals()` unconditionally, while returning **raw pool price** (token0/token1) with no inversion/decimal normalization.
        - No boolean like `assetIsToken1` / `invert` persisted on init.
        - No test that flips token address order (asset < reserve vs asset > reserve) and asserts identical quoted price.
        - Assets and reserves have **different decimals**, but adapter never adjusts for `decimals(token0) ≠ decimals(token1)`.
        - Consumer code compares `marketRate` vs `redeemRate` and **blindly picks the lower** without validating units/scale.
    - **Fix pattern:**
        - **Canonicalize orientation** in the adapter:
            - Sort `(token0, token1)` on init; set `invert = (asset == token0)` (or whichever orientation you standardize).
            - When computing price, **invert** the X96 price if needed so output is always “asset quoted in reserve” (or vice versa), and **document** that contract’s price semantics.
            - Normalize for decimals: scale by `10**(decimals(quote) - decimals(base))` so returned price uses a **declared, constant precision** (e.g., 1e18).
            - Set `decimals()` to a **fixed oracle precision** (e.g., 18) or to the **quote unit you documented**, not blindly to `asset.decimals()`.
        - In consumers:
            - Validate unit consistency once (e.g., require adapter precision = 1e18).
            - Prefer **band checks** over unconditional min: ensure `marketRate` is sane (non-zero, within ±X% of model/TWAP), else ignore.
        - Tests:
            - Parameterize tests over both token orders and disparate decimals (e.g., 6 vs 18) and assert identical redeem outputs.
    - **Questions to ask next time:**
        - “What exactly does the oracle return—**quote/base or base/quote**? In what **fixed precision**?”
        - “If the pool returns token0/token1, do we **invert** when the asset is token0? Where is that flag stored?”
        - “Are we adjusting for **decimal differences** between tokens before emitting the price?”
        - “Do any consumers assume the oracle is in **asset decimals**? Should we standardize to 1e18 instead?”
        - “Factory returns a pool for both `[a][b]` and `[b][a]`. How do we guarantee we interpret orientation **consistently** across deployments?”
- [ ]  **Discrete, post-hoc fee extraction lets redeemers dodge liabilities (“free-rider” exit)**
    - **Symptom:** Fees owed from **leverage holders → bond holders** are collected **later** (e.g., via a quarterly auction). Until the auction finalizes and transfers funds out, redeemers compute their payout from **full pool reserves/TVL**, so a leverage user can **withdraw just before fee transfer** and avoid paying their period share—shifting the entire burden to remaining holders.
    - **Where to look (fee accrual/settlement & redemption paths):**
        - Fee mechanics: period timers, auction/settlement modules, and when reserves are **actually decremented** (e.g., `transferReserveToAuction` after auction end).
        - Redemption math for the liable side (e.g., **lev** token): formulas using `tvl`, `assetSupply`, `bondSupply` that **don’t subtract pending fees** (e.g., `redeemRate = ((tvl - bondSupply*TARGET)/assetSupply)*PRECISION`).
        - Any **“pendingFees”, “accruedFees”, “feeBuffer”** fields and whether they’re deducted from TVL in redeem/mint.
        - User-level accounting: presence/absence of **cumulative fee index / checkpoint** (`userIndex`, `globalIndex`) to charge each user’s **pro-rata accrued** fee on exit.
        - Withdrawal gates around settlement windows: cooldowns, snapshots at period start/end, or **withdraw-while-auction-pending** allowances.
    - **Red flags:**
        - Fees are **recognized only at auction finalization**; until then, TVL used in redeem math includes the portion destined for bond fees.
        - No global **fee liability index** and no per-user `lastIndex` → exits don’t settle accrued obligations.
        - No **pending fee reserve** reducing redeemable reserves; `tvl`/reserves stay whole until funds are transferred.
        - Long periods (e.g., quarterly) with **unrestricted withdrawals** right up to settlement.
        - Consumers of the pool (UIs/contracts) compute `redeemAmount` without subtracting **unpaid period fees**.
    - **Fix pattern:**
        - **Accrue continuously**: maintain a global **feePerShare index** (per second/block) and charge each redeemer `shares * (globalIndex - userIndex)` on exit; update `userIndex` on mint/redeem.
        - **Pre-reserve fees:** when a period starts (or as they accrue), increment `pendingFees` and **exclude it from TVL** in all pricing/redemption paths; optionally hold in a dedicated **fee buffer**.
        - **Snapshot & pro-rata**: snapshot supply at period start; on redeem, take a proportional cut for the period’s fee even if auction not finalized.
        - **Window controls:** add a short **cooldown/blackout** before auction end or apply an **exit fee equal to accrued liability** during the pending window.
        - In settlement, transfer from **fee buffer** not from general reserves, so redeemers cannot evade by timing.
    - **Questions to ask next time:**
        - “When are fees **accrued vs. settled**? Are reserves/TVL reduced **before** redemptions can use them?”
        - “Do we have a **cumulative fee index** so each exit pays its accrued share regardless of timing?”
        - “Does the redeem formula **subtract pending/accrued fees** from TVL?”
        - “Can users withdraw during an **auction-pending** window, and if so, what prevents free-riding?”
        - “What happens if many users exit just before settlement—who pays the remaining fee liability?”
- [ ]  **Permissionless epoch flip + flash-mint lets newcomers capture period rewards (“instant farmer”)**
    - **Symptom:** A public “start period / start auction / rollover” function checkpoints a **global shares-per-token** index, and rewards are later distributed based on that index. Because there’s **no snapshot of user balances at the period boundary** (and no warm-up), an attacker can **mint/flash-borrow**, trigger the rollover, and immediately **distribute + claim** the entire coupon for that period in one transaction, then burn/repay—leaving real holders with nothing.
    - **Where to look (epochs/checkpoints/distributor math):**
        - Period rollover: functions like `startAuction()`, `startEpoch()`, `rollover()` that call something like `bondToken.increaseIndexedAssetPeriod(sharesPerToken)` and update `lastDistribution`.
        - Distribution entrypoint: `distribute()` that uses `previousPeriod = currentPeriod - 1` and reads auction totals; verify whether it can be called **right after** rollover by the **same caller**.
        - Reward accounting: distributor `claim()` and helpers such as `getIndexedUserAmount(user, balance, period)` that **loop periods and multiply a passed-in `balance`** by `sharesPerToken`—i.e., **not** a per-period balance snapshot/TWAB.
        - Token balance updates vs. index: check whether **mint/burn/transfer** update any `userIndex`/`lastUpdatedPeriod` *at the time of balance change*, or if claims rely on current `balanceOf(user)`.
        - Access control & timing: is `startAuction` permissionless? Are there guards like keeper-only, min blocks, or **cutoff windows** before rollover?
    - **Red flags:**
        - Rollover is **`external` and permissionless**, gated only by `block.timestamp` windows.
        - `claim()`/`getIndexedUserAmount(...)` use **current balance** as a parameter and iterate `for (i = user.lastUpdatedPeriod; i < period; i++)`, with no **per-period balance snapshot**.
        - No **warm-up / min holding time**; newly minted tokens are immediately eligible for the current checkpoint.
        - It’s possible to **`startAuction()` → `distribute()` → `claim()`** in the same tx.
        - No **flash-loan protection** (e.g., TWAB or deposit timestamp checks).
    - **Fix pattern:**
        - **Snapshot balances at boundary:** on rollover, snapshot total supply and **per-user balances** (ERC20Votes-style checkpoints or TWAB). Claims must use **the snapshotted balance**, not the live one.
        - **Index + userIndex updates:** track a global `accRewardPerShare` and per-user `rewardDebt`; **update on every balance change** so a user cannot mint right before rollover to capture past accrual.
        - **Warm-up/cutoff:** require deposits/transfers to be **older than the period start** (e.g., `depositTimestamp < lastDistribution`) to be eligible; or add a **cooldown** before eligibility.
        - **Gate the rollover:** make rollover **keeper/governance-only** or enforce a **delay** between rollover and `distribute()` (e.g., next block / min seconds) so both can’t happen atomically.
        - **Fee/reward buffer:** pre-accrue and hold rewards in a buffer that’s **excluded from TVL** and **allocated by snapshot**, preventing instant capture by transient capital.
    - **Questions to ask next time:**
        - “Who can **flip epochs**? Can a user mint and flip in the **same transaction**?”
        - “Are balances used for distribution **snapshotted** at rollover or are we multiplying **current** balances by historical indices?”
        - “Do **mint/burn/transfer** paths update a **userIndex/rewardDebt** so claims are invariant to timing?”
        - “Is there a **warm-up** (min holding time) or **cutoff window** before/after rollover?”
        - “Can `startAuction` and `distribute` be called back-to-back? If yes, why can’t an attacker flash-borrow to steal the period’s rewards?”
- [ ]  **Vault “max in” joins leave leftovers stranded in router (no post-join refund)**
    - **Symptom:** Integration transfers **maxAmountsIn** from the user to an intermediary (router), calls Balancer `joinPool`, and **doesn’t reconcile balances after**. Because Balancer pools decide the **actual in** and the Vault only pulls what it needs, any **unused tokens remain on the router** and are never sent back—effectively locking user funds.
    - **Where to look (Balancer V2 joins/routers):**
        - Router helpers like `joinBalancerPool()`, `joinBalancerAndPlaza()`, `joinBalancerAndPredeposit()`.
        - Token flow before join: `safeTransferFrom(user, router, maxAmount)` and approvals to Vault.
        - Balancer Vault call: `IVault.joinPool(poolId, sender=router, recipient=router, JoinPoolRequest{ assets, maxAmountsIn, userData, fromInternalBalance })`.
        - After the join: check for **balance reconciliation** (router token balances vs. pre-join; refunds to `msg.sender`).
    - **Red flags:**
        - Code comments assume “we deposited all `maxAmountsIn`”, but **Balancer docs**: “**Pool** decides actual amounts; Vault enforces maximums.”
        - No accounting of **actual in** (e.g., `before - after` on each token) and no **refund** path.
        - Using generic join kinds (`ALL_TOKENS_IN_FOR_EXACT_BPT_OUT`, `JOIN_KIND_INIT`, etc.) without aligning to **EXACT_TOKENS_IN** semantics or without tracking **PoolBalanceChanged** event.
        - Router custody pattern: pull tokens into router first (instead of letting Vault pull directly from user) + **missing cleanup**.
    - **Fix pattern:**
        - **Reconcile & refund:** For each asset, record `pre = IERC20(asset).balanceOf(address(this))` before `joinPool`; after join, compute `used = pre + pulledFromUser - IERC20(asset).balanceOf(address(this))`; **refund `max - used`** to the user.
        - Prefer **EXACT_TOKENS_IN_FOR_BPT_OUT** join kinds when possible; then verify `actualIn == providedIn`; otherwise revert or refund residuals.
        - If UX allows, **avoid router custody**: approve Vault to pull from the **user** directly (`FundManagement{ sender=user }`) so leftovers never hit router. If router must custody, enforce a **post-join sweep**.
        - Add unit tests that simulate pools **accepting less** than `maxAmountsIn` and assert **no residual** stays on router.
    - **Questions to ask next time:**
        - “Who ultimately **pulls** tokens for the join—Vault from user or router? If router, where do **unused tokens** go?”
        - “Do we **measure actual in** after `joinPool` and refund the difference per asset?”
        - “Which **join kind** is used? Does it guarantee exact-in behavior, or can the pool choose a smaller in?”
        - “Do we rely on **events** (`PoolBalanceChanged`) or on **pre/post balance diffs** to compute actual in?”
- [ ]  **Time-rate fees multiplied by *current* balance (no checkpointing) over/under-charge**
    - **Symptom:** A continuous fee is computed as `fee = currentReserveBalance * feeRate * (now - lastFeeClaimTime) / YEAR`, using the **current** pool balance at claim time, not the **time-weighted** balance. Large deposits/withdrawals right before `claimFees()` cause **massive over/under-collection**.
    - **Where to look (treasury/fee accrual paths):**
        - Fee claimers: `claimFees()`, `getFeeAmount()`, variables like `lastFeeClaimTime`, `fee`, `SECONDS_PER_YEAR`, `PRECISION`.
        - All code that changes the **base** of the fee (reserve token balance): mint/burn/create/redeem/join/exit, prize/auction payouts, protocol sweeps.
        - Any routers/helpers that *temporarily* custody reserves (pre/post balances not accounted in fee accrual).
    - **Red flags:**
        - Formula uses **`IERC20(reserveToken).balanceOf(address(this))` at claim time** to represent the whole elapsed period.
        - No “**accrue**” or “**checkpoint**” call in user flows that modify reserves (create/redeem/auction settlement/joins).
        - Single timestamp `lastFeeClaimTime` without tracking a **fee index** or per-balance epoch data.
        - Fee parameter updates mid-period with no proration (e.g., `fee = newRate` and still using `lastFeeClaimTime`).
        - Comments like “continuous fee” but implementation is **snapshot × delta-time** with changing base.
    - **Fix pattern:**
        - **Accrue on every balance change**: before any mutation of reserves, run `accrue()`:
            - `feeAccrued += reserveBalance * (block.timestamp - lastTs) * feeRate / YEAR;`
            - `lastTs = block.timestamp;`
            - Then mutate `reserveBalance` (implicitly via transfers), not the formula.
        - Or use a **fee index** (time integral of rate) and per-pool “principal index”:
            - `feeIndex += feeRate * (t - lastTs) / YEAR; lastTs = t;`
            - On each balance change: `feeAccrued += reserveBalance * (feeIndex - lastFeeIndexOnBalanceChange); lastFeeIndexOnBalanceChange = feeIndex;`
        - If calling `claimFees()` inside user flows is desired, **avoid external transfers** during user actions; instead, accrue to storage and payout in a separate call to prevent reentrancy/coupling.
        - Add tests: (1) large deposit just before claim ⇒ fee equals **time-weighted average**, not full-period current balance; (2) large withdrawal just before claim doesn’t under-charge; (3) fee-rate change mid-period prorates correctly.
    - **Questions to ask next time:**
        - “When the reserve balance changes, do we **accrue up to now** before changing it?”
        - “Do we compute fees from a **time integral of balance × rate**, or from a single snapshot?”
        - “Can anyone game fees by **depositing/withdrawing right before claim**?”
        - “What happens if `fee` changes mid-period—do we split the accrual windows?”
        - “Are there flows (auctions/joins/bridges/sweeps) that change reserves **without** passing through the fee accrual hook?”
- [ ]  **Misaligned piecewise threshold causes discontinuous price jump (96→100) and wealth transfer**
    - **Symptom:** A two-regime pricing formula switches at a fixed `COLLATERAL_THRESHOLD`, but the boundary doesn’t align with the multiplier used in the “low-collateral” branch. Example (Plaza): with `multiplier = 80%` and `BOND_TARGET_PRICE = 100`, using `COLLATERAL_THRESHOLD = 120%` makes Bond price jump from ≤96 to 100 at the switch, hurting Lev holders even as TVL rises. The mathematically continuous threshold should be `1 / multiplier = 125%`, not 120%.
    - **Where to look (AMMs with tranches/bonds/levered tokens):**
        - Redemption/mint math: functions like `getRedeemAmount`, `simulateRedeem`, `create/mint` that compute rates from `tvl`, `bondSupply`, `levSupply`, `BOND_TARGET_PRICE`, `multiplier` (e.g., `POINT_EIGHT`), and `COLLATERAL_THRESHOLD`.
        - Constants/config: check `COLLATERAL_THRESHOLD`, `BOND_TARGET_PRICE`, and any “allocation multipliers” for below-threshold regimes.
        - Branching logic: piecewise sections `if (collateralLevel <= THRESHOLD) {...} else {...}` for both BOND and LEVERAGE paths.
        - Equality choice: use of `<=` vs `<` at the boundary and any later `min(redeemRate, marketRate)` clamps.
    - **Red flags:**
        - Hardcoded `COLLATERAL_THRESHOLD` not equal to `BOND_TARGET_PRICE / multiplierPrice` (in normalized units). With `multiplier = 0.8`, the continuous boundary is **125%**, not **120%**.
        - Price in low-collateral branch is `redeemRate = tvl * multiplier / assetSupply` and high-collateral branch returns a flat target (e.g., `= 100`), with no continuity proof or test at the switch.
        - Boundary uses `<=` causing a one-sided step; tiny TVL change flips regimes and creates a multi-point jump.
        - No unit/invariant tests around the threshold (e.g., values just below/above 120%) to assert **continuity/monotonicity** and **no loss for Lev when TVL ↑**.
    - **Fix pattern:**
        - **Align the threshold to ensure continuity:** set `COLLATERAL_THRESHOLD = ceilDiv(BOND_TARGET_PRICE, multiplier)` in the same units (e.g., 125% for 100/0.8).
        - Compute the boundary **from parameters**, not as a magic number:
            - `threshold = BOND_TARGET_PRICE * PRECISION / multiplier;`
        - Consider a **continuous interpolation** around the boundary (small buffer/ramp) or choose `redeemRate = min(targetPrice, tvl*multiplier/assetSupply)` only if it’s proven continuous.
        - Add invariants/tests:
            - `redeemRate(threshold−ε) ≈ redeemRate(threshold+ε)` (continuity).
            - If `tvl` increases with supplies fixed, **Lev value must not decrease**.
            - No arbitrage/wealth transfer from micro-toggling across the boundary.
    - **Questions to ask next time:**
        - “Is the piecewise curve **continuous** at the switching point? Show the algebra.”
        - “What’s the **exact** boundary that makes `lowBranch(t*) == highBranch(t*)`? Is `COLLATERAL_THRESHOLD` set to that value?”
        - “Why `<=` vs `<` at the boundary? What happens for `collateralLevel == THRESHOLD`?”
        - “Do we have unit tests for values just **below/above** the threshold and for **monotonicity** wrt TVL?”
        - “Could a trader profit (or Lev holders lose) by **nudging TVL across the boundary** with tiny trades/deposits?”
- [ ]  **Failed epochs still credited: period rollover pushes non-zero `sharesPerToken` even when no funds arrived**
- **Symptom:** Period/epoch rollover function (e.g., `increaseIndexedAssetPeriod`) snapshots the *previous* period’s `sharesPerToken` using a default/global value, regardless of whether the emission/auction actually succeeded. Claim logic then counts that failed period toward users’ accruals, causing “paper” rewards without backing and draining future distributions.
- **Where to look (staking/auctions/distributors with indexed accounting):**
    - Token with indexed rewards: functions like `increaseIndexedAssetPeriod`, `advanceEpoch`, `snapshot`, or `checkpoint` that push to `previousPoolAmounts[]` / `epochs[]` alongside `globalPool.sharesPerToken` / `index`.
    - Auction/fee collection end flow: `endAuction` / `finalize` → transfer of reward tokens to the pool/distributor vs. failure branches (`FAILED_UNDERSOLD`, `FAILED_POOL_SALE_LIMIT`).
    - Period turn-over entry points callable by anyone: `startAuction()` / `startEpoch()` that call the index advance.
    - Distributor claim path: `getIndexedUserAmount(...)` (loop from `lastUpdatedPeriod` to `currentPeriod`) and guards like `couponAmountToDistribute`, `NotEnoughSharesBalance`, `NotEnoughSharesToDistribute`.
- **Red flags:**
    - Rollover writes `previousPoolAmounts.push({sharesPerToken: globalPool.sharesPerToken})` **before** knowing whether funds for the prior period actually landed.
    - `sharesPerToken` initialized to a non-zero default (e.g., “2.5 USD”) and reused across periods unless explicitly overwritten.
    - Anyone can advance periods (`startAuction`) independent of settlement, letting an attacker/early claimer “bank” an extra (unfunded) period into their accrual.
    - No invariant ensuring `sum(distributed_shares) ≤ sum(actual_tokens_received_for_periods)`.
    - No per-period state tying `sharesPerToken` to an observed incoming amount; pushing “expected” instead of “actual”.
- **Fix pattern:**
    - **Bind accrual to settlement:** Only write the *prior* period’s `sharesPerToken` after funds are received/allocated (e.g., during `distribute()`), not when starting the next auction/epoch.
    - **Explicit inputs:** Change rollover to accept `prevPeriodSharesPerToken` (applied to `previousPoolAmounts`) and `nextPeriodTargetSharesPerToken` separately—or compute `prevPeriodSharesPerToken = receivedAmount / totalSupplyAtSnapshot`.
    - **Failure = zero:** If the prior auction/epoch failed, persist `sharesPerToken = 0` for that period before advancing.
    - **Ordering & guards:** Reorder flows so `endAuction(SUCCEEDED)` → transfer → pool accounts funds → distributor sets `couponAmountToDistribute` → **then** `increaseIndexedAssetPeriod`. Disallow advancing if prior period’s settlement hasn’t been finalized.
    - **Access control / cooldown:** Restrict period-advance or add a delay/snapshot guard to prevent flash/drive-by rollovers.
    - **Accounting invariant tests:** Add tests asserting `∑claims <= ∑funds_received` across periods and that failed periods contribute zero to `getIndexedUserAmount`.
- **Questions to ask next time:**
    - When the period advances, is the value pushed for the *previous* period derived from **actual received tokens** or from a constant/config?
    - Can anyone advance the period before settlement, or is there a state machine enforcing `ended && settled`?
    - Does `getIndexedUserAmount` include failed periods? How do claims behave if the distributor has `0` balance but `sharesPerToken > 0` for that period?
    - Is there an invariant test that total claimable over all users never exceeds tokens actually transferred in?
    - On failure branches (`undersold`, `limit hit`), where is `sharesPerToken` set to zero and persisted **before** the next period begins?
- [ ]  **Proportional trim uses integer math that changes bid price (ratio drift exploit)**
- **Symptom:** In oversubscribed auctions, the “remove excess” routine scales down the *lowest* bid by multiplying both legs (e.g., `sellCouponAmount` and `buyReserveAmount`) with an integer “proportion”. Truncation/rounding alters the ratio `sell/buy`, so a bidder who entered at a *higher* price can end up with an *effective lower* price after trimming, and can repeat the maneuver to grow size without paying the intended price.
- **Where to look (auctions/priority-queue partial fills):**
    - Functions named like `removeExcessBids`, `trimLast`, `rebalance`, `partialFillLowest`, `capOversubscription`.
    - Math patterns:
        - `proportion = amountToRemove * PRECISION / sellAmount;` then
        - `newSell = sellAmount - sellAmount * proportion / PRECISION;`
        - `newBuy = buyAmount - buyAmount * proportion / PRECISION;`
    - Check invariants around price: `newSell/newBuy` vs original `sellAmount/buyAmount`. Look for inconsistent rounding between legs or different precisions.
    - Data structures: sorted arrays/heaps of bids (lowest last), “slotSize” caps, and any code that replaces the tail bid when a better one enters.
- **Red flags:**
    - Integer division used twice with truncation; no single-source-of-truth for price when resizing.
    - No assertion that price is preserved (or stays at least as expensive as before) post-trim.
    - Ability for the displaced/entering bidder to improve *effective* price after trimming (`newBuy` increases or doesn’t shrink proportionally).
    - Lack of post-condition checks like `newBuy * sellAmount == buyAmount * newSell` (within tolerance).
    - Bidders can split/sequence bids to trigger repeated trimming and accumulate advantage.
- **Fix pattern:**
    - **Derive one leg from the other using the original price:** compute `newSell` first, then set `newBuy = ceilDiv(newSell * buyAmount, sellAmount)` (or vice-versa) to preserve (or conservatively worsen) the price for the bidder.
    - Use `mulDiv` with explicit rounding direction (e.g., round **up** on the leg favorable to the bidder) so trimming can only **worsen** the bidder’s effective price, never improve it.
    - Alternatively, avoid proportional trims: drop the losing bid entirely or fill to exact slot boundaries that preserve price using rational math.
    - Add an invariant: after trimming, `newSell/newBuy >= oldSell/oldBuy` for buy-side payments (or the correct inequality for your convention).
    - Unit tests for corner cases: small amounts (1 wei), near-full trims, repeated insertions/replacements, and mixed decimal assets.
- **Questions to ask next time:**
    - When partially filling or trimming a bid, how do we guarantee the **price ratio is preserved** (or becomes strictly worse for the bidder)?
    - Which leg is the price-defining numerator/denominator? Do we derive the other leg from that ratio with consistent rounding?
    - Can repeated trimming via strategic bids increase the *buy* leg for the same *sell* leg (or vice versa)?
    - Do we have assertions/tests that detect ratio drift due to integer truncation at extreme values (tiny `amountToRemove`, 1-wei legs, large precision scalars)?
- [ ]  **Cap-limited downstream deposit strands LP/BPT on router (no reconciliation/refund)**
- **Symptom:** A multi-step helper mints LP/BPT (e.g., via Balancer join) and then calls a cap-limited `deposit(...)` on a second contract. The callee **clips** the requested `amount` to the remaining cap and `transferFrom(msg.sender, ...)` only the clipped amount. Since the router is `msg.sender`, the **excess LP/BPT stays on the router** and is never returned to the user, causing silent funds loss.
- **Where to look (routers/helpers that chain actions):**
    - Router functions like `joinXAndDeposit/Predeposit/Stake`, `zapAndStake`, `mintAndLock`, etc., that:
        1. receive user tokens / mint LPs;
        2. call a second contract with **caps** (`reserveCap/depositCap/positionCap`) or dynamic acceptance.
    - Callee deposit paths with clipping logic, e.g.:
        - `if (reserveAmount + amount >= reserveCap) { amount = reserveCap - reserveAmount; }`
        - And then `IERC20(token).safeTransferFrom(msg.sender, address(this), amount);`
    - Ownership flow: check if callee resolves `recipient = onBehalfOf == address(0) ? msg.sender : onBehalfOf;` but **pulls** from `msg.sender` (the router) instead of the end user.
    - After-call balance handling in router: is there a **before/after** LP/BPT balance check on the router with refund to the user?
- **Red flags:**
    - Callee **does not return** the actually-used amount; router **assumes** the full requested amount was deposited.
    - Router approves `amount` to callee and calls `deposit(amount, onBehalfOf=user)` but never checks `routerBalance(token)` to refund leftovers.
    - Cap/saturation/rounding paths in callee (e.g., “fill to cap”) + callee using `transferFrom(msg.sender, ...)`.
    - Any helper performing: join/mint → approve → deposit; **no** sweep/refund step afterward.
    - Similar risk surfaces: Curve/Uni/Camelot add-liquidity then stake-with-cap; staking gauges/vaults with per-pool caps; LRT/LST predeposit campaigns.
- **Fix pattern:**
    - **Query cap first** and pass `min(request, capRemaining)` to avoid over-request.
    - Have callee **return `usedAmount`**; router refunds `requested - usedAmount` to the user.
    - In router, add **balance deltas** around the call:
        - `before = token.balanceOf(address(this));`
        - call deposit
        - `after = token.balanceOf(address(this));`
        - `leftover = after - before; if (leftover > 0) transfer leftover to user;`
    - Prefer “**pull from user**” design: router only sets allowance from **user** to callee (Permit2/permit), so any leftover remains with the user, not the router.
    - Emit events and/or revert if leftover is non-zero (safer default), depending on product UX.
- **Questions to ask next time:**
    - Does the deposit target **clip** or **cap** the requested amount? If yes, how do we learn the **actual used** amount?
    - Who is `msg.sender` for the callee’s `transferFrom`—the router or the user? Where do leftovers physically sit?
    - After multi-step zaps, does the router **sweep/refund** any residual LP/BPT and input tokens back to the user?
    - Are there **edge tests** for “cap just reached” and “over-cap by 1 wei” scenarios?
    - If reverting on leftover is undesirable, what’s the **explicit refund path** and event users/tools can rely on?
- [ ]  **Burn-all path bricks leverage side: zero supply → permanent create() DoS**
- **Symptom:** The redeem/burn path lets **all** leverage tokens be burned. Later, `create(TokenType.LEVERAGE, ...)` (or equivalent) requires a **non-zero** `assetSupply/levSupply` to compute rates and **reverts** (`ZeroLeverageSupply`). Result: once lev supply hits zero, **no one can buy lev** again (core market dead).
- **Where to look (leveraged/binary-tranche AMMs, R-DAMMs, bonding-curve pools):**
    - Mint/price path: `getCreateAmount` / `previewMint` / `getMintRate` that divides by `assetSupply` or explicitly reverts on zero supply.
    - Redeem/burn path: `_redeem` / `burn` calls for LEVERAGE branch that **do not** enforce a minimum remaining supply (e.g., `lToken.burn(msg.sender, depositAmount)` without checks on post-burn supply).
    - Collateral-level logic gates with thresholds (e.g., 120%) that switch formulas and assume `levSupply > 0`.
    - Init/bootstrap code: is there a special case to **reboot** pricing from `levSupply == 0`?
- **Red flags:**
    - Guards like `if (assetSupply == 0) revert ZeroLeverageSupply();` exist **only** on the mint side, not mirrored on redeem.
    - No “virtual supply/shares” or seed-liquidity to prevent `levSupply` from reaching zero.
    - No check like `require(depositAmount < lToken.totalSupply())` in the lev redeem path.
    - Tests cover zero-bond scenarios but **not** zero-lev scenarios.
- **Fix pattern:**
    - **Supply floor:** Disallow burning the last wei: `require(depositAmount < lToken.totalSupply(), "min lev supply");` or maintain a protocol-owned dust position.
    - **Virtualization:** Introduce **virtual supply/reserves** (non-zero constants) used in rate math to avoid division by zero while keeping user supply burnable.
    - **Bootstrap branch:** Allow `create(LEVERAGE, ...)` when `levSupply == 0` using a deterministic initial rate derived from TVL/bond state (and test for fair pricing).
    - **Circuit breaker:** If a redeem would drop lev supply to zero, route through admin/auction or revert with guidance.
- **Questions to ask next time:**
    - Can **any** user burn the last lev token? What invariant prevents `levSupply == 0`?
    - If lev hits zero, is there a **documented bootstrap** path for price/rate (no division by zero)?
    - Do we have unit/integration tests for **redeem all lev → attempt to mint lev**?
    - Who owns the protocol dust/virtual shares, and are they excluded from user PnL/fees?
    - Are threshold-based pricing branches safe under `levSupply == 0` (no hidden div-by-zero or stuck state)?
- [ ]  **Finalize uses “not-ended” gate → snapshot can be frontrun/backrun**
- **Symptom:** A “finalize/snapshot” function (e.g., `setBondAndLeverageAmount`) is gated with a **`depositNotEnded`** modifier and reads live contract balances. Users can still `deposit`/`withdraw` in the same window, so a griefer can **front-/back-run** governance’s finalize call to skew totals. After the window ends, finalize can’t be called again → **permanently wrong supplies/allocations** and unfair downstream rewards.
- **Where to look (bootstraps/pre-deposit/IDO/LBP/snapshot flows):**
    - “Configuration/finalization” functions that compute token amounts/supplies from **`balanceOf(address(this))`** or mutable state (e.g., `reserveAmount`) and are **callable before the phase actually ends**.
    - Time/phase guards on:
        - `finalize()/setXAmounts()/snapshot()` (uses `checkDepositNotEnded` / `block.timestamp < end`).
        - `deposit()/withdraw()` (still enabled up to or beyond the same end).
    - Single-shot flows where finalize **cannot be re-run** after the end due to strict modifiers.
    - Any place where later pricing/creation rates rely on the finalized totals (e.g., initial `bondSupply/levSupply`, `sharesPerToken`).
- **Red flags:**
    - Finalize guarded by **“not ended”** instead of **“ended”**; deposits/withdrawals still open concurrently.
    - Reads of **live** balances in finalize without freezing/caching during an earlier checkpoint.
    - No “freeze”/pause step (e.g., `depositsPaused = true`) prior to computing allocations.
    - Finalize is **non-idempotent** and **one-shot** (can’t correct after phase end).
    - Inconsistent inequalities: deposits allowed while `block.timestamp < end`, finalize allowed while **the same condition holds**.
- **Fix pattern:**
    - Gate finalize with **`depositEnded`** (e.g., `require(block.timestamp >= depositEndTime)`), or
    - Two-step close:
        1. `closeDeposits()` (pauses `deposit/withdraw`, records `endSnapshot = reserveAmount/balance`),
        2. `finalize()` computes from **snapshotted** values only.
    - Make finalize **idempotent** (can safely re-run until a `finalized` flag is set).
    - Alternatively, compute allocations on-chain **at deposit end** automatically (e.g., via a single `finalize()` callable only after end; deposits/withdrawals revert then).
    - Consider **commit→reveal** for governance finalize to reduce MEV windows.
- **Questions to ask next time:**
    - Can anyone still **change balances** (deposit/withdraw) while finalize/snapshot is callable?
    - Is finalize **after** end (`>= endTime`) and are deposits/withdrawals **blocked** before finalize reads balances?
    - Are the **finalization inputs snapshotted** or derived from live balances?
    - If finalize happens with wrong inputs, can it be **corrected** (idempotent or re-runnable) or is it **one-shot**?
    - Do unit tests include **frontrun/backrun** around finalization and verify identical results predeposit vs postlaunch creators?
- [ ]  **Sliding “lastDistribution = now” causes epoch drift and yield leakage**
    - **Symptom:** Periodic auctions/distributions treat `lastDistribution` as a moving wall-clock anchor (`lastDistribution = block.timestamp`). Each start delays the next by the lateness of the current one. Over quarters this **drift compounds**, shortening effective accrual windows and **underpaying bond holders**.
    - **Where to look (epochs/auctions/rewards scheduling):**
        - Epoch gates: `require(lastDistribution + distributionPeriod < block.timestamp)` and updates: `lastDistribution = block.timestamp`.
        - Modules named like `startAuction/startEpoch/rollover/distribute/rebase/harvest` that use **`block.timestamp` as the new baseline**.
        - Any **quarterly/weekly** reward logic (`distributionPeriod`, `auctionPeriod`) and how “period start” is computed.
        - Pause/failure paths that still update `lastDistribution` to `now`.
    - **Red flags:**
        - Setting the next baseline with **`last = block.timestamp`** instead of **`last += period`**.
        - No **catch-up loop** for missed periods (e.g., keeper delays, congestion): `while (last + period <= now) last += period`.
        - Window checks like `last + period + auctionPeriod >= now` with no bound on cumulative slippage.
        - Comments implying fixed quarters but math using floating timestamps.
    - **Fix pattern:**
        - Anchor epochs to the schedule, not the clock: set `lastDistribution = lastDistribution + distributionPeriod` (or advance by multiples via a loop).
        - If you need tolerance windows, use **grace periods** but still advance by scheduled increments.
        - Emit both `scheduledStart` and `actualStartedAt`; use `scheduledStart` for future gating.
        - Add tests: simulate N late starts with random delays; assert total scheduled time advanced by `N * distributionPeriod` and no reduction in accrued rewards.
    - **Questions to ask next time:**
        - If the keeper is late by X minutes, does the **next period** shift by X too? After a year of delays, how big is the drift?
        - On auction failure/pause, do you still update `lastDistribution`? Should you?
        - Should multiple missed periods be **rolled forward** in one tx?
        - Are rewards based on **scheduled** or **actual** timing? Which timestamp drives accrual math?
- [ ]  **Floor-div “slot size” lets attackers fill max bids yet force UNDERSOLD**
    - **Symptom:** Auctions cap bids with a **fixed slot size** `slotSize = totalBuyCouponAmount / maxBids` (integer division). Bids must be an exact multiple of `slotSize`. When `totalBuyCouponAmount % maxBids != 0`, the **remainder is lost**. A spammer can submit exactly `maxBids` bids of size `slotSize`, hitting the bid limit while leaving the remainder unfunded, so `currentCouponAmount < totalBuyCouponAmount` and the auction ends **FAILED_UNDERSOLD** despite “full” participation.
    - **Where to look (slot sizing / bid caps / auction close):**
        - Auction math: `slotSize()` (or equivalent) and **any integer division** of `totalBuyCouponAmount / maxBids`.
        - Bid validators: checks like `require(sellCouponAmount % slotSize == 0)` or “must be even division of slot”.
        - End conditions: `endAuction()` comparisons such as `currentCouponAmount < totalBuyCouponAmount` and how `currentCouponAmount` is accumulated.
        - List maintenance: `removeExcessBids()` and rolling out lowest bids when max capacity is reached (precision/rounding in proportional trims).
        - Constants/decimals: token decimals vs internal `PRECISION` when computing slots.
    - **Red flags:**
        - Using **floor division** for per-slot sizing without handling the **remainder** (no `ceilDiv`).
        - Max bids reached check tied to **count** of bids, not to **sum** covering target with remainder bucket.
        - Acceptance rule “bid must be multiple of `slotSize()`” with **no special last-slot** allowance.
        - End check uses strict `<` vs allowing a one-slot tolerance; or sums in mixed units (coupon vs reserve) causing off-by-one shortfalls.
        - `slotSize()` can be **zero** if `totalBuyCouponAmount < maxBids` (catastrophic DoS).
    - **Fix pattern:**
        - Compute slots with **ceil division**: `slotSize = ceilDiv(totalBuyCouponAmount, maxBids)`, or
        - Keep `slotSize = floorDiv(...)` but introduce a **remainder bucket**: allow one final bid up to `slotSize + remainder`, or relax multiple-of-slot rule for the last slot.
        - Alternatively, drop slot multiple requirement and enforce **per-auction cap** using total filled amount `min(totalBuyCouponAmount - filled, bidAmount)`.
        - At close, compare **normalized totals** in the same units and consider **tolerance** no larger than `(maxBids - 1)` lost units if floor sizing is retained (better: eliminate the loss).
        - Add invariant tests: for any `totalBuyCouponAmount` and `maxBids`, an attacker submitting `maxBids` minimal valid bids **cannot** make a funded auction fail.
    - **Questions to ask next time:**
        - What happens when `totalBuyCouponAmount` is **not divisible** by `maxBids`? Who covers the **remainder**?
        - Can a bidder place a **larger final slot** to absorb the remainder, or is the multiple-of-slot rule absolute?
        - Is `slotSize()` **recomputed** after proportional removals/rounding? Can `slotSize()` be **zero**?
        - Does `endAuction()` use the **same unit/precision** as bids when comparing to `totalBuyCouponAmount`?
        - Can a single actor fill all `maxBids` with **exact-slot** orders and still keep the auction below target? If yes, how is that prevented?
- [ ]  **Hard-coded BPT token in multi-pool router strands user funds**
    - **Symptom:** A Balancer router intended to work with many pools tracks a **single immutable BPT** (`balancerPoolToken`). When joining any other pool, the router measures balance deltas and sets approvals against the **wrong token**, so `balancerPoolTokenReceived == 0` while the actual BPT from the joined pool sits in the router. Subsequent pre-deposit uses the zero amount, leaving the real BPT **stuck in the router**.
    - **Where to look (routers / pool joins / BPT accounting):**
        - Router constructor: immutable fields like `IERC20 public immutable balancerPoolToken;`.
        - Join helpers: `joinPool` wrappers that do `balanceBefore/After` on a **global** token instead of the pool’s BPT derived from `poolId`.
        - Follow-ups: approvals and `deposit(...)` calls that pass the **returned delta** from the prior step.
        - Vault interfaces: usage of `IVault.joinPool(...)` without fetching the corresponding **pool/BPT address** for the provided `poolId`.
        - Any path that supports “multiple Balancer pools” but keeps **one** BPT token in state.
    - **Red flags:**
        - `balancerPoolToken` (singular/immutable) used across all joins.
        - No call to resolve pool token per call (e.g., `vault.getPool(poolId)` → `poolAddress` → `IERC20(poolAddress)`).
        - Balance delta measured against the wrong token; zero delta returned while tokens actually arrived.
        - Approval made on the wrong token; deposit uses zero amount; **no refund path** for router-held BPT.
    - **Fix pattern:**
        - Derive BPT **per join**: `(_, address pool,) = vault.getPool(poolId)` (or equivalent) and treat `IERC20 poolToken = IERC20(pool);`.
        - Compute `received = poolToken.balanceOf(this) - before` and approve/deposit **that** token/amount.
        - Add safety checks: revert if `received == 0`; return leftovers to caller if downstream cap blocks full deposit.
        - Remove/avoid global immutable BPT for multi-pool flows; pass pool token through the call stack.
        - Integration tests: joining two different pools back-to-back should not strand BPT; assert router ends with **zero** BPT.
    - **Questions to ask next time:**
        - How do you map `poolId` → **BPT ERC-20** address at runtime? Is there any case where token order or specialization changes this?
        - Are **balance deltas** and **approvals** computed against the **same** token you just received?
        - What happens if the downstream contract (e.g., PreDeposit) accepts **less** than received—do you **refund** the remainder?
        - Can the router handle **multiple poolIds** in a single tx without persisting unintended balances?
        - Do you have tests that join pool A then pool B and ensure the router doesn’t retain BPT from A or B?
- [ ]  **Decimal mismatch nullifies market-rate guard**
    - **Symptom:** A calculated `redeemRate` uses one precision (e.g., `PRECISION = 1e6` / 6dp), while `marketRate` from an external oracle (Balancer/Chainlink/AMM adapter) is 18dp (wad). Code does `if (marketRate != 0 && marketRate < redeemRate) redeemRate = marketRate;` but compares **unscaled values**, so the condition is effectively always false (or always true), and the market cap/floor path is never taken.
    - **Where to look (pricing, redemption, oracle adapters):**
        - Redemption math (e.g., `getRedeemAmount`, `simulateRedeem`, `redeem`, `create`) where `redeemRate` is derived with local constants like `PRECISION = 1e6`, `BOND_TARGET_PRICE * PRECISION`.
        - The branch that merges oracle “market” pricing (TWAP / Chainlink / Balancer) into redemption: code like
        `if (marketRate != 0 && marketRate < redeemRate) redeemRate = marketRate;`
        - Normalization utilities: `.normalizeAmount(srcDecimals, dstDecimals)` usage and **targets**. Verify the chosen normalization target equals the unit of `redeemRate` (shares decimals vs oracle decimals vs reserve token feed decimals).
        - Oracle adapters that **hardcode 18dp** (Balancer/AMM math “works in 18 decimals”) and the callsite that **assumes** 6dp or token decimals.
        - Cross-token “rate” comparisons (bond vs lev), especially when one path is 6dp and the other pulls an 18dp oracle.
    - **Red flags:**
        - `PRECISION = 1e6` locally, while `marketRate` is normalized to 18dp (`1e18`) and then compared to `redeemRate` without re-scaling.
        - Using reserve feed decimals for normalization of a **bond** market rate (or vice versa).
        - Different decimal bases for “rate” vs “amount” in the same function; passing a single `oracleDecimals` param to multiple unrelated comparisons.
        - Comments like “Balancer math works with 18 dec” but branch compares against a 6dp number.
    - **Fix pattern:**
        - Pick a **single canonical scale** for rates (e.g., `SHARES_DECIMALS` or `COMMON_DECIMALS`) and **normalize both** `redeemRate` and `marketRate` to it **before** comparison.
        - At the boundary where `marketRate` is fetched, convert:
        `marketRate = normalize(marketRate, marketDecimals, RATE_DECIMALS);`
        - Plumb explicit `rateDecimals` through helpers (`getRedeemAmount(...)`) and **document** expected units. Avoid inferring decimals from reserve feeds when comparing rates for **bond/lev tokens**.
        - Add invariants/tests: for representative inputs where market < model rate (after proper scaling), the branch must take the market rate; include 6dp vs 18dp fuzz cases.
    - **Questions to ask next time:**
        - What **exact decimals** is `redeemRate` expressed in? What about `marketRate` returned by this adapter/feed?
        - Where is the **normalization** done? Is it to **the same target** before any `<` / `>` comparison?
        - Are you normalizing to **token decimals**, **oracle feed decimals**, or a **shares/rate decimals** constant? Why?
        - Do unit tests assert both branches (market overrides / model rate used) under **mismatched decimal** scenarios (6↔18)?
- [ ]  **Last-second bid inflates sell total to trip pool-sale cap (perma-fail auctions)**
    - **Symptom:** `endAuction()` compares an ever-growing `totalSellReserveAmount` against a cap `(pool.reserveBalance * poolSaleLimit)/100`. Any bidder can submit a very large bid right before expiry; `bid()` immediately bumps `totalSellReserveAmount`. On finalize, the auction hits `FAILED_POOL_SALE_LIMIT` and refunds—cheap to repeat, effectively DoS’ing coupon distributions.
    - **Where to look (auction matching/finalization):**
        - Auction state & tallies: `totalSellReserveAmount`, `totalBuyCouponAmount`, `currentCouponAmount`.
        - Order intake: `bid(...)`—does it **add to** `totalSellReserveAmount` on submission (pre-matching)?
        - Pruning/sizing: `removeExcessBids()`—does it bound totals to the **accepted set** or can oversized bids still inflate the global total?
        - Finalization: `endAuction()`—failure branch like
        `if (totalSellReserveAmount >= (IERC20(sellReserveToken).balanceOf(pool) * poolSaleLimit)/100) state = FAILED_POOL_SALE_LIMIT;`
        - Refund path: `claimRefund()`—zero/low penalty makes the attack cheap.
        - Pool hooks: `Pool.transferReserveToAuction(...)`—is the cap enforced **there** or only via fail state?
    - **Red flags:**
        - `totalSellReserveAmount` represents **requested** (tentative) sell, not the **matched/accepted** sell but is still used in the end-cap check.
        - No guard in `bid()` to reject/clip a bid that would push the matched sum over the cap.
        - Cap check references **pool balance * limit** rather than the sum of **top-N winning** bids after trimming.
        - No anti-sniping window / commit-reveal / auction extension for late bids.
        - Full refunds on failure; no cancellation or griefing cost.
    - **Fix pattern:**
        - Compute cap compliance on the **accepted set** only: at finalize, sort/trim to winners, compute `matchedSell = sum(winners.sellReserve)`, then enforce `matchedSell <= cap`; proportionally clip the **last** winning tranche if needed instead of failing the auction.
        - On intake, reject or **clip** incoming bid to `remainingCap = cap - matchedSellSoFar` (with precise proportional adjustment and price invariants).
        - Separate “requested” vs “matched” tallies (`requestedSell`, `matchedSell`), and only the latter participates in end checks.
        - Add anti-sniping (extend end time if bid in last X seconds) or freeze the order book in the last block.
        - Introduce small non-refundable fee/slash for bids that are later refunded due to failure to reduce griefing.
        - Unit tests: last-block jumbo bid should not move state to `FAILED_POOL_SALE_LIMIT`; it should be clipped or rejected while auction succeeds.
    - **Questions to ask next time:**
        - When is `totalSellReserveAmount` updated—**pre** or **post** matching/selection? Does it represent **tentative** or **final** allocation?
        - Is the pool-sale cap enforced during **matching** or only via a **fail** branch at finalize?
        - Can a single late bid flip the auction to failure despite a valid winning set under the cap?
        - Are refunds costless? What’s the griefing cost to repeatedly fail auctions?
        - Do you extend the auction or freeze submissions in the final seconds to prevent last-second manipulation?
- [ ]  **Unit-of-account mismatch (USD vs USDC) skews collateral level & pricing**
    - **Symptom:** TVL and prices are computed in **USD** (e.g., `getOraclePrice(reserveToken, USD)`), while targets/settlement use **USDC** (e.g., `BOND_TARGET_PRICE = 100` and coupons paid in USDC). When USDC ≠ $1, formulas like `collateralLevel = TVL(USD) / (bondSupply * 100 USDC)` silently mix units, misclassifying the regime (≤/≥ threshold) and producing wrong mint/redeem quotes.
    - **Where to look (pricing/oracles/auctions):**
        - Pool math: `getCreateAmount`, `getRedeemAmount`, `simulateCreate`, `simulateRedeem`—inputs/returns and constants (`BOND_TARGET_PRICE`, `COLLATERAL_THRESHOLD`) and which currency they assume.
        - Oracle reader: `getOraclePrice(token, USD)` vs any path that should return **USDC**; normalization helpers (`normalizeAmount`, `toBaseUnit`).
        - Coupon/auction flow: how `sharesPerToken` (denominated in USDC) and `couponAmountToDistribute` are derived; any conversions to/from USD.
        - Global constants/enums: does code ever treat “USD” symbolically equal to the coupon token (USDC)?
    - **Red flags:**
        - Hard-coded `BOND_TARGET_PRICE = 100` used directly against TVL(USD) without multiplying by a **USDC/USD** rate.
        - Comments/docs say “bond = 100 **USDC**” while core math divides/multiplies by `... * 100` alongside `getOraclePrice(..., USD)`.
        - Threshold/comparison logic (e.g., 120% collateral level) mixing a USD numerator with a USDC-denominated denominator.
        - No explicit USDC/USD oracle (or it’s assumed 1e18).
    - **Fix pattern:**
        - Pick a single **unit of account** (USD or the coupon token) and normalize **all monetary terms** into that unit before comparisons.
            - If USD: convert USDC legs by `amountUSDC * (USDC/USD price)` wherever `BOND_TARGET_PRICE` or coupons appear.
            - If USDC: convert TVL by dividing USD prices by `(USDC/USD price)` so TVL is in USDC.
        - Make unit explicit in types/vars: `tvlUsd`, `tvlUsdc`, `targetPriceUsdc`, `priceUsd`.
        - Thread a `fxUsdcPerUsd` (or `usdPerUsdc`) from an oracle into `getCreateAmount/getRedeemAmount` and apply consistently.
        - Add invariants/tests: simulate USDC at 0.90 and 1.05; ensure regime selection and outputs remain economically correct.
    - **Questions to ask next time:**
        - What is the **unit** of `BOND_TARGET_PRICE`, thresholds, and coupon `sharesPerToken`? Are those ever compared with values in another unit?
        - Do you have an explicit **USDC/USD feed**? Where is it applied?
        - If the coupon token changes (e.g., USDT/DAI), does the math still hold without modifying constants?
        - In backtests with USDC depeg scenarios, do mint/redeem decisions (and collateral-level regime switches) stay consistent?
- [ ]  **Using `safeIncreaseAllowance` on tokens with “infinite allowance to vault” causes approval overflow & exit DoS**
    - **Symptom:** Exit/withdraw paths revert when the code calls `SafeERC20.safeIncreaseAllowance` for a token that **returns `type(uint256).max` allowance** for a specific spender (e.g., Balancer BPT → Vault). The helper computes `oldAllowance + value` and overflows, bricking functions like `exitPlazaAndBalancer` / pool exits.
    - **Where to look (routers/adapters interacting with vaults):**
        - Router code around **pool exit**: e.g., `_exitBalancerPool` / `exitPool` helpers that try to “approve BPT to Vault”.
        - Any **approval before `joinPool`/`exitPool`** or similar vault calls (Balancer, sometimes specialized AMMs).
        - Tokens with non-standard **allowance overrides** (Balancer’s `BalancerPoolToken.sol` returns max allowance for `getVault()`).
        - OZ `SafeERC20` usage sites, especially `safeIncreaseAllowance`/`safeApprove` on **LP/BPT tokens**.
    - **Red flags:**
        - `balancerPoolToken.safeIncreaseAllowance(address(balancerVault), balancerPoolTokenIn)` before `exitPool(...)`.
        - Comments like “approve to vault” on BPT/LP tokens (Balancer **does not need** approval; allowance is infinite).
        - No guard for `if (token.allowance(this, vault) == type(uint256).max) { skip }`.
        - Approvals done **every exit** instead of once, or on tokens with documented infinite allowance semantics.
    - **Fix pattern:**
        - **Do not approve** BPT to the Balancer Vault; **remove** `safeIncreaseAllowance` entirely for BPT → Vault path. (BPT overrides `allowance()` to `type(uint256).max` for the vault.)
        - If keeping a generic path, add a **guard**: `if (token.allowance(address(this), spender) != type(uint256).max) { SafeERC20.forceApprove(token, spender, amount); }`.
        - Prefer **idempotent** approval logic (zero-then-set or `forceApprove`) and **skip** when spender is a vault with infinite allowance semantics.
        - Add a unit test that calls `safeIncreaseAllowance(vault, 1)` on a real/mainnet-fork BPT to ensure it **does not** revert (by skipping).
    - **Questions to ask next time:**
        - Does this LP/BPT token **override** `allowance(owner, spender)` (e.g., returns max for a vault/manager)?
        - Are we calling **`safeIncreaseAllowance`** anywhere with a spender that might already have **max allowance**?
        - Do we really need approval for this spender, or does the protocol’s **authorization model** (vault) already allow pulls?
        - Can we centralize approvals (constructor/init) or remove them entirely for known tokens to avoid **hot-path** approvals?
        - Do our CI tests include a **fork test** trying the exact approve path against the real vault/token implementation?
- [ ]  **Divide-then-multiply truncation makes redeem rate round to zero**
    - **Symptom:** Formulas shaped like `((A - B) / C) * PRECISION` **truncate to 0** when `A - B < C`, causing `redeemRate == 0` and downstream `ZeroAmount()` reverts during otherwise valid redemptions (common with low TVL or large supply).
    - **Where to look (pricing/math paths):**
        - Token economics math: `Pool.getRedeemAmount`, `simulateRedeem`, `simulateCreate`, fee/yield formulas.
        - Any rate calc that mixes totals & supplies: leverage/bond redeem paths, NAV share price, index/share-per-token updates.
        - Library calls doing integer math before scaling (look for `(... / denom) * PRECISION` patterns).
    - **Red flags:**
        - Order `(... / assetSupply) * PRECISION` instead of `mulDiv(..., PRECISION, assetSupply)`.
        - Missing guard when `numerator = tvl - bondSupply * TARGET` can be small or negative.
        - Inconsistent scaling (mix of `toBaseUnit`/`normalizeAmount` plus a late divide).
        - Use of plain `/` with large denominators and no rounding control.
    - **Fix pattern:**
        - **Multiply before divide** using a 256-bit safe helper:
        `redeemRate = Math.mulDiv((tvl - bondSupply * BOND_TARGET_PRICE), PRECISION, assetSupply);`
        - If under-collateralization is possible, **guard/clamp**: revert when `tvl < bondSupply * TARGET` or saturate at zero per design.
        - Standardize to a **single precision unit** (e.g., 1e6/1e18 “WAD/RAY”) at function boundaries; avoid re-scaling mid-formula.
        - Where upward bias is safer, use `Math.mulDiv(..., ..., ..., Math.Rounding.Ceil)` or add `+ denom - 1` before division (only if consistent).
        - Add property tests: for `tvl > bondSupply * TARGET && assetSupply > 0`, assert `redeemRate > 0`.
    - **Questions to ask next time:**
        - Do any rate formulas divide **before** scaling? Can the numerator be **smaller than** the denominator in normal states?
        - What’s the **intended decimals** of the rate (6/18)? Are all operands normalized to that before math?
        - Could `tvl - bondSupply * TARGET` be **negative** at edges? What’s the desired behavior then?
        - Are we using `Math.mulDiv` / `FullMath.mulDiv` to avoid overflow when multiplying first?
        - Do tests include **low-TVL / high-supply** scenarios to catch truncation-to-zero cases?
- [ ]  **Push-refunds with blacklistable tokens can brick eviction and force cheap fills**
    - **Symptom:** Evicting the lowest bid (or order) performs a **push `transfer`** refund of the quote token (e.g., USDC). If the recipient is **blacklisted/paused** or otherwise reverts on `transfer`, the eviction reverts, **blocking new bids** from entering. An attacker first fills the queue with many low-price bids and then ensures the current lowest-bidder address is blacklisted, so the queue can’t rotate; their underpriced bids survive to settlement, draining reserves.
    - **Where to look (auctions/orderbooks/queues):**
        - Bid eviction paths: functions like `_removeBid`, `popLowest`, `rebalanceBids`, `pruneOrders` that do refunds.
        - Refund logic using `IERC20.transfer` / `safeTransfer` directly to user during **state mutation**.
        - Any system with **max active bids/orders** (fixed-size heap/array) that must evict one to admit another.
        - Tokens configured as quote/refund assets: **USDC/USDT or owner-controlled ERC20s** with blacklist/pause/freeze hooks.
    - **Red flags:**
        - `safeTransfer(buyToken, bidder, refundAmount)` **inside** the eviction branch (no pull fallback, no try/catch).
        - No handling for **transfer-reverting tokens** (blacklist, pause, sanctions, fee-on-transfer).
        - Admission logic like `if (count == MAX) { _removeLowest(); }` with no escape if refund fails.
        - Settlement bounded by **pool-sale-limit**, so blocking competition lets attacker keep many **underpriced** wins.
        - Assumptions like “refunds can’t fail” or tests using only vanilla ERC20 mocks.
    - **Fix pattern:**
        - Switch to a **pull-refund** model: on eviction, **record a refundable credit** (`refunds[bidder] += amount`) and emit an event; bidder later calls `claimRefund()`. Do **not** push value during critical state transitions.
        - Add **admin escape hatch** to reassign stuck refunds (e.g., change refund token, or allow re-routing after grace period) and to **skip/void** toxic bids whose refunds cannot be delivered.
        - For centralized tokens, consider **allowlists** or **Permit2/pull-from** models for incoming funds, but always treat **outgoing** as potentially reverting.
        - Wrap external token calls in **try/catch** only on **non-critical** paths; never let a refund failure block core progress.
        - Add invariant tests: “no single failing refund can prevent new bids or auction end,” and fuzz with a token that **reverts on transfer to specific addresses**.
    - **Questions to ask next time:**
        - Can a **single refund failure** (due to blacklist/pause/fee-on-transfer) **block queue rotation** or auction finalization?
        - Do we ever **transfer out** funds to arbitrary users **during** mutations (admit/evict/settle), or is everything claimable later?
        - What happens if the quote token **changes behavior mid-epoch** (e.g., USDC blacklist event)? Is there a **graceful degradation** path?
        - Are there **per-bid refunds** recorded on-chain, with idempotent `claimRefund()` and admin retries?
- [ ]  **Sale-limit checks ignore accrued/claimable fees → auctions flip state or slash fees**
    - **Symptom:** Auction “success” test uses `balanceOf(pool) * poolSaleLimit / 100` at **end time**, without netting out **claimable/accrued fees**. Claiming fees right before `endAuction()` can flip success → failure (FAILED_POOL_SALE_LIMIT). Conversely, a “successful” auction that pulls reserves **shrinks the fee base**, drastically cutting fee proceeds measured off current pool balance.
    - **Where to look (AMM tranches / coupon auctions / market makers):**
        - Auction finalize path (e.g., `Auction.endAuction()`): condition like
        `if (totalSellReserveAmount >= (IERC20(reserveToken).balanceOf(pool) * poolSaleLimit) / 100) { fail; }`
        - Pool fee logic: time-based/continuous fee accrual using `balanceOf(pool)` (e.g., `getFeeAmount()` → `(balance * fee * dt) / YEAR`) and `claimFees()` that **reduces** the pool balance.
        - Auction start path (e.g., `Pool.startAuction()`): no **snapshot** of sellable reserves or fee base; uses live balance later.
        - Transfer path (e.g., `Pool.transferReserveToAuction()`): pulls reserves after success decision, further shrinking fee base.
    - **Red flags:**
        - Live `balanceOf(pool)` in sale-limit math; no `snapshotBalance` at auction start.
        - Fee model that computes owed fees from **current** balance since `lastFeeClaimTime`, not piecewise segmented by balance changes.
        - No reservation/escrow of “max sellable” amount at auction start; auctions compete with fees for the same balance.
        - Comments like “limit auction to ≤90% of pool balance” with no mention of **net of fees** or “available for sale”.
        - Tests don’t cover ordering: (1) claim fees → end auction; (2) end auction → claim fees; (3) high bid near limit ±1 wei.
    - **Fix pattern:**
        - **Snapshot & reserve** at start: compute `sellCap = min(poolSaleLimit% * availableReserves, availableReserves - feeBuffer)` where `availableReserves = balance - accruedFees(now)`; store `sellCap` and decrement it on accepted bids. Finalize comparing `totalSellReserveAmount <= sellCapSnapshot`.
        - **Piecewise fee accrual:** Update fee state on **every balance-changing event** (create/redeem/auction pull) so fees are computed on the correct historical balances; or track **fee shares** independent of raw pool token balance.
        - **Atomic ordering:** In `endAuction()`, **settle fees first** (book or transfer), then evaluate success against **net reserves**; or vice-versa but use the same ordering consistently and test both paths.
        - **Explicit buffers/guards:** Add `minReserveBuffer` (e.g., for fees + safety) and require `totalSellReserveAmount <= sellCapSnapshot - minReserveBuffer`.
- [ ]  **Naive DEX-pool selection in oracle adapter (tick spacing / fee ignored) enables attacker-chosen pool**
    - **Symptom:** Price adapter “auto-discovers” a pool by iterating candidate tick spacings (or fee tiers) and picks the first match—often the tightest spacing—without checking pool class (stable/volatile), fee tier, liquidity depth, or age. An attacker can deploy a same-pair pool with tiny tick spacing / ultra-low fees and thin liquidity; the adapter locks to it and the market price becomes easy to skew.
    - **Where to look (UniswapV3/Velodrome/Aerodrome CL oracles):**
        - Oracle adapter init/selection: functions like `getPool()`, `_selectPool()`, `initialize(...)` that call `factory.getPool(tokenA, tokenB, tickSpacing)` or loop `for spacing in spacings`.
        - Hardcoded arrays: `uint24[] spacings = [1, 50, 200, 2000]` (or fee tiers) with “pick-first-that-exists” logic.
        - Lack of vetting around: minimum liquidity TVL, observation cardinality, pool age, pool fee tier/class (stable vs volatile).
        - Price path: `latestRoundData()` → reads TWAP from the chosen pool; check no fallback/anchor or sanity bounds vs reference feed.
        - Deployment wiring: is the **pool address immutable/whitelisted** or re-computed every call from a factory?
    - **Red flags:**
        - “Prefer tighter tick spacing” comment/logic for volatile pairs (e.g., Bond/USD) without any **fee-tier or volatility-class** filter.
        - No check like `require(liquidity >= minLiq)` / `poolTvl >= minUsd` / `poolAge >= minAge` / `obsCardinality >= minCardinality`.
        - Accepts **any** pool the factory returns; no governance allowlist; no multi-pool median/weighted aggregation.
        - Single-pool TWAP with short `twapInterval` and no **deviation clamp** vs anchor (Chainlink/median of venues).
        - Pool selection runs **on every read**, allowing mid-flight pool swaps if an attacker spins up a “better-matching” pool.
    - **Fix pattern:**
        - **Pin by config, not discovery:** Store an explicit, governance-approved `pool` (and class/fee tier) in the adapter; updates gated and time-delayed.
        - **Eligibility gating:** Before accepting a pool: `feeTier in allowed`, `tickSpacing in allowed for pairType`, `poolAge >= X`, `obsCardinality >= Y`, `liqUsd >= Z`.
        - **Volatility-aware selection:** For non-correlated pairs, **do not** favor the smallest tick spacing; prefer the standard “volatile” spacing/fee tier used by the venue.
        - **Liquidity-aware pricing:** If using multiple pools, compute a **liquidity-weighted median/mean** TWAP; drop outliers; require inter-venue agreement within a max deviation.
        - **Anchors & clamps:** Compare TWAP to a trusted anchor (e.g., Chainlink/other oracle); revert or clamp when `|twap - anchor| > maxDeviationBps`.
        - **Staleness & safety:** Enforce `heartbeat`, `min observations`, `minVolume` over the window; reject pools that don’t meet thresholds.
        - **Tests:**
            - Attacker deploys thin-liquidity, low-fee, tiny-spacing pool → adapter must **not** select it.
            - Governance rotates pool → requires delay & emits event; reads continue from old pool until activation.
            - Deviation/TVL thresholds: fuzz TVL and price; verify adapter reverts/clamps on manipulations.
    - **Questions to ask next time:**
        - How is the **pool chosen**—hardcoded, allowlisted, or auto-discovered each call?
        - Does selection check **fee tier, tick spacing, pool class (stable/volatile), age, and liquidity**?
        - What **minimum TVL / observation cardinality / TWAP window** is enforced?
        - Is there an **anchor oracle** and a **max deviation** gate between venues?
        - Can an attacker **front-run pool selection** by deploying a “better-matching” pool (tighter spacing / lower fee)?
        - Are pool changes **time-delayed** with the old pool used until the delay elapses?
- [ ]  **Period snapshot uses stale `sharesPerToken`, mis-crediting coupon claims**
    - **Symptom:** When rolling to a new distribution period (e.g., `Pool.startAuction()`), the token’s period snapshot (`globalPool.previousPoolAmounts[i].sharesPerToken`) is recorded from the **old, stored** `globalPool.sharesPerToken` instead of the **current** configured value for the just-ended period. If governance changed `sharesPerToken` since the last rollover, bondholders’ claim math uses the wrong per-period share.
    - **Where to look (period indexing / coupon accrual):**
        - `Pool.startAuction()`: call to `bondToken.increaseIndexedAssetPeriod(sharesPerToken)` right after computing the new period’s target coupon amount.
        - `BondToken.increaseIndexedAssetPeriod(uint256 sharesPerToken)`: push to `globalPool.previousPoolAmounts` and assignment to `globalPool.sharesPerToken`; verify which value is stored for the period being finalized.
        - Any setter: `Pool.setSharesPerToken(...)` (and guards like `NotInAuction`)—does governance change take effect **before** the snapshot?
        - Claim paths: `Distributor.claim()` → `BondToken.getIndexedUserAmount(...)` loop over `previousPoolAmounts[i].sharesPerToken`.
    - **Red flags:**
        - Snapshot code does:
        `previousPoolAmounts.push({ sharesPerToken: globalPool.sharesPerToken })` **then** `globalPool.sharesPerToken = newValue;`
        (i.e., it records the **old** value for the just-ended period).
        - Comments assume `sharesPerToken` is constant across periods, but there’s a public setter.
        - Rollover happens only at auction start; any interim parameter change is not latched into the period being closed.
        - No invariant/check that “snapshot value equals the effective `sharesPerToken` intended for the prior period”.
    - **Fix pattern:**
        - **Record what you intend to pay for the period you are closing**: push `sharesPerToken` **argument** (or an explicit `closingSharesPerToken`) into `previousPoolAmounts`:
            - `previousPoolAmounts.push({ ..., sharesPerToken: sharesPerToken /*param*/ })`
            - Then assign `globalPool.sharesPerToken = sharesPerToken` for the **new** open period.
        - Alternatively, flip the order: set `globalPool.sharesPerToken = newValueForNextPeriod` **after** pushing a struct that uses an explicitly captured **current/closing** value.
        - Emit events with both `closingSharesPerToken` and `newSharesPerToken` to audit state transitions.
        - Add guards/tests:
            - If `setSharesPerToken` was called since last rollover, assert the snapshot uses that updated value.
            - Unit test: change `sharesPerToken`, start auction, then verify `previousPoolAmounts[last].sharesPerToken == newConfiguredValue` and that claims match expectations.
    - **Questions to ask next time:**
        - What **exact value** represents the per-token coupon for the **period being closed** vs. the **period being opened**? Where is each stored?
        - Can governance change `sharesPerToken` between periods? If yes, **how is that value latched** into the snapshot at rollover?
        - Does the claim loop (`for i = lastUpdatedPeriod; i < period; i++`) rely solely on `previousPoolAmounts[i].sharesPerToken`? Is that value ever stale?
        - Are there tests where `setSharesPerToken` is called right before `startAuction()` validating claim correctness?
        - Is there an invariant like: “Sum of distributed coupons for period P equals `sharesPerToken(P) * supplyAtSnapshot(P)`?”
- [ ]  **Assumed USD Chainlink pairs (e.g., wstETH/USD, stETH/USD) don’t exist → oracle reverts**
    - **Symptom:** Oracle adapters call `getOraclePrice(token, USD)` (or direct `latestRoundData`) for tokens that **don’t have a USD Chainlink feed** (e.g., wstETH, stETH). Calls revert at runtime, bricking price queries and any function that depends on them (e.g., pool pricing, TVL, redemptions).
    - **Where to look (oracle adapters / feed wiring):**
        - Adapters like `BalancerOracleAdapter.latestRoundData()` and helpers that loop pool tokens and fetch `token/USD` directly.
        - Feed registries/mappings: `OracleFeeds.priceFeeds(token, USD)` (or equivalent). Check if it’s ever set for LSTs.
        - Normalization utilities: `getOracleDecimals(token, USD)`, `.normalizeAmount(...)`—these often assume a valid aggregator.
        - Initialization paths: adapter `initialize()` functions that accept only a pool/factory but **don’t validate required feeds** for all component tokens.
    - **Red flags:**
        - Hardcoded assumption that **every** pool token has a `token/USD` Chainlink pair.
        - No **composite route** support (e.g., `wstETH -> stETH -> ETH` via on-chain rate * `ETH/USD`).
        - No try/catch or graceful fallback if `priceFeeds[token][USD] == address(0)`; missing explicit “unsupported token” check.
        - LSTs present in pools (stETH, wstETH, rETH, cbETH) but only USD feed lookups implemented.
        - Adapters that compute per-token prices in a loop and **revert** on any missing feed (one bad token halts everything).
    - **Fix pattern:**
        - Implement **derived pricing paths** for LSTs:
            - `wstETH/USD = wstETH/stETH (unwrap rate) * stETH/ETH (Chainlink) * ETH/USD (Chainlink)`.
            - Or `stETH/USD = stETH/ETH (Chainlink) * ETH/USD (Chainlink)`.
        - Add a **routing layer**: per-token config that defines feed routes (direct feed, composite via base asset, TWAP fallback) with decimals accounted for.
        - Validate in `initialize()` (and on pool updates) that **all tokens** in the pool have a resolvable price path; otherwise revert with a clear error and/or skip unsupported tokens per design.
        - Narrow scope: if only the **reserve token** price is needed, don’t fetch **every pool token** price; compute BPT price using pool math that avoids USD feeds for non-reserve tokens, or pre-map only required assets.
        - Add unit tests for: pools including stETH/wstETH; missing direct USD feeds; composite path correctness; revert messaging; and per-token decimals normalization.
    - **Questions to ask next time:**
        - Do all assets we touch actually have a **USD aggregator** on the target chain? If not, what’s our **route**?
        - For LSTs, are we using **on-chain exchange rates** (e.g., wstETH↔stETH) plus Chainlink **ETH/USD**? Are the decimals and staleness handled?
        - What’s the behavior when a token is **unsupported**? Do we fail loudly at init, or crash at runtime inside a loop?
        - Are adapters **overfetching** (pricing every pool token) when only a subset is needed?
- [ ]  **Per-intent health check ignores cumulative pending PnL → multi-intent drain**
    - **Symptom:** Margin/health validation includes only the *current* intent’s price-adjustment PnL (intent price vs. market price) and ignores other pending intents for the same account. Attackers submit many adverse-priced intents that each pass health individually but together drive the account deeply negative after settlement, letting the counterparty withdraw inflated profit.
    - **Where to look (intents/order flow/margin):**
        - Intent admission path (e.g., `Market.update` / `submitIntent`) → invariant/health checks (e.g., `InvariantLib.validate`, `…validateUpdate`, `…checkCollateralization`).
        - Price-adjustment/guarantee structs (e.g., `Guarantee`, `PendingOrder`, `PriceAdjustment`) and how “new” vs. “pending” guarantees are applied.
        - Settlement rollups that realize `intentPrice - oraclePrice` PnL across *multiple* intents in the same epoch/batch.
        - Limits like `maxPriceDeviation`, intent `nonce/group/expiry`, and whether multiple outstanding intents per account are allowed concurrently.
    - **Red flags:**
        - Validation uses a single `newGuarantee` or “this intent” delta: `adjustedCollateral = collateral + pnl(newIntent)` with no aggregation over `pendingIntents[account]`.
        - No per-intent **reserved margin** / lock; no cap on outstanding intents; admission doesn’t settle previous intents or reject if any are pending.
        - Phrases like “latest”, “current”, or `pending.latestVersion` are used to compute health, rather than summing *all* unsettled entries.
        - PnL from `intentPrice` vs. reference price accounted only at settlement, not reserved at admission.
        - Off-by-policy windows (same epoch/price version) allow many intents to queue before the first settles.
    - **Fix pattern:**
        - **Aggregate pending PnL** in health checks: compute `collateral + Σ PnL(all pending intents under worst-allowable price)` ≥ maintenance.
        - **Reserve margin per intent** (lock collateral) and decrement available collateral on admission; release on settlement/cancel.
        - **One-pending-intent** per account per market (or per side) or hard cap with strict sequencing; require settle/cancel before new admission.
        - **Batch settle-then-admit**: settle all prior pending intents for the account before accepting a new one in the same update.
        - Include **worst-case** price adjustment based on `maxPriceDeviation` during admission (not just current oracle).
        - Add invariant tests covering multiple intents within one epoch that individually pass but collectively fail after settlement.
    - **Questions to ask next time:**
        - Can an account have **multiple pending intents** simultaneously? If so, **where** is their combined PnL reserved in health?
        - Does the health check **sum over all pending** `Guarantee/Intent` entries or only the current one?
        - Is there a **per-intent margin lock** (available collateral tracker) that reduces spendable/withdrawable balance upon admission?
        - What prevents submitting N intents within one **price/version window** before any settlement occurs?
        - Are there **caps** on outstanding intents per account/market? Are earlier intents auto-settled or canceled on new submissions?
        - Does `maxPriceDeviation` create an **adversarial PnL** window that must be budgeted at admission?
        - On settlement, could realized PnL make the account **retroactively under-margined** relative to what admission allowed? If yes, how is that prevented?
- [ ]  **Mismatched margin contexts (pending vs. settled) enable “safe” withdrawal → instant liquidation**
    - **Symptom:** Normal updates check margin against a **pending/current** position (e.g., `currentPositionLocal`) while liquidation checks maintenance against the **latest/settled** position (e.g., `latestPositionLocal`). Users can reduce position + withdraw collateral “safely” per the pending state, then be liquidated immediately based on the settled state.
    - **Where to look (perps/intents engines):**
        - Update admission path (withdraw / position change): functions like `Market.update(...)` → `InvariantLib.validate(...)`.
        - Liquidation gate: e.g., `InvariantLib.validate(..., protected=true)`, `…maintained(...)` checks.
        - Position contexts: any `currentPosition`, `pending`, or “pre-commit” vs. `latestPosition`, “post-commit/settled”.
        - Epoch/versioning: use of `latestOracleVersion` vs. “current epoch” projections; pending invalidation flags.
        - Collateral path: where `collateral.add(priceAdjustment)` is computed only for pending/current, not latest.
    - **Red flags:**
        - Margin check uses **pending/current** position; liquidation uses **latest** position.
        - Collateral withdrawal allowed **before** settlement of the position decrease.
        - Comments/vars like “expected at commit”, “current epoch”, “pending” in margin checks.
        - Asymmetric price-adjustment application (intent price override applied only in one path).
        - No “settle-then-withdraw” requirement; no lock on collateral while decrease is uncommitted.
    - **Fix pattern:**
        - **Unify basis**: use the **same position context** for margin and maintenance (either both latest+worst-case-pending or force settlement first).
        - **Settle-before-withdraw**: require settling prior epochs/pending intents before allowing collateral reductions.
        - **Reserve margin for pending changes**: lock collateral until the position change commits; compute health as `latestPosition ± worst-case(pending)` consistently for both update and liquidation.
        - Apply **identical price-adjustment logic** (intent price vs. oracle) in both normal and liquidation checks.
        - Add tests: reduce position + withdraw in same epoch → should not become liquidatable solely due to context switch.
    - **Questions to ask next time:**
        - Which position (latest vs. current/pending) is used in **every** margin/maintenance check path?
        - Can users **withdraw** based on a **pending** position that might be invalidated at commit?
        - Are liquidation checks **byte-for-byte consistent** with update-time margin checks (same position, same price, same adjustments)?
        - Is there a **settlement gate** (or margin reservation) before permitting collateral decreases?
        - How are **price overrides** (intent price / deviation windows) applied across both update and liquidation paths?
- [ ]  **Guaranteed intents + deferred fee booking let users withdraw collateral before fees hit**
- **Symptom:** Intent-sourced orders are guaranteed (no invalidation), but trading fees for those orders are only applied at checkpoint/settlement. Users can open → close via intents and then withdraw **all** collateral before fees post; once the epoch commits, fees are charged to a zeroed account (bad debt) while referrer/protocol still receive their cut.
- **Where to look (perps/intents engines):**
    - Update admission & health: functions like `Market.update(...)` → `InvariantLib.validate(...)` (or equivalent) that compute margin/maintenance and **do not** subtract pending taker/protocol/referral fees.
    - Settlement pipeline: libraries like `CheckpointLib` (or “advance/settle/commit” logic) where fees are actually realized.
    - Intent/guarantee structs: e.g., `Guarantee` / `invalidation == 0` (“guaranteed”) paths that allow position changes even when the opening epoch isn’t committed yet.
    - Withdrawal paths permitting collateral decrease while **pending fees exist**.
    - Referral accounting: when/where `claimable` referral fees are minted/credited vs. when the payer’s fee liability is applied.
- **Red flags:**
    - Margin check collat uses `local.collateral` ± price adjustments, but **omits `pendingFeeLiability`** from intents in current epoch.
    - Comments like “fees applied on commit/settlement” without a matching **pre-book/lock** in update-time checks.
    - Guaranteed intents (`invalidation == 0`) let users fully close positions and withdraw in the **same epoch**.
    - Referral/protocol fees become claimable **before** the payer’s fees are deducted from collateral.
    - No “settle-before-withdraw” guard; no cap like “max withdraw ≤ (freeCollateral − pendingFees)”.
- **Fix pattern:**
    - **Include pending fees in health:** compute `effectiveCollateral = collateral − sum(pending taker/protocol/referral fees for guaranteed/pending intents)` in **both** margin and maintenance checks.
    - **Lock fees on admit:** reserve/escrow the **worst-case fee** per pending intent at update-time; release/adjust on settlement.
    - **Synchronize accounting:** ensure the position/fee context used for normal updates matches liquidation checks; apply identical fee debits in both.
    - **Withhold fee payouts:** delay referral/protocol claims until the payer’s fee debit is settled/paid; or pay from a separate reserve that is funded only after successful debit.
    - **Gating:** require settling current epoch (or flushing pending guaranteed intents) before allowing large collateral withdrawals or full closes.
    - **Tests:** open→close via intents, then withdraw full collateral within same epoch; assert withdrawal is limited by pending fees and that no bad debt or free referral claim arises.
- **Questions to ask next time:**
    - Are **all** pending/guaranteed intents’ fees subtracted from collateral during update-time health checks?
    - Can users reduce/zero positions and withdraw before fee realization? What stops them?
    - When are referral/protocol fees **claimable** vs. when the payer’s fee liability is **booked**?
    - Do liquidation and normal update paths use the **same** collateral-minus-fees basis?
    - What is the **worst-case** fee assumption per intent (slippage, partial fills), and is it reserved upfront?
- [ ]  **Liquidation path allows position *increases* (protected updates) enabling 2⁶²−1 grief & market drain**
- **Symptom:** During liquidation (`protected=true`), the engine only checks `pending.negative == latest.magnitude()` (i.e., “pending full close”) and then proceeds, but does **not** forbid positive deltas. A liquidator can *increase* the target account’s position (even to storage max like `2**62 - 1`), bypassing normal margin/size checks. The next tiny price move yields outsized PnL that can bankrupt the market.
- **Where to look (perps/liquidation engines):**
    - Liquidation admission in invariants: e.g., `InvariantLib.validate(...)` protected branch, the “no pending zero-cross” / “full close” logic.
    - Update entrypoints carrying a `protected`/`isLiquidation` flag (e.g., `Market.update(..., protected)`), especially the code that computes/accepts `delta` for the liquidated account.
    - Position math structs (`Position`, `Pending`, `Magnitude`) and checks comparing `latestPosition`, `pending.negative/positive`, and `currentPosition`.
    - Global/market accounting that trusts the updated position size without re-applying risk limits in protected mode.
    - Numeric limits: signed/fixed types (e.g., 62-bit) and any saturation/upper-bound guards on position size.
- **Red flags:**
    - Protected path only verifies a “pending equals latest magnitude” condition, but **never** enforces `delta ≤ 0` (no increase) for the liquidated taker.
    - Comments like “liquidate with full close” while code still accepts arbitrary `delta` (including positive) in the same branch.
    - Margin/maintenance/size caps explicitly **skipped** when `protected==true`.
    - Ability to submit liquidation updates that set `pending.positive > 0` or grow `currentPosition`/`targetPosition`.
    - Lack of explicit cap: `abs(newPosition) ≤ abs(latestPosition)` (or `= 0` for force-close) under liquidation.
- **Fix pattern:**
    - **Monotone decrease under liquidation:** In `protected` updates, require `delta ≤ 0` for taker (or `abs(new) ≤ abs(latest)`), and **reject** any positive change. If `pending.negative == latest.magnitude()` (full close pending), **freeze** further taker size changes (allow only `delta = 0`).
    - **Force close semantics:** Optionally compute `delta = -latest.magnitude()` internally for liquidation, ignoring external `delta`, to guarantee a pure close.
    - **Re-enable risk gates:** Even in protected mode, enforce position upper bounds and sanity checks (overflow, max size).
    - **Invariant parity:** Use the same position basis for liquidation and normal checks (e.g., `latestPosition`), and assert no path can increase magnitude in protected flows.
    - **Tests:**
        - (a) Close-pending then liquidation with positive `delta` → **revert**.
        - (b) Liquidation cannot set position above max or above `latest`.
        - (c) Tiny price move after liquidation cannot produce unbounded PnL due to liquidation-grown size.
- **Questions to ask next time:**
    - In `protected` updates, is **any** positive `delta` possible for the liquidated account?
    - Do we ever skip margin/maintenance/size validations under liquidation? If yes, what guards replace them?
    - Can “pending full close” state be used to smuggle a positive `delta`?
    - Are position numeric bounds (bit-width caps) enforced at all write points, including liquidation?
    - Do liquidation and normal update paths validate against the **same** notion of position (latest vs current/pending)?
- [ ]  **Memory-context overwrite drops credits when `account == beneficiary` in settle flows**
- **Symptom:** Calling a settle/checkpoint routine with `account` equal to the fee/beneficiary (e.g., `coordinator`) mints/credits shares to storage inside the loop, but a final `_saveContext` writes an outdated **memory** snapshot back to storage and **overwrites** the just-credited balance → the beneficiary silently loses profit shares.
- **Where to look (vaults/settlers with “Context” patterns):**
    - Settle/checkpoint loops that compute `profitShares` then call `_credit(...)` (or `_debit(...)`) during iteration.
    - Helpers like `_credit(address, UFixed)` that **read+write storage directly** (e.g., `_accounts[addr].read()` / `.store()`), while the caller also maintains a `Context memory context` with `context.local`/`context.global`.
    - Finalizers such as `_saveContext(context, account)` that unconditionally do `_accounts[account].store(context.local)` after earlier storage mutations.
    - Any path where `beneficiary` (fees/referrals/coordinator) **can equal** the `account` being settled.
- **Red flags:**
    - `_credit(account, shares)` writes storage but **does not** update `context.local.shares`.
    - `_saveContext` does `if (account != address(0)) _accounts[account].store(context.local);` regardless of interim storage changes.
    - Comments like “credit coordinator” in-settle while the settle target can be the same address.
    - Mixed “global” vs “local” accounting with `address(0)` sentinel plus per-account stores in one function.
    - No test case for `settle(account = coordinator/feeRecipient/referrer)`.
- **Fix pattern:**
    - **Single source of truth during settle:** make `_credit` accept and mutate the live `Context` when `account == context.account` (e.g., `context.local.shares += shares`) and only hit storage for other addresses; or
    - **Ordering fix:** call `_saveContext` **before** storage credits, or skip the final `store(context.local)` when `account == beneficiary` and you’ve already credited storage; or
    - **Unified writes:** refactor `_credit/_debit` to *only* buffer into `context` during settle and persist once in `_saveContext`; or
    - **Guard:** disallow `settle(account == beneficiary)` if refactor isn’t feasible (least-preferred).
    - **Tests:** add property tests ensuring credited shares survive `_saveContext` for `account ∈ {beneficiary, coordinator, referrer}`.
- **Questions to ask next time:**
    - Do any helpers mutate storage mid-settle while a **memory** context will later be stored over them?
    - Can `account` passed to `settle` equal the fee sink (coordinator/beneficiary/referrer)?
    - Does `_credit/_debit` ever update `context.local` to keep memory and storage in sync?
    - Is `_saveContext` conditionalized to avoid overwriting recent storage-side credits?
    - Are there similar flows (fee accrual, referral payout, performance fees) that mix context writes and storage writes in one call?
- [ ]  **Liquidation path ignores intent price-adjustment PnL → unfair liquidations / unliquidatable debt**
- **Symptom:** Margin admission (normal update) adds a **priceAdjustment** from intent price vs. oracle price to collateral, but the **protection/liquidation** path checks health **without** that adjustment. Result: users can be liquidated while actually healthy (pending profit from intents omitted), or liquidation can revert while they’re unhealthy (pending loss omitted).
- **Where to look (perps with signed intents/guarantees):**
    - Invariant/validation library: the branch for `protected == true` (liquidation) vs. normal admission.
        - E.g., `InvariantLib.validate(...)`: look for `maintained(...)` checks using `context.local.collateral` **alone** and `latestPositionLocal` only.
    - Admission path for intents: where `newGuarantee.priceAdjustment(oraclePrice)` (or equivalent) is **added** to `collateral` when checking `margined(...)`.
    - Guarantee/Intent math: helpers computing `(intentPrice - oraclePrice) * pendingMagnitude` or similar.
    - Position sources used in each path: `currentPositionLocal` (latest + pending) in margin checks vs. `latestPositionLocal` in liquidation checks.
    - Pending queues across epochs: any loop/accumulator over **pending guarantees** that’s **skipped** in the liquidation branch.
- **Red flags:**
    - Liquidation health uses only `context.local.collateral` (no sum of pending, non-invalidated intent PnL).
    - Margin admission does `collateral.add(priceAdjustment)`; liquidation uses raw collateral.
    - Guaranteed intents (`invalidation == 0`) exist but their PnL isn’t considered in protection.
    - Divergent position bases: `currentPosition` for margin vs. `latestPosition` for liquidation, with no intent deltas applied for protection.
    - No tests for: (a) near-edge accounts with profitable pending intents being liquidated, (b) accounts with lossful pending intents **not** liquidating.
- **Fix pattern:**
    - **Unify health calculation:** create a single “effective collateral/position” function used by both admission and protection that:
        - Adds the **sum of price adjustments** for all **non-invalidated** pending intents from `(latest+1 … current)` to collateral.
        - Optionally applies **pending intent position deltas** (e.g., closing intents) to the position used for liquidation health so protection reflects guaranteed updates.
    - Or, require settlement (or intent netting) before liquidation when non-empty guaranteed intents exist (weaker UX).
    - Cap/check aggregation to avoid overflow and ensure deterministic ordering across epochs.
    - Add tests: profitable pending-intent → must **not** liquidate; lossful pending-intent → **must** liquidate; mixed multi-intent scenarios.
- **Questions to ask next time:**
    - Does the liquidation path include **the same** price-adjustment (intent PnL) applied during margin admission?
    - Are **guaranteed** intents (not invalidatable) explicitly incorporated into liquidation health (both collateral and position)?
    - Are we summing **all** pending adjustments across epochs or just the “current” one?
    - Do margin and liquidation share a single health helper, or are there two divergent implementations?
    - What happens if pending intent closes half the position—does liquidation still check against the **pre-close** size?
- [ ]  **Over-close pending blocks liquidation (pending close > latest size)**
- **Symptom:** Accounts become **temporarily un-liquidatable** when the total pending close (negative pending) exceeds the last committed position size. Liquidation updates (protected orders) always revert due to an “over-close” guard that’s relaxed for intents but **still enforced** for liquidations.
- **Where to look (perps with intents & protected/liq updates):**
    - Invariant/validation library for protected (liquidation) path:
        - Check the branch where `protected == true` (or equivalent) and any guard like
        `if (pending.neg() > latest.magnitude() && pending.invalidation != 0) revert OverClose`.
    - Admission rules for **intents vs non-intents**:
        - Intents typically set `invalidation == 0` (guaranteed), allowing `pending.neg() > latest.magnitude()`; non-intent/liquidation uses `invalidation == 1`, **forbidding** it.
    - Special cases:
        - Code that special-cases **zero-crossing** during liquidation (e.g., “must be empty order”) but **omits** a similar carve-out for **over-close** states.
    - Pending queue across epochs: confirm whether liquidation logic can progress when there’s already excessive pending negative.
- **Red flags:**
    - A hard revert on `pending.neg() > latest.magnitude()` applied **unconditionally** to protected updates.
    - Intents can create `pending.neg() > latest`, but liquidation has **no escape hatch** (no “empty order” allowance) until settlement.
    - No fallback to **settle before liquidate** when over-close prevents liquidation.
    - Tests missing for: (a) large pending close + price move → liquidation should succeed; (b) liquidation while `pending.neg() > latest`.
- **Fix pattern:**
    - Mirror the zero-cross rule: when `protected == true` and `pending.neg() > latest.magnitude()`, **require liquidation order to be empty** (no additional close) instead of reverting, allowing the liquidation call to proceed.
    - Alternatively, allow the engine to **settle / net** pending first (or simulate netting) for liquidation health checks so protection isn’t blocked by bookkeeping.
    - Add invariant helper that computes an **effective latest position** under guaranteed pending closes for liquidation-only checks, without permitting new size changes.
    - Add tests where pending close exceeds latest: liquidation must not revert, and must not modify size except to execute protection semantics.
- **Questions to ask next time:**
    - Can intents legally produce `pending.neg() > latest.magnitude()`? If so, what is the **liquidation** behavior in that state?
    - Does the liquidation path have a **non-reverting** path (e.g., empty order) when over-close is present, like it often does for zero-cross?
    - Is there a mechanism to **settle/net** pending before protection or to **virtually** account for it in the liquidation check?
    - Do margin/maintenance checks use the **same** position basis across normal vs. protected updates in over-close scenarios?
- [ ]  **Unvalidated “asset container” lets attacker spoof constituents and drain via double-counted deltas**
- **Symptom:** A function accepts an arbitrary “pod/vault/pool” address, pulls its asset list, snapshots `balanceOf(this)` **per index**, performs an action, then computes received amounts as `after - before` **per index**. A malicious adapter returns **duplicate tokens** and/or **attacker-controlled config (e.g., zero lock duration)**, so the same token is counted multiple times and the contract credits/transfers **2× (or N×)** what it actually received.
- **Where to look (wrappers/lockers/migrators/adapters):**
    - Entry points that take an external “container” param: `debondAndLock(address pod)`, `unwrap(address vault)`, `migrate(address pool)`, `zapIn(address lp)`.
    - Calls into untrusted interfaces like `getAllAssets()`, `getConfiguration()`, `assets(uint256)`, `underlying()` followed by:
        - Pre/post loops capturing `_balancesBefore[i] = IERC20(_tokens[i]).balanceOf(address(this))`.
        - Post loops computing `_receivedAmounts[i] = balanceOf(this) - _balancesBefore[i]`.
    - Any place that assumes **asset arrays are unique & well-formed** (length==2, no duplicates, ordered).
    - Lockers that let the external pod/config dictate **duration/penalties** (e.g., zero duration enabling immediate withdrawal).
- **Red flags:**
    - No whitelist/registry check: `_pod` can be **any address**.
    - No verification that returned assets are **unique**, non-zero, and non-duplicated.
    - Arithmetic performed **per index** rather than **per token**, enabling dupes to sum extra credit.
    - Trusting external config (duration, fees) from the pod to govern internal lock/withdraw logic.
    - Using array-sized local buffers (`_tokens`, `_balancesBefore`, `_receivedAmounts`) driven by **untrusted length**.
    - Missing invariant checks like `tokens.length == expected` or `tokensSet.size == tokens.length`.
- **Fix pattern:**
    - **Authorize the container:** Require `_pod` to be created by a known **factory/registry** (`require(factory.isPod(_pod))`), or hard-code supported pods.
    - **Deduplicate & aggregate:** Build a **per-token map** of deltas: `mapping(address=>uint256 received)`; compute *_before* once per unique token, then after the action compute `_received[token] = after - before`; reject if any token repeats.
    - **Validate shape:** Enforce `tokens.length` within expected bounds; `all tokens != address(0)`; `no duplicates`; `decimals/symbol sanity` if useful.
    - **Own your policy:** Don’t read lock duration or withdrawal gates from untrusted pod config. Enforce **min lock >= protocol constant**, fee policy, and withdrawal schedule **in your contract**.
    - **Defensive accounting:** Prefer **pull** accounting (track what you transfer out) over balance-delta when feasible; otherwise bounds-check that `sum(received) == actual net inflow`.
- **Questions to ask next time:**
    - Can a user pass **any** container address? How is a pod/vault **authenticated**?
    - Do we **assume** returned asset arrays are unique/ordered? What happens with **duplicates**?
    - Is received-amount computed **per index** or **per token**? Do we **aggregate** deltas by token?
    - Who sets lock duration/fees—**us** or the external pod? Can the pod trick us into **instant unlocks**?
    - Do we cap/validate the **number of assets** and guard against **unbounded memory arrays** from untrusted inputs?
- [ ]  **First-mint goes to a live EOA (not “dead shares”), enabling ERC-4626 vault inflation via rounding cycles**
- **Symptom:** Factory “seeding” mints the minimum deposit to the **caller** instead of a burn/dead address. With tiny supply, an attacker nudges `totalAssets` (e.g., by donating rewards), then uses deposit/withdraw rounding (e.g., deposit `2*assets-1` → mint 1 share; withdraw 1 asset → burn 1 share) to ratchet price per share up, siphoning value from later depositors.
- **Where to look (ERC-4626 vaults / autocompounders / factories):**
    - Factory bootstrap hooks like `AutoCompoundingPodLpFactory._depositMin`, `initialize()`, `seed()`, `mintBootstrapShares()` that call vault `deposit(minimumDepositAtCreation, _msgSender())`.
    - Vault math in `deposit/mint/withdraw/redeem`, especially conversions: `convertToShares`, `convertToAssets`, `previewDeposit/Withdraw`, and whether they **round down/up**.
    - Any path that increases `totalAssets` **without mint/burn** (auto-compounding rewards, fee skim, token donations).
    - Deployment-time guards: is the **first deposit restricted** and are shares minted to a **sink** (e.g., `address(0)` / `0xDead`)?
- **Red flags:**
    - `factory._depositMin(...)` → `vault.deposit(minimumDepositAtCreation, _msgSender())` (live recipient).
    - No “minimum liquidity” lock (à la Uniswap’s `MINIMUM_LIQUIDITY`) or **virtual assets/shares**.
    - Rounding allows `deposit(2*A-1)` to mint exactly 1 share and `withdraw(1)` to burn 1 share when `totalSupply` is tiny.
    - `totalAssets` can tick up from external transfers/rewards before supply grows (share price jump with no dilution).
    - First deposit not gated (`require(totalSupply()==0 ? onlyFactory : true)`), allowing front-run/MEV on bootstrap.
- **Fix pattern:**
    - **Burn the seed:** On bootstrap, mint the minimum shares to a dead sink (e.g., `address(0)` / `0xdead`) funded by factory-provided assets:
        - Factory pulls assets from itself (not user), approves vault, then `vault.deposit(minimum, DEAD_ADDRESS)`.
        - Or directly `_mint(DEAD_ADDRESS, MIN_SHARES)` while transferring/locking equivalent assets in the vault.
    - **Use virtual balances:** Adopt **virtual assets/shares** (e.g., Solmate-style ERC4626) so conversions use `totalAssets + VIRTUAL_ASSETS` and `totalSupply + VIRTUAL_SHARES` to neutralize low-supply rounding attacks.
    - **Gate first deposit:** `require(totalSupply()==0 ? msg.sender==factory : true)` and/or a one-shot `initialize()` that sets price and burns seed shares atomically.
    - **Rounding discipline:** Ensure `deposit` mints `shares = assets * totalSupply / totalAssets` (rdn) and `mint` pulls `assets = shares * totalAssets / totalSupply` (rup) using consistent `mulDiv` with explicit rounding modes; add min-amount guards to block 1-wei ratchets when supply is tiny.
    - **Test vectors:** Unit tests where:
        - Seed shares go to EOA vs burn address.
        - Exogenous `totalAssets` increase (reward/donation) before/after first user deposit.
        - The `2*A-1` / `+1 asset` cycle fails to inflate when virtual balances or dead shares are in place.
- **Questions to ask next time:**
    - Who receives the **very first shares**? Can they ever be **redeemed**?
    - Can `totalAssets` **increase without minting** (rewards/donations)? How are low-supply rounding edges handled?
    - Do we implement **virtual assets/shares** or a **minimum-liquidity burn** to set an initial exchange rate?
    - Are first-deposit and bootstrap steps **atomic and access-controlled** to the factory?
    - Do `previewDeposit/Withdraw` and `convertTo*` use **explicit rounding** and reject dust operations that could ratchet price?
- [ ]  **Fee deposit of non-reward tokens reverts when no stakers (LEAVE_AS_PAIRED_LP_TOKEN-style DoS)**
- **Symptom:** Any action that triggers “process fees → deposit into rewards” reverts when `totalShares == 0` because the distributor rejects deposits of tokens that aren’t the `rewardsToken` (e.g., paired LP). With “leave as paired LP” flags, no one can stake or withdraw/debond—perma-DoS.
- **Where to look (EVM/Solidity):**
    - Fee pipelines: functions named like `_processPreSwapFeesAndSwap`, `_processFees`, `distributeFees`, `harvest`, `compound`.
    - Rewards/Distributor contracts: `_depositRewards`, `depositFromPairedLpToken`, `notifyRewardAmount`, `queueNewRewards`, and any `require(totalShares > 0)` or `require(stakers > 0)` guard.
    - Config flags and branches: `LEAVE_AS_PAIRED_LP_TOKEN`, `leaveAsLp`, `skipSwap`, `payInKind`, `payInLP`, `keepLP`.
    - Staking share state: `totalShares`, `totalSupply`, `numStakers`; places where fee processing runs **before** stake minting or **before** first staker exists.
    - LP paths: V2/V3 add/remove liquidity helpers; check for dust/“dead shares” that keep pool balance > 0 and continually trigger the LP-branch.
- **Red flags:**
    - Rewards deposit code path: `if (token != rewardsToken) require(totalShares > 0)` (or revert) with **no buffering/fallback**.
    - Fee processing executed on **stake/debond** before shares are minted (ordering: process fees → mint shares).
    - No “pending buffer” for non-reward tokens when there are zero stakers; no “flush” function once stakers exist.
    - Hard switch to “leave as LP” without a bootstrap staker or sentinel share (e.g., no seed mint to a burn address).
    - No recover path: inability to bypass or defer `depositFromPairedLpToken` when `totalShares == 0`.
- **Fix pattern:**
    - **Buffer, don’t revert:** When `token != rewardsToken && totalShares == 0`, accumulate into `pendingTokenBalances[token]` and return; add `flushPending()` that forwards once `totalShares > 0`.
    - **Reorder calls:** Mint stake shares first, then process fees, so `totalShares > 0` by the time of deposit.
    - **Bootstrap:** On enabling “leave as LP,” require `totalShares > 0`, or seed 1 wei share to a burn address (not EOAs) to avoid reverts.
    - **Config guard:** At init or when toggling the flag, `require(!leaveAsLp || totalShares > 0)`.
    - **Operational fallback:** Allow skipping fee processing when no stakers, or swap LP → `rewardsToken` if buffering is not supported.
- **Questions to ask next time:**
    - What happens if fees arrive before the **first** staker? Is there a buffer or does it revert?
    - Does the fee pipeline ever deposit **non-reward tokens** (LP, paired asset, basket token) into the distributor? Under what flags?
    - Is fee processing executed **before** staking mints shares or **before** debond burns shares?
    - Can anyone “flush” buffered fees once `totalShares > 0`? Is there a keeper hook?
    - If an AMM pool leaves **dust LP**, will that continually select the LP branch and brick flows when `totalShares == 0`?
- [ ]  **Auto-compounder ignores leftover leg → perpetual imbalance → LP add reverts (stuck funds/DoS)**
- **Symptom:** Zap logic does: reward → paired, swap ~half to pTKN, then `addLiquidity`. If a prior `addLiquidity` failed (or attacker donates pTKN), leftovers accumulate on one side. On subsequent epochs the algo still swaps ~half of *new paired* (ignoring current pTKN balance), worsening the imbalance; `addLiquidity` min-outs fail repeatedly → compounding halts and tokens get stuck.
- **Where to look (EVM/Solidity vaults/zaps):**
    - Auto-compounders / zaps: functions like `_tokenToPodLp`, `_pairedLpTokenToPodLp`, `compound()`, `harvest()`, `autoCompound()`.
    - “Half-split” or “optimal swap” helpers: `_getSwapAmt`, `optimalSwapAmount`, `quoteZap`, `sqrt`based formulas using pool reserves.
    - Add-liquidity calls: `addLiquidity`, `addLiquidityV2/V3`, check `amountAMin/amountBMin` (slippage) derivation.
    - Balance resets after swap: code that re-reads `balanceOf(this)` and then feeds *all* balances into `addLiquidity`.
    - Handling of leftovers/donations: any path that tolerates or clears imbalanced residuals (or lack thereof).
- **Red flags:**
    - Swap amount function uses **only `_amountIn`** (new paired) + pool reserves, **not** current contract balances of pTKN/paired; e.g.
    `swapAmt = f(reserves, _amountIn)` with no `balanceOf(pTKN)` input.
    - Comments like “swap half then add LP” or fixed ratio split; no branch that “consumes leftover of the other leg first”.
    - After a swap: code sets
    `_podAmountOut = pod.balanceOf(this); _pairedRemaining = IERC20(paired).balanceOf(this) - fees;`
    and immediately tries `addLiquidity(...)` with those **raw** balances.
    - Add-liquidity min-outs computed **proportionally** to provided amounts (same % slippage on both legs). This fails when inputs are far from pool ratio.
    - No “buffer”/threshold to accumulate until a balanced add is possible; no one-sided-liquidity zap path.
    - No protection against grief-donations (attacker can transfer extra pTKN to skew ratios).
- **Fix pattern:**
    - **Balance-aware swap sizing:** compute swap based on **current balances + new input** to target pool ratio. Example (V2 50/50):
        - Let `Pa, Pb` be pool reserves; `xa = pairedBal`, `xb = pTknBal`. Choose direction to swap from the **excess leg** toward the deficit to achieve `(xa'/xb') ≈ Pa/Pb` (use exact formula or iterative quote with router’s `getAmountOut`).
        - If leftovers exist on pTKN side, first set `pairedToUse = min(xa, xb * Pa/Pb)` and **skip or reduce** the new swap accordingly.
    - **One-sided add support:** when imbalance is large, use a router zap routine (swap optimal portion internally) or perform two-step: swap from excess to deficit until within tolerance, then add.
    - **Adaptive min-outs:** derive `amountAMin/amountBMin` from **expected post-swap balances** + conservative slippage, not a flat percentage on imbalanced inputs.
    - **Leftover handling:** keep per-token buffers (e.g., `_pendingPaired`, `_pendingPod`) and only attempt add when `|ratio - target| < eps`; otherwise store and return early. Expose `flush()` to retry.
    - **Donation resistance:** cap how much of an *unexpected* token balance is considered per cycle (e.g., clamp to previous-cycle-balance + harvest delta), or sweep donations to treasury.
    - **Failure path hygiene:** on `addLiquidity` failure, **don’t** perform another “half swap” next time; recompute from fresh balances; reset allowances.
- **Questions to ask next time:**
    - Does the swap sizing consider **existing** pTKN/paired balances on the contract, or only the new harvest amount?
    - What happens if I donate 1 unit of one leg? Will future compounds keep making the ratio worse?
    - Do you ever **skip** swapping when the other leg already has enough to pair? Any buffer/threshold before attempting LP?
    - How are `amountAMin/amountBMin` computed—proportional to inputs, or from an expected ratio? What if inputs are 1:3 and pool is 1:1?
    - Is there a one-sided add or zap helper, or do you rely solely on two-sided exact adds?
    - On repeated `addLiquidity` failure, is there a backoff/flush, or will the strategy loop forever with increasing imbalance?
- [ ]  **LP fair-price math ignores wrapper ratios / wrong denomination → mispriced LP (oracle drift)**
- **Symptom:** Oracle that prices an LP (e.g., spTKN) against a base token assumes `pairedLpTKN == base` or 1:1 conversion. When `pairedLpTKN` is a *wrapper* (pods, Fraxlend pairs, ib/wrapped tokens), it misplaces the conversion factor and/or inverts a ratio, yielding systematically high/low LP prices. Downstream lending/health checks over/underestimate value.
- **Where to look (Solidity, AMM oracles / LP vaults):**
    - Files with LP pricing helpers using V2 “fair price” math: `_calculateSpTknPerBase`, `_priceBasePerPTkn`, `_getReserves`, `sqrt(k)`, `totalSupply()`.
    - Any path that converts between **pTKN ↔ base**, then to **LP ↔ paired**: functions like `_calculateBasePerPTkn`, `_calculateSpTknPerBase`.
    - Wrapper conversions:
        - Pods / share tokens: methods like `getPodPerBasePrice()`, CBR (collateral:backing ratio), unwrap fees.
        - Fraxlend pairs / yield shares: `convertToAssets()`, `convertToShares()`.
    - Denomination flips: code that does `price = 1e36 / price` (or similar “reverse” step) after mixing units.
    - Decimals normalization: `kDec`, `IERC20Metadata(...).decimals()`, raising 10**(18/2), etc.
- **Red flags:**
    - Assumes `pairedLpTKN/baseTKN == 1` (or never derives that price at all).
    - Applies wrapper conversion **after** inverting the LP price, or at the wrong stage of the fair-price formula.
    - Uses `sqrt(k * priceBasePerPTkn)` without adjusting for **paired/base** via the wrapper’s asset↔share ratio.
    - Order-of-ops smells: compute `spTKN/paired`, then **invert first**, *then* multiply by wrapper ratio; or divide by a ratio outside the square root when it must affect the under-the-root product of token prices.
    - Hand-rolled decimal scalars like `10 ** (18 / 2)` with integer division; `kDec` not matching reserve decimals; LP `totalSupply()` decimals mismatched.
    - No tests toggling `BASE_IS_POD / BASE_IS_FRAX_PAIR` (or equivalent flags), or token order in the LP.
- **Fix pattern:**
    - Be explicit about **units** for every variable (e.g., “`_priceBasePerPTkn18` = base per 1 pTKN (18d)”). Keep a comment trail.
    - Compute LP fair price in a consistent quote:
        1. Get `pTKN/base` and `paired/base`. If `paired` is a wrapper, derive `paired/base` via `cbr = convertToAssets(1e18)` (or pod CBR/unwrap) and apply it **at the right step**.
        2. For V2 50/50 LP: `fair(base/LP) = 2 * sqrt( k * priceToken0(base) * priceToken1(base) ) / LPSupply`, with correct decimals/scalars.
        3. Only then invert if you need `LP/base` vs. `base/LP`—never mix inversion with conversion steps.
    - If `paired == base`, set `paired/base = 1e18`; otherwise, **don’t** assume 1:1.
    - Keep conversion factors under the square root when they affect component **prices**, and outside when they affect **final denomination**; write unit tests to pin this.
    - Add tests for:
        - `BASE_IS_POD` and `BASE_IS_FRAX_PAIR` on/off.
        - Token order flipped in the LP (token0/token1 swap).
        - Random reserve/CBR/decimals fuzz to catch scaling errors.
- **Questions to ask next time:**
    - What are the **exact units** of each intermediate (`pTKN/base`, `paired/base`, `base/LP`)? Where do we invert, and why?
    - Is `pairedLpTKN` ever a **share/wrapper** token? If yes, where is `convertToAssets/CBR/unwrapFee` applied—under the sqrt or after?
    - If I set `paired != base`, do we still compute `paired/base`, or silently assume 1:1?
    - Do decimals for reserves, LP supply, and price scalars align? Any `10**(18/2)` integer division bugs?
    - Do tests cover both podded and Fraxlend-paired cases, and fail if we swap LP token order?
- [ ]  **Exact-out swap + “minOut” flag misused → leftover debt token not forwarded (funds stuck after unwind)**
- **Symptom:** During leverage removal / flash-unwind, the contract does an `exactOut` swap to source the repay amount. If a user-provided `_podSwapAmtOutMin` is set, the code treats it as the **target `exactOut`** (not a lower bound), over-acquires debt tokens. Leftover calculation then only compares the **pre-swap** debt (`_pairedAmtReceived`) vs. `repayAmount`, ignoring the newly swapped debt—so extra debt tokens remain in the contract and aren’t returned to the user.
- **Where to look (Solidity, leverage unwind / flashloan callbacks):**
    - “Remove leverage” paths with steps: withdraw collateral → have `{podToken, debtToken}` → swap `podToken → debtToken` → repay flash → **return leftovers**.
    - Functions like `_swapPodForBorrowToken`, `_acquireBorrowTokenForRepayment`, `_removeLeveragePostCallback` (or similarly named) that:
        - Call `swapV2SingleExactOut` / “exactOutput” style swaps.
        - Take a `_minOut` / slippage param and **repurpose** it as the exact output amount.
    - Leftover accounting blocks that set something like
    `_borrowAmtRemaining = _pairedAmtReceived > _repayAmount ? _pairedAmtReceived - _repayAmount : 0;`
    instead of using **final balance – repay**.
- **Red flags:**
    - Passing `_podSwapAmtOutMin` directly as the `exactOut` argument:
    `swapV2SingleExactOut(pod, debt, podInMax, _podSwapAmtOutMin == 0 ? targetNeeded : _podSwapAmtOutMin, ...)`.
    - Leftover debt computed **from the pre-swap amount** (`_pairedAmtReceived`) instead of checking the **post-swap balance**.
    - No final `IERC20(debt).balanceOf(address(this))` delta check after `repay` to forward residue to the user.
    - Comments/state machine claim “send leftover borrowedTKN to owner,” but no code path actually transfers swapped surplus when `_minOut` is used.
    - Asymmetric handling of leftovers: pod residue returned, debt residue ignored.
- **Fix pattern:**
    - Keep semantics clear:
        - If you want a **slippage floor**, use `exactIn` with `amountOutMin`, or keep `exactOut == targetNeeded` and use an **input cap** (`amountInMax`) for slippage control. Don’t overload a minOut arg as the exactOut target.
    - Compute leftovers from **final balances**:
        - After the swap and **after** `repayAmount` is transferred, set
        `borrowLeft = IERC20(debt).balanceOf(address(this));`
        then `safeTransfer(user, borrowLeft)`.
        - Similarly return remaining pod tokens.
    - In `_swapPodForBorrowToken` return both: consumed pod and *actual* debt acquired; avoid inferring via pre-balance.
    - Emit explicit events for leftover returns; add invariant tests where `_podSwapAmtOutMin > targetNeeded`.
- **Questions to ask next time:**
    - Is `_minOut` ever used as **exactOut**? What happens if it’s **larger** than the actual need?
    - After repay, do we recompute `IERC20(debt).balanceOf(this)` and forward **all** residue?
    - Are there tests where:
        - `pairedAmtReceived < repayAmount` **and** `_podSwapAmtOutMin > targetNeeded`?
        - Sandwich/fill variations cause **over-acquisition**?
    - Does the API clearly separate **slippage control** (minOut / maxIn) from **neededOut** (repay delta)?
    - Are leftovers for **both** tokens (pod & debt) always returned in every branch (success/fallback)?
- [ ]  **Assuming V2 reserves align with caller token order (r0↔t0) → wrong quotes/swaps**
- **Symptom:** Optimal swap / single-sided add-liquidity math uses `(r0, r1)` from `getReserves()` as if `r0` belongs to `_t0` and `r1` to `_t1`. On Uniswap V2–style AMMs, reserves are returned in **pair.token0/token1** order (sorted by address), not in your argument order. Mismap causes bad `_getSwapAmt`, failed adds, slippage reverts, or exploitable mispricing.
- **Where to look (Solidity, V2 routers/oracles/helpers):**
    - Helpers like `_getSwapAmt`, `quote`, “optimal swap-in” functions used for:
        - Single-sided LP add flows (`token → LP` pipelines).
        - Auto-compounders that split one token into two before adding V2 liquidity.
    - Any adapter/wrapper returning `(reserve0,reserve1)` without also returning `token0`/`token1`.
    - Local code that calls:
        - `pair.getReserves()` (directly or via adapter) and then picks reserve by comparing to `_t0/_t1` **without** checking `pair.token0()`.
    - Oracles/fair-LP-price routines that do `k = r0*r1` and mix with prices assuming r0 maps to a specific token.
- **Red flags:**
    - Code like:
        
        ```solidity
        (uint112 r0, uint112 r1) = adapter.getReserves(adapter.getV2Pool(t0, t1));
        uint112 r = (swapT == t0) ? r0 : r1; // assumes r0↔t0
        
        ```
        
    - Adapters that expose `getReserves(pool)` but **not** `token0(pool)` / `token1(pool)`.
    - No call to `pair.token0()` or a local `sortTokens(t0,t1)` before mapping reserves to the caller’s token order.
    - Math uses `_t0/_t1` decimals with `r0/r1` without reconciling address-sorted order.
- **Fix pattern:**
    - Always reconcile reserves to the caller’s token order:
        
        ```solidity
        address pool = adapter.getV2Pool(t0, t1);
        (uint112 r0, uint112 r1) = adapter.getReserves(pool);
        address token0 = IUniswapV2Pair(pool).token0();
        (uint112 rT0, uint112 rT1) = (t0 == token0) ? (r0, r1) : (r1, r0);
        // now rT0 aligns with t0, rT1 aligns with t1
        
        ```
        
    - Or locally `sortTokens(t0,t1)` and swap `(r0,r1)` if `t0 != token0`.
    - Wrap this mapping in a small helper and reuse everywhere you consume reserves.
    - Add unit tests where `_t0 > _t1` by address and ensure outputs match a known-good Uniswap formula/`Router02` quote.
- **Questions to ask next time:**
    - Do we ever assume `r0` corresponds to our `_t0`? Where?
    - Do our adapters expose `token0/token1`, or do we need to call `pair.token0()` directly?
    - Are reserve-based calculations (optimal swap-in, fair LP price, quotes) covered by tests when `_t0` and `_t1` are passed in **reverse address order**?
    - Do we normalize decimals **after** aligning reserves to the intended token order?
    - Are there any places mixing V2 reserves with prices/decimals of mismatched tokens (e.g., using `r0` with `_t1` decimals)?
- [ ]  **“Slippage stacking”: lowering user `amountOutMin` before routing the swap**
- **Symptom:** Router/zapper takes a user-provided `minOut`/`amountOutMin` and then *reduces it again* with an internal/default slippage (e.g., `minOut = minOut * (1 - finalSlipBps)`), passing a looser threshold to the DEX. MEV can sandwich to the new floor; users get less than their stated minimum.
- **Where to look (Solidity, DEX routers/zappers/adapters):**
    - Swap helpers that accept both a user `minOut` and a stored `_defaultSlippage` / pool-specific slippage map:
    `swapV2Single`, `swapV3Single`, `swapV3SingleExactOut`, zap “token→token” paths.
    - Zap/Router modules with per-pool settings like `_slippage[pool]` plus a `_defaultSlippage`.
    - Any UniswapV3 `exactInputSingle` or `exactInput` call where `amountOutMinimum` is derived from user input **and then** multiplied by `(1000 - slip)/1000` (or `1e4 - slipBps`).
    - “Safety” wrappers that recompute `minOut` from a quote **and** still accept a user `minOut`, then pick the smaller (looser) one.
- **Red flags:**
    - Code like:
        
        ```solidity
        uint256 finalSlip = _slippage[pool] > 0 ? _slippage[pool] : _defaultSlippage;
        dex.swapV3Single(_in, _out, fee, amountIn, (amountOutMin * (1000 - finalSlip)) / 1000, to);
        
        ```
        
    - Variables named `finalSlip`, `defaultSlippage`, `tolerance`, `safetyBps` applied **after** receiving `amountOutMin` from the user.
    - Comments like “tighten/loosen user slippage,” or “ensure success by relaxing minOut.”
    - Unit tests that only check “swap doesn’t revert,” not that **received ≥ user minOut**.
- **Fix pattern:**
    - Do **not** relax the user’s `amountOutMin`. If both a user min and an internal slippage are present, enforce the stricter one:
        
        ```solidity
        uint256 internalMin = quoteOut * (1e4 - slipBps) / 1e4;
        uint256 minOutToPass = userMinOut == 0 ? internalMin : Math.max(userMinOut, internalMin);
        
        ```
        
    - Alternatively, support **either** `minOut` **or** `slippageBps`, not both; revert if both are set.
    - Document semantics: “User `minOut` is absolute and passed through untouched.”
    - Add tests where the on-chain min sent to the DEX equals the user-specified min when provided.
- **Questions to ask next time:**
    - Do we ever modify a user-provided `minOut` before calling the DEX?
    - If we compute an internal min from quotes, do we take the **max(userMin, internalMin)** or the min?
    - Are there pool-specific slippage knobs that silently undercut user protection?
    - Do tests assert `received >= userAmountOutMin` for adversarial price movements and sandwich scenarios?
    - For V3 exactInput/exactOutput paths, do we pass `amountOutMinimum`/`amountInMaximum` exactly as the user intended?
- [ ]  **Stale “leftover” math after switching fee regime causes false liquidation reverts**
- **Symptom:** Liquidations compute `_leftoverCollateral` using an *optimistic* (clean) fee, then switch to a *dirty* fee for the actual transfer but keep comparing the **old** leftover against dirty thresholds (e.g., `minCollateralRequiredOnDirtyLiquidation`). This mismatch triggers `revert` even though the true leftover (under dirty fee) is sufficient.
- **Where to look (Solidity, lending/liquidation paths):**
    - Liquidation flows that branch on fee tiers or modes (e.g., *clean* vs *dirty* liquidation, auction vs direct repay): functions named like `liquidate`, `_liquidate…`, `_settleLiquidation`, `_compute…ForLiquidator`.
    - Integrations with external lenders (Fraxlend-style) that expose `cleanLiquidationFee`, `dirtyLiquidationFee`, and `minCollateralRequiredOnDirtyLiquidation`.
    - Code that first computes `_optimisticCollateralForLiquidator` (or similarly named preliminary amounts) and a `_leftoverCollateral = userBal - optimistic`, then later recomputes `_collateralForLiquidator` with a different fee **without** recomputing `_leftoverCollateral`.
- **Red flags:**
    - Variables/branches named `optimistic`, `prelim`, `estimate`, `clean` fee used for an initial leftover calculation, followed by a second computation using `dirty` fee, but guards still check the *first* leftover:
        
        ```solidity
        _leftover = userBal - optimisticForLiq;                // uses clean fee
        _collateralForLiq = exactWithDirtyFee(...);            // final path
        if (_leftover <= 0) { ... }
        else if (_leftover < minRequiredDirty) revert;         // compares stale value
        // true leftover should be: userBal - _collateralForLiq
        
        ```
        
    - Comparisons that mix units or regimes: `_leftover` (clean-based) vs `minCollateralRequiredOnDirtyLiquidation` (dirty-based).
    - Comments like “recompute with dirty fee” followed by **no** update to any dependent variables.
    - Boundary logic using `<=`/`<` around fee thresholds without re-deriving dependent values.
- **Fix pattern:**
    - After determining the **final** liquidation fee regime (dirty vs clean) and computing `_collateralForLiquidator`, **recompute** dependent values used in guards:
        
        ```solidity
        _collateralForLiquidator = _calcWithFinalFee(...);
        int256 trueLeftover = userBal.toInt256() - _collateralForLiquidator.toInt256();
        if (trueLeftover <= 0) { ... }
        else if (trueLeftover < minCollateralRequiredOnDirtyLiquidation.toInt256()) revert BadDirtyLiquidation();
        
        ```
        
    - Alternatively, compute both candidate amounts up-front (clean & dirty), choose one, and only then do *all* comparisons with a single consistent set.
    - Add unit tests at the precipice (e.g., values where clean leftover just above 0 but dirty leftover clears `minRequiredDirty`) to ensure no false reverts.
- **Questions to ask next time:**
    - Do we compare any guard/threshold against a value computed under *different* fee/price assumptions than the one actually used to transfer funds?
    - After we switch from “clean” to “dirty” (or estimate → final), do we recompute *all* derived variables (`_leftover`, needed shares, min-collateral checks)?
    - Are rounding modes (floor/ceil) consistent between preliminary and final computations? Could rounding flip a boundary check?
    - Do tests cover both sides of the boundary (just-barely-pass & just-barely-fail) for each fee regime?
    - If external modules (e.g., Fraxlend) define multiple liquidation fee paths, are our internal thresholds and comparisons aligned to the **chosen** path only?
- [ ]  **Balance-based feeing re-charges leftovers on retries**
- **Symptom:** Protocol/reward fees are computed from the **current token balance** each harvest (e.g., `balanceOf(this) - _protocolFees`) so if a downstream swap/add-liquidity fails and leaves intermediate tokens in the contract, the next run charges a fee again on those leftovers. Over time, user rewards shrink and fee totals exceed the intended percentage of *newly accrued* rewards.
- **Where to look (Solidity, reward pipelines):**
    - Harvest/compound flows that: rewards → swap to intermediate (e.g., `PAIRED_LP_TOKEN`) → split/swap → add LP → stake.
    - Functions named like `_processRewards…`, `_tokenTo…`, `_…ToPodLp`, `_paired…To…`, or fee handlers `_protocolFees`.
    - Code that sets the swap amount via `IERC20(token).balanceOf(address(this))` or `balance - _protocolFees` rather than a **delta since last step**.
    - Branches where the *reward token equals the intermediate token* (e.g., `reward == PAIRED_LP_TOKEN`) and where try/catch on swaps leaves balances untouched on failure.
- **Red flags:**
    - Single, global `_protocolFees` (or similar) used across multiple tokens/sources.
    - Fee base derived from **current balance** instead of fresh inflow:
        
        ```solidity
        uint256 bal = IERC20(paired).balanceOf(address(this));
        uint256 fee = (bal * protocolFeeBps) / 1e4;  // ← charges old leftovers too
        
        ```
        
    - Comments like “reset here to local balances to accommodate any residual leftover” but **fees still computed on total balance**.
    - No `balanceBefore / balanceAfter` snapshot around each receive/swap leg.
    - On swap failure: nothing quarantines/tagging leftovers (no `unprocessed[ token ]`), yet the next cycle fees them again.
- **Fix pattern:**
    - **Charge on deltas only.** Snapshot `before = balanceOf(token)` prior to the action that *creates* fee-able amounts; compute `newInflow = balanceOf(token) - before` and apply fee to **newInflow**.
    - Maintain **per-token staging**:
        - Track `unprocessed[token]` and only fee `newInflow`, then attempt conversion using `unprocessed[token]`.
        - If a downstream step fails, **do not add fee again**; leave `unprocessed[token]` untouched for the next attempt.
    - Alternatively, **fee after success**: apply the protocol fee *only once the conversion to the target LP/tokens succeeds*; on failure, skip feeing that batch.
    - Unit tests:
        1. Simulate swap failure leaving intermediates; next harvest should not increase total fees for the same leftovers.
        2. Case where `rewardToken == intermediate` to ensure no double-fee.
- **Questions to ask next time:**
    - Is the fee base “total balance” or “newly received since last accounting”?
    - What happens to leftovers if an intermediate swap/add-liquidity fails—are they quarantined and *excluded* from future fee calculations?
    - Are fees tracked **per token** and per batch, or is there a single accumulator that can cause cross-batch double charging?
    - Do we ever have `rewardToken == intermediateToken` paths, and are those specially handled?
- [ ]  **Utilization-delta–gated interest accrual lets new deposits capture pre-accrued yield**
- **Symptom:** A depositor mints shares at an **old exchange rate** (no interest just applied), then a subsequent action finally accrues interest globally, boosting the vault’s share price so the new depositor earns yield for time they weren’t staked. MEV bots can sandwich: *deposit → force accrue → withdraw*.
- **Where to look (Solidity lending / wrappers around Fraxlend-style pairs):**
    - Functions named `addInterest`, `previewAddInterest`, `_addInterest`, `convertToShares/convertToAssets`, `deposit/withdraw` call sites.
    - Guards like `minURChangeForExternalAddInterest` (utilization change threshold) and code that **skips accrual** unless `Δutilization > threshold`.
    - Places computing utilization with **stale snapshots**: `totalBorrow.amount`, `totalAsset.totalAmount(...)`, `currentRateInfo` read from storage **without** first applying pending interest.
    - Wrapper vaults that “broadcast accrue” to children (e.g., `_updateInterestAndMdInAllVaults`) but call `addInterest(false)` and rely on rate-change rather than time/preview math.
    - Share pricing variables (`_cbr`, “exchange rate”, “share value”) used by `convertToShares` **before** forcing accrual.
- **Red flags:**
    - `addInterest` path like:
        
        ```solidity
        uint256 prevUR = _prevUtilizationRate;
        uint256 newUR = (UTIL_PREC * totalBorrow.amount) / totalAsset.totalAmount(externalAssetVault);
        if (prevUR != 0 && abs(newUR - prevUR) < prevUR * minURChange / UTIL_PREC) {
            // skip accrual
        }
        
        ```
        
        …where `totalBorrow`/`totalAsset` were **last updated at the prior accrual**, so `newUR ≈ prevUR` and accrual keeps being skipped.
        
    - `deposit()` → calls an “update all vaults” loop → **still no accrual** → immediately mints shares with old rate.
    - No use of `previewAddInterest`/on-the-fly interest math before mint/burn.
    - Accrual decisions based on **rate change** instead of **time elapsed** or **previewed interest**.
- **Fix pattern:**
    - **Force interest application before any share-changing operation** (mint/burn/borrow/repay): call a `preview…` or internal `_calculateInterest` and **apply** it (or incorporate the previewed result directly into conversions).
    - If keeping a skip gate, compute the gate with **fresh, virtualized totals** (include pending interest first), or **gate on time** (e.g., accrue if `block.timestamp > lastAccrual`), not on `ΔUR`.
    - Prefer a **global index model** (cumulative interest factor) that `convertToShares/Assets` uses immediately, so accrual timing can’t be gamed.
    - Add tests: (1) long idle period → deposit → trigger accrue → ensure depositor doesn’t profit from pre-deposit interval; (2) MEV sandwich attempt yields ~0 extra.
- **Questions to ask next time:**
    - Does `deposit/withdraw/borrow/repay` **apply pending interest** (or use a previewed exchange rate) *before* converting assets↔shares?
    - Is accrual skipped by a **min utilization delta**? If so, are the inputs **fresh** or are they the exact snapshots from the last accrual?
    - Can a user deposit, then in the **same block or next tx** cause accrue and withdraw a risk-free gain?
    - Why not use `previewAddInterest/_calculateInterest` outputs for conversion paths? If not, what prevents stale rates?
    - Are wrapper vaults that “update all children” relying on `addInterest(false)` with gates, instead of a deterministic accrue/preview?
- [ ]  **Accepting yield-bearing stake tokens without a reward path traps emissions**
- **Symptom:** A wrapper/voting/gauge contract lets users stake *receipt* tokens (e.g., staking LP `spTKN`) that themselves accrue external rewards. Those reward tokens are paid to the wrapper’s address during stake/unstake/hold—but the wrapper has no `claim/harvest/sweep` logic, so emissions accumulate and become permanently stuck.
- **Where to look (governance wrappers, gauges, voting pools):**
    - Staking entrypoints: `stake`, `unstake`, `deposit`, `withdraw`, `vote`, `lock`, `delegate` that transfer user tokens **to the wrapper**.
    - Token allowlists/adapters: modules like `IStakingConversionFactor`, “conversion factor” registries, or plugin systems that permit multiple token types (`pTKN`, `spTKN`, LP/shares).
    - Reward mechanics on the staked token: does the token (or the staking pool behind it) distribute rewards to `tokenHolder` or require an explicit `getReward/claim`? Look for calls to external staking pools that mint/send reward tokens.
    - Absence of handler methods on the wrapper: no `claimRewards`, no reward forwarding, no `sweep(address token)`, no accounting to redistribute rewards to stakers.
- **Red flags:**
    - Whitelisting of yield-bearing receipt tokens (LP staking shares, vault shares, `spTKN`) as stakeable **without** any reward-claim or distribution code in the wrapper.
    - Comments/params like `ConversionFactorSPTKN`, `ConversionFactorPTKN`, “paired staking token”, but the staking wrapper only tracks balances/votes and never touches reward tokens.
    - Transfers like `IERC20(spTKN).transferFrom(user, address(this), amount)` with no subsequent `claim`, `onRewardPaid`, or `sweep`.
    - Reward tokens visible via `IERC20(reward).balanceOf(wrapper) > 0` in tests or after staking cycles, but no path to move them.
- **Fix pattern:**
    - **Prevent** staking of externally-rewarding receipt tokens unless the wrapper implements reward harvesting and pro-rata distribution (or immediate forwarding to the depositor).
    - If supporting them, add a **harvest module** that:
        - Calls upstream `claim/getReward`, or passively sweeps tokens sent to `address(this)`.
        - Tracks reward indices (cumulative per-share) and lets stakers `claim()` proportionally.
        - Provides `sweep(address token, address to)` (guarded) for accidental tokens, with clear policy.
    - Alternatively, require staking the **underlying** (non-rewarding) token and keep reward flows outside the voting wrapper.
    - Add tests: stake `spTKN`, simulate reward accrual, assert wrapper can harvest and distribute; assert zero stranded balances after `unstake`.
- **Questions to ask next time:**
    - Can any **yield-bearing** tokens be staked here (LP shares, staking receipts, vault shares)? If yes, **where do their rewards go**?
    - Does the wrapper ever call `claim/getReward` or maintain a reward index for redistribution?
    - What happens if reward tokens are **sent directly** to the wrapper during transfers/holds—can users withdraw or claim them?
    - Is there an **admin/emergency sweep** and a documented policy to avoid silent value accumulation?
- [ ]  **Charging fees on “intended” amounts instead of actual fills overcharges users**
- **Symptom:** A protocol deducts an open/deposit/exit fee using the *requested* or *desired* input (e.g., `pairedLpDesired`, `_amountIn`) before routing swaps/add-liquidity. Execution leaves leftovers (due to slippage, ratios, or min-out), but the fee isn’t recomputed on the **actual used** amount, so users pay fees on tokens that weren’t utilized.
- **Where to look (zaps/leverage/routers):**
    - Leverage/Zap callbacks: functions like `_addLeveragePostCallback`, `_zap`, `_lpAndStakeInPod`, `_tokenToPodLp`, `_addLiquidity*`.
    - Any path that:
        1. Takes a user “desired” amount,
        2. Applies a fee (`openFeePerc`, `protocolFee`, etc.),
        3. Performs swaps / add-liquidity,
        4. Returns **leftover** tokens (refunds), but **doesn’t** adjust the fee.
    - Exact-out swaps or ratio-constrained LP mints where leftovers are common. Look for return values such as `(_used, _pairedLpUsed, _leftover)` that are ignored by fee accounting.
- **Red flags:**
    - Fee computed as `fee = desired * feePerc / 1000` **before** calling the execution function that returns used/leftover.
    - Variables named `desired`, `target`, `maxIn`, `minOut` used for fee math instead of `used`, `spent`, or fill amounts.
    - Post-exec leftovers (`_pairedLeftover`, refunds) present, but fee not re-evaluated.
    - Comments like “reduce amount by fee, then LP” with no reconciliation after the LP call.
- **Fix pattern:**
    - Compute fee on the **actual consumed** amount:
        1. Tentatively cap input (optionally), execute swaps/LP to get `(used, leftover)`.
        2. `fee = used * feePerc / DENOM`.
        3. Transfer fee; refund leftovers to the user.
    - If fee must be taken upfront (e.g., transfer ordering constraints), take a **max-fee hold**, then reconcile after execution and refund excess.
    - Unit tests: craft cases with intentional leftovers (mismatched pTKN/paired ratios, high slippage) and assert `chargedFee == used * feePerc`.
- **Questions to ask next time:**
    - Is the fee based on **what the user intended** to spend or what the contract **actually spent**?
    - Do we get `used/leftover` values from downstream calls (swaps/LP adds), and are fees reconciled to `used`?
    - In failure/partial-fill paths, are fees reversed or re-computed?
    - Are there code paths with exact-output swaps or ratio-constrained LP where leftovers are likely every time? How is fee handled there?
- [ ]  **Missing reward settlement before share changes lets newcomers siphon accrued rewards**
- **Symptom:** A vault/auto-compounder updates user shares (deposit/mint/redeem/burn) **before** pulling & distributing accrued rewards from an external rewarder (gauge/staking pool). A fresh depositor can trigger distribution, then immediately withdraw to capture rewards earned by prior stakers. (E.g., rewards sit in `TokenRewards` until a transfer triggers `setShares`, but `_processRewardsToPodLp` doesn’t first `distribute()`/harvest, so the joiner benefits.)
- **Where to look (vaults, wrappers, auto-compounders):**
    - Deposit/mint and withdraw/redeem functions in:
        - Auto-compounders wrapping LP or staking tokens (`_processRewards*`, `_harvest`, `_compound`).
        - Gauge/rewarder wrappers and voting-escrow staking proxies.
        - ERC-4626-like vaults that rely on external rewarders (MasterChef, Gauge, `TokenRewards`, `Rewarder`, `getReward`, `distribute`).
    - Any path that:
        1. Computes shares from `totalAssets/totalSupply`,
        2. Transfers/stakes base tokens (which may implicitly trigger reward distribution),
        3. **Only later** claims/harvests rewards (or never harvests if “no balances”).
- **Red flags:**
    - No explicit `harvest()/getReward()/distribute()` **before** changing `totalSupply` or user shares.
    - Reward distribution happens implicitly on token transfer hooks (e.g., `setShares` in a reward token) and deposits are executed **prior** to a harvest.
    - Multi-reward tokens loop calls `_tokenToPodLp()` only when `_bal > 0`; if all reward balances are zero locally, distribution is skipped even though the external rewarder holds pending rewards.
    - Comments like “harvest happens later in compound” with deposits allowed now.
    - Tests don’t simulate: deposit → (pending rewards exist externally) → immediate redeem; or don’t check a newcomer’s ability to leave with extra yield.
- **Fix pattern:**
    - **Settle first, then mutate shares.** On every deposit/mint and redeem/burn path:
        1. Pull/distribute all pending rewards from the external rewarder(s).
        2. Convert/compound them (or at least account them) into `totalAssets`.
        3. Only then compute/mint/burn shares.
    - If gas-sensitive, maintain an **`accRewardPerShare`**style index and update it on each action; newcomers’ shares start after the index bump so they don’t get prior rewards.
    - Guard zero-balance short-circuits: if local reward balances are zero, still **force-distribute** from the rewarder if it tracks pending rewards.
    - Symmetry: apply the same settlement on **redeem/withdraw** to avoid quitters escaping with unaccounted rewards.
- **Questions to ask next time:**
    - Before changing `totalSupply` or user shares, do we **harvest/distribute** all pending rewards from external rewarders?
    - Can a deposit itself trigger distribution (via transfer hooks), letting the depositor claim **pre-existing** rewards?
    - What happens if local reward balances are zero but the rewarder has **claimable** rewards for this wrapper?
    - Are multi-reward tokens all harvested atomically before share math?
- [ ]  **Round-down shares math underquotes exact redemptions → repay shortfall/DoS**
- **Symptom:** When converting a target **asset** amount into **shares** to redeem (e.g., to repay a flash loan or debt exactly), code uses a **round-down** path (`convertToShares` / `toShares(..., roundUp=false)`), then later redeems with another **round-down** path (`redeem` → `toAmount(..., roundUp=false)`). Double truncation yields **fewer assets than needed**, causing “insufficient repay” and revert in leverage unwind / flash-repay flows.
- **Where to look (lending wrappers, leverage managers, ERC-4626/Fraxlend integrations):**
    - Functions that must obtain **at least X assets** from a share redemption:
        - Leverage removal helpers like `_acquireBorrowTokenForRepayment`, `_swapPodForBorrowToken`, or “self-lending” branches that go `assetNeeded → convertToShares → redeem`.
        - ERC-4626 / Fraxlend bridges: `convertToShares/previewWithdraw/redeem`, `toShares` / `toAmount`.
        - “Exact repay” or “exact flash-repay” code paths that compute a share amount for a precise asset liability.
    - Libraries/utilities performing conversions:
        - `convertToShares(_assets)`, `toAssetShares(..., /*roundUp?*/ false, /*preview?*/ true)`.
        - Subsequent `redeem(shares, ...)` that defaults to **roundDown**.
- **Red flags:**
    - Using `convertToShares(targetAssets)` **without** a `roundUp` option or cushion, immediately followed by `redeem(shares)` to meet a **hard** `repayAmount`.
    - Comments like “get enough shares to repay exactly X” while calling round-down conversions twice.
    - No post-redeem balance check/top-up before `transfer(repayAmount)`.
    - Tests only cover “happy path” balances, not boundary cases where `assets % pricePerShare != 0`.
- **Fix pattern:**
    - **Round up when targeting assets.** Prefer `previewWithdraw(targetAssets)` (which implies rounding up shares) or `toShares(..., roundUp=true)`, or add a minimal **+1 share** / **dust cushion**.
    - After redeem, **verify** `assetBalance >= repayAmount`; if not, perform a small top-up swap.
    - Avoid double truncation: if `convertToShares` rounds down, ensure `redeem` uses `roundUp=true` (if API supports), or compensate with a cushion.
    - Add invariants/tests: for random targets, redemption must always yield `>= targetAssets`.
- **Questions to ask next time:**
    - When converting **assets → shares** to meet an **exact** liability, do we **round up** or add a cushion?
    - Are we applying **two** round-downs (assets→shares, then shares→assets) in sequence?
    - Do we recheck balance after redeem **before** attempting the repay transfer?
    - Do preview methods (`previewWithdraw`, `previewRedeem`) align with runtime rounding and fee behavior?
    - What happens at extreme ratios/low liquidity where truncation is most visible?
- [ ]  **Oracle-based minOut lets attackers revert compounding → strand rewards, then siphon them on next deposit**
- **Symptom:** Compounding routine swaps rewards → pod token with a **fixed, oracle-derived minOut (e.g., 5% slippage)**. An attacker momentarily moves price beyond that bound so the swap reverts; rewards remain idle in the compounder. The attacker then **deposits**, waits for a later successful compound, and withdraws a cut of **old** rewards they didn’t earn.
- **Where to look (auto-compounders / reward roll-ups):**
    - Reward compounding paths that do: `reward → pairedLP → pod (pTKN) → addLiquidity → stake`, e.g. functions like `_processRewardsToPodLp`, `_pairedLpTokenToPodLp`, `_tokenToPodLp`.
    - Any **minOut calculated from an oracle price** (e.g., `podOracle.getPodPerBasePrice()` with a hardcoded “5%” discount) passed to a swap adapter (Uniswap V2/V3).
    - TokenRewards / distributor flows that **don’t distribute before** compounding (so old rewards can sit in the compounder).
    - Flags like **`LEAVE_AS_PAIRED_LP_TOKEN`** or code paths that leave reward tokens in a non-reward form when a swap fails.
    - Try/catch branches around swaps that **swallow the revert** and simply keep balances in the compounder.
- **Red flags:**
    - Hardcoded slippage (e.g., 5%) applied to an **oracle** quote, not to an on-chain TWAP or pool spot with buffer.
    - No fallback: if swap fails, rewards **stay in contract**; no partial conversion, no reattempt logic, no distribution checkpointing.
    - Compounding only triggers on **user deposit/redeem** (not periodic), letting entrants time deposits right after a forced-revert.
    - Lack of **epoch accounting** (no snapshot) so new shares participate in rewards accrued **before** they arrived.
    - `LEAVE_AS_PAIRED_LP_TOKEN=true` while that token **isn’t** in the rewards whitelist, so stranded value persists.
- **Fix pattern:**
    - **Distribute first, then compound.** Settle TokenRewards to the compounder *before* any swaps; snapshot a pre-compound “reward index” so only existing shares earn that batch.
    - Make slippage **adaptive** and TWAP-aware (e.g., dynamic bands from pool TWAP/volatility) and/or allow **keeper-supplied minOut** with caps.
    - On swap failure: write a **“pendingRewards” bucket** and **do not** let new deposits share past buckets; require an explicit keeper `compound()` to roll them forward.
    - Support **partial compounding**: if pTKN leg fails, still convert/distribute the portion that can succeed; otherwise revert the whole compound **without** changing share distribution.
    - Introduce **cooldown/epoch gating**: deposits after a failed compound start earning from the **next** epoch only.
- **Questions to ask next time:**
    - Is minOut based on **oracle** or **on-chain TWAP**? Can an attacker push spot beyond the band to force a revert?
    - What happens to rewards when the swap reverts—are they **distributed** or **left idle** in the compounder?
    - Do new deposits get exposure to **previously accrued** rewards (no snapshot/epoch)?
    - Is there a **keeper-only** compound entrypoint vs. auto-compound on user flows?
    - Does the system handle **LEAVE_AS_PAIRED_LP_TOKEN** coherently (whitelisted as reward, or converted later)?
- [ ]  **Threshold-triggered fee processing can be gamed via direct token top-ups**
- **Symptom:** Reward fee conversions only run once a **min-fee threshold** is reached (e.g., `_bal >= min`). An attacker first **stakes** when fees are *below* the threshold (so no rewards are processed), then **pushes the fee token directly** to the pod to cross the threshold, and finally **unstakes** in the same tx to capture rewards they didn’t help generate (can be amplified with a flash loan).
- **Where to look (index/pool fee → rewards pipelines):**
    - Fee processing entrypoints like `_processPreSwapFeesAndSwap`, `swapFeesToRewards`, `distributeFees`, or similar in the **Pod/Index** contract.
    - Places computing fee balance with `IERC20(feeToken).balanceOf(address(this))` (or equivalent) and comparing to a **minimum** before swapping/distributing.
    - Staking flows (`stake/unstake`, `setShares`) that **trigger fee processing** inline without first distributing previously accrued rewards or snapshotting.
    - Any path where **arbitrary external transfers** to the pod (e.g., `transfer(pod, x)`) can **raise the fee balance** seen by the threshold check.
- **Red flags:**
    - Fee accounting driven by **raw token balances** instead of **internal tracked deltas** (e.g., `pendingFees += feesGenerated;`).
    - No **mandatory distribute/snapshot** before share changes (stake/unstake/deposit/redeem).
    - Threshold checks that run during **user actions** rather than on **keeper/cron-only** functions.
    - Lack of **flash-resistance** (no cool-down, no min stake duration, no block delay on new shares).
    - Comments or config like `minFeeToProcess`, `LEAVE_AS_PAIRED_LP_TOKEN`, or “process fees on interaction” combined with no guard against **donations**.
- **Fix pattern:**
    - **Track fees by internal accounting**, not `balanceOf`. Only protocol actions mutate `pendingFees`; ignore unsolicited transfers (or treat them as donations to a separate bucket).
    - **Distribute (or snapshot an index) before any share changes.** On stake/unstake, first call `distribute()` (or mint/burn against a global reward index) so newcomers can’t claim past accruals.
    - Move fee processing to a **keeper-only `compound()`**, or gate user-triggered processing with **epoch boundaries** so rewards belong to the prior epoch’s stakers.
    - Add **cooldown / min-stake-seconds** or block delay to prevent same-tx stake→trigger→unstake capture.
    - If keeping a threshold, compute against **tracked pendingFees** deltas; do **not** rely on raw token balances.
- **Questions to ask next time:**
    - “Can fee processing be triggered by **raw token balances** that include **unsolicited transfers**?”
    - “Do we **distribute/snapshot rewards before** accepting new stake or processing unstakes?”
    - “Is fee conversion gated to **keeper/epoch** or can any user trigger it inline with share changes?”
    - “Do we have a **cooldown** so stake→trigger→unstake in one tx can’t capture prior rewards?”
    - “If there’s a **min threshold**, what prevents an attacker from topping it off right after staking?”
- [ ]  **Transfer-tax applied during burns causes recursive deflation and bad totalSupply**
- **Symptom:** With `hasTransferTax=true`, a portion of each transfer is **burned** via `_burn(...)`, but `_burn` (or a burn implemented as `transfer(address(0), x)`) routes through the **taxed transfer path** again. The burn amount itself gets taxed/burned **recursively**, making `totalSupply` drift **lower than intended** and overcharging user fees. Downstream math (e.g., mint-on-bond proportional to `totalSupply`) becomes incorrect.
- **Where to look (ERC20 + fee/tax + index/pod tokens):**
    - Central transfer hook like `_update/_transfer` in the token (e.g., **DecentralizedIndex**): check if it unconditionally applies **transfer tax** for *any* `from/to`, including `address(0)`.
    - Burn paths: `_burn`, `burn`, `burnFrom`, or fee handlers calling `_burn(...)` (e.g., `_processBurnFee`), and any “burn-by-transfer” patterns (`_transfer(from, address(0), ...)`).
    - Conditionals around `hasTransferTax`, `feeOnTransfer`, and fee splits (protocol fee vs burn) to see if **internal fee leg** triggers the same taxed path.
- **Red flags:**
    - `_burn` internally calls `_update/_transfer` which re-applies tax logic.
    - Fee code that does `transfer(address(this))` (protocol fee) **then** `_burn(...)` for the burn leg without **tax exclusion**.
    - No exclusions like `if (from==address(0) || to==address(0)) skip tax`.
    - Comments like “burn is just a transfer to zero” but the tax code doesn’t exclude the zero/burn address.
    - `totalSupply` used in pricing/minting (bonding) while the burn mechanism can **double-count** supply reduction.
- **Fix pattern:**
    - **Never tax burns/mints.** Add guards in the taxed transfer hook: `if (from==address(0) || to==address(0)) { /* mint/burn */ return raw; }`.
    - If fee logic calls `_burn`, ensure `_burn` **does not** re-enter the taxed transfer path (use a **separate internal burn** that adjusts balances and `totalSupply` without calling the taxed hook).
    - If burns are implemented via `transfer(address(0))`, **exclude the burn address** from tax (`isFeeExempt[address(0)]=true`) or branch out before the tax calculation.
    - Add an **inFeeProcessing** boolean (reentrancy-style guard) to skip fee/tax calculation during internal fee legs.
    - Unit-test invariants: (1) single user transfer with burn fee reduces `totalSupply` by **exactly** the configured burn portion once; (2) fee handler that burns X reduces supply by X—not more.
- **Questions to ask next time:**
    - “Does `_burn` (or burn-by-transfer) go through the same taxed `_transfer/_update` path?”
    - “Are `address(0)` transfers and mints/burns **excluded** from tax logic?”
    - “Can fee sub-transfers (protocol-fee, burn-leg) recursively trigger the tax hook?”
    - “Which math uses `totalSupply` (bonding/mint shares)? What breaks if it’s deflated more than intended?”
- [ ]  **Hardcoded external addresses brick multi-chain deployments**
- **Symptom:** Contracts that integrate Uniswap/routers/oracles work on one network but revert or misroute on others (e.g., `NonfungiblePositionManager` calls fail, pools not found, approvals to a non-existent address). Deploy succeeds but features like “lock V3 NFT”, “mint/collect”, “zap” silently fail on some chains.
- **Where to look (EVM DEX/interop code):**
    - Constructors and `immutable` fields for third-party endpoints: `INonfungiblePositionManager`, Uniswap V2/V3 `Router`, `Factory`, `Quoter`, `WETH`, Chainlink feeds, Fraxlend pairs, Curve registries.
    - “Addresses” or “Constants” files (e.g., `Addresses.sol`, `Config.sol`) with literals like `0xC364...` (V3 NFPM), `0xE592...` (V3 Router), `0xC02a...` (WETH).
    - Any helper/locker/zapper modules (e.g., `V3Locker`, `Zapper`, `DexAdapter`) that set protocol addresses in the constructor.
    - Proxy initializers that pin addresses during `initialize()`, and libraries that assume a single chain (no `chainid()` branching).
- **Red flags:**
    - `constant/immutable` third-party addresses hardcoded (e.g., `V3_POS_MGR = INonfungiblePositionManager(0xC364...)`).
    - Comments like “mainnet address” with no per-chain override.
    - No owner/governance setter or registry hook to update endpoints.
    - Tests run only on a single chain config; CI lacks chain matrix (Mainnet/Arbitrum/Base/etc.).
    - No on-chain sanity checks (code size/codehash/interface support) when setting external endpoints.
- **Fix pattern:**
    - **Inject, don’t hardcode.** Pass third-party addresses via constructor/initializer and store them; avoid `constant`.
    - Add a **governance-gated setter** (`onlyOwner`/timelock) to update endpoints; emit events (`AddressUpdated(name, old, next)`).
    - Validate the new address: `require(addr.code.length>0)`, interface checks (e.g., `supportsInterface`/`try positions(...)`), optional codehash allowlist.
    - Optionally use a **chain registry** pattern: `AddressProvider` mapping `bytes32 key -> address`, updatable by governance.
    - Guard critical ops with explicit checks: `require(factory.getPool(token0, token1, fee) != address(0), "POOL_NOT_DEPLOYED")`.
    - Build tests across networks (forks) and add a deploy-time checklist asserting configured addresses per `block.chainid`.
- **Questions to ask next time:**
    - “Which chains are we targeting, and what are the **per-chain addresses** for NFPM/routers/oracles/WETH?”
    - “Can these endpoints change (upgrades, v3.1, custom deployments)? Who can **rotate** them safely?”
    - “Do we verify that the configured address actually implements the expected interface on this chain?”
    - “If the endpoint is wrong, do we **fail fast** (clear revert) instead of silently sending funds/approvals to a black hole?”
    - “Are our deployments/tests parameterized by `chainid()` and do we run a **multi-chain test matrix**?”
- [ ]  **Uninitialized “sender” field causes `transferFrom(address(0), ...)` reverts**
- **Symptom:** Flows that allow “user provides up to X tokens” (e.g., `_userProvidedDebtAmtMax`) revert during repayment because the code pulls from `_props.sender` which was never set, so `transferFrom` is attempted from `address(0)`.
- **Where to look (Leverage/flash-callback paths):**
    - Structs like `LeverageFlashProps` / `PositionProps` passed through `flashLoan` → callback (e.g., `_removeLeveragePostCallback` / `_acquireBorrowTokenForRepayment`).
    - Places that build/encode `_props` vs. places that **consume** `_props.sender`; compare fields that are set (`owner`) vs. assumed (`sender`).
    - Any logic gating “user-provided top-up” for repays or fees: `_userProvidedDebtAmtMax`, `_userProvided...` params, and the `transferFrom(_props.sender, ...)` lines.
    - Initializers/constructors that assign `owner` but never `sender`, and helpers that use `_msgSender()` but don’t write it into the struct.
- **Red flags:**
    - `_props.sender` read but never written (no assignment from `msg.sender` or function argument).
    - Struct populated in one function, consumed in another after being ABI-encoded/decoded, with missing fields.
    - `transferFrom` using a struct field instead of a directly passed `from` param, and no `require(_props.sender != address(0))`.
    - Tests only covering the swap path to source tokens, not the **user-supplied top-up** path.
- **Fix pattern:**
    - **Plumb the caller identity.** On entry (e.g., `removeLeverage`), set `_props.sender = msg.sender;` before encoding, or pass `from` explicitly to the callee that does `transferFrom`.
    - Add guards: `require(_props.sender != address(0), "SENDER_0");` and `require(_userProvidedDebtAmtMax == 0 || IERC20(token).allowance(_props.sender, address(this)) >= _userProvidedDebtAmtMax, "NO_ALLOW");`.
    - Prefer using `_props.owner` consistently if “owner pays” is the intended model, or **explicitly choose** between `owner` vs `msg.sender` in the API and document it.
    - Unit test matrix:
        - With `_userProvidedDebtAmtMax > 0`: allowance present → succeeds; no allowance → reverts with clear error.
        - With `_userProvidedDebtAmtMax = 0`: path skips `transferFrom` entirely.
        - Round-trip encode/decode of `_props` preserves `sender`.
- **Questions to ask next time:**
    - “Who is the **payer** for top-ups—`owner`, `msg.sender`, or a designated `payer`? Is that identity consistently propagated through callbacks?”
    - “Do we ever call this via a relayer or contract wallet? If so, does `sender` differ from `owner` and is that supported?”
    - “Do we have a revert reason if `sender == address(0)` or allowance is missing?”
- [ ]  **Add-liquidity assumes no transfer-tax; self-lending breaks when PAIRED_LP_TOKEN is fee-on-transfer**
- **Symptom:** `addLiquidityV2` reverts for self-lending pods when the paired token (e.g., podded fTKN) charges a transfer tax. Tokens sent into the pod arrive short, but the code still tries to add liquidity with the **pre-tax** amounts.
- **Where to look (Pods/LP add paths):**
    - `IndexUtils.addLPAndStake` → `DecentralizedIndex.addLiquidityV2` → `DEX_HANDLER.addLiquidity` (Uniswap V2 adapter).
    - Config where `PAIRED_LP_TOKEN` is itself a pod/fToken and `hasTransferTax == true`.
    - Any place that:
        - Transfers `_pairedLPTokens`/`_pTKNLPTokens` into the pod, then **passes the same numbers** to the router as desired/min amounts.
        - Does **not** compute `balanceDelta` after transfer before calling add-liquidity.
    - Router/adapter code: UniswapV2 add-liquidity paths typically **don’t** have “supportingFeeOnTransferTokens” variants—look for assumptions that exact desired amounts will be available.
- **Red flags:**
    - Using the “intended” amounts for both `amountDesired` and `amountMin` without recomputing from actual balances received by the pod.
    - No handling for deflationary/FOT tokens (no balance-before/after pattern).
    - Self-lending pods where `PAIRED_LP_TOKEN` is a pod with transfer tax enabled.
    - Reverts only on chains/pods where transfer-tax is toggled on.
- **Fix pattern:**
    - **Balance-delta accounting:** After `safeTransferFrom`, compute the actual amounts received (`received = balAfter - balBefore`) for both legs and use those in add-liquidity. Derive `amountMin` from `received` and slippage.
    - **Pair-direct mint flow (Uniswap V2):** Skip router’s pull; send tokens directly to the pair (`transfer`), then call `pair.mint(recipient)`. This uses actual received amounts and naturally supports FOT tokens. Apply a post-mint LP min check.
    - **Disable or gate:** If supporting FOT is out of scope, reject self-lending configs where `PAIRED_LP_TOKEN` is FOT (`hasTransferTax==true`), with explicit guardrails and tests.
    - **Tests:** Add unit tests with a mock fee-on-transfer token as `PAIRED_LP_TOKEN` to prove success on taxed adds and to cover both legs taxed/untaxed.
- **Questions to ask next time:**
    - “Can `PAIRED_LP_TOKEN` ever be deflationary or fee-on-transfer in any supported configuration?”
    - “Do we recompute from **actual** balances received before calling add-liquidity, or do we trust requested amounts?”
    - “Does our DEX adapter support FOT on swaps **and** liquidity adds? If not, should we mint directly via the pair?”
    - “What’s our policy if only one leg is taxed? Are min-amounts derived from ‘received’ or from user inputs?”
- [ ]  **Config toggles that mint “free APY”: front-running `setYieldConvEnabled` jumps share price**
- **Symptom:** Turning on compounding (`yieldConvEnabled=true`) triggers a one-shot harvest in `_processRewardsToPodLp` that **bumps `_totalAssets` sharply**. Attackers who **deposit just before** the toggle (while compounding is off and `_totalAssets` is stale) mint shares too cheaply, then exit after the owner’s tx executes, pocketing the windfall.
- **Where to look (Vaults/Auto-compounding paths):**
    - `AutoCompoundingPodLp.setYieldConvEnabled` and any owner/governance flag that gates harvesting/compounding.
    - `_processRewardsToPodLp`: early return when disabled, and `_totalAssets += _lpAmtOut` when enabled.
    - ERC-4626 flows: `deposit/mint/previewDeposit/previewMint` and their use of `_totalAssets` **before** any harvest.
    - Reward collection sources (e.g., `TokenRewards.claimReward`) that can leave undistributed rewards pending while deposits still mint shares.
- **Red flags:**
    - Any **boolean gate** (e.g., `yieldConvEnabled`) that prevents accrual/harvest during deposits, then later **applies all accrued yield at once**.
    - Deposits that **don’t call a harvest/accrual** step (or don’t compute with a post-harvest NAV) before computing shares.
    - Owner setters are callable **without pausing deposits**, timelock, or commit-reveal.
    - Comments like “enable later” / “harvest on toggle” with no front-run protection.
- **Fix pattern:**
    - **Harvest-before-mint:** On every `deposit/mint`, if pending rewards > 0 **or** the gate was recently flipped, run `_processRewardsToPodLp` (or an inexpensive accrual update) first so share price reflects current NAV.
    - **Atomic enable + harvest:** In `setYieldConvEnabled(true)`, **perform an immediate harvest** inside the same call; **pause deposits** during the tx (or require the contract to be paused before toggling and unpaused after).
    - **Timelock/commit-reveal:** Make toggles two-step with a delay; watchers can’t sandwich cheap deposits in the same block/mempool window.
    - **Locked-profit / smoothing:** Adopt a Yearn-style locked-profit mechanism so sudden gains vest into share price over time, reducing toggle MEV.
    - **Min-share guards:** Require `minSharesOut` on deposit and compute against **post-harvest** preview to block unfavorable pre-harvest pricing.
    - **Tests:** Fuzz scenarios where rewards accumulate while disabled; simulate mempool ordering: attacker `deposit` → owner `setYieldConvEnabled(true)` (harvest) → attacker `redeem`. Assert zero free gain.
- **Questions to ask next time:**
    - “Can any governance switch cause `_totalAssets` or share price to jump? If yes, what prevents **pre-toggle deposits** minting underpriced shares?”
    - “Do `deposit/mint` paths **always** account for pending rewards (harvest/accrue) before computing shares?”
    - “Is there a pause/guardrail around toggles or a **mandatory harvest** within the toggle?”
    - “Do previews (`previewDeposit/previewMint`) reflect the **same state** that the tx will use (post-harvest)?”
- [ ]  **Wrong utilization denominator lets lenders spam `addInterest` to hike borrower APR**
- **Symptom:** Utilization is computed as `borrowed / unborrowed` (cash-only) instead of `borrowed / (borrowed + unborrowed)`. This inflates observed utilization deltas and **forces `_addInterest` to run even when nothing changed**, letting anyone repeatedly update rates and overcharge borrowers.
- **Where to look (Fraxlend-style pairs / vault integrations):**
    - `FraxlendPairCore.addInterest` and any wrappers that gate accrual via utilization deltas.
    - Helpers like `_totalAssetAvailable(...)` and **how they’re fed into** `_prevUtilizationRate` / `_newUtilizationRate`.
    - Calls that do:
    `totalAssetsAvailable = totalAsset.totalAmount(externalAssetVault) - totalBorrow.amount;`
    then `util = totalBorrow.amount / totalAssetsAvailable`.
    - Threshold logic that compares `abs(newUR - prevUR)` to a fraction of `prevUR` to decide skipping vs. accruing.
- **Red flags:**
    - Utilization formula that **excludes borrowed** from the denominator, i.e., `UR = borrow / cash` instead of `borrow / (cash + borrow)`.
    - Multiple, inconsistent UR sources (e.g., `_prevUtilizationRate` vs `_newUtilizationRate`) mixing different denominators.
    - Accrual skip condition tied to **percent change of UR** rather than elapsed time or true economic change.
    - Anyone-can-call `addInterest()` without rate smoothing or cooldowns.
- **Fix pattern:**
    - **Correct UR:** `UR = totalBorrow.amount / totalAsset.totalAmount(externalAssetVault)` (where `totalAsset` already includes both cash and borrow). If you must compute cash separately, do `denom = cash + totalBorrow.amount`.
    - **Single-source-of-truth:** Factor UR computation into one pure/internal function and reuse everywhere.
    - **Defensive accrual gating:** Base accrual trigger primarily on **time elapsed** (e.g., min accrual period) and/or **balance changes**, not tiny UR jitters; add a **cooldown** or dampening.
    - **Precision guards:** Use consistent scaling (e.g., 1e5/1e18) and avoid division order that exaggerates small changes; clamp UR to `[0, 1e18]`.
    - **Tests:**
        - With fixed cash/borrow, call `addInterest` across many blocks and assert **no accrual** when UR is unchanged.
        - Fuzz cash/borrow; verify `UR_cashOnly` ≠ `UR_total` and that only the corrected formula preserves skip behavior.
- **Questions to ask next time:**
    - “Is utilization defined as **borrow / (cash + borrow)** everywhere, and do helpers reflect that?”
    - “Can any user call `addInterest` and cause rate jumps without state changes? Is there a cooldown or time-weighted rate update?”
    - “Do `_prevUtilizationRate` and `_newUtilizationRate` use the **same** denominator and precision?”
    - “What invariant/unit tests protect that `abs(newUR - prevUR)` is ~0 when balances don’t change?”
- [ ]  **Order-dependent liquidation bonus drains last liquidator**
- **Symptom:** Early partial liquidators get full bonus while the final liquidator receives less than the advertised bonus (or even < principal), because payouts exceed a fair pro-rata share of remaining collateral.
- **Where to look (lending/liquidation modules):**
    - Functions that compute per-call liquidation payout: `liquidate`, `_calcLiquidationPayout`, `_seize`, `_distributeLiquidation`, `_calcSeizeTokens`.
    - Codepaths distinguishing “full/clean” vs “partial/dirty” liquidation: variables like `cleanLiquidationFee`, `dirtyLiquidationFee`, `protocolLiquidationFee`, `leftoverCollateral`, `optimisticCollateralForLiquidator`.
    - Any branch that pays **“all remaining collateral”** or computes payout as `min(remainingCollateral, repayAmount * (1 + fee))`.
    - Math that uses **current** remaining `C` (collateral) and `D` (debt) but **does not** cap `payout ≤ C * (repay/D)` on each partial.
    - Unit tests (or lack thereof) for **order-independent** partial liquidations.
- **Red flags:**
    - Pattern like:
        
        ```solidity
        uint256 optimistic = repayInCollat * (LIQ_PRECISION + cleanFee) / LIQ_PRECISION;
        int256 leftover = int256(userCollat) - int256(optimistic);
        uint256 payout = leftover <= 0
            ? userCollat     // give all remaining
            : repayInCollat * (LIQ_PRECISION + dirtyFee) / LIQ_PRECISION;
        // protocol fee then taken from payout
        
        ```
        
    - Fee application **before** checking pro-rata availability (`payout > C * x / D`).
    - Protocol fee taken from *theoretical* payout instead of the **capped** payout.
- **Questions to ask next time:**
    - “If several partial liquidations occur in different orders, do total payouts and each liquidator’s payout remain the same?” (Order-independence)
    - “Is each partial payout capped by the liquidator’s pro-rata share: `min(gross, C * x / D)`? Where is that enforced?”
    - “When collateral is insufficient to fund bonuses, is the shortfall spread **pro-rata** across all liquidators, or does the last one eat the loss?”
    - “Are protocol fees computed from **capped** payout or from pre-cap ‘gross’ payout?”
    - “Do you ever transfer ‘all remaining collateral’ to a liquidator as a shortcut? Under what predicate?”
- **Fix pattern:**
    - Compute the scheduled gross: `gross = x * (1 + fee)`.
    **Cap** by pro-rata: `cap = min(gross, C * x / D)`, then `payout = cap - protocolFee(cap)`.
    - Remove “give all remaining collateral” branches; always respect the cap.
    - If desired, add a **no-loss guard** for partials (`require(cap >= x)` unless doing a full liquidation).
- [ ]  **Oracle double-applies exit/debond fees in chained conversions**
- **Symptom:** Reported price for an LP/staked token (e.g., spTKN per base) is biased high (or base per pTKN biased low) because the debond/unwrap fee is subtracted twice along the price path.
- **Where to look (oracle + wrapper math):**
    - Oracle pricing funnels that hop through wrappers: `_MinimalOracle::_calculateBasePerPTkn`, `_calculateSpTknPerBase`, `_checkAndHandleBaseTokenPodConfig`.
    - Fee helpers in the oracle: `_accountForCBRInPrice`, `_accountForUnwrapFeeInPrice`, `_apply{Buy/Sell/Burn}Fee`.
    - Underlying wrapper/vault math used by the oracle:
        - Pods/vaults: `WeightedIndex.convertToAssets/convertToShares` (and whether these already net debond/unwrap fees).
        - ERC-4626‐like `previewRedeem/previewWithdraw` vs `convertToAssets` semantics (do they include fees?).
    - Conditional base handling: branches like `BASE_IS_POD` / `BASE_IS_FRAX_PAIR` that may reapply fees after a conversion already netted them.
    - Any place that takes a “gross” price, then calls a function that returns a **net-of-fee** amount, and then manually subtracts the fee again.
- **Red flags:**
    - Sequence like: `price = accountForCBR(convertToAssets(...)); price = accountForUnwrapFee(price);`
    - Comments like “convert pTKN→TKN (CBR)” followed by “now subtract unwrap/debond fee” while `convertToAssets` already removed it.
    - Mixing *ratio* adjustments (CBR) with *fee* adjustments without clear gross/net contracts.
    - Applying both `convertToAssets` (net) **and** `(1 - fee)` or `/ (1 - fee)` again.
    - Duplicated logic for base and collateral (e.g., `BASE_IS_POD` re-runs the same fee adjustments).
    - Lack of invariants/tests for round-trips (e.g., TKN→pTKN→TKN equals TKN*(1 - fee) exactly once).
- **Questions to ask next time:**
    - “Do `convertToAssets/preview*` already include exit or debond fees? Where is that documented?”
    - “Are we ever multiplying by `(1 - fee)` (or dividing by it) after calling a function that already returned a net amount?”
    - “For the base-is-wrapped branch, do we reuse the same fee adjustment twice (once in conversion, once in ‘unwrap fee’)?”
    - “What’s the contract of each helper: does it return *gross* or *net* values? Is that enforced with tests?”
    - “Do unit tests assert that enabling a non-zero debond fee changes price exactly once (not twice) and that a TKN→pTKN→TKN round-trip applies the fee once?”
- **Fix pattern:**
    - Decide a single source of truth for fee application:
        - **Option A:** Rely on `convertToAssets/preview*` (net), then **remove** subsequent manual fee adjustments.
        - **Option B:** Use **gross** conversions (that do not include fees) and apply the fee exactly once in the oracle.
    - Clearly separate **ratio (CBR)** vs **fee** adjustments and document gross/net expectations in function names (`_Net`, `_Gross`).
    - If base can be a pod/frax pair, compute a fee/CBR factor **once** (e.g., `cbr := convertToAssets(1e18)`) and avoid reapplying unwrap logic downstream.
- [ ]  **Chainlink min/max bounds misused → accept clamped/orphaned prices during crashes**
- **Symptom:** Oracle treats `minAnswer`/`maxAnswer` as simple validity checks (`answer < min || answer > max`), which never trigger when Chainlink **clamps** at the bound after a massive move (e.g., LUNA-style crash). Protocol continues using an inflated/deflated price instead of flagging it as invalid.
- **Where to look (Chainlink adapters & price guards):**
    - On-chain oracle adapters: `ChainlinkSinglePriceOracle`, `PriceOracle.sol`, `_OracleAdapter.sol`.
    - Lines around `latestRoundData()` usage and bound checks: `answer`, `updatedAt`, `answeredInRound`, plus calls to `minAnswer()` / `maxAnswer()`.
    - Proxies vs aggregators: `AggregatorV3Interface(Proxy).aggregator().minAnswer()` and `.maxAnswer()`; verify you’re reading bounds from the actual aggregator, not a proxy without them.
    - Any “bounds + heartbeat + deviation” guardrail logic; places that compare `int256 answer` directly to `minAnswer/maxAnswer` without a **proximity** buffer.
- **Red flags:**
    - Code like `if (answer > max || answer < min) invalid = true;` with no “near-bound” treatment.
    - No handling for “clamped” scenarios (answer == minAnswer or == maxAnswer for multiple rounds).
    - Missing or weak staleness checks (`updatedAt`, `answeredInRound < roundId`, heartbeat).
    - No cross-check against secondary sources (TWAP/DEX quote) or sanity bounds after a big move.
    - Assuming `minAnswer` means “lowest possible market price” rather than “aggregator-configured floor”.
    - Not accounting for decimals/sign on `int256` answer, or not normalizing units before comparisons.
- **Questions to ask next time:**
    - “What happens if the market price moves **past** Chainlink’s configured min/max? Do we detect a clamp at the bound?”
    - “Do we treat `answer == minAnswer` (or within ε of it) as suspicious and trigger circuit breakers?”
    - “Are we enforcing both heartbeat staleness and round integrity (`answeredInRound >= roundId`)? Any grace windows?”
    - “Do we have a secondary sanity check (e.g., capped deviation vs. last good price / DEX TWAP) when near bounds?”
    - “Are we reading `minAnswer/maxAnswer` from the **underlying aggregator**, not just the proxy?”
- **Fix pattern:**
    - Replace strict bound tests with **proximity/clamp detection**:
        - Mark price invalid if `answer <= minAnswer + gap` or `answer >= maxAnswer - gap` for N consecutive rounds (gap sized in **asset units** after decimals normalization).
        - Require **freshness**: `block.timestamp - updatedAt <= heartbeat`, and `answeredInRound >= roundId`.
        - Add **LastGoodPrice / circuit-breaker**: if near-bound or invalid, fall back to last good price and/or pause sensitive actions (mint, borrow, LTV bumps).
        - Optional cross-check: compare to conservative DEX TWAP; if deviation > threshold, treat as invalid.
    - Document semantics: `min/maxAnswer` are **aggregator config bounds**, not market limits; treat bound hits as potential clamps, not valid prices.
- [ ]  **Asymmetric CBR delta math over-shrinks utilization on decreases**
- **Symptom:** When a vault’s CBR (assets per share) drops, code computes the change as `(prev/current) - 1` and applies it as a “percent decrease”, causing outsized cuts (e.g., 1.5→1 reported as 50% instead of 33%) and can zero out `vaultUtilization`/totals.
- **Where to look (ERC-4626 aggregators / vault-of-vaults / lending routers):**
    - Vault metadata updaters: functions named like `_updateAssetMetadata…`, `_recomputeUtilization`, `_syncVaultCbr`, `_onInterestAccrued`.
    - Any place reading `IERC4626.convertToAssets(PRECISION)` or storing “CBR”, “asset/share ratio”, “exchangeRate”, “pricePerShare”.
    - Accounting state touched together: `vaultUtilization[vault]`, `_totalAssetsUtilized`, `_totalAssets`, `utilizationByVault`.
    - Fraxlend-style pairs or wrappers where utilization depends on share→asset conversion.
- **Red flags:**
    - Decrease branch uses `((PRECISION * prev) / current) - PRECISION` (i.e., `prev/current - 1`) to derive a **decrease** percentage.
    - Same delta formula used for both up/down moves (symmetric), or “flip sign” logic without changing the ratio orientation.
    - Applying percent deltas to utilization instead of recalculating via a direct scale factor (`current / prev`).
    - No tests for down moves like `1.5e27 → 1e27` (33%) or `1.5e27 → 0.75e27` (50%); rounding not examined.
    - Updating both `vaultUtilization` and `_totalAssets` with the same mistaken delta, compounding error.
- **Questions to ask next time:**
    - “When CBR decreases from `prev` to `curr`, do we compute the new utilization as `old * curr / prev`, or do we apply a ‘percent change’? If the latter, is the decrease formula `1 - curr/prev` or mistakenly `prev/curr - 1`?”
    - “Do unit tests assert correct behavior for both increases **and** decreases (1.5→1 and 1.5→0.75) with exact expected percents (33%, 50%)?”
    - “Are we double-applying the delta (e.g., to utilization and totals) or mixing deltas with separate absolute recomputations?”
    - “What guards exist for extreme drops (bad debt, haircuts) and rounding direction (floor/ceil)?”
- **Fix pattern:**
    - Prefer **direct rescaling**: `newUtilization = (oldUtilization * currCBR) / prevCBR;` (same for any mirrored totals) to avoid delta math entirely.
    - If using deltas, make them directionally correct:
        - Increase: `deltaUp = (PRECISION * curr / prev) - PRECISION`.
        - Decrease: `deltaDown = PRECISION - (PRECISION * curr / prev)` (i.e., `1 - curr/prev`), **not** `prev/curr - 1`.
    - Normalize all ratios to a single `PRECISION` and document rounding (favor floor to avoid inflation of utilization).
- [ ]  **Asymmetric CBR delta math over-shrinks utilization on decreases**
- **Symptom:** When a vault’s CBR (assets per share) drops, code computes the change as `(prev/current) - 1` and applies it as a “percent decrease”, causing outsized cuts (e.g., 1.5→1 reported as 50% instead of 33%) and can zero out `vaultUtilization`/totals.
- **Where to look (ERC-4626 aggregators / vault-of-vaults / lending routers):**
    - Vault metadata updaters: functions named like `_updateAssetMetadata…`, `_recomputeUtilization`, `_syncVaultCbr`, `_onInterestAccrued`.
    - Any place reading `IERC4626.convertToAssets(PRECISION)` or storing “CBR”, “asset/share ratio”, “exchangeRate”, “pricePerShare”.
    - Accounting state touched together: `vaultUtilization[vault]`, `_totalAssetsUtilized`, `_totalAssets`, `utilizationByVault`.
    - Fraxlend-style pairs or wrappers where utilization depends on share→asset conversion.
- **Red flags:**
    - Decrease branch uses `((PRECISION * prev) / current) - PRECISION` (i.e., `prev/current - 1`) to derive a **decrease** percentage.
    - Same delta formula used for both up/down moves (symmetric), or “flip sign” logic without changing the ratio orientation.
    - Applying percent deltas to utilization instead of recalculating via a direct scale factor (`current / prev`).
    - No tests for down moves like `1.5e27 → 1e27` (33%) or `1.5e27 → 0.75e27` (50%); rounding not examined.
    - Updating both `vaultUtilization` and `_totalAssets` with the same mistaken delta, compounding error.
- **Questions to ask next time:**
    - “When CBR decreases from `prev` to `curr`, do we compute the new utilization as `old * curr / prev`, or do we apply a ‘percent change’? If the latter, is the decrease formula `1 - curr/prev` or mistakenly `prev/curr - 1`?”
    - “Do unit tests assert correct behavior for both increases **and** decreases (1.5→1 and 1.5→0.75) with exact expected percents (33%, 50%)?”
    - “Are we double-applying the delta (e.g., to utilization and totals) or mixing deltas with separate absolute recomputations?”
    - “What guards exist for extreme drops (bad debt, haircuts) and rounding direction (floor/ceil)?”
- **Fix pattern:**
    - Prefer **direct rescaling**: `newUtilization = (oldUtilization * currCBR) / prevCBR;` (same for any mirrored totals) to avoid delta math entirely.
    - If using deltas, make them directionally correct:
        - Increase: `deltaUp = (PRECISION * curr / prev) - PRECISION`.
        - Decrease: `deltaDown = PRECISION - (PRECISION * curr / prev)` (i.e., `1 - curr/prev`), **not** `prev/curr - 1`.
    - Normalize all ratios to a single `PRECISION` and document rounding (favor floor to avoid inflation of utilization).
- [ ]  **Vault rebalancing skips global interest sync → stale utilization & mispriced rates**
- **Symptom:** Functions that change LendingAssetVault (LAV) asset balances (whitelist or owner admin paths) don’t trigger the system-wide interest/metadata refresh, leaving FraxlendPair utilization and rates computed from stale totals. Interest accrues inaccurately vs. the “deposit/withdraw” paths that do sync first.
- **Where to look (ERC-4626 routers / vault-of-vaults / Fraxlend integrators):**
    - LAV-style contracts: `whitelistWithdraw`, `whitelistDeposit`, `depositToVault`, `redeemFromVault`, `rebalance`, `sweep`, `allocate`, `deallocate`.
    - Any function that moves assets between the aggregator and underlying pairs/vaults without going through the public `deposit/withdraw` entrypoints.
    - Cross-check callers of `_updateInterestAndMdInAllVaults` (or similarly named “sync” hooks) vs. **all** state-changing functions that affect vault balances.
    - FraxlendPair integration points using `addInterest`, `previewAddInterest`, or utilization math that depends on LAV balances.
- **Red flags:**
    - Functions update accounting (`vaultDeposits`, `vaultUtilization`, `_totalAssetsUtilized`, `_totalAssets`) and maybe call `_updateAssetMetadataFromVault(_vault)` **but never** call the global sync `_updateInterestAndMdInAllVaults()`.
    - Admin/whitelist-only pathways that bypass user `deposit/withdraw` and thus skip interest sync.
    - Utilization/rate guards computed **before** a potential sync (ordering issue).
    - Tests only cover `deposit/withdraw` paths for interest sync, no tests for whitelist/admin asset moves.
- **Questions to ask next time:**
    - “Which entrypoints can change LAV’s asset balances? Do **all** of them call the global interest/metadata sync (or `previewAddInterest`) before updating accounting?”
    - “Are utilization/rate calculations anywhere using totals that could be stale if a sync is skipped?”
    - “Do admin or whitelist flows (rebalancing, vault migrations) produce identical interest outcomes as the user flows in tests?”
    - “Is there a modifier (e.g., `syncInterest`) consistently applied to every balance-changing function?”
- **Fix pattern:**
    - Call `_updateInterestAndMdInAllVaults(address(0))` at the **start** of every function that changes LAV or vault balances (including whitelist/admin functions).
    - Alternatively, call `previewAddInterest`/`addInterest` on all paired vaults first, then update local accounting.
    - Add a `modifier syncInterest()` used by `deposit`, `withdraw`, `whitelistDeposit/Withdraw`, `depositToVault`, `redeemFromVault`, and any rebalance route.
- [ ]  **Reward compounding assumes paired asset is fTKN, breaks for podded fTKN (pfTKN)**
- **Symptom:** Auto-compounding on “advanced” self-lending pods (where the paired LP token is a *podded* fTKN) reverts or yields 0 output. Rewards remain un-compounded; events like `TokenToPairedLpSwapError` may fire; vault’s totalAssets doesn’t grow despite pending rewards.
- **Where to look (Auto-compounders / yield routers around self-lending):**
    - `AutoCompoundingPodLp.sol`:
        - `_tokenToPodLp` → `_tokenToPairedLpToken` → `_pairedLpTokenToPodLp`.
        - Branches guarded by `IS_PAIRED_LENDING_PAIR` and calls to `_depositIntoLendingPair(...)`.
    - Config that signals “advanced” self-lending: `LeverageManager.initializePosition(..., _hasSelfLendingPairPod)` and how/if that flag is consumed in the compounding path.
    - Type assumptions for `pod.PAIRED_LP_TOKEN()`:
        - Is it cast/treated as `IFraxlendPair` unconditionally?
        - Does code attempt Fraxlend deposit when the paired token is actually an *index/pod* (`IDecentralizedIndex`)?
    - Any place that converts **reward → borrow asset → fTKN** but **never fTKN → pfTKN** (bond/mint step missing).
- **Red flags:**
    - Comments mentioning “advanced self-lending” / “podded as well” with no corresponding code path in the auto-compounder.
    - Conditionals like `if (IS_PAIRED_LENDING_PAIR) { _swapOutputTkn = IFraxlendPair(_paired).asset(); ... _depositIntoLendingPair(...) }` with **no branch** for `paired` being a *pod token* (pfTKN).
    - Treating `PAIRED_LP_TOKEN` as a Fraxlend pair address when it’s actually an ERC20 pod (will revert or silently do the wrong thing).
    - No tests covering reward conversion when `paired` is a pod over fTKN (pfTKN).
- **Questions to ask next time:**
    - “Can `PAIRED_LP_TOKEN` be anything other than an `IFraxlendPair` (e.g., an `IDecentralizedIndex` over fTKN)? If yes, what’s the conversion pipeline?”
    - “Where is the step from **fTKN → pfTKN** implemented? Is it via bonding/minting into the pfTKN pod or via a DEX route with pfTKN liquidity?”
    - “Do we introspect the paired token type at runtime (interface detection) or rely on a single boolean like `IS_PAIRED_LENDING_PAIR`?”
    - “Are slippage/fee parameters and approvals set for the additional hop (bonding into pfTKN), and are transfer-tax fees accounted for on that hop?”
- **Fix pattern:**
    - Introduce a **paired-asset adapter** interface (e.g., `IPairedConverter`) with implementations for:
        - **Fraxlend pair:** reward → borrow asset → deposit → fTKN.
        - **Podded fTKN:** reward → borrow asset → deposit → fTKN → **bond into pfTKN** (handle pod fees/tax) *or* DEX swap to pfTKN if deep liquidity exists.
        - **Plain ERC20:** direct swap.
    - Detect paired token type via interface support (try/catch or ERC-165-style) rather than a single boolean.
    - Surface config: pod address for pfTKN, bonding min-outs/slippage, and fee-on-transfer handling on bonding/mint.
- [ ]  **Per-hop swap caps strand intermediate tokens in V2 multihops**
- **Symptom:** After reward → pairedLP multi-hop swaps, the contract accumulates a non-zero balance of the intermediate token (path[1]). `totalAssets` under-grows; future compounds never drain the dust because the second hop only uses `min(balance, maxSwap[path[1]])`.
- **Where to look (auto-compounders / routers / vault rebalancers):**
    - Swap helpers: `_swap`, `_swapV2`, `_tokenToPairedLpToken` (or equivalents) that build `path` arrays and call a DEX adapter (e.g., `swapV2Single`).
    - Config maps like `swapMaps[in][out]` and `maxSwap[token]` that gate per-token amounts.
    - Any logic that does `amountOut = firstHop(...); intermediate = amountOut or IERC20(path[1]).balanceOf(this); secondHop(intermediate, ...)` with a cap on `intermediate`.
- **Red flags:**
    - Applying `maxSwap` **again** on the intermediate hop (`if (maxSwap[path[1]] > 0 && intermediate > max) intermediate = max;`) while *not* re-using the leftover later.
    - First hop uses exact-in and second hop uses only a fraction, leaving the remainder in the contract indefinitely.
    - No “sweep leftover” / “drain intermediate” step after multihop; no admin recovery for non-reward tokens.
    - Unit tests only cover single-hop or unlimited-amount swaps; no assertions that `balanceOf(path[1]) == 0` post-swap.
- **Questions to ask next time:**
    - “Is `maxSwap` intended to limit *input* only, or also intermediate hops? What happens to the remainder?”
    - “Do we ever loop/split to process the leftover intermediate balance in subsequent calls, or is it permanently stranded?”
    - “Is there a periodic sweep that converts any non-tracked intermediate balances back to the paired/reward token?”
    - “Are minOuts re-scaled when capping amounts mid-path? Could proportional minOut cause silent value loss?”
    - “Do we need `maxSwap` on path[1] at all, or should risk controls live on the *first* hop only?”
- **Fix pattern:**
    - Prefer **cap only the first hop** (`path[0]`) and **always swap 100% of the intermediate token balance** on the second hop:
        
        ```solidity
        uint256 inter = IERC20(path[1]).balanceOf(address(this));
        IERC20(path[1]).safeIncreaseAllowance(adapter, inter);
        _amountOut = adapter.swapV2Single(path[1], path[2], inter, _amountOutMin, address(this));
        
        ```
        
    - If a per-hop cap is mandatory, **iterate**: split the intermediate balance into chunks and loop until drained or gas limit reached; otherwise **return leftover explicitly** and handle/sweep it immediately after.
    - Add a **post-swap sweep** that converts any residual `path[1]` into the target (or into the reward token) to keep balances clean.
    - Test multihop with `maxSwap[path[1]]` set and assert `IERC20(path[1]).balanceOf(this) == 0` after compounding.
- [ ]  **USD-only quote path silently assumes stables = $1 in cross-asset oracles**
- **Symptom:** Oracle computes `underlying/base` by dividing `underlying/USD` (from a pool via USDC/DAI routing) by `base/USD` (from a DIA/CL USD feed), implicitly treating the stablecoin leg as exactly $1. During depegs, `underlying/base` is materially wrong (over/under-prices collateral/borrow).
- **Where to look (Oracle composition / price routers):**
    - Composite oracles that expose `getPriceUSD18` or similar and then derive `tokenA/tokenB` via division: e.g., `spTKNMinimalOracle::_getDefaultPrice18`, `_calculateBasePerPTkn`.
    - DIA/Chainlink wrappers (e.g., `DIAOracleV2SinglePriceOracle`, `UniswapV3SinglePriceOracle`) and code paths referencing `BASE_CONVERSION_DIA_FEED`, `BASE_CONVERSION_CL_POOL`, `BASE_IN_CL`.
    - Any logic that builds a USD price for one leg from **DEX** (token vs USDC) and a USD price for the other leg from **Chainlink/DIA**, without an explicit stablecoin/USD adjustment.
    - String-based DIA lookups like `symbol + "/USD"` and comments defaulting base to `1e18` (“return only quote pool price without any base conversion”).
- **Red flags:**
    - Comments or code assuming stables = $1 (e.g., default `_basePrice18 = 1e18`) while the *other* leg uses a USDC or DAI pool.
    - Mixing numeraires: one leg via UniswapV3 against USDC/DAI, the other via DIA/CL `/USD`, with no `stable/USD` factor.
    - No handling for stablecoin depeg events; no “bad data” on stablecoin USD feeds; no `stableUsdFeed` wiring.
    - Division chain like: `price = (tokenA/USD_from_pool) / (tokenB/USD_from_oracle)` where pool “USD” is actually USDC.
- **Questions to ask next time:**
    - “What is the **common numeraire** for both legs—USD, ETH, or a specific stable? Are **both** legs quoted versus the **same** numeraire?”
    - “If a DEX pool uses USDC/DAI as quote, where do we account for `USDC/USD`? Do we have a CL feed for it or a basket?”
    - “What happens during a stablecoin depeg? Do we mark data bad or adjust with a `stable/USD` feed?”
    - “Do we ever default `_basePrice18 = 1e18`? Under what conditions, and is that safe?”
    - “Are there tests that simulate a 10–20% depeg of USDC/DAI and verify oracle correctness?”
- **Fix pattern:**
    - Ensure **both legs share the same numeraire** and include a **stable/USD conversion** when needed:
        - If using DEX `token/USDC` for one leg and DIA/CL `/USD` for the other, multiply the DEX leg by `USDC/USD` (from Chainlink) or prefer `token/WETH` and `base/WETH` paths via on-chain pools.
    - Do **not default** USD conversion to `1e18` when a stablecoin pool is involved; wire `stableUsdFeed` and multiply/divide accordingly.
    - Add **bad-data rules**: if stable/USD feed is stale or missing, fail closed for cross-asset prices.