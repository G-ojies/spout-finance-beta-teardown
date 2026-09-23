# On-chain checks, Solana devnet, 23 Sep 2026 (public RPC api.devnet.solana.com)

## Program IDs (from the beta bundle, PROGRAM_IDS)
| Name | Program | Upgrade authority | Last upgrade (UTC) |
|---|---|---|---|
| orders | SPoRXgsB4gWZmWPwyndoRWQrmZXKUc7o7oPMdkkGRcG | 7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp | 2026-09-11 20:52:53 |
| vault | spvaDgABYdpFKyatqo4Jr3nfwvVgWF5BbFxKFkzN3Am | 7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp | 2026-09-11 20:53:26 |
| onchainId (KYC) | SKYCVrkX3mQaHwZcLrUtuvzii43kma7kBUim5MaQm6k | BD29wQ5Tj7b1MEFqquRxASsTqoN38oCrjxpU4riTZx7C | 2026-06-22 15:39:16 |
| stork | stork1JUZMKYgjNagHiK2KdMmb42iTnYe9bYUCDUk8n | ECAM9u3xtSY2LZ1bsHRnx8PFWh4B7WVL1icNrk9MpZ8K | (slot 410333097) |

None of the four programs exist on Solana **testnet** (getAccountInfo returns null). They exist on **devnet** only.
The bundle defaults NEXT_PUBLIC_SOLANA_RPC to https://api.devnet.solana.com and uses the Circle devnet USDC mint 4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU.

## Vault program activity (all 338 transactions, 4 Aug to 23 Sep)
Instruction names seen in logs: InitializeVault, DepositCollateral, BorrowStablecoin, RepayDebt, MigrateLpPoolState.
Daily counts: spike of 74 on 9 Sep (admin migration), 22 on 14 Sep, 26 on 18 Sep, 4 on 23 Sep.
Successful BorrowStablecoin and RepayDebt transactions exist on 22 and 23 Sep, so the borrow path is live again after the 9 Sep migration breakage reported by earlier testers.
Newest vault tx on 23 Sep 11:12 UTC was an admin `MigrateLpPoolState` signed by 7N31cE8B (lending pool state is being prepared).

## spAsset mints (11, from the bundle)
All 11 are owned by the legacy SPL Token program (Tokenkeg...), decimals 9, extensions: none.
Freeze authority on every mint: 7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp (the same hot key that upgrades programs).
Mint authorities are per-mint PDAs. No Token-2022, therefore no transfer hook, contrary to docs/security-and-compliance.

Mints: 4Kfbn6ct.., 6AQa4pfP.., 9igwHtsP.., 9w3epdiz.., BMj7m9Jp.., Cr4GetXz.., E65hukXH.., FJwVgQpr.., Hm36ttF2.., HMTQEQBQ.., vtPfjmCo..

## /api/market-data/instruments (unauthenticated, 23 Sep 2026)
Returns 10 instruments. AAPL is absent. The client bundle still carries a hardcoded fallback for AAPL with price 228.
| Symbol | estBorrowCostBps | last close (22 Sep) |
|---|---|---|
| BSOL | 0 | 16.29 |
| GLD | 6 | 400.08 |
| GOOG | 54 | 347.28 |
| GS | 86 | 949.69 |
| IBIT | 55 | 48.82 |
| MSTR | 58 | 167.31 |
| NVDA | 0 | 228.85 |
| PFE | 7 | 27.93 |
| SMCI | 0 | 41.525 |
| XOM | 80 | 158.68 |

## Docs prerender bug (spout.finance/docs/*)
Every dollar figure beginning with $1 or $2 is corrupted in the served HTML because the prerender inserts the rendered page with a `String.replace` whose replacement string contains `$1`/`$2`, which JavaScript treats as capture-group references.
Evidence: "$10m" renders as `<div id="root">0m`, "$2,184" renders as `</div>,184`.
Extra `<div id="root">` opens per page: lending-tranches 8, liquidation-example 9, loss-waterfall 4, scenarios 3, insurance-fund 2 (should be 1).
