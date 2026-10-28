# Hifi DAO Proposal 21 Incident Brief

Oct 28, 2026 · @spreek

Proposal 21 is a hostile takeover. From 30 Oct 2026 21:39:47 UTC anyone can execute it: it sends all WETH collateral in the BalanceSheet and a fresh 25,000,000 HIFI to a one-owner Safe. It also lets anyone sweep the other \~$1M of collateral (mostly HC1) to that Safe, and nothing on-chain can cancel it. Two clocks matter now: a defensive proposal must be created before about 01:15 UTC on 29 Oct, and depositors must withdraw before execution.

## Deadlines

The first deadline is about three hours away as of 22:12 UTC on 28 Oct. Block times are estimated at 12.035 s from the last 50,000 blocks, so treat the 01:15 cutoff as a safe margin, not an exact time.

| Deadline (UTC) | What happens | Who must act |
| --- | --- | --- |
| Oct 29, 2026 \~01:15 (hard limit \~01:44) | Last moment to create a defensive proposal whose vote snapshot predates the 25M mint. After this, the attacker can vote any new proposal with \~28M HIFI | Team or any delegate with 50,000+ HIFI |
| Oct 30, 2026 \~05:10 (block 26,303,403) | Voting opens on proposal 22 (10B HIFI mint + timelock takeover) | All delegates: vote Against |
| Oct 30, 2026 21:39:47 | Proposal 21 becomes executable by anyone. All 8.79 WETH collateral and 25M new HIFI go to the attacker's Safe in one transaction | Depositors: withdraw collateral before this |
| Nov 4, 2026 \~05:30 (block 26,339,403) | Voting on proposal 22 closes | All delegates |
| Nov 13, 2026 21:39:47 | Proposal 21 expires if still unexecuted (14-day timelock grace period) | n/a |

## What proposal 21 does

Proposal 21 swaps the BalanceSheet's code for a version that sends all existing collateral to a hardcoded address, then mints 25M HIFI to that same address. It passed 2,945,598 For to 0 Against (quorum 2,839,677) and was queued on 28 Oct at 21:39 UTC. Its whole description reads: "Upgrade BalanceSheet implementation and distribute HIFI tokens".

| # | Target | Call | Effect |
| --- | --- | --- | --- |
| 0 | ProxyAdmin `0x025e…08c4` | `upgradeAndCall(BalanceSheet 0x4524…513b, BalanceSheetV2 0xb77b…584d, harvestIdle(WETH))` | Replaces the BalanceSheet code and immediately sweeps its WETH |
| 1 | HIFI `0x4b92…b07e` | `mint(0x1485…63fd, 25,000,000 HIFI)` | Adds 17.6% to the 141,983,852 supply, paid to the attacker's Safe |

**The flaw is in how "idle" is defined.** The proposed BalanceSheetV2 (verified on Etherscan, deployed by the proposer on 14 Oct) differs from the live implementation `0x783c…bf5e` only in the lines summarised here:

```text
harvestIdle(asset)                      -- callable by anyone
  idle = asset.balanceOf(BalanceSheet) - aggregateTracked[asset]
  send idle to DAO_OPS_WALLET           -- constant 0x14856a1E…63Fd = the mint recipient

aggregateTracked[asset]                 -- NEW storage: 0 for every asset at upgrade,
                                        -- raised only by deposits made after it
```

Because the tracker starts at zero, every deposit made before the upgrade counts as idle. The function's own comment claims "funds tracked through vault positions are never touched", which is false on day one. The upgrade calls `harvestIdle(WETH)` in the same transaction, and anyone can call it for every other collateral token afterwards.

The same group tried this twice before: proposal 18 (same upgrade, 50M mint to the same Safe, defeated) and proposal 20, whose text promised to "recover \~8.8 WETH of idle untracked balances" (defeated).

## What is at risk

Proposal 21 reaches all $1.02M of collateral in the BalanceSheet at Hifi's own oracle prices, not only the WETH. Almost all of it is the HC1 position that secures the protocol's one live loan.

| Asset | Held for | Amount | Value at Hifi oracle | Taken by |
| --- | --- | --- | --- | --- |
| HC1 (Horse Collateral 1) | Vault `0x546f…9f4a`, which owes 458,662 hUSDC | 1,000,000 | $1,000,000 (fixed $1.00) | Anyone calling `harvestIdle(HC1)` after execution |
| WETH | 16 vaults; 6.0 in `0xfca5…83cf`, 2.5605 in `0xbdf9…f105` | 8.793957 | $20,735 | Proposal 21 itself |
| SHEETp | 4 vaults | 17.156807 | $335 | `harvestIdle(SHEETp)` |
| SPARK | No vault (sent to the contract directly) | 50,005,899 | not priced | `harvestIdle(SPARK)` |
| HIFI | All holders | +25,000,000 (17.6%) | $257k market cap at $0.001828 | The mint in proposal 21 |

**The HC1 loss lands on the lender.** 461,178 of the 461,180 hUSDC outstanding (bond hUSC27Mar26, maturing 26 Mar 2027) sit in wallet `0xc4b6…b6e7`. Once the HC1 leaves the BalanceSheet, that loan can no longer be liquidated on-chain, so the lender is unsecured on-chain. The HC1 token cannot be frozen: its owner `0xfe9b…c03a` can mint and burn, nothing else. Its real backing is the off-chain agreement behind it.

**Out of reach of proposal 21 and of governance:** the 639,903 USDC in `0xc4b6…b6e7` (an ordinary wallet), $1,903 of USDC in matured hUSDC reserves (those bonds are owned by other wallets, not the timelock), $9 in the HifiPool, and about $24 of token approvals to the BalanceSheet.

**After a takeover:** the timelock also owns the price oracle `0xaff0…7efe` and the Fintroller. With the attacker's \~28M votes, a later proposal could set any price and mint unlimited HIFI. There is little extra cash in the protocol today for that to reach, so the further loss falls mainly on HIFI holders and future depositors.

**The reported $1.53M TVL, reconciled:** $1,021k collateral + $461k hUSDC outstanding + $1.9k bond reserves = about $1.48M on-chain. I could not place the remaining \~$45k.

## Simulations

Every call below was run with `eth_simulateV1` against Ethereum mainnet state at block \~26,294,100 (28 Oct, \~22:05 UTC), with the block time moved past the eta. The caller is an unrelated address, `0x0000…fa11d`, which shows that anyone can execute.

| # | Simulated call | Result |
| --- | --- | --- |
| 1 | `execute(21)` at eta − 60 s | Reverts: timelock not yet passed |
| 2 | `execute(21)` at eta + 60 s | Succeeds. BalanceSheet upgraded to `0xb77b…584d`; 8.793957 WETH to the Safe; 25,000,000 HIFI minted to the Safe |
| 3 | Next block: `harvestIdle` on SHEETp, HC1, SPARK | All succeed: 17.156807 SHEETp, 1,000,000 HC1 and 50,005,899 SPARK to the Safe |
| 4 | Next block: vault `0xfca5…83cf` withdraws its 6.0 WETH | Reverts (`SafeErc20LowLevelCall`): the WETH is gone |
| 5 | `execute(21)` with the BalanceSheet's WETH set to 0 | Reverts: `harvestIdle` finds nothing, so the whole proposal fails, mint included |
| 6 | Same, with the WETH set to 1 wei | Succeeds: anyone can re-arm execution by sending 1 wei of WETH |

Rows 5 and 6 matter for depositors: withdrawing every WETH blocks execution only until someone sends 1 wei. It protects the funds, not the mint. Rows 5 and 6 override WETH9's `balanceOf` storage (mapping slot 3) for the BalanceSheet.

## Who is behind it

The proposer, the Safe and all three voters act as one operator. It spent about 3 ETH on HIFI between 10 and 16 Oct and still holds every token. The voters were funded through Tornado Cash, and the For votes landed about six minutes before the vote closed. All times are UTC, newest first.

| When (2026) | What happened |
| --- | --- |
| 28 Oct 21:39 | Proposer `0xdf32…1543` queues proposal 21 |
| 28 Oct 21:30–21:31 | Proposer and three voters vote For within one minute: 113,599 + 956,600 + 940,311 + 935,087 = 2,945,598 HIFI vs quorum 2,839,677. No Against votes |
| 22 Oct 01:19 | Proposer creates proposal 21 (vote snapshot block 26,257,964) |
| 16 Oct 21:59–22:00 | All three voters self-delegate within the same minute |
| 16 Oct 08:57 | Each voter sends 0.1 ETH to `0xc609…b228`, all in the same minute |
| 15–16 Oct | Voters buy their HIFI in 8–9 Uniswap swaps each, 0.05–0.2 ETH per swap |
| 15 Oct 06:58–07:09 | Each voter (`0x16d4…7af8`, `0xd05b…809d`, `0x65d0…41ed`) receives 0.9947 ETH from the Tornado Cash 1 ETH pool `0x47ce…2936` |
| 15 Oct 02:33 | Proposer creates proposal 20 (same actions as 21); defeated with no votes |
| 14 Oct 08:00–08:01 | Proposer deploys BalanceSheetV2 `0xb77b…584d` and creates proposal 19 (cancelled next day) |
| 14 Oct 07:44–07:57 | Safe creator `0x1388…96c` sends the proposer 113,599.45 HIFI and 0.019 ETH: the proposer's first activity |
| 10 Oct 19:15–19:30 | Safe creator deploys Safe `0x1485…63fd` (one owner, `0x7808…9ec9`, never used) and creates proposal 18: same upgrade plus a 50M mint to that Safe. Defeated with 113,599 For |
| 10 Oct 18:53–18:54 | Safe creator receives 0.1 ETH from `0x689a…ce97` and buys 113,599.45 HIFI on Uniswap |

In July a different address, `0x99d5…a965`, proposed handing every admin role to itself (proposal 17). It was defeated with 24,270,829 Against. The wallets that voted Against then still hold about 13.5M votes today, but none of them voted on proposal 21. The largest, `0xc4b6…b6e7` (10.4M votes in July), now holds none. I found no on-chain link between the July address and this group.

## Proposal 22

Proposal 22 is a second takeover attempt presented as a defence. Its text says "the Protocol Manager has decided to temporarily lock governance", but its two actions hand control to its own proposer:

| # | Target | Call | Effect |
| --- | --- | --- | --- |
| 0 | Timelock `0xac46…47f7` | `setPendingAdmin(0xe107…cf6f)` | The proposer can then accept admin of the timelock, which controls every Hifi contract governance owns |
| 1 | HIFI `0x4b92…b07e` | `mint(0xe107…cf6f, 10,000,000,000 HIFI)` | About 70 times the current supply |

The proposer `0xe107…cf6f` was funded on 28 Oct at 08:59 UTC with 0.36 ETH from `0x3f3e…207b`, an address with 123,163 transactions (likely an exchange or swap service). At 09:13 it bought 54,400 HIFI, just over the 50,000 proposal threshold, and proposed at 09:15. I found no on-chain link to the proposal 21 group.

Voting runs from about 05:10 UTC on 30 Oct (block 26,303,403) to about 05:30 UTC on 4 Nov (block 26,339,403). The snapshot falls before the 25M mint. Even so, the proposal 21 group's 2.95M votes alone exceed the 2.84M quorum. If nobody votes Against, it passes.

## Response options

Nothing on-chain can stop proposal 21. The governor has no guardian. Only the proposer can cancel, or anyone once the proposer's votes fall below 50,000 HIFI, and the attacker holds 113,599. What can still be protected is the collateral, the timelock, and the vote on proposal 22. In deadline order:

1. **Before \~01:15 UTC on 29 Oct: create a defensive proposal.** It needs a proposer with at least 50,000 HIFI delegated.
   - Its vote snapshot (creation + 13,140 blocks) falls just before the earliest possible mint, so the attacker could vote only \~2.95M against the \~13.5M the July defenders hold.
   - It would execute around 6 Nov. The earliest follow-up using the 25M votes would execute around 8 Nov.
   - Candidate actions (the team's call): point the timelock's admin at a team multisig, or upgrade the governor to one with a cancel guardian; move the HIFI minter away from the timelock.
   - It cannot stop proposal 21, which executes about six days earlier.
2. **Before 21:39 UTC on 30 Oct: depositors withdraw.**
   - WETH: vault `0xfca5…83cf` (6.0 WETH, no debt, owner `0xe97f…64b8`) can withdraw now. Vault `0xbdf9…f105` (2.5605 WETH, owner `0xd10c…1146`) owes 2,507 hUSDC, so it must repay first or withdraw only the excess. The other 14 WETH vaults hold under 0.1 WETH each.
   - SHEETp: four vaults, 17.16 SHEETp in total.
   - HC1: vault `0x546f…9f4a` owes 458,662 hUSDC, so most of its HC1 cannot leave. The HC1 issuer and the lender `0xc4b6…b6e7` should agree now that any HC1 swept to the Safe is void and will be replaced. The HC1 owner can mint replacements.
3. **Now: publish a notice** on Hifi's channels (X, Discord, Medium, Tally). It should tell depositors to withdraw, warn HIFI holders about the 25M mint, and state whether proposal 22 is the team's.
4. **30 Oct 05:10 to 4 Nov 05:30 UTC: vote Against proposal 22.** It must be defeated whatever else happens. If it passed, it would be executable around 6 Nov, before the defensive proposal.
5. **Exchanges and incident responders:** flag the Safe, its owner, the proposer and the three voters (addresses below). HIFI trades mainly on Uniswap V4 (HIFI/USDC) and Mercado Bitcoin. SEAL 911 can coordinate.
6. **Attribution leads:** `0x689a…ce97` (funded the Safe creator), `0x3f3e…207b` (funded proposal 22's proposer; a high-volume service that may hold KYC), `0xc609…b228` (received 0.1 ETH from each voter), and the three 1 ETH Tornado Cash withdrawals on 15 Oct between 06:58 and 07:09 UTC.

## Addresses

All on Ethereum mainnet; each address links to Etherscan.

| Side | Role | Address |
| --- | --- | --- |
| Attacker | Recipient Safe (`DAO_OPS_WALLET`), one owner, never used | [0x14856a1ed89340de0478a5df8a18fa5638da63fd](https://etherscan.io/address/0x14856a1ed89340de0478a5df8a18fa5638da63fd) |
| Attacker | Safe owner (no transactions) | [0x7808e3474f4eea68e272b5fc3e208574be009ec9](https://etherscan.io/address/0x7808e3474f4eea68e272b5fc3e208574be009ec9) |
| Attacker | Proposer of 19, 20, 21 | [0xdf32c5c5f4465eded23bccf7a959565e466c1543](https://etherscan.io/address/0xdf32c5c5f4465eded23bccf7a959565e466c1543) |
| Attacker | Safe creator, proposer of 18 | [0x13883d12df822dcd66d272237fefbf668710b96c](https://etherscan.io/address/0x13883d12df822dcd66d272237fefbf668710b96c) |
| Attacker | Voter (956,600 HIFI) | [0x16d4d6a441d3b89cfa462a4417fdbefa8a7d7af8](https://etherscan.io/address/0x16d4d6a441d3b89cfa462a4417fdbefa8a7d7af8) |
| Attacker | Voter (940,311 HIFI) | [0xd05b3a76f42daa1c60e531d456ec6d47d9cc809d](https://etherscan.io/address/0xd05b3a76f42daa1c60e531d456ec6d47d9cc809d) |
| Attacker | Voter (935,087 HIFI) | [0x65d0c36abc673c6db8716ea53c1c1c61b1f641ed](https://etherscan.io/address/0x65d0c36abc673c6db8716ea53c1c1c61b1f641ed) |
| Attacker | Malicious BalanceSheetV2 (proposal 21) | [0xb77baf19d5f512beba4ac0353f29b6da6332584d](https://etherscan.io/address/0xb77baf19d5f512beba4ac0353f29b6da6332584d#code) |
| Attacker | Earlier BalanceSheetV2 (proposal 18) | [0x722979660d70d39fbd88460e270799e813098b3f](https://etherscan.io/address/0x722979660d70d39fbd88460e270799e813098b3f) |
| Lead | Funded the Safe creator | [0x689ab689fac6b073607416687c6618c32d71ce97](https://etherscan.io/address/0x689ab689fac6b073607416687c6618c32d71ce97) |
| Lead | Received 0.1 ETH from each voter | [0xc609d6eaa48d225bd4e328571404460867c2b228](https://etherscan.io/address/0xc609d6eaa48d225bd4e328571404460867c2b228) |
| Proposal 22 | Proposer | [0xe1073a50b831b4dcac5b62f2c88f26393384cf6f](https://etherscan.io/address/0xe1073a50b831b4dcac5b62f2c88f26393384cf6f) |
| Proposal 22 | Funder (high-volume service) | [0x3f3ee0a9cac2d01db44001eca3e8382fbe40207b](https://etherscan.io/address/0x3f3ee0a9cac2d01db44001eca3e8382fbe40207b) |
| Hifi | Governor (GovernorBravoDelegator) | [0xef0a0421ea43b602e5be35e9018dd3e34bcee007](https://etherscan.io/address/0xef0a0421ea43b602e5be35e9018dd3e34bcee007) |
| Hifi | Timelock (2-day delay) | [0xac46db50b44bbef8dc25f778359e1834248147f7](https://etherscan.io/address/0xac46db50b44bbef8dc25f778359e1834248147f7) |
| Hifi | ProxyAdmin | [0x025e32ad3491cc1cb8934c41b5059a216aff08c4](https://etherscan.io/address/0x025e32ad3491cc1cb8934c41b5059a216aff08c4) |
| Hifi | BalanceSheet proxy | [0x452467a37f7a0c1ea8432a52b8bbe3cc31e9513b](https://etherscan.io/address/0x452467a37f7a0c1ea8432a52b8bbe3cc31e9513b) |
| Hifi | BalanceSheet live implementation | [0x783c0b27871fcc9ce8b52cca392ee4dfa573bf5e](https://etherscan.io/address/0x783c0b27871fcc9ce8b52cca392ee4dfa573bf5e#code) |
| Hifi | HIFI token | [0x4b9278b94a1112cad404048903b8d343a810b07e](https://etherscan.io/address/0x4b9278b94a1112cad404048903b8d343a810b07e) |
| Hifi | Fintroller | [0xee46e4b672286392fcb5308a58dcdb8f0bfca8c3](https://etherscan.io/address/0xee46e4b672286392fcb5308a58dcdb8f0bfca8c3) |
| Hifi | Price oracle | [0xaff0bb2f5c6e773cee07e9465968f91ca12a7efe](https://etherscan.io/address/0xaff0bb2f5c6e773cee07e9465968f91ca12a7efe) |
| Hifi | Live bond hUSC27Mar26 | [0x6da6f7c9d4f96cf2b80c11051711668048cfe378](https://etherscan.io/address/0x6da6f7c9d4f96cf2b80c11051711668048cfe378) |
| Affected | WETH vault (6.0) / its owner | [0xfca5bb05dabe94ad77d652b07c1304997d7683cf](https://etherscan.io/address/0xfca5bb05dabe94ad77d652b07c1304997d7683cf) / [0xe97f2d37f9612b702d1a2eb5288bd62de95864b8](https://etherscan.io/address/0xe97f2d37f9612b702d1a2eb5288bd62de95864b8) |
| Affected | WETH vault (2.5605) / its owner | [0xbdf9679eb2a76b6cb4bc9b65f55c9f4734bbf105](https://etherscan.io/address/0xbdf9679eb2a76b6cb4bc9b65f55c9f4734bbf105) / [0xd10ce1caaabd338258392a7cf6c662fd09421146](https://etherscan.io/address/0xd10ce1caaabd338258392a7cf6c662fd09421146) |
| Affected | HC1 vault (borrower) | [0x546ffc2806dc67090f48d501f963b6f0487b9f4a](https://etherscan.io/address/0x546ffc2806dc67090f48d501f963b6f0487b9f4a) |
| Affected | hUSDC lender (461,178 hUSDC) | [0xc4b69c22ad426d6a51f15c5bf5a72733e8beb6e7](https://etherscan.io/address/0xc4b69c22ad426d6a51f15c5bf5a72733e8beb6e7) |
| Affected | HC1 token / its owner | [0xa4273c363dce9544ec838b0043ddc2060cea99d8](https://etherscan.io/address/0xa4273c363dce9544ec838b0043ddc2060cea99d8) / [0xfe9b25dadea53e63c93c1c497df8f1e3af33c03a](https://etherscan.io/address/0xfe9b25dadea53e63c93c1c497df8f1e3af33c03a) |

## Sources

Figures come from Ethereum mainnet reads (`eth_call`, `eth_getLogs`, `eth_simulateV1`) and the Etherscan API, taken at block 26,294,140 (28 Oct 2026, 22:12 UTC). The HIFI ($0.001828) and ETH ($2,358.66) prices are from CoinGecko the same evening; collateral values use Hifi's own oracle.

- [Proposal 21 queued](https://etherscan.io/tx/0x175a62f2f308c645d350dab8b187b69d4fe8ea4eef9f5800d2dd276d2977a71b)
- For votes on proposal 21: [proposer](https://etherscan.io/tx/0x38ca34b54248f904654bba2eccfc165dc819925ffe121ce848418ab969568cc3), [voter 1](https://etherscan.io/tx/0xb93c261c272740fa1c38362278c0287b1bf1b47844328d9f8cf22356aa312f00), [voter 2](https://etherscan.io/tx/0xd90e6736522acfc5f49a5221b1f1347e7a1d475b5ccb02e570ce9b45c8fa0077), [voter 3](https://etherscan.io/tx/0xf76df2a7e92af4d7eb64d781eaf9d4618a03edcc921bb33e7f56074a462e753e)
- [BalanceSheetV2 deployed by the proposer](https://etherscan.io/tx/0x8e12484c79e7c5f3dc7b602ee0e75b2447b41975724f461c36c7d2058181ad48) and its [verified source](https://etherscan.io/address/0xb77baf19d5f512beba4ac0353f29b6da6332584d#code)
- [Recipient Safe created by the proposal 18 proposer](https://etherscan.io/tx/0x4377e2576dd41f8c8f8bfe517e38d72fb0060e512c74b8b4143d9e06c570bcdc)
- [Proposal 19 cancelled](https://etherscan.io/tx/0xdb836ab22bed356925fa1c0c7239b7a51e38648002ddd21e5d9d644169dee92c)
- [Proposal 16 queued](https://etherscan.io/tx/0x23ba286f979548db5203d2b52f7ce8447e88069c6788e16febd3f6be2016dae3), the last queue before proposal 21
