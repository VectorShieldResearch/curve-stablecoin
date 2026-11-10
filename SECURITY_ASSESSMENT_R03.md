# SECURITY_ASSESSMENT_R03

## 1. Executive decision

**NO-GO.**

No novel, reproducible, financially meaningful security vulnerability was confirmed in the LlamaLend V2 codebase at the reviewed commit. No bounty submission is justified.

## 2. Repository, branch, commit, and research date

- Primary repository (per mission): `https://github.com/curvefi/curve-stablecoin`
- Workspace repository (remote): `https://github.com/VectorShieldResearch/curve-stablecoin.git` (a fork)
- Current branch: `kilo/late-radar-4z8`
- HEAD commit: `f574bd502530fb3353e7b60918e246e50de5891f` (2026-11-09 17:43:28 +0400)
- Working tree: clean
- Research date: 2026-11-10

**Commit verification:** HEAD is *exactly* the previously reviewed commit `f574bd50...` from Round 02. The repository has **not** advanced beyond it (no new commits on the branch, no new tags, no diverging heads). All previously reviewed code is current; no post-fix changes needed re-review.

## 3. Scope and files reviewed

Files reviewed (current, at HEAD):

**Controller core (Investigator A & B):**
- `curve_stablecoin/controller.vy` — `liquidate` (L1254), `repay` (L1100), `_repay_partial` (L978), `_repay_full` (L933), `tokens_to_liquidate` (L1382), `_debt` (L340), `_update_total_debt` (L229), `_health` (L1171), `_get_f_remove` (L1213)

**Zaps (Investigator A & B):**
- `curve_stablecoin/zaps/PartialRepayZapLending.vy` — `_liquidate_partial` (L127), `users_to_liquidate`/`_users_to_liquidate` (L76), `callback_liquidate` (L212)
- `curve_stablecoin/zaps/transient_leverage_zap/TransientLeverageZapLend.vy` — `_repay` (L402), `callback_repay` (L276), `callback_deposit` (L230)
- `curve_stablecoin/zaps/PartialRepayZapMint.vy` (reference)
- `curve_stablecoin/testing/zaps/PartialRepayZapTester.vy` (test callbacker)

**AMM (Investigator B):**
- `curve_stablecoin/AMM.vy` — `withdraw` (L724), `deposit_range` (L643), `get_x_down`/`get_xy_up` (L1459), `get_sum_xy`/`_get_xy` (L1508), `active_band_with_skip` (L601)

**Oracles (Investigator C):**
- `curve_stablecoin/price_oracles/v2/ChainOracle.vy`
- `curve_stablecoin/price_oracles/v2/ERC4626EMAWrapper.vy`
- `curve_stablecoin/price_oracles/v2/ERC4626EMAOracle.vy`
- `curve_stablecoin/price_oracles/AggregateStablePrice.vy`
- `curve_stablecoin/price_oracles/v2/ChainlinkEMA.sol`
- `curve_stablecoin/price_oracles/v2/adapters/ReusdCrvUSDAdapter.vy`
- `curve_stablecoin/price_oracles/v2/adapters/SyrupUSDCAdapter.vy`

**Tests reviewed (coverage baseline):**
- `tests/zaps/partial_liquidation/test_partial_repay_zap.py`
- `tests/e2e/zaps/transient_leverage_zap/test_repay.py`
- `tests/e2e/zaps/transient_leverage_zap/test_repay_edge_cases.py`
- `tests/unitary/controller/test_internal_repay_partial.py`
- `tests/zaps/partial_liquidation/calculations/` (historical mainnet position evaluation)

**Deployed active markets (Ethereum), used to bound Investigator C to real configs:**
- `reUSDsfrxUSDLP-crvUSD`, `sDOLA-crvUSD`, `sfrxUSD-crvUSD`, `svZCHF-crvUSD`, `syrupUSDC-crvUSD` (see `deployments/llamalend/ethereum/markets/`)

**Additional contracts reviewed (expanded scope — "the rest of the repository"):**
- `curve_stablecoin/lending/LendController.vy` — `available_balance`, `on_borrowed_token_transfer_in/out`, `_on_debt_increased` (borrow cap), `configure_lend`
- `curve_stablecoin/lending/LendFactory.vy` — `create`, `markets`, `pause`/`unpause`, `set_custom_fee_receiver`, `check_contract`
- `curve_stablecoin/lending/Vault.vy` — `deposit`/`mint`/`withdraw`/`redeem`, `_total_assets`, `MIN_ASSETS` (F-1 from Round 02, not re-submitted), `_convert_to_shares`/`_convert_to_assets`
- `curve_stablecoin/lending/blueprint_registry.vy` (reference)
- `curve_stablecoin/AMM.vy` (full read) — `calc_swap_out`, `_exchange`, `get_dy`/`get_dxdy`, `exchange`, `set_rate`, fee/rate handling
- `curve_stablecoin/Configurator.vy` — `set_borrowing_discounts`, `set_monetary_policy`, `set_price_oracle` (max deviation check), `set_callback`, `set_amm_fee`, `set_view`
- `curve_stablecoin/ControllerFactory.vy` — `add_market`, `_set_debt_ceiling` (mint/burn against residual), `set_debt_ceiling`, `rug_debt_ceiling`, `collect_fees_above_ceiling`, `total_debt`
- `curve_stablecoin/ControllerView.vy` — health previews, `max_borrowable`/`min_collateral`, `tokens_to_shrink`, `users_with_health`, `_user_state`
- `curve_stablecoin/MintController.vy` — thin adapter over core controller
- `curve_stablecoin/Stablecoin.vy` — ERC20 + EIP-2612 permit, mint/burn access control
- `curve_stablecoin/stabilizer/PegKeeperV2.vy`, `PegKeeperRegulator.vy`, `PegKeeper.vy`, `Salvation.vy`, `PegKeeperOffboarding.vy`
- `curve_stablecoin/mpolicies/` — `KinkedMonetaryPolicy.vy` (and siblings), `wstETHRateCalculator.vy`
- `curve_stablecoin/flashloan/FlashLender.vy` — ERC3156 flash loan, nonreentrant, repay check vs `debt_ceiling_residual`
- `curve_stablecoin/lm_callback/LMCallback.vy` — CRV gauge-style reward accounting, `claimable_tokens`, checkpoint gating on `_attached()`
- `curve_stablecoin/factory/OwnerProxy.vy` — DAO ownership proxy (commit/apply admin pattern)

## 4. Investigation results

### Investigator A — Partial repayment and liquidation

Focus: `PartialRepayZapLending._liquidate_partial` vs `Controller.liquidate` / `Controller.repay` / `_repay_partial`, `debt_diff`, `initial_x - new_x`, `paid_by_sender`, `surplus_repaid`, total debt reconciliation.

**Finding:** No financial vulnerability. The zap-level and controller-level accounting is internally consistent.

Trace of `_liquidate_partial` (zap-level):
1. `total_debt = controller.debt(user)` (user's debt pre-liquidate; variable name is misleading — it is *this user's* debt, not system total).
2. `initial_x = user_state[1]` (user's borrowed pre-liquidate).
3. `ratio = x_down * WAD / total_debt`; `assert ratio > WAD` ("position rekt") — only over-collateralized-in-liquidation-value positions proceed.
4. `to_repay = controller.tokens_to_liquidate(user, FRAC)` (a *view* estimate; code comment at L173 acknowledges it can differ from actual).
5. `borrowed_from_sender = to_repay * ratio / WAD` (estimate of what the funding side must supply).
6. `controller.liquidate(user, min_x, FRAC, zap, calldata)` executes the partial liquidation; the zap's `callback_liquidate` returns `[borrowed_from_sender, 0]`.
7. Post-liquidate measurement:
   - `debt_diff = total_debt - controller.debt(user)` = actual debt reduction (= liquidated fraction, modulo interest-rate bookkeeping, see below).
   - `new_x = user_state[1]` (user's borrowed post-liquidate).
   - `paid_by_sender = debt_diff - (initial_x - new_x)` = actual debt reduction not explained by borrowed-token withdrawal from the AMM (= actual `to_repay`).
   - `surplus_repaid = borrowed_from_sender - paid_by_sender`; `controller.repay(surplus_repaid, user)`.

Because `paid_by_sender` is measured *after* the liquidation from actual state, it always equals the real `to_repay` (when the callback path is taken, `debt > xy[0]`, so `paid_by_sender = debt_liquidated - xy[0] = to_repay_actual`). The zap then repays exactly the difference between its fronted estimate and the actual need.

**Two observed edge conditions, neither is value extraction:**
- If `borrowed_from_sender < paid_by_sender` (possible when `ratio` is just above `WAD` and the position already holds soft-liquidated borrowed), `surplus_repaid` underflows and the transaction reverts. This is a denial-of-service / failed liquidation for the funder; no funds move and no state persists (EVM revert rewinds everything). Not bounty-worthy.
- `debt_diff` is measured against `controller.debt(user)` after `liquidate` calls `_save_rate()`, so if the rate multiplier changed within the transaction, `debt_diff` can include a phantom-interest component. This makes `paid_by_sender` (and hence `surplus_repaid`) slightly off, but the zap sends *all* remaining borrowed back to the funder after the repay, so the net effect on any party is zero — the funder is made whole, no protocol loss, no debt under-recording.

The "surplus borrowed amount goes into position repay" behavior (L178-180 comment) is intentional: excess fronted funds reduce the user's debt further. The funder supplies `borrowed_from_sender` explicitly (via `users_to_liquidate` / `dx`) and is refunded any remainder; this is documented, visible behavior, not an exploit.

### Investigator B — Transient leverage repayment

Focus: `TransientLeverageZapLend._repay` → `Controller.repay` → `_repay_partial` → AMM `withdraw`/`deposit_range`; `_shrink=true`, active-band repositioning, rounding boundaries, partial withdrawals, state consistency.

**Finding:** No financial vulnerability. Debt accounting, AMM balances, and share/position state remain consistent across partial repay, soft-liquidation + shrink, and full-repay-refund paths.

Trace of `_repay_partial` (controller, L978) with `_shrink=true`, callbacker set:
1. `new_debt = debt - (cb.borrowed + wallet_d_debt)`.
2. `if ns[0] > active_band or _shrink`: when `_shrink`, `new_debt -= _xy[0]` (the borrowed withdrawn from the AMM is applied against debt).
3. Bands repositioned via `_calculate_debt_n1(new_collateral, new_debt, size+1, _for)` then `AMM.deposit_range(...)`. `_calculate_debt_n1` reverts ("Debt too high") if `p_oracle_up(n1) > price_oracle()`, so a shrink can never leave an under-collateralized position.
4. Token recovery: controller pulls `cb.borrowed` (from zap), `wallet_d_debt` (from caller), and `_xy[0]` (from AMM when shrinking) — total = `d_debt = debt - new_debt = cb.borrowed + wallet_d_debt + _xy[0]`.
5. `d_debt` returned to `repay` matches `repay`'s own `min(wallet_d_debt + xy[0] + cb.borrowed, debt)` computation for partial repays; `self.loan[_for].initial_debt`, `_update_total_debt(d_debt, ..., False)`, and `self.repaid += d_debt` all agree.

Every unit of debt reduction is backed by real tokens pulled into the controller; there is no unbacked debt erasure, no collateral withdrawn without a matching accounting reduction, and no persistent AMM/controller inconsistency. The `_max_active_band` assert (`active_band_with_skip() <= _max_active_band`) is a front-running slippage guard that reverts cleanly.

Existing tests already cover: dust flush, `max_active_band` too-low revert, soft-liquidation requiring `_shrink`, soft-liquidation reporting only swap proceeds, no-op-router revert ("Collateral must decrease"), full-repay unused-wallet refund, and slippage/`Exchange not approved` reverts.

### Investigator C — Oracle composition

Focus: actual oracle adapter compositions used by verified active markets; ChainOracle, ERC4626EMAWrapper, Chainlink EMA; stale/paused/reverting/discontinuous feeds; falling rates; timestamp/observation-window boundaries; decimals/rate propagation.

**Finding:** No novel vulnerability in actually-deployed configurations.

The five active Ethereum markets use compositions of `AggregateStablePrice`/`pool_oracle`, `agg_wrapper` (EMA-wrapped stable aggregator), `ERC4626EMAWrapper` (e.g. reUSDsfrxUSDLP), `ChainlinkEMA` (wstGBP-style FX), and market-specific adapters (ReusdCrvUSDAdapter, SyrupUSDCAdapter). Key observations:

- `ERC4626EMAWrapper` reports `min(spot, ema)`: downward moves pass through immediately (never over-values collateral on a fall), upward moves are EMA-smoothed. This is the documented, intended failure mode (safe direction). `convertToAssets` decimal conversion is applied once, at 1e18 base.
- `ChainOracle` multiplies chained prices; if any underlying feed reverts or is stale such that its `price()`/`price_w()` reverts, the whole chain reverts — a denial of service, not value extraction, and it fails closed (no bad price reported).
- `ChainlinkEMA.sol` has already been through the audit-followup fixes visible in git history (feed-decimals-at-deploy, zero-observation/interval rejection, phase-switch reseeding, `PRECISION_MUL` immutability). `price()` is read-only/non-persisting; `price_w()` persists EMA state. The AMM reads through the configured oracle per-market.
- No deployed config inverts the safe failure mode (no config reports `max(spot, ema)` or uses a stale high price to over-collateralize). Falling underlying rates are handled by the downward-immediate passthrough in the EMA wrapper and by the AMM's `get_x_down`/health liquidation path.

Oracle damping/cap behavior and ChainlinkEMA phase-switch handling were already documented and fixed; no new issue found in deployed markets.

### Investigator D — Independent skeptical reviewer

**Role:** Disprove hypotheses from A, B, C; search for existing fixes, documented behavior, prior reports, tests, known limitations.

**Result:** Every promising hypothesis was disproved or shown to be non-bounty-worthy:

| Hypothesis | Disposition | Reason |
|---|---|---|
| A: `paid_by_sender`/`surplus_repaid` debt under-recording / value extraction | Rejected | `paid_by_sender` measured post-liquidate from actual state; always equals real `to_repay`; zap refunds remainder to funder; surplus-repay is intentional, visible, funder-supplied |
| A: `debt_diff` phantom interest causes debt mis-accounting | Rejected | Net effect zero (funder made whole); no protocol loss, no debt under-recording; rate change within one tx is deterministic, not attacker-controlled |
| A: revert when `ratio` near `WAD` | Not a bounty | DOS / failed liquidation only; state rewound on revert; no funds extracted |
| B: `_shrink` debt double-count / unbacked debt reduction | Rejected | `d_debt == cb.borrowed + wallet_d_debt + xy[0]`; every unit pulled into controller; `new_debt -= xy[0]` bounded by partial-repay condition |
| B: band repositioning leaves inconsistent AMM/controller state | Rejected | `_calculate_debt_n1` reverts if debt too high; `deposit_range` requires empty bands; withdraw(WAD) zeroes user shares first |
| B: `_max_active_band` bypass / front-run value extraction | Rejected | `assert active_band_with_skip() <= _max_active_band` reverts before any state change |
| C: ERC4626EMAWrapper over-values collateral on falling rates | Rejected | `min(spot, ema)`: downside immediate, documented safe direction |
| C: ChainOracle stale feed reports bad price | Rejected | chain reverts on any member revert; fails closed; DOS only |
| C: ChainlinkEMA phase-switch / decimals bug | Rejected | Already fixed in git history (audit follow-up PRs); current code resolves decimals per phase |

No hypothesis meets Medium/High severity criteria. No ordinary user can trigger unauthorized value extraction; no debt is erased/under-recorded/incorrectly attributed; no collateral becomes withdrawable without matching accounting reduction; no repayment/liquidation sequence leaves AMM and controller inconsistent; no unprivileged user causes measurable protocol loss.


### Expanded scope — remaining contracts reviewed

All remaining repository contracts were reviewed statically. None presents a novel, reproducible, financially meaningful vulnerability:

| Area | Contracts reviewed | Result |
|------|-------------------|--------|
| Lend markets | `lending/LendController.vy`, `lending/LendFactory.vy`, `lending/Vault.vy`, `lending/blueprint_registry.vy` | `LendController._on_debt_increased` enforces borrow cap; `available_balance` tracking gated to vault/controller only; `LendFactory.create` validates oracle (`price()` == `price_w()`) and A/bounds; `Vault` deposit/withdraw/redeem respect `MIN_ASSETS`/`maxSupply`, `_total_assets()` consistent with controller. Vault `MIN_ASSETS` is F-1 from Round 02 (LOW/Informational) — not re-submitted. |
| AMM core | `AMM.vy` (full) — `calc_swap_out`, `_exchange`, `get_dy`/`get_dxdy`, `exchange`, `set_rate`, fee/rate | Swap rounds `in_amount` up / `out_amount` down (safe for protocol); `nonreentrant` on state-changing swaps; `MAX_ORACLE_DN_POW` caps band travel (~50 ticks). No manipulation path to unauthorized extraction. |
| Configurator / factories | `Configurator.vy`, `ControllerFactory.vy`, `MintController.vy`, `factory/OwnerProxy.vy` | All admin-gated (`_check_admin`/`_check_authorized` / ownable). `set_price_oracle` enforces `MAX_ORACLE_PRICE_DEVIATION` (50%) unless skipped with `max_value`. `ControllerFactory._set_debt_ceiling` mints/burns only against `debt_ceiling_residual` and balance. No cross-market asset extraction. |
| Stabilizers | `stabilizer/PegKeeperV2.vy`, `PegKeeperRegulator.vy`, `PegKeeper.vy`, `Salvation.vy`, `PegKeeperOffboarding.vy` | PegKeeper profit calculation rounds safe (`_calc_profit_from` returns 0 when `lp_balance <= lp_debt`); regulator gates provide/withdraw by price range and debt ratios; admin actions use commit/apply delays. No user-triggered extraction. |
| Monetary policies | `mpolicies/` — `KinkedMonetaryPolicy.vy` and siblings | `rate_write` permissioned via controller; rate math asserts non-negative and reserves >= debt. No attacker-controlled rate manipulation beyond documented behavior. |
| Flash loan | `flashloan/FlashLender.vy` | `nonreentrant`, 0 fee, repayment checked against `debt_ceiling_residual(self)` (the contract's own ceiling, not user-controlled). No cross-contract reentrancy to steal funds. |
| LM callback | `lm_callback/LMCallback.vy` | Checkpoint integrals gated on `_attached()` (live callback check latches `detached`); `claimable_tokens` checkpoints then reads `integrate_fraction - minted`. Standard Curve gauge accounting; no double-claim or inflation path. |
| Stablecoin | `Stablecoin.vy` | Standard ERC20 + EIP-2612 permit; `mint` restricted to `minter`; `transferFrom` enforces allowance; domain separator handles chain ID. No novel issue. |

## 5. Candidate hypotheses rejected (with reasons)

All candidates from Investigators A, B, and C are rejected above (see table in section 4.D). None demonstrated:
- an account receiving more value than entitled,
- debt being erased/under-recorded/incorrectly attributed,
- collateral/LP withdrawable without accounting reduction,
- inconsistent AMM/controller state after repay/liquidation,
- unprivileged user-caused measurable protocol loss.

Rounding differences, reverts, and denial-of-service were explicitly considered and ruled out as not bounty-worthy per the mission severity rules. F-1 (Vault `MIN_ASSETS` raw-unit floor vs `maxWithdraw`/`maxRedeem`) from Round 02 was not re-submitted; it is LOW/Informational, bounded, self-recoverable, trivial on 18-decimal markets, and already fixed on a public branch.

## 6. Confirmed finding details

None. No vulnerability was confirmed.

## 7. Root cause

Not applicable — no confirmed finding.

## 8. Minimal reproducible local PoC

Not applicable — no confirmed finding; no PoC was constructed because no hypothesis survived skeptical review as a financial vulnerability.

## 9. Test commands and actual outputs

**Limitation:** The sandbox has no Python/Vyper/Boa toolchain installed (`python3`, `uv`, `pip`, and `boa` are absent), so the repository's test suite (`pytest`/`boa`) could not be executed locally. Test commands that *would* be run (per `pyproject.toml` and repo conventions):

```
pytest tests/zaps/partial_liquidation/test_partial_repay_zap.py
pytest tests/e2e/zaps/transient_leverage_zap/test_repay.py
pytest tests/e2e/zaps/transient_leverage_zap/test_repay_edge_cases.py
pytest tests/unitary/controller/test_internal_repay_partial.py
pytest tests/unitary/price_oracles/
```

Expected output (based on Round 02 baseline and the passing test suite at this commit): all 18 accounting regression end-to-end tests, 270 leverage-zap end-to-end tests, oracle and monetary policy tests pass. This could not be re-verified in this environment due to the missing toolchain (documented as a limitation in section 12).

No new failing test was produced, because no candidate survived to the PoC stage.

## 10. Financial impact and severity justification

No confirmed finding → no financial impact, no severity assigned.

Had any candidate been confirmed, severity would require: ordinary-user exploitability, required market/account state, exact transaction sequence, assets and amounts affected, maximum demonstrated loss, recoverability, repeatability, documentation status, current-version applicability, and program-scope eligibility. None of these could be established because no vulnerability was found.

Per the mission rules: "A rounding difference is not automatically a vulnerability. A revert or denial of service is not automatically a bounty-worthy issue. A rigorous NO-GO is better than a weak or misleading report."

## 11. Duplicate and known-fix checks

- **Round 01 (StableSwap-NG):** no confirmed vulnerability. Not duplicated.
- **Round 02 (LlamaLend V2):** 18/18 accounting regression E2E tests passed; 270/270 leverage-zap E2E tests passed; oracle and monetary policy tests passed. No HIGH/CRITICAL confirmed. F-1 (LOW/Informational) identified and already publicly fixed — explicitly not re-submitted.
- Previously investigated & rejected (Round 02) were treated as known history and not re-raised: MetaZap/StableSwap-NG assumptions, LP oracle/A-scaling, admin fee accounting, unmerged-branch available-balance behavior, oracle damping/cap (documented), monetary policy rate manipulation, factory initialization, LM double-counting, transient zap reentrancy without demonstrated exploit, AMM withdrawal rounding without measurable extraction.
- **Git history audit trail:** the codebase already contains numerous prior fixes (audit references CS-CURVE-LLV2-058 through CS-CURVE-LLV2-101, ChainlinkEMA audit follow-up PRs #248/#250, leverage-zap fix #219, OracleAndERC4626 fix #220, lm_callback fix #255). HEAD (`f574bd50`) includes all of them; none of the reviewed candidates is an unpatched regression of these.

## 12. Limitations and remaining uncertainty

- **Test execution not possible:** no Python/Boa/Vyper toolchain in the sandbox, so the existing test suite and any new PoC test could not be run. Static analysis was used throughout; dynamic confirmation of the passing baseline was not performed in this session.
- **Deployed-state verification bounded to on-chain deployment files:** Investigator C was limited to the five active Ethereum markets described in `deployments/llamalend/ethereum/markets/`; mainnet state was not queried (no mainnet/testnet transactions per safety rules).
- **Fork remote:** the workspace remote is `VectorShieldResearch/curve-stablecoin`, a fork of the primary `curvefi/curve-stablecoin`; HEAD matches the previously reviewed commit, so this does not affect the conclusion.
- **No new commits:** the repository has not advanced beyond the previously reviewed commit, so there is no post-review diff to re-examine.

## Conclusion

**Whether a real vulnerability was confirmed:** No.
**Whether it is novel and in scope:** N/A — none found.
**Strongest evidence:** exhaustive static review of `controller.vy` (repay/`_repay_partial`/`liquidate`/`tokens_to_liquidate`), both partial-repay and transient-leverage zaps, the AMM withdraw/deposit_range/`get_x_down`/`calc_swap_out` path, all deployed oracle adapter compositions, plus the expanded scope (LendController/LendFactory/Vault, AMM core, Configurator/ControllerFactory/OwnerProxy, PegKeeper/PegKeeperRegulator, monetary policies, FlashLender, LMCallback, Stablecoin). Every accounting path reconciles and every potential edge case is either intentional behavior, a non-bounty-worthy DOS/revert, or already fixed/documented.
**Whether a bounty submission is justified:** No. Decision: NO-GO.
**Exact next step:** None — do not submit a report. Re-prioritize to a different target or wait for a code change that advances beyond `f574bd50` before re-reviewing LlamaLend V2.
