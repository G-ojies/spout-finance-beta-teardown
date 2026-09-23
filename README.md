# Spout Finance beta teardown: the 0% loan is a covered call nobody priced

**Hands-on review of beta.spout.finance on Solana devnet, 23 September 2026, plus a 13-day regression check against the P0s other testers reported on 10 September.**

Reviewer: Great Ojietohamen ([@G-ojies](https://github.com/G-ojies)). Tester number **253**, wallet `3qqWgYcbTpN3K3YbWep9SqyuZ82oGkeW9pncgEgyfe7o`, session 18:03 to 19:00 UTC (US market open throughout).
Submitted for the Superteam Earn "Spout Finance Product Feedback" bounty. Public thread: [x.com/Great_ojies/status/2102839537571434635](https://x.com/Great_ojies/status/2102839537571434635). Raw evidence is in [`evidence/`](evidence/).

---

## TL;DR

Spout's pitch is clean: post tokenized US stocks, borrow stablecoins at 0%, the protocol writes covered calls on the pool to pay lenders. The beta proves the plumbing (Alpaca custody, Stork pricing, a vault program that deposits, borrows and repays) but the product still tells the user four different stories about what the loan costs, and several of the safety claims in the docs are not what is deployed.

Ten things that matter most, in order:

| # | Finding | Severity | Status on 23 Sep |
|---|---|---|---|
| 0 | **Nothing has filled for 37 hours.** The last fill on the NVDA mint (`FulfillBuyOrderKycGated` + `MintKycGated` + `mintTo`) was 22 Sep 05:05 UTC. Since then 36 NVDA buy orders across two full US sessions, including my own at 18:31 UTC today, sit as unfilled `PlaceBuyOrder` while the UI told every buyer "Your purchase has been confirmed. You own 0.02 NVDA" and the operator key swept their USDC from escrow to treasury within three minutes. No shares means the Borrow page shows 0 for every asset, so borrowing is untestable through the app right now. | Critical (P0, live) | New, verified on-chain 23 Sep 18:45 UTC |
| 1 | "0% interest" is contradicted by the app's own `estBorrowCostBps` column, which is 0 for the three most volatile names (NVDA, SMCI, BSOL) and 86 bps for Goldman Sachs. Real cost of a 2σ weekly call program on NVDA is roughly 8 to 10% of collateral per year, which is 16 to 20% per borrowed dollar at 50% LTV. | Critical (model disclosure) | New analysis, live data |
| 2 | Docs say spAssets are Token-2022 with a KYC transfer hook. All 11 devnet mints are legacy SPL Token with no extensions. Freeze authority on every mint is the same hot key that upgrades the programs. | High (security claim) | Verified on-chain today |
| 3 | The beta is on **devnet**, not testnet. None of the four programs exist on testnet. Every wallet defaults to mainnet, so first-time users fail silently. | High (onboarding) | Verified on-chain today |
| 4 | AAPL is missing from `/api/market-data/instruments` (10 rows, not 11) and the bundle still ships a hardcoded AAPL fallback at $228 while AAPL trades far above that, so the ticket over-promises shares. My own $5 order: modal said 0.02 AAPL at $228.50, chain executed 0.0148 AAPL at $337.05. | High (money) | **Still broken** 13 days after first report, reproduced with sig `oHZuKS...WKqEt` |
| 5 | Every worked example in the docs is numerically corrupted: "$10m" renders as "0m", "$2,184" as ",184", because the prerender uses `String.replace` with a replacement containing `$1`/`$2`. Extra `<div id="root">` tags break the HTML. | Medium (trust) | **Still broken** |
| 6 | Borrow path is back: `BorrowStablecoin` and `RepayDebt` succeed on 22 and 23 Sep after program upgrades on 11 Sep 20:52 UTC. | Fixed at program level, untestable for me (no fills) | Verified on-chain |
| 7 | Assignment force-closes the loan. Docs: proceeds "first cover any outstanding debt". The persona who borrowed to pay rent has their loan repaid without consent the first week the stock rips. | High (product design) | Docs-verified |
| 8 | "Path B" collateral (xStocks, Ondo) cannot be productive: Spout does not hold those shares at Alpaca, so it cannot write calls on them. Either those 0% loans are subsidized by spAsset borrowers or Path B has to go. The app has no Path B flow. | High (economics) | Docs-verified, app-verified |
| 9 | 2.0x leverage = max LTV = entry Health Factor of 1.08 (4% buffer assets) to 1.25 (12.5% buffer). A 7.4% drop liquidates the "safest" collateral. The liquidation fee equals the buffer, so the safest assets carry the thinnest cushion and the cheapest liquidation for the protocol. | High (risk UX) | Docs-verified; app shows no per-asset threshold |
| 10 | One hot key (`7N31cE8B...`) is upgrade authority for orders and vault, freeze authority on all 11 mints, and signs admin migrations (latest `MigrateLpPoolState` 23 Sep 11:12 UTC). Fine for devnet, not for a mainnet holding user equity. | High (mainnet blocker) | Verified on-chain today |

---

## 1. What I tested and how

Environment: Solana devnet, browser wallet via Privy on Brave on Linux, funded from the Solana and Circle faucets. Session log with signatures:

| Step | Action | Result | Evidence |
|---|---|---|---|
| A1 | Beta gate (email + passcode) | Code accepted first try; card issued at 18:03 UTC | `02-card.png` |
| A2 | Card issued, tester number | NO. 253, card labelled "SOLANA TESTNET" | `02-card.png` |
| A3 | Connect wallet | KYC identity created and `SetVerified` by hot key `7N31cE8B` at 18:22:09 UTC, one second after connecting, with no verification step shown to me. Sig `27SNMFSnE2oqpM5tgmSyrgLnH8emhoz9fyrNi8jtPUtGRyXrJFUe2Cfo2PuKzdSqoiNhV6R54HPdi7yDadU6C51N` | `04-wallet-sheet-devnet.png` |
| A4 | Reload after login (session persistence) | Still broken: header reverts to "Connect" on every reload even though the Privy session persists; clicking Connect reconnects immediately, so the dead-button half of the 10 Sep report is fixed, the drop itself is not | `05-tour-step1.png` |
| B6 | Funding | Funded externally: 5 SOL from the Solana faucet at 18:31:09 UTC, 20 USDC from the Circle faucet at 18:23:07 UTC. The in-app "Claim Faucets" button was not re-tested. | |
| C9 | AAPL list price vs ticket | List and ticket both show **$228.50** with a 0.37%/yr borrow cost; neither value exists in the instruments API, so both are hardcoded fallbacks. AAPL is appended last in the list, after XOM (`16-sell-tab-zero-shares.png`) | `12-aapl-purchase-confirmed-228.png` |
| C10 | Buy $5 AAPL, 18:33:42 UTC | Modal: "You own 0.02 AAPL, Bought at $228.50". On-chain `PlaceBuyOrder` log: Stork price **$337.045**, staleness 167 s, order = 5 USDC for **0.014834809 AAPL**. Executed price 47.5% above the quote, 32% fewer shares than the ticket implied. No 0.20% fee deducted. | sig `oHZuKSrTMPmvh4vwZCPT92h86pyRg7MYAywYC7PBn9VDZBXJ3P8iAcsG96azbpE4L8rJjntvXGcPHsD6vXWKqEt` |
| C11 | Buy $5 NVDA, 18:31:58 UTC | Modal: "0.02 NVDA at $225.50". On-chain: Stork $225.59, staleness 182 s, 5 USDC for 0.022164103 NVDA. Log also prints `DEVNET: KYC verification bypassed (mock)`. USDC left my wallet to escrow `7YBM7UQi...` in the same transaction, before any fill. | sig `47MPZVx6ABwdHA3LhnRbv3dzFU5NVBiA7kHPtr77rRaNXcRuWbLcrBgh6Q4keqhBW389vjaxRa8wteFkwp2widbU` |
| C12 | Escrow sweep | My 5 USDC per order was moved from the escrow `ADZRYU7F...` to the treasury `Cr2A3ck6...` (owner `Ee7LLsoS...`, balance 47,105 devnet USDC after mine) at 18:34:34 and 18:36:06 UTC, signed by `7YBM7UQi...` with a durable nonce, before any shares were minted to me. | sigs `67FPDXX6EKqqpR6Z...`, `66xjoByTbv4HTwYe...` |
| C13 | Sell NVDA | Impossible: the Sell tab shows "Shares owned 0, Shares value ~$0.00" for both AAPL and NVDA, twenty minutes after the Buy modal said "You own 0.02 AAPL" (`16-sell-tab-zero-shares.png`) | |
| C14 | Cancel an open order | Impossible: Portfolio shows both orders as "Order received" with no Open Orders view and no cancel control anywhere (`14-portfolio-activity-order-received-no-cancel.png`). Times shown as 19:31 and 19:33 with no timezone (they are UTC+1 local). | |
| D18 | Deposit NVDA + borrow max | Blocked: no shares were ever minted (see row 0 of the TL;DR). Borrow panel with 0 collateral shows an LTV slider 0 to 50% and a Health Factor gauge from 1.00 to ∞ (`13-borrow-page-zero-shares.png`). | |
| D20 | Repay | Blocked by D18. | |
| D19 | Leverage slider | Still stuck at 1.0x: drag, click on 2.0x label and keyboard do nothing, no disabled state (`17-leverage-slider-stuck.png`) | |
| E23 | Ask Spout | "What is my liquidation threshold for NVDA?" returned the generic health-factor paragraph with no number (`18-ask-spout-liquidation-threshold.png`). "Are liquidations executed outside US market hours?" returned: "No. All liquidations occur during US market hours only (9:30 AM to 4:00 PM ET). The buffers for each asset are sized to absorb overnight and weekend price gaps. For digital-asset-linked ETFs like IBIT and BSOL ... the buffers are set higher to account for weekend exposure." (`19-ask-spout-liquidation-hours.png`) | |

Independent checks I ran from the outside (all reproducible with the public devnet RPC, see `evidence/onchain-checks-2026-09-23.md`): program existence on devnet vs testnet, upgrade authorities and upgrade timestamps, instruction names across all 338 vault transactions, the token program and authorities on all 11 spAsset mints, the unauthenticated instruments endpoint, and the served HTML of every docs page.

---

## 2. Product insight

### 2.1 Four surfaces, four prices for "free"

- Landing page and Borrow page: **0%, always**.
- Buy list, live today (screenshot `evidence/screenshots/03-buy-list-borrow-cost.png`, taken directly under the banner "0% Interest on borrowing. Always, no matter the market conditions") and the same values from `/api/market-data/instruments`: `estBorrowCostBps` of **0** (NVDA, SMCI, BSOL), 6 (GLD), 7 (PFE), 54 (GOOG), 55 (IBIT), 58 (MSTR), 80 (XOM), **86 (GS)**.
- Leverage tooltip: "higher leverage means more borrower cost".
- FAQ: assignment costs "around 0.5% annualized across the portfolio".

The column is inverted relative to reality. Covered-call cost is capped upside, and capped upside is largest on the names that move most. NVDA, SMCI and BSOL at 0 bps and Goldman at 86 bps is not a model output. It reads like a placeholder, and it is the number a user sees before committing collateral.

What the model actually implies: the docs' own tranche example needs $30k of weekly premium on a $10m pool, which at 50% LTV means roughly $20m of locked collateral and about 0.15% per week, or 7.8% per year, of gross premium. Weekly calls on large-cap tech that pay 0.15% are roughly 2 standard deviations out of the money. Historically, a 2σ weekly overwrite on NVDA gives up on the order of 8 to 10% of collateral value per year in capped upside, concentrated in a handful of weeks. Per borrowed dollar at 50% LTV that is 16 to 20%. Kamino, which holds about 83% of tokenized-stock lending on Solana, charges an explicit 5 to 9% USDC borrow APR on xStocks collateral. Spout is cheaper in flat weeks and much more expensive in the weeks a stockholder cares about.

None of this makes the product bad. It makes "0%" the wrong headline. **Recommendation:** replace the borrow-cost column with a per-asset "expected upside given up per year" derived from the live strike rule, show it on the ticket, and show the realized number per position after every cycle.

### 2.2 The personas in the docs are the wrong customers for the mechanism

- "The long-term bull who wants real leverage" is the worst possible customer for a systematic call overwrite. The strategy sells exactly the right tail they are holding for. Auto-roll rebuys at a higher price after assignment, so a bull who is right ends every strong week with fewer shares (the docs' own Carol scenario).
- "The holder who needs cash" gets deleveraged without consent: on assignment, proceeds "first cover any outstanding debt". The cash they borrowed to cover rent is effectively clawed back into the loan the first week the stock rips, and if they still need the cash they have to re-borrow after auto-roll.

**Recommendation:** an "on assignment, keep my loan open" setting that re-borrows against the rebought shares automatically, and honest persona copy: the ideal borrower is someone who expects flat-to-mildly-up markets and would write calls anyway.

### 2.3 Path B cannot exist as written

Docs "Getting Started" offers Path B: deposit xStocks or Ondo tokens as collateral with no Spout KYC. Spout can only write covered calls on shares it holds at Alpaca. Third-party tokens are backed at other custodians, so they generate no premium. Either those loans are paid for by spAsset borrowers' premium (cross-subsidy that the tranche math never mentions) or they are not 0%. The beta has no Path B flow at all. **Recommendation:** remove Path B from the docs until there is an in-kind conversion route (Ondo now offers in-kind conversion) that lands the shares at Alpaca.

### 2.4 Leverage puts users on the liquidation line by design

At 2.0x the app is buying with the maximum 50% LTV. Health Factor at entry is liquidation LTV divided by 50%: 1.08 for a 4% buffer asset, 1.18 for NVDA, 1.25 for a 12.5% buffer asset. The steadiest asset liquidates on a 7.4% drop; Goldman has moved more than that in a single earnings session. The liquidation fee equals the buffer, so the protocol earns least exactly where the cushion is thinnest, and the borrower gets the least warning. **Recommendation:** default the slider to 1.5x, show liquidation price and HF live on the buy ticket, and decouple the liquidation fee from the buffer.

### 2.5 Feature recommendations, ranked

1. **Cost transparency on the ticket:** strike distance, expiry, expected premium, expected upside given up, and the historical assignment frequency for that asset, before "Confirm".
2. **Cycle receipts:** after each Friday, show per position the strike written, whether it was assigned, premium credited, and shares before and after. Docs promise this; the app has no per-cycle view.
3. **Choose your overwrite ratio:** let borrowers pick 100% overlay (0% interest, QYLD-style) or 50% overlay with a small explicit APR (JEPI-style). Same engine, and it neutralises the "0% is a lie" critique.
4. **Keep-loan-open on assignment** (2.2).
5. **Network detection:** read the wallet's cluster, block mainnet wallets with a one-line fix, and rename "Testnet" to "Devnet" everywhere (card, tour, blog post).
6. **A faucet button that actually airdrops** SOL and test USDC, or remove the button.
7. **Tour matches the product:** cut the Senior/Junior steps until Earn ships.
8. **Unlock timer:** after repayment, show the date and time the collateral leaves the cycle.
9. **Mainnet governance:** Squads multisig plus timelock on upgrade, freeze and mint authorities, and verifiable builds with a published IDL.
10. **Publish the backtest** the docs lean on ("a loss large enough to reach Senior has not occurred in any historical scenario we have tested"), with basket weights and the strike rule.

---

## 3. DeFi and tokenization analysis

### 3.1 Custody: what "SIPC-covered" means for a wallet

The app shows "Held 1:1 at Alpaca Securities · Reserves 100.2%". Alpaca now custodies roughly 94% of all tokenized US equity backing, so Spout's counterparty risk is the whole sector's counterparty risk. SIPC protects the account holder at the broker, up to $500k. The account holder is Spout's issuing entity, not the wallet holding spNVDA. The FAQ's "SIPC-covered" wording implies per-user protection that does not exist. Say it plainly: shares are segregated at a FINRA broker; SIPC applies to the omnibus account.

### 3.2 Token control: docs vs chain

Docs (Security & Compliance): spAssets use Token-2022 with a transfer hook, "tokens cannot move to non-verified wallets". Devnet today: all 11 mints are legacy SPL Token, extensions empty, freeze authority `7N31cE8B...`. KYC is enforced at mint time (the onchainId program) and nowhere after. On my wallet the identity was created and marked verified by the operator hot key one second after I connected (sig `27SNMF...C51N`), with no verification flow, so the "Verified on-chain" badge in the beta means "the operator's key said so". Tokens move freely once issued. Whether mainnet will use Token-2022 is unknowable from the beta, which is exactly the thing a beta should let testers verify. A freeze authority on a hot key also means one leaked key can freeze every holder.

### 3.3 The loss waterfall has three versions

- Loss Waterfall page: Insurance Fund, then Junior, then Senior.
- "Who this is for" page: insurance fund, "then treasury, then pool socialization".
- Settlement Flow: assignment cost is netted against premium before distribution, which means lenders bear assignment cost, while the Assignment page and FAQ say the borrower bears it (shares sold at strike, fewer shares after auto-roll). Both cannot be true. If lenders bear it, Junior's 32% target is a fair-weather number. If borrowers bear it, the "0.5% annualized" figure needs a source.

### 3.4 Buffer sizing against a single-name gap

Insurance fund target is 2% of the lending pool. Pool is half the collateral. A 30% up-week in SMCI (it has had several) against a strike 9% out with 0.15% premium costs about 21% of the SMCI collateral. If SMCI is 10% of collateral, the loss is 2.1% of collateral, which is 4.2% of the pool: more than twice the insurance fund in one week, straight into Junior. The docs call this "has not occurred in any historical scenario". Publish the scenario set.

### 3.5 Weekly cycles and 24/7 tokens

spAssets trade around the clock on Solana. The underlying trades 6.5 hours a day. Liquidation requires selling shares at Alpaca, which can only happen in session. Stork keeps publishing off-hours, so the health factor can cross 1.00 on Sunday night with no way to act until Monday 09:30 ET. The docs say health-factor checks "continue at reduced frequency" off-hours and leave execution timing open; the in-app assistant told me flatly that "all liquidations occur during US market hours only" and that buffers are "sized to absorb overnight and weekend gaps" (`19-ask-spout-liquidation-hours.png`). With buffers of 4 to 12.5% on single names, a 10% earnings gap on the 4% buffer asset is bad debt for the pool by construction. State the rule and size buffers to overnight gap history, not intraday moves.

### 3.6 Regulatory posture

Spout is a FinCEN-registered MSB, not a broker-dealer. The margin-like credit is offered against securities to non-US persons, who also eat 30% dividend withholding. The SEC's 22 September carve-out for on-chain stock trading venues is a tailwind, but the beta does not show how W-8BEN, Reg S, or a US-person block will be handled. Superteam's audience is largely non-US; this is the first question they will ask.

### 3.7 Competitive position

Kamino already has $23 to 53m of tokenized-stock collateral at explicit rates with soft liquidations. Spout's edge is not "0%", it is that the collateral works for the lender. The honest framing: "borrow at 0% cash cost, pay in capped upside; here is the number". That is a product Kamino cannot copy without an options desk. The dishonest framing invites the exact teardowns this bounty has already produced.

---

## 4. UX feedback

Prioritised by how many users hit it before their first successful action. Rows marked with a screenshot were reproduced on 23 Sep; the rest are carried from earlier public testers and marked as such.

| Pri | Where | Issue | Fix |
|---|---|---|---|
| P0 | First buy | Wallet on mainnet, app on devnet: silent failure or `[object Object]` toast (earlier testers; not re-tested) | Detect cluster, show a blocking banner |
| P0 | Insufficient balance sheet | "Claim Faucets" does not fund the wallet (earlier testers; not re-tested) | Real airdrop or link out and say so |
| P0 | AAPL ticket | Quote from hardcoded $228 fallback, fill at oracle price; user gets fewer shares than promised | Remove fallback, error if the row is missing |
| P1 | Login | Reload drops the wallet connection while the Privy session persists (reproduced 23 Sep) | Rehydrate wallet list |
| P1 | Buy vs Sell state | Buy modal says "You own 0.02 AAPL"; Sell tab for the same wallet says "Shares owned 0" (`16-sell-tab-zero-shares.png`). Earlier testers also saw sell confirmations using buy copy; I could not sell to check. | One source of truth for holdings |
| P0 | Order confirmation | "Your purchase has been confirmed. You own 0.02 AAPL" appears the instant the order is signed (`09-nvda-purchase-confirmed.png`, `12-aapl-purchase-confirmed-228.png`), while the chain shows only `PlaceBuyOrder`, USDC moved to escrow then treasury, and no mint. The keeper has minted nothing since 22 Sep 05:05 UTC. The Borrow page then shows 0 shares for every asset (`13-borrow-page-zero-shares.png`). | Show "Order placed, waiting for fill" until the mint lands, alert ops when the keeper stalls |
| P1 | Borrow panel | Health Factor gauge runs 1.00 to ∞ with no per-asset threshold; Ask Spout cannot state the NVDA threshold either (`18-ask-spout-liquidation-threshold.png`); no liquidation price on the buy ticket | Use the docs formula, show threshold and liquidation price everywhere |
| P1 | Leverage slider | Unresponsive on 23 Sep too: no drag, no click, no disabled state (`17-leverage-slider-stuck.png`) | Make it work or hide it |
| P1 | Cancel order | On 23 Sep there is no cancel control at all: Activity shows "Order received" and nothing else, so 5 USDC per unfilled order is stuck with no exit (`14-portfolio-activity-order-received-no-cancel.png`) | Restore cancel and make it refund |
| P1 | Tour + Earn | Tour still pitches Senior/Junior; Earn is "coming soon" with a "Borrow at 0%" button that leads to a page where nothing can be borrowed (`15-earn-coming-soon.png`) | Trim the tour |
| P2 | Repay | No unlock date after full repayment | Show next cycle close |
| P2 | Portfolio | $0.00 / No Holdings for minutes after a fill | Poll balances after fill |
| P2 | Copy | "Your receive" (typo in the order summary), "0%interest" in tour step 5, every page titled "Buy \| Spout Finance" | |
| P2 | Wallet sheet | Address shown truncated (3qqWgY…fe7o) but a copy icon now exists; the sheet says "DEVNET TEST FUNDS" while the card, tour and blog say "Testnet" (`04-wallet-sheet-devnet.png`) | Pick one network name |
| P2 | Privy modal | The wallet line shows "US$571.00" for a devnet wallet holding 5 test SOL and 20 test USDC: devnet SOL is being priced at the mainnet SOL price | Hide fiat value on devnet |
| P2 | Amount input | `-5abc` becomes $5; `0.001` collapses the panel with no minimum shown (min buy is $2, min sell $1 in the bundle) | Validate and show minimums |
| P2 | Docs | Every $1x and $2x figure corrupted; `app.spout.finance` link in Getting Started dead; Fees says "no origination fee", Who-this-is-for says "one-time origination fee"; Lending page says "0% deposit/withdraw fees", Fees page says 0.20% withdrawal; landing says "no lockups on either side", Junior has a 45-day notice | Fix the prerender (use a replacer function), reconcile copy |
| P3 | Charts | Range buttons (1D/1W/1M/1Y) do nothing | |
| P3 | Sort | Label says Price or Newest, order is alphabetical with AAPL appended last | |

Positive: the insufficient-balance sheet is the clearest copy in the app, the wallet sheet's "Spout never takes custody" line is good, and the vault program's instruction set (InitializeVault, DepositCollateral, BorrowStablecoin, RepayDebt) is small and readable in Explorer.

---

## 5. Regression check: 10 September P0s, re-tested 23 September

Earlier public teardowns (WolfurX, cyberhooman, kaminariouji) reported five P0s on 10 September. Status today:

| P0 | Then | Now |
|---|---|---|
| Borrow API 500 (`CollateralType: unexpected length 213`) | Broken since 9 Sep migration | **Fixed at the program level** (upgraded 11 Sep 20:52 UTC; other wallets borrowed and repaid on 22 and 23 Sep). **Untestable for me**: with no fills since 22 Sep 05:05 UTC I hold no spAssets to deposit. |
| AAPL stale quote (instruments batch omits AAPL) | 28 to 31% share shortfall | **Still broken.** 10 rows returned today, fallback price 228 still in the bundle. My $5 order: modal said $228.50 and 0.02 AAPL, chain says $337.05 and 0.0148 AAPL (sig `oHZuKS...WKqEt`). |
| Order fills | Keeper filled in 5 to 11 s during market hours | **Regressed.** Zero fills on the NVDA mint since 22 Sep 05:05 UTC; 36 orders queued behind mine, all swept to treasury. |
| Cancel order never closes | 0 of 10 cancels since June | **Worse**: the cancel control is gone; orders show "Order received" with no action |
| Session lost on reload | Reproduced 5 times | **Still broken** (reproduced 23 Sep, tester 253) |
| Mainnet wallet silent failure | `[object Object]` toast | Not re-tested (wallet was on devnet throughout) |
| Docs `$1`/`$2` corruption | Reported | **Still broken** (checked every docs page today) |
| Leverage slider stuck | Reported | **Still broken** (`17-leverage-slider-stuck.png`) |

---

## 6. Appendix: reproduction notes

- **Devnet vs testnet:** `getAccountInfo` for each of `SPoRXgs...`, `spvaDgA...`, `SKYCVrk...`, `stork1JU...` returns an executable account on `api.devnet.solana.com` and `null` on `api.testnet.solana.com`.
- **Mint check:** `getAccountInfo` with `jsonParsed` on each of the 11 mints in the bundle: owner `Tokenkeg...`, `extensions: []`, `freezeAuthority: 7N31cE8B...`.
- **Docs bug:** `curl -s https://spout.finance/docs/lending-tranches | grep -c '<div id="root">'` returns 8. The source string "$10m" becomes `<div id="root">0m`, "$2,184" becomes `</div>,184`, which is the JavaScript `$1`/`$2` special replacement pattern. Fix: `html.replace(marker, () => rendered)`.
- **Instruments:** `curl -s https://beta.spout.finance/api/market-data/instruments | jq '.instruments | length'` returns 10, no AAPL, as of 23 Sep 12:45 UTC.
- **Borrow activity:** `getSignaturesForAddress` on the vault program, then `getTransaction` logs: `Instruction: BorrowStablecoin` on 22 Sep 23:xx UTC and 23 Sep 08:xx UTC, both succeeded.

Everything in this report was collected between 12:30 and 19:00 UTC on 23 September 2026 on Solana devnet with test tokens. No real funds were used. Not financial advice.
