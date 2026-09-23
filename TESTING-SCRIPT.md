# Hands-on testing script (60 to 90 minutes)

Deadline: **23 Sep 2026, 22:59 UTC** (Earn listing). Do this first: DM **@SpoutHelp** on Telegram for the beta email + passcode. Say you are a Superteam Earn participant and the deadline is tonight.

Use a fresh throwaway wallet (Phantom or Backpack). Switch the wallet to **Devnet** before you start (the app is devnet, not testnet). Fund it: 1 to 2 SOL from https://faucet.solana.com and 20 USDC from https://faucet.circle.com (select Solana devnet).

Keep a screenshots folder. Name files `01-gate.png`, `02-card.png`, ... Note the exact UTC time of every action. Paste every transaction signature into README.md where it says `[[FILL]]`.

## A. Entry (10 min)
1. Open https://beta.spout.finance. Screenshot the gate. Enter email + passcode. Note: does "Code accepted." appear? Any lag?
2. Card screen: enter a name, screenshot the card with your **user number**.
3. Log in (Privy). Note which wallet icon shows, whether the email OTP arrives quickly.
4. **Reload the page.** Does the header revert to "Connect" and go dead? (Earlier testers reported this as a P0 on 10 Sep.) Record fixed / not fixed.
5. Take the tour. Note the "0%interest" spacing typo (step 5) and whether steps 8 to 9 still describe Senior/Junior lending. Then open Earn: still "coming soon"? Screenshot.

## B. Wallet and funding (10 min)
6. Try to buy $100 of NVDA with an empty wallet. Click **Claim Faucets**. Does anything actually arrive in the wallet? (Reported non-functional.) Screenshot before/after, record wallet balance from the RPC.
7. Open the wallet sheet. Is the full address copyable, or only truncated? Screenshot.
8. Put the wallet on **Mainnet** deliberately, attempt a $2 buy, and record exactly what the UI shows (earlier testers saw a toast reading `[object Object]`). Then switch back to Devnet.

## C. Buy and sell (15 min)
9. Open **AAPL**. Compare the list price, the ticket price, and the chart tooltip. As of 23 Sep the instruments API returns 10 symbols and AAPL is missing, so the ticket likely still quotes a hardcoded $228. Screenshot all three numbers.
10. Buy **$5 AAPL** at 1.0x. Sign. Record: signature, time, quoted shares vs minted shares (check the token balance in the wallet or on Solana Explorer, devnet). Compute the gap.
11. Buy **$5 NVDA**. Same record. Note whether a fee line appears anywhere (docs say 0.20%).
12. If the market is closed when you test (it opens 13:30 UTC, closes 20:00 UTC), note whether the confirmation modal says "Your purchase has been confirmed" before the fill. Note when the USDC left your wallet vs when shares arrived.
13. Sell **half** of your NVDA. Does the confirmation modal say "purchase"/"Bought"/"Total paid" instead of sold/received? Screenshot.
14. Place a buy, then **Cancel** it in Open Orders. Does it actually cancel and refund? Check the escrow/refund on-chain. (Reported broken since June.)
15. Type `-5abc` and `0.001` in the amount box; record what happens.

## D. Borrow and repay (20 min)
16. Go to Borrow. Select NVDA collateral. Screenshot the panel: Available Borrow, Health Factor, Liquidation price, Est. borrower cost/yr, Auto buyback toggle.
17. Work out what the app's Health Factor means: at max borrow, does it show ~2.00 (collateral/debt) or ~1.18 (docs formula with the 58.8% threshold)? Screenshot the number at 25% and 50% LTV.
18. Borrow the **maximum** against your NVDA. Record signature. Check the vault program logs on Explorer (`DepositCollateral`, `BorrowStablecoin`).
19. Try the **leverage slider** on the Buy page (1.25x, 1.5x, 2.0x). Does it move? Keyboard? Screenshot.
20. Borrow more, then **Repay** part, then repay all. Record signatures. After full repayment, does the UI tell you when collateral unlocks (docs say next cycle close)? Is there any unlock date shown?
21. Does the app show the strike, expiry, or premium for the cycle your collateral is enrolled in? Record yes/no and screenshot.
22. Open Portfolio right after a fill. Does it show $0.00 / No Holdings for a few minutes while the balances endpoint already has the shares?

## E. Ask Spout (5 min)
23. Ask: "What is my liquidation threshold for NVDA?" and "Are liquidations executed outside US market hours?" Paste both answers verbatim.

## F. Final records for README.md
- User number, build hash (bottom of page or console), wallet address, date/time window (UTC).
- Every signature from steps 10, 11, 13, 14, 18, 20.
- Status per earlier-reported P0: session reload, mainnet silent failure, AAPL stale quote, cancel order, borrow API. Mark each **fixed / still broken / could not test**.
