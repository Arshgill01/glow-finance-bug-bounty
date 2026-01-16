# Glow Finance Bug Bounty Report

## Executive Summary
I have identified three significant vulnerabilities in the Glow Finance protocol, ranging from Critical to Medium severity. The most severe issue allows for a Denial of Service (DoS) on liquidations, potentially leading to protocol insolvency. The other issues involve bypasses of security constraints (`DENY_WITHDRAWALS` and `DENY_DEPOSITS`) intended to secure managed margin accounts.

## 1. Critical: Liquidation Denial of Service via Stale Oracle Updates

### Description
The protocol uses inconsistent staleness thresholds for Oracle updates versus Account Valuation.
*   `MAX_ORACLE_STALENESS`: **60 seconds** (Used to accept new prices).
*   `MAX_PRICE_QUOTE_AGE`: **30 seconds** (Used to validate prices during liquidation).

An attacker can submit an oracle update that is **40 seconds old** (valid for update) but too old for valuation. When `liquidate_begin` is called, the `valuation()` function detects the 40s age, marks the position as "Stale" (`ErrorCode::OutdatedPrice`), and `verify_unhealthy` subsequently returns `ErrorCode::StalePositions`. This error causes the liquidation transaction to revert.

By continuously submitting stale updates (e.g., every 30 seconds with 40s latency), an attacker can keep their account in a "Stale" state, preventing liquidation indefinitely while the asset price potentially crashes.

### Impact
*   **Direct funds at risk:** Protocol insolvency. Underwater accounts cannot be liquidated, leading to bad debt.
*   **Likelihood:** High. Easy to exploit if the attacker controls the timing of oracle updates (e.g., in a Pull Oracle model).

### Recommendation
Ensure the valuation staleness threshold is at least as permissive as the update threshold.
*   **Fix:** Increase `MAX_PRICE_QUOTE_AGE` to `60` seconds (matching `MAX_ORACLE_STALENESS`).

---

## 2. High: `DENY_WITHDRAWALS` Constraint Bypass

### Description
The `AccountConstraints::DENY_WITHDRAWALS` flag is designed to prevent margin account owners from withdrawing funds to their personal wallets (e.g., when the account is managed by a Vault).
*   The `withdraw` instruction (Pool -> Margin) correctly enforces this by forcing the destination to be the Margin Account's ATA.
*   However, the `transfer_deposit` instruction (Margin ATA -> Wallet) **does not check** `DENY_WITHDRAWALS`. It only checks `DENY_TRANSFERS`.

### Impact
*   **Constraint Bypass:** A user can withdraw funds from a restricted, vault-managed margin account by first moving them to the Margin ATA (allowed) and then to their wallet (via `transfer_deposit`), defeating the security control.

### Recommendation
Add a check for `DENY_WITHDRAWALS` in `transfer_deposit_handler` when the source is the Margin Account.

---

## 3. Medium: `DENY_DEPOSITS` Constraint Not Enforced

### Description
The `AccountConstraints::DENY_DEPOSITS` flag is documented to "Deny deposits by this margin account (e.g. into a vault or some other pool)".
However, the `margin_pool::deposit` instruction does not check this constraint on the signing `MarginAccount`.

### Impact
*   **Constraint Bypass:** A restricted margin account can still deposit funds into a Margin Pool, violating the constraint set by the manager/vault.

### Recommendation
Add a check for `DENY_DEPOSITS` in `margin_pool::deposit_handler` if the depositor is a Margin Account.
