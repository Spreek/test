# Swell hot-key compromise: incident report

Oct 22, 2026 · @Spreek&#32;

An unknown actor has controlled Swell's operational hot keys since 14:25 UTC on 21 Oct, including the swETH/rswETH oracle bot. Since 23:02 UTC they have raised the swETH and rswETH exchange rates by 1% an hour, the on-chain maximum, with no ETH behind the increase.

They also redirected rswETH's EigenLayer rewards to their own wallet. They deployed contracts that would sweep about 2,513 ETH from four Swell withdrawal and deposit contracts if an upgrade were ever approved. As of 01:20 UTC on 22 Oct, no upgrade is queued and no third-party venue has been drained.

Every fact below is on-chain and linked. That this is a key compromise rather than an insider is our inference.

## Rate markups so far

swETH is 3.0% above its last clean rate (1.133254 ETH) and rswETH 2.0% above (1.082464 ETH). Each step was the largest the contracts allow, taken within minutes of the hourly limit reopening. The next steps can land from 02:09 UTC (swETH) and 02:10 UTC (rswETH).

| UTC | Token | Rate, ETH per token | Step | Unbacked reserves, cumulative | Tx |
| --- | --- | --- | --- | --- | --- |
| 22 Oct 01:10 | rswETH | 1.093288 to 1.104221 | +1.00% | about 232 ETH | [0x2fbb4c74](https://etherscan.io/tx/0x2fbb4c74b2be98fb6972b1d43add83e8b3be972ca500bf7910dd35b5ef22dc6a) |
| 22 Oct 01:09 | swETH | 1.156030 to 1.167591 | +1.00% | 349.2 ETH | [0xf4fc0889](https://etherscan.io/tx/0xf4fc0889b742ebc816e2fc8ad064460c56fdea5648d64199b34bec05bf060e6e) |
| 22 Oct 00:04 | swETH | 1.144584 to 1.156030 | +1.00% | 231.7 ETH | [0x709bef86](https://etherscan.io/tx/0x709bef86097a992a7fc43522de34fe68ad0ff7225384336409cdb402624d4361) |
| 21 Oct 23:35 | rswETH | 1.082464 to 1.093288 | +1.00% | about 115 ETH | [0xe0d6433f](https://etherscan.io/tx/0xe0d6433fdfc8a7dbd1f8e107ef9cdb75ad059eccd6cc95874c91217b17280b42) |
| 21 Oct 23:02 | swETH | 1.133254 to 1.144584 | +1.00% | 115.4 ETH | [0x3f03c9af](https://etherscan.io/tx/0x3f03c9afa2e929dc272c1f16253810c35d71d018bac8e0137a276b47ec49a871) |

The DEX market has followed only about half of it: the Uniswap v3 swETH/WETH pool showed 1.1441 at 00:58 UTC.

## How it works

The bot key alone sets the reserves behind both tokens, and nothing on-chain checks the field it is inflating.

- **Fake reserves.** Each snapshot adds about 116 ETH to `transitioning`, a balance nobody can verify on-chain. The amounts are not multiples of 32 and come with zero fees. The swETH step at 00:04 is +1.0000% to 14 decimal places.
- **No outside check.** `ExternalV3ReservesPoROracle` on the swETH [RepricingOracle](https://etherscan.io/address/0x289d600447a74b952ad16f0bd53b8eaaac2d2d71) is 0x0, so the proof-of-reserve check is off. The reference rate the oracle compares against reuses the same bot-reported balances.
- **Stale beacon data.** The swETH snapshot slot has moved only 5 slots per snapshot since 01:33 UTC on 21 Oct. It is now about 23.5 hours behind.
- **Limits the actor works within.** swETH's `maximumRepriceDifferencePercentage` is 1% and `minimumRepriceTime` is 3,600 s, both unchanged since 2023.
- **Why Swell cannot simply mark it back down.** A rate decrease makes the oracle call `AccessControlManager.lockdown()`, which pauses everything, withdrawals included.
- **Redemptions are under the actor's control.** swEXIT pays the lower of the request-time and processing-time rate, and only the bot's snapshot processes requests. Arbitrage requests 32832 to 32835 (19.4 swETH) filed at the inflated rates have not been processed.

## Wallets involved

The actor's funds sit in one hub wallet and a few helpers. It took the gas left in 30 Swell hot keys, which shows the whole hot-key store is exposed, not just the oracle bot. No exchange, mixer or bridge appears in the inbound funding.

| Address | Who | What it did |
| --- | --- | --- |
| [0x9a475595b1e73e5f63b9eaefb46b6d78b8ce0f71](https://etherscan.io/address/0x9a475595b1e73e5f63b9eaefb46b6d78b8ce0f71) | Actor's hub wallet, first funded 21 Oct 23:03 | Collected gas swept from the hot keys on Ethereum (0.92 ETH), Arbitrum, HyperEVM and Base. Paid the bot's gas and is now the EigenLayer rewards claimer for 40 rswETH StakerProxies. Deposited 0.03 ETH and 1,730 USDC to Lighter, bridged USDC with Across and funded Hyperliquid accounts. |
| [0x1bda625d03c0c1b968e53b7450789e8be264b975](https://etherscan.io/address/0x1bda625d03c0c1b968e53b7450789e8be264b975) | Actor's deployer, given 0.1 ETH by the bot at 14:25 | Deployed the four sweep payloads below, plus [0x4aa19843d3359194e5cb729067b153fc8ff3295d](https://etherscan.io/address/0x4aa19843d3359194e5cb729067b153fc8ff3295d), a contract with sweep and drain functions. |
| [0xd184df90a137b237dc17db02dd4840300e46d9dc](https://etherscan.io/address/0xd184df90a137b237dc17db02dd4840300e46d9dc), [0xa3e7a9ffef733c3de17f3c4ebdfc65084cde4a84](https://etherscan.io/address/0xa3e7a9ffef733c3de17f3c4ebdfc65084cde4a84), [0xd39f0ec0c7bb54905006878cddecce262fa47dc7](https://etherscan.io/address/0xd39f0ec0c7bb54905006878cddecce262fa47dc7), [0xa4f1c1406d142d1c641c338086d7e09cb67e7819](https://etherscan.io/address/0xa4f1c1406d142d1c641c338086d7e09cb67e7819) | Upgrade payloads for swEXIT, the swETH DepositManager, rswEXIT and the rswETH DepositManager | Each records the live implementation (`prev()`) and adds `sweepETH()` to the sink. None is installed: implementation slots are unchanged. |
| [0xbe95afc3ee77e0ebe257357d2db791a6913b12da](https://etherscan.io/address/0xbe95afc3ee77e0ebe257357d2db791a6913b12da) | Sink | Hard-coded recipient in all four payloads. Empty and unused. |
| [0x78fad532b9f69e0c48cc6aa415a2b88f1c47e773](https://etherscan.io/address/0x78fad532b9f69e0c48cc6aa415a2b88f1c47e773) | Collection wallet | Received 37.07 HYPE swept from 15 keys on HyperEVM at 14:54, and 0.079 ETH from the bot at 23:03. Never spent. |
| [0x196f026dac96a43b405dbf364f6a8a4afc62e384](https://etherscan.io/address/0x196f026dac96a43b405dbf364f6a8a4afc62e384), [0xf693282ee2000a5fc1305f49308fd996d2315f6a](https://etherscan.io/address/0xf693282ee2000a5fc1305f49308fd996d2315f6a) | Funded by the bot at 14:26 (0.02 and 0.15 ETH) | 0xf693282e was swept back into the hub. 0x196f026d later received USDC from an actor helper on HyperCore. |
| [0x2481a87fbe9b48700283dbcf2a860c7ea1de5e56](https://arbiscan.io/address/0x2481a87fbe9b48700283dbcf2a860c7ea1de5e56), [0xe2bc03dbaaaacecb01cf2c34457f73de3e0a30f8](https://arbiscan.io/address/0xe2bc03dbaaaacecb01cf2c34457f73de3e0a30f8) | Helpers funded by the hub on Arbitrum | Small deposits to Hyperliquid. |
| [0xb29f1af8720e5fd23aa745559d1deaae3d9cda17](https://etherscan.io/address/0xb29f1af8720e5fd23aa745559d1deaae3d9cda17) | Swell oracle bot (compromised) | All five markups, 40 `setClaimerFor` calls, and the transfers that funded the actor's wallets. |
| 0xd469cc70c1b6d25b81eefd34200bf382af0fa096 | Swell hwHLP strategist on HyperEVM (compromised) | Our trace shows $392,891 moved out of the HLP vault into hwHLP sub-account 0x8a83f1f7ba4dede601b689f307cbe7f79f731628, where it still sits. |
| [0x3a5144e9239bfc3f4988d522dd60175042fd1043](https://arbiscan.io/address/0x3a5144e9239bfc3f4988d522dd60175042fd1043) | Swell AtomicSolverV3 solver (compromised) | Ran an Arbitrum `redeemSolve` and passed its 98.45 USDC spread to the hub. |
| [0xd94cd91e155a30b36ded4e021e8f696fd1cb5e3e](https://etherscan.io/address/0xd94cd91e155a30b36ded4e021e8f696fd1cb5e3e) | Swell Prime Alpha USD strategist (compromised) | Two $400 swap round trips inside the vault, which look like a probe. |

Every Swell key swept by the actor, for rotation. The chains column lists where we saw the sweep. HyperEVM-only keys have no role identified.

```csv
address,swept_on,role
0xb29f1af8720e5fd23aa745559d1deaae3d9cda17,eth,swETH/rswETH oracle bot
0x562b97212ce3913967df115f931b8577fbb14fbe,eth hyperevm base,hwUSD accountant updater
0x80c6421286d04ee66ade50cd9ad263727cee1caa,eth hyperevm base,hwUSD strategist
0x809311c6170d7e1285b28bfa152a464532bb3e1b,eth hyperevm base,hwUSD strategist
0x24df26a7d791365a01c3b03cab6ac6e819a533fd,eth hyperevm base,hwHLP and hwUSD strategist
0x305cd7f7d38c435a52794627ace0317cac9bf1b0,eth hyperevm,hwHLP accountant updater
0xf07f4e345be0f851500586bd0dabe992d8bfd304,eth,hwHLP strategist
0xd469cc70c1b6d25b81eefd34200bf382af0fa096,hyperevm,hwHLP strategist
0x3a5144e9239bfc3f4988d522dd60175042fd1043,eth arb,AtomicSolverV3 solver
0xd124372ffea78438c8ea798d25fb064589c86167,eth arb,Prime USD accountant updater
0xd94cd91e155a30b36ded4e021e8f696fd1cb5e3e,eth arb,Prime Alpha USD strategist
0xbfe49d970b8a8ffd97dbef292f3832091abf4f54,eth arb,Prime USD strategist and FeeSplitter
0xb7bb4aeb4387bf4fecce4a6fd53ff65dfa8c6d54,eth arb,Autonomous Prime USD strategist and CCTP relayer
0x1fb5c021b54bb68181232075d52ca909927e0776,eth arb,Test Vault accountant and RolesAuthority admin
0x691ddad682ec414da25718cf05366fa3586c98a4,eth,SubAccountWithMerkleVerification operator
0x9e175a78855ad69d002055d26060c1931a68c363,eth,Swell v1 SWNFTv3 rate key
0x1d4553dbe048ee8556f1549556abb55b344e516b,eth hyperevm base,idle ops key
0x5f518d95fef188519ac94e69ecc835b8033ddad7,eth,idle ops key
0x0d70f4b86592a3dd9cb467fd1754719cb31c3cdc,eth arb,idle ops key
0xe03dad8cab308d7324c86b0f0b8cd28c57712ed1,hyperevm,
0xdf0abbf428135a766fd6f36dc83f8acf7b2c0f0c,hyperevm,
0xdae1a2383586405fa28ba3022131ebc086636e89,hyperevm,
0xcb4961779b06317d51f08084a80b9d474c5d9c3b,hyperevm,
0xc06874427c79fde284eece0de21573256f88903e,hyperevm,
0x8f92010d76abcba30a8f644c3347d8dced9772a9,hyperevm,
0x83c5a8754b87b3f327d7f24a54ba66d8f9fa9596,hyperevm,
0x643bdd764053c96e43f9408f05da44ef8a323beb,hyperevm,
0x4734fb20fa04f4ae68ad34e30b9b98ec5742ed5a,hyperevm,
0x3d957d089a14edf9edfb74348f8d38dc56319628,hyperevm,
0x02f733d4dee640569e01e5527f51dc5ef3b93e67,hyperevm,
```

## Funds at risk

The largest exposure is about 2,513 ETH (about $6.2M at $2,476) in four Swell withdrawal and deposit contracts. Reaching it needs an upgrade through Swell's 7-day [Timelock](https://etherscan.io/address/0xba829003e6b5dd858fbed3945719939a6dc3073b), run by its 4/7 [Safe](https://etherscan.io/address/0x20fdf47509c5efc0e1101e3ce443691781c17f90). The Timelock's last event was 23 Sep, so nothing is queued.

| Pot | Size | How the actor could reach it | Status at 01:20 UTC |
| --- | --- | --- | --- |
| [swEXIT](https://etherscan.io/address/0x48c11b86807627af70a34662d4865cf854251663) | 1,324.5 ETH, owed to processed withdrawals | Install payload 0xd184df90 through the Timelock | Not queued |
| [rswEXIT](https://etherscan.io/address/0x58749c46ffe97e4d79508a2c781c440f4756f064) | 875.6 ETH, owed to processed withdrawals | Install payload 0xd39f0ec0 through the Timelock | Not queued |
| [swETH DepositManager](https://etherscan.io/address/0xb3d9cf8e163bbc840195a97e81f8a34e295b8f39) | 299.7 ETH free | Payload 0xa3e7a9ff, or today: redeem swETH at the inflated rate and process it with the bot key | No actor redemption yet |
| [rswETH DepositManager](https://etherscan.io/address/0x5e6342d8090665be14eeb8154c8a87b7249a4889) | 13.3 ETH free | Payload 0xa4f1c140, or redemption as above | No actor redemption yet |
| EigenLayer rewards for 40 rswETH StakerProxies | Not sized | Claimer already set to the hub wallet | Nothing claimed yet |
| hwHLP sub-account 0x8a83f1f7ba4dede601b689f307cbe7f79f731628 (HyperCore) | $392,891 USDC | Already moved there by a compromised strategist; whether it can leave is unverified | Still in place |
| [Morpho rswETH/msETH market](https://app.morpho.org/ethereum/market/0x0d55c325847ed87d53506c2aca7de046cb59d8c22928fd55fb2790c4811d20db) | About 34 ETH borrowable | Oracle reads raw `getRate()`, LLTV 94.5%. An outsider profits only after about 19% more inflation | No market events |

Other venues pricing off `getRate()` hold under 45 ETH between them: Curve rswETH pools, Gyro swETH pools, Euler and Ion. So far only small MEV backruns have hit them. Aave, Spark and Fluid list neither token.

## Suggested actions

The first two Swell steps stop the hourly markup and the redemption path, so they come before everything else.

**Swell**

1. Pause bot methods and withdrawals on both AccessControlManagers: [swETH 0x625087d7](https://etherscan.io/address/0x625087d72c762254a72cb22cc2eca40da6b95eac) and [rswETH 0x796592b2](https://etherscan.io/address/0x796592b2092f7e150c48643da19dd2f28be3333f). `submitSnapshotV2` reverts while bot methods are paused.
2. Revoke every role held by the oracle bot and by each key in the rotation list, then rotate the whole hot-key store. The sweep's fee pattern suggests the actor may be running on the bot's own host, so treat that host as compromised.
3. Reset the EigenLayer rewards claimer on the 40 rswETH StakerProxies from a clean key.
4. Safe signers on [0x20fdf475](https://etherscan.io/address/0x20fdf47509c5efc0e1101e3ce443691781c17f90) and [0xc80e9ffc](https://etherscan.io/address/0xc80e9ffcc3774f1fe5a925bda8f846fa4301eae8): reject anything that schedules an upgrade to the four payload addresses. The hot keys were funded from 0xc80e9ffc, so check signer devices too.
5. Secure the $392,891 in hwHLP sub-account 0x8a83f1f7ba4dede601b689f307cbe7f79f731628 and remove the compromised strategists from its Merkle permissions.
6. Publish the last clean rates, swETH 1.133254 and rswETH 1.082464, so integrators can pin them.

**EigenLayer**

1. Watch for reward claims by 0x9a475595b1e73e5f63b9eaefb46b6d78b8ce0f71 on behalf of the rswETH StakerProxies at the [RewardsCoordinator](https://etherscan.io/address/0x7750d328b314effa365a0402ccfd489b80b0adda), and hold them where possible.

**Integrators**

1. Morpho rswETH/msETH curators and allocators: stop new supply and borrows until the rate is fixed.
2. Anyone pricing swETH or rswETH with `getRate()` should pin the clean rates. This covers Curve and Gyro rate-provider pools, Swell earnETH's accountant, and consumers of RedStone's `rswETH_FUNDAMENTAL` feed.

**Venues and bridges (Lighter, Hyperliquid, Across)**

1. Flag the hub wallet 0x9a475595b1e73e5f63b9eaefb46b6d78b8ce0f71 and helpers 0x2481a87fbe9b48700283dbcf2a860c7ea1de5e56, 0xe2bc03dbaaaacecb01cf2c34457f73de3e0a30f8, 0x196f026dac96a43b405dbf364f6a8a4afc62e384 and 0x78fad532b9f69e0c48cc6aa415a2b88f1c47e773.
2. Bring in SEAL 911 to coordinate if Swell cannot be reached quickly.

## Timeline

The actor was active from 14:25 UTC on 21 Oct, more than eight hours before the first markup. All times are UTC.

| When | What happened | Tx |
| --- | --- | --- |
| 22 Oct 01:10 | rswETH marked up 1% | [0x2fbb4c74](https://etherscan.io/tx/0x2fbb4c74b2be98fb6972b1d43add83e8b3be972ca500bf7910dd35b5ef22dc6a) |
| 22 Oct 01:09 | swETH marked up 1% | [0xf4fc0889](https://etherscan.io/tx/0xf4fc0889b742ebc816e2fc8ad064460c56fdea5648d64199b34bec05bf060e6e) |
| 22 Oct 00:49 | Hub deploys [0x867d0baa31e50aaf9ae6c6276c9320229a90e1a3](https://etherscan.io/address/0x867d0baa31e50aaf9ae6c6276c9320229a90e1a3), a forwarder into the live swETH implementation | [0xd44a087c](https://etherscan.io/tx/0xd44a087c12276a829082549113b6ebe2dcaa8c997647cdd2eba964c211e62dd2) |
| 22 Oct 00:04 | swETH marked up 1%; pending arbitrage withdrawals left unprocessed | [0x709bef86](https://etherscan.io/tx/0x709bef86097a992a7fc43522de34fe68ad0ff7225384336409cdb402624d4361) |
| 21 Oct 23:35 to 23:45 | hwHLP strategist moves $392,891 from the HLP vault to sub-account 0x8a83f1f7 | 0xd08fee26 (HyperEVM) |
| 21 Oct 23:35 | rswETH marked up 1%, 62 minutes after its routine 22:33 reprice | [0xe0d6433f](https://etherscan.io/tx/0xe0d6433fdfc8a7dbd1f8e107ef9cdb75ad059eccd6cc95874c91217b17280b42) |
| 21 Oct 23:19 to 23:29 | Bot makes the hub the EigenLayer rewards claimer on 40 rswETH StakerProxies | [first](https://etherscan.io/tx/0x3e188daaf74ac966abcf65bb08ae965f567fbe0b8a029bef1d5c606d23056e7d), [last](https://etherscan.io/tx/0xb2ac2cc4ee69b86e6d26d524796de1682818b94eedfd79c9f2636c1491aebbd9) |
| 21 Oct 23:19 to 23:26 | Hub pays the bot's gas, then deposits 0.03 ETH and 1,730 USDC to Lighter | [0x3b91dd8e](https://etherscan.io/tx/0x3b91dd8e8267f87b60e84f65eaf55d7706470298df5c92fc89718d49ff4045b0), [0x21036063](https://etherscan.io/tx/0x21036063003c94da98d97f7a5b90fe3e3c58cc1d441224c521a6922e02b36d81) |
| 21 Oct 23:03 to 23:09 | Gas swept from 20 hot keys into the hub on Ethereum, then Arbitrum, HyperEVM and Base | [0x04cf1c2d](https://etherscan.io/tx/0x04cf1c2da5dd3287560e450c1fd840b70cc8e239e426955bf3a98e6231aae43c), [0x6fe5ba66](https://etherscan.io/tx/0x6fe5ba66ba0bd3c5da9aa9974ad05f956a2806682ba3b15805510f9bd0600b6e) |
| 21 Oct 23:03 | Bot sends 0.079 ETH to the collection wallet | [0x4cc14fe8](https://etherscan.io/tx/0x4cc14fe8d5b04eef55906d1714db96d3d254e1383e30ca9ca121494f4810abbb) |
| 21 Oct 22:59 to 23:03 | Bot mints 0.06 swETH, files a withdrawal at the old rate, marks swETH up 1% (23:02) and claims the ETH back | [0x51c42020](https://etherscan.io/tx/0x51c420201ba7cb922230bbb502d066341d883b03740fa8ad5edad747321fb415), [0x3f03c9af](https://etherscan.io/tx/0x3f03c9afa2e929dc272c1f16253810c35d71d018bac8e0137a276b47ec49a871), [0xba136bbf](https://etherscan.io/tx/0xba136bbf3c8832495c3d8aa5e824ac7a124d431d52c35079026bb47398a0db75) |
| 21 Oct 14:54 | First HyperEVM sweep: 37.07 HYPE from 15 keys to the collection wallet | 0x6af0bf55 (HyperEVM) |
| 21 Oct 14:28 | Deployer publishes the four sweep payloads | [deployer](https://etherscan.io/address/0x1bda625d03c0c1b968e53b7450789e8be264b975) |
| 21 Oct 14:25 | Bot funds the deployer (0.1 ETH) and two other fresh wallets | [0x1dd86d34](https://etherscan.io/tx/0x1dd86d3494fe58a688be82931924b7a27dd5f10cfdf33ae14c485e7262871f56) |
| 21 Oct 01:49 | Last normal swETH reprice at 1.133254. The 07:49, 13:49 and 19:49 runs never happened | [0x6dbcd3fb](https://etherscan.io/tx/0x6dbcd3fb0e2abfcdef90ed9c3207d11247599b271097bedb88e58351ab3f69bc) |

The full funding trace, with every swept-key transaction across four chains, is available on request.
