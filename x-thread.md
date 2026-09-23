# X thread draft (post after README is filled and the repo is public; tag @SpoutFi)

1/
Spent today inside the @SpoutFi beta (Solana devnet, tester no. 253): bought two stocks, read every docs page and every order on-chain.

The plumbing is real. The keeper is asleep. The "0% interest" headline is not. Full teardown with signatures: https://github.com/G-ojies/spout-finance-beta-teardown

2/
The app's own borrow-cost column, live today: NVDA 0 bps, SMCI 0 bps, BSOL 0 bps, Goldman Sachs 86 bps.

Covered-call cost is capped upside. Capped upside is largest on the names that move most. This column is upside down.

3/
What the docs' own tranche example implies: ~0.15% weekly premium on collateral, which is a ~2σ weekly call.

A 2σ overwrite on NVDA historically gives up ~8 to 10% of collateral per year. At 50% LTV that is 16 to 20% per borrowed dollar. Kamino charges 5 to 9%, explicitly.

4/
The persona problem. The docs' first customer is "the long-term bull who wants leverage". A systematic call overwrite sells exactly the tail they are holding for, and auto-roll rebuys higher after every assignment. Right about the stock, fewer shares each strong week.

5/
Docs: spAssets are Token-2022 with a KYC transfer hook, "tokens cannot move to non-verified wallets".

Devnet today: all 11 mints are legacy SPL Token, zero extensions, freeze authority on the same hot key that upgrades the programs. KYC gates the mint and nothing after it.

6/
The beta is called "Solana Testnet" on the card and in the launch post. None of the four programs exist on testnet. They are on devnet, and the wallet sheet inside the same app says "DEVNET TEST FUNDS". Pick one.

7/
Still broken 13 days after other testers flagged it: AAPL is missing from the instruments API and the bundle ships a hardcoded $228.50. My $5 order: modal said "You own 0.02 AAPL at $228.50". Chain says Stork price $337.05, 0.0148 AAPL, oracle 167 s stale. Sig oHZuKS…WKqEt

8/
The bigger one: nothing has filled for 37 hours. Last NVDA mint was 22 Sep 05:05 UTC. Since then 36 buy orders, mine included, sit unfilled while the UI says "purchase confirmed" and the operator key sweeps the USDC to treasury in 3 minutes. No shares, so nobody can borrow.

9/
Small one, but it says something about QA: every dollar figure starting with $1 or $2 in the docs is corrupted ("In a 0m pool", "NVDA falls to 02"). Cause: String.replace with "$1" in the replacement string. Every worked example on the site is unreadable in the served HTML.

10/
Three things I would ship before mainnet:
• the real "upside given up per year" per asset, on the ticket, not 0%
• a "keep my loan open on assignment" setting, because today assignment force-repays you
• Squads multisig + timelock on upgrade, freeze and mint authorities

11/
The honest pitch is stronger than the marketing one: "0% cash cost, you pay in capped upside, here is the number." That is a product Kamino cannot copy without an options desk. Full report + evidence: https://github.com/G-ojies/spout-finance-beta-teardown
