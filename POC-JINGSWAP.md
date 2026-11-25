# PoC — strengthening claimed-sub2 I-1: single-key `jing-swap` is a proven drain, not just slippage

**Bounty:** mwvrpx3a633f201e8caa (Juice savings + trading safes, auth-helpers-v3, Jing router v6-1)
**Target claim:** claimed-sub2 §I-1 (Iskander, filed as **Info**) — `jing-swap` lets a single key swap the
whole balance at once, bypassing the threshold/cooldown model; Iskander measured a sandwich only on the
deep DLMM venue (victim −0.90%, attacker round trip **unprofitable**) and claimed no theft path.
**This PoC:** the stronger version is **PROVEN on a mainnet fork**. Routed through the thin Velar venue via
the unsigned split fields, the same single-key swap loses **86.9%** and a sandwich attacker extracts
**+89,967.63 STX net** — ≈90% of everything the safe sold. That is a theft path, i.e. a Low/Medium
candidate, not an Info.

---

## 1. The code path (single key → whole balance → attacker-chosen venue)

`repo/contracts/juice-trading.clar`, `jing-swap` (define-public at **L2088**):

- Auth is one key, either way:
  - a passkey signature over `build-jing-swap-hash` (L2129-2137), or
  - the admin path `(try! (is-authorized none))` (none branch of the `sig-auth` match, ~L2147).
- It **never** calls `would-exceed-stx-threshold` / `would-exceed-sbtc-threshold` (defined L583/L592;
  called by `stx-transfer` L665, the FT transfer L817, and staking L973 — the pending-op + cooldown flow).
  A plain transfer of the same size by the same key would be forced into a 2-key + cooldown pending op.
- It forwards every routing field verbatim to the router, e.g. for `"stx-sbtc"` (L2166-2174):

```clarity
(try! (as-contract? ((with-stx amount))
  (try! (contract-call? ROUTER-SBTC-STX swap-stx-for-sbtc amount jing-amount
    limit-price update fallback amm-amounts amm-mins min-out zest-route
  ))
))
```

`repo/contracts/juice-safe-auth-helpers-v3.clar`, `build-jing-swap-hash` (**L67-85**) — the signed
SIP-018 payload contains ONLY:

```
topic: "jing-swap", auth-id, action, amount, limit-price, min-out
```

**Unsigned** (chosen at send time by whoever constructs the call — and `min-out` itself is chosen by the
signing key): `jing-amount`, `update`, `fallback`, `amm-amounts`, `amm-mins`, `zest-route`.

Router side (`jing/contracts/sbtc-stx/swap-router-sbtc-stx-jing-v6-1.clar`): `swap-stx-for-sbtc`
(L519) asserts only that the split sums to `amount` (L545-550) and at the end that the total gained is
≥ `min-out`. Per-leg minimums come from the unsigned `amm-mins`, floored to u1 by `amm-floor` (L253).
With `amm-amounts = { velar: amount, others: 0 }`, `amm-mins = 0`, `min-out = 0`, the **entire balance**
executes on one thin venue with effectively no price protection.

## 2. Venue survey for the sBTC/STX pair (live mainnet state, 2026-11-25)

| Venue (router leg) | Contract | Depth / status |
|---|---|---|
| Jing core | markets-sbtc-stx-jing-v6 | limit-order leg; price-bounded by signed `limit-price`, and `jing-amount` can be set to 0 |
| Zest CLMM | clmm-pool-v1 pool u3 | live since 2026-11-24; not used here |
| DLMM | Bitflow dlmm pools stx-sbtc v1/v2/v3 | **dead for this pair**: router swap reverts `err u2001` (`dlmm-swap-router-v-1-2` ERR_NO_RESULT_DATA — no reachable liquidity), fork-verified |
| XYK | Bitflow `xyk-pool-sbtc-stx-v-1-1` (0.518 BTC / 118,181 STX) | **dead for the STX side**: `xyk-core-v-1-2 swap-y-for-x` reverts `err u5` (`token-stx-v-1-2` ERR_INVALID_PRINCIPAL_SIP_010 — its `transfer` asserts the recipient is a standard principal; the pool is a contract principal), fork-verified directly and via the router |
| **Velar** | `SP20X3DC5R091J8B6YPQT638J8NR1W83KN6TN5BJY.univ2-pool-v1_0_0-0070` | **live and thin: 184,000 STX / 80,456,736 sats (0.8046 BTC)** (live `/balances` read); constant-product; executes end-to-end with the safe's native STX |

The dead legs don't mitigate anything — they just revert. Velar is the live extraction path, and the
unsigned split lets the single key put 100% of the swap there.

## 3. Fork execution (the numbers)

Environment: `@stacks/clarinet-sdk` 4.0.0, mainnet fork via `[repl.remote_data]` against api.hiro.so
(fork at chain tip, 2026-11-25). Pattern per Iskander's accepted approach: **the live router called
directly with the exact arguments `jing-swap` forwards** — here the live
`swap-router-sbtc-stx-jing-v6` (see §5 for why v6, not v6-1). The "safe" is one simnet account selling
its whole STX balance of 100,000 STX; the attacker is a second account. All swaps use
`amm-amounts = { velar: amount, zest/dlmm/xyk: 0 }`, `amm-mins = 0`, `min-out = 0`, `jing-amount = 0`,
`fallback = none` — every one of those values is either unsigned or set by the single key itself.

### Baseline (no attack)

```
BASELINE: safe STX 100000000000000 -> 99900000000000 (sold 100000000000),
sBTC received (router out) = 28290216 sats = 0.28290216 BTC; sBTC bal = 28290216
```

Safe sells 100,000 STX → **28,290,216 sats**.

### Attack (sandwich on Velar, same victim call)

```
FRONT-RUN: attacker sold 400000000000 uSTX -> 55078460 sats (sBTC bal 0 -> 55078460)
VICTIM (attacked): safe sold 100000000000 uSTX -> 3706342 sats = 0.03706342 BTC; sBTC bal = 3706342
BACK-RUN: attacker sold 55078460 sats -> 489967634295 uSTX (router out);
          attacker STX 100000000000000 -> 100089967634295
ATTACKER NET (balance-measured): 89967634295 uSTX = 89967.634295 STX | victim received 3706342 sats
```

| | sBTC received by safe |
|---|---|
| Baseline | 28,290,216 sats (0.28290 BTC) |
| Sandwiched | 3,706,342 sats (0.03706 BTC) |
| **Victim loss vs baseline** | **−24,583,874 sats = −86.9%** |

**Attacker net profit: +89,967.63 STX** (balance-measured, net of all Velar fees, on 400,000 STX of
front-run capital) — equal to **89.97% of the safe's entire sold amount**. (The attacker's take exceeds
the victim's excess loss at spot because the back-run also extracts value the victim's own price impact
left in the pool — standard thin-pool sandwich accounting; all figures above are as executed.)

Cross-check (closed-form constant-product on the live reserves, fee 0.3–1.3%): victim attacked output
≈0.037 BTC, attacker net ≈ +90–95k STX — the fork execution matches the math.

## 4. Why this clears the bar I-1 didn't

- I-1's only measurement was the deep DLMM venue on a different pair (STX/USDCx), where the sandwich
  round trip lost ~1,402 STX. On the venue the unsigned split actually lets the key choose for
  sBTC/STX — Velar, 184k STX deep — the identical attack is worth **+89,968 STX**.
- Preconditions are exactly the ones I-1 already establishes: ONE compromised passkey (or admin) —
  the key class that for a plain transfer is forced into 2-key + cooldown — plus ordinary,
  recoverable front-run capital. No second key, no cooldown window in which the owner can veto,
  no threshold accounting anywhere in the path.
- The extraction is not hypothetical slippage: it is a completed, profitable round trip executed
  against live mainnet state, with the victim's loss and the attacker's gain both balance-measured.

Suggested severity: **Low–Medium** (single-key precondition keeps it out of High; ~87% of the swapped
balance extractable in one call is well past Info). Fix direction is the one I-1 already names: count
`amount` against the threshold on the passkey path, or sign the route split (`amm-amounts`, `fallback`)
and enforce a non-zero floor under `min-out` relative to a reference price.

## 5. Method notes / honest limits

- **v6 vs v6-1:** `swap-router-sbtc-stx-jing-v6-1` is **not yet deployed** (Hiro contract-interface
  check 2026-11-25: v6-1 → 404, v6 → 200), and neither is `juice-trading` itself — this bounty reviews
  pre-deploy code. The PoC therefore executes against the live v6 router. Per `v61.diff` (221 lines),
  the v6→v6-1 changes are confined to the **Zest leg's orientation/bounds**; the split-swap entry
  points, the split-sum assert, the per-leg min handling, and the XYK/VELAR/DLMM venue constants are
  unchanged (verified by diffing the local sources: `swap-stx-for-sbtc` signature and
  `XYK_CORE`/`XYK_POOL`/`VELAR_POOL` constants identical). The forwarded call `jing-swap` makes is the
  same call, with the same arguments, on both versions.
- **Direction proven:** `stx-sbtc` via Velar. The reverse direction and the other pairs were not
  executed; the mechanism (unsigned split + self-chosen `min-out` + no threshold check) is identical
  by code for all six actions.
- **Not attempted:** Zest leg with a crafted tick walk (unnecessary — Velar already proves the drain);
  deploying `juice-trading` itself in simnet (the accepted Iskander pattern — calling the live router
  with the exact forwarded arguments — was used instead, so the known clarinet 3.20
  `with-staking` deploy limit does not arise).
- Front-run size (400k STX) is illustrative, not minimal: the closed-form grid shows the round trip
  stays profitable across front-run sizes (10k–400k STX) and victim sizes (50k–250k STX) at every fee
  assumption tested (0.3% / 1.0% / 1.3%).

## 6. Reproduce

```
cd poc/
npm install
npx vitest run tests/velar-drain.test.ts   # baseline + attack, prints all figures above
npx vitest run tests/diag.test.ts          # XYK u5 and DLMM u2001 dead-leg evidence
```

Files: `poc/tests/velar-drain.test.ts` (the PoC), `poc/tests/diag.test.ts` + `diag2.test.ts`
(venue diagnostics), raw logs `poc/velar.log`, `poc/diag.log`, `poc/diag2.log`.
