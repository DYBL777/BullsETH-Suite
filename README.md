# BullsEth: 30-Draw Price Prediction Suite

**Base · BSC · Arbitrum One**

A 30-draw price prediction game where players predict the ETH, BNB, or BTC price at draw resolution. The closest prediction wins. No randomness. No external yield protocol. Pure USDC.

BullsEth is part of a new category of on-chain game where prize economics are governed autonomously rather than fixed at deployment. The Eternal Seed establishes a pot floor that rises through the active season and cannot be redistributed during play. A scheduled draw-30 bonus accumulation directs a percentage of every weekly pool toward the season finale, making it a designed economic event rather than a windfall.

The breathing engine adjusts prize rates in real time based on pot health. The OG endgame layer makes long-term commitment economically rational. We believe this constitutes a protocol architecture with no direct precedent in existing DeFi prize protocols. The DYBL Foundation welcomes being directed to prior art.

## The Suite

| Contract | Chain | Feed | USDC | Sequencer |
|---|---|---|---|---|
| `BullsEth` | Base Mainnet | ETH/USD `0x71041ddd...` | Native Circle `0x833589fC...` | `0xBCF85224...` |
| `BullsBNB` | BSC Mainnet | BNB/USD `0x0567F232...` | Binance-Peg `0x8AC76a51...` | address(0), no L2 sequencer on BSC |
| `BullsBTC` | Arbitrum One | BTC/USD (see Deployment Notes) | Native Circle `0xaf88d065...` | `0xFdB631F5...` |

All three contracts share identical game logic. Only the price feed, stablecoin address, sequencer feed, and brand names differ. See Deployment Notes for address verification requirements.

## How to Play

**1. Register during the pregame window.**
Pay a $10 USDC commitment deposit. This reserves your place and enables ticket purchases. No whitelist. First-come-first-served. The game starts when enough players have committed and a minimum time has elapsed.

**2. Submit your prediction every 72 hours.**
Before each draw closes, submit a price prediction: your best estimate of the ETH (or BNB or BTC) price in dollars and cents, for example $2,056.33. You can change your prediction until the draw lock window. If you miss a draw, an auto-pick is assigned based on the previous week's resolved price.

**3. The draw resolves.**
One keeper transaction reads the Chainlink Price Feed. All predictions are ranked by absolute distance from the resolved price. The closest 1% of entries win the T1 (jackpot) pool. The next 5% share T2. The next 6-9% share T3. Prizes are claimable immediately.

**4. Repeat for 30 draws.**
The season runs for 30 draws at 72-hour intervals, roughly 90 days. Prize pools grow as the breathing engine releases more of the pot in later draws. Draw 30 is the season finale with the largest prize pool.

**Want OG status?** Pay $600 USDC upfront before the season starts. OGs have a targeted return of their committed capital at season close, calibrated to the OG-to-player ratio. Lower OG concentration targets a 90% return. Higher concentration scales down to a 30% floor.

## Game Parameters

**Ticket price:** $10 USDC per draw
**OG upfront cost:** $600 USDC, with targeted return at season close
**Maximum draws:** 30
**Draw cycle:** 72 hours (~90-day season)
**Draw resolution:** Single keeper transaction via `resolveWeek()`. No VRF callback.

Prize tiers per draw:

- **T1 (Jackpot):** Closest 1% of entries
- **T2:** Next 5% of entries
- **T3:** Next 6 to 9% of entries, graduated by draw number

Draw 30 is the season finale. The accumulated surplus above the OG endgame obligation distributes in a single draw, which the breathing engine is designed to make the largest prize event of the season. A `DRAW30_PRIZE_RESERVE` of $5,000 USDC is maintained throughout the season as a minimum prize floor at draw 30, above and beyond the OG obligation.

## How Draws Resolve

Draws resolve via a single keeper transaction that reads the Chainlink Price Feed. The feed reading serves as both the oracle input and the deterministic basis for ranking predictions. No VRF. No callback. No pending request.

This means draw resolution has no external timing dependency beyond the feed itself. A single call reads the price, ranks every prediction by distance from that price, and closes the draw. To our knowledge, no prior on-chain prediction game has used a price feed as the sole winner arbiter in this way. The DYBL Foundation welcomes being directed to prior art.

## Protocol Primitives

### 1. The Breathing Engine

An autonomous closed-loop controller adjusts the prize rate every draw based on the pot's trajectory toward the endgame obligation. From draw 8 onward, a 24-iteration geometric binary search solver computes the maximum safe breath: the highest payout rate that leaves the pot solvent at draw 30 assuming zero future revenue. The solver uses an exponential moving average of ticket revenue as its forward estimate.

The engine eases the prize rate when the pot is healthy and tightens when it needs to conserve. The first deployed season includes owner override via timelocked rail adjustment functions.

### 2. The OG Endgame Layer

Upfront OGs commit $600 before the season starts. The breathing engine calibrates throughout the season toward returning their committed capital at draw 30, alongside whatever prizes they won during the game. The target return scales with the OG-to-player ratio at launch: 90% at 20% OG ratio or below, declining to a 30% floor at 100% OG ratio.

At draw 30, the holdback uses the 29-draw running-average OG ratio, the same accumulator `closeGame()` reads. The reserved amount and the final payout computation use identical state by design, eliminating the over-reserve-to-treasury path in normal deployments.

### 3. The Four-Step Dormancy Waterfall

If the game enters dormancy at any point, remaining funds distribute in strict priority order:

1. **OG principal first.** Committed capital returned to upfront OGs.
2. **Casual final-draw tickets second.** Most recent participants refunded their last draw cost.
3. **Pro-rata surplus third.** Remaining pot distributed proportionally to all recent participants, where the pot allows.
4. **Treasury last.** Any residual after all player obligations.

Players are protected in order of commitment. The treasury takes nothing ahead of players in any dormancy scenario.

### 4. The Pregame Registration System

No whitelist. No gas war. Players register during the pregame window in first-come-first-served order. Upfront OG slots are allocated by timestamp with a 72-hour decline window, giving any player time to review before committing. A commitment payment is required before ticket purchases begin, creating a meaningful entry signal and reducing low-cost spam registrations. The pregame closes when the minimum player threshold is reached and a minimum elapsed time has passed. Any player can trigger the game start once those conditions are met.

### 5. The Geometric Binary Search Solver

`_solveGeometricBps()` runs 24 iterations of binary search over the breath space to find the maximum rate that keeps the pot solvent forward-projecting to draw 30 with no assumed future revenue. The forward simulation in `_simGeomPot()` compounds the net pot drain over remaining draws. The solver emits a `SolverDistressSignal` event if the pot is already below the solvency floor, supporting external monitoring without blocking the draw.

### 6. The Running-Average Holdback

The draw-30 holdback is computed from the 29-draw `ogRatioBpsAccumulator` running average rather than a fixed ceiling. A guard in `_finalizeWeekCore()` excludes draw 30's ratio from the accumulator so the holdback and `closeGame()` read the same 29-draw state by design. This is documented in the contracts as the single-source-of-truth mechanism.

## Keeper and Automation Role

Draws advance via Chainlink Automation. The keeper calls `checkUpkeep()` to detect when a draw is ready to resolve, then calls `performUpkeep()` to execute. All batch processing functions are permissionless as a fallback: if the keeper is unavailable, any address can call the batch functions to keep the game moving.

The keeper has no special privileges. It cannot access funds, change parameters, or influence outcomes. It is a scheduler, not a privileged actor.

## Architecture

```
Player prediction (price in cents)
        |
resolveWeek() reads Chainlink Price Feed (latestRoundData)
        |
_matchAndCategorize() assigns T1 / T2 / T3 tier
        |
_distributePrizesCore() computes per-winner allocation
        |
_calculatePrizePools() runs breathing engine via _checkAutoAdjust()
        |
_finalizeWeekCore() checks draw 30, updates accumulator (draws 1-29 only)
```

No VRF. No external yield protocol. No upgradeability. No proxy. No admin key on player funds.

## Timeframe Adaptability

The 30-draw sprint is a design choice, not a constraint. `TOTAL_DRAWS` is the only parameter governing season length. The breathing engine, OG endgame, dormancy waterfall, and prize structure scale automatically. The same codebase supports:

- 30-draw sprint (this contract, roughly 90 days)
- 52-draw season (the NearestTheETH 1Y parent, roughly one year)
- Custom cadence, any TOTAL_DRAWS with any DRAW_COOLDOWN

Ticket price, OG upfront cost, and player capacity are also constructor parameters. A team deploying this can configure `TICKET_PRICE`, `OG_UPFRONT_COST`, `MAX_PLAYERS`, and `MIN_PLAYERS_TO_START` for their target market without changing any game logic.

## Risks and Known Limitations

**Smart contract risk.** This contract has not yet been professionally audited. Internal review has resolved 40+ findings but is not a substitute for an independent security audit. The DYBL Foundation is actively seeking professional audit engagement. Do not deploy on mainnet before professional audit completion.

**BTC/USD feed address.** Two candidate addresses for the BTC/USD feed on Arbitrum were identified during development. Neither has been independently confirmed. The BullsBTC contract must have this verified at docs.chain.link before deployment. See Deployment Notes.

**Stablecoin risk.** BullsBNB uses Binance-Peg USDC, which is a Binance-bridged stablecoin and not native Circle USDC. Native Circle USDC is not available on BSC via CCTP. Players and deployers should understand this distinction.

**Chainlink dependency.** Feed staleness, sequencer downtime, and circuit-breaker bounds are all guarded in the contract. However, prolonged oracle outages or feed deprecations are external risks. If the game stalls or becomes unviable, the owner can propose dormancy via a 24-hour timelock. Once activated, all remaining funds distribute to participants according to the four-step waterfall.

**OG return is targeted, not guaranteed.** The breathing engine strives toward the targeted OG return. If the pot falls below the required floor, `closeGame()` pays pro-rata from what is available and emits an `EndgameShortfall` event. OGs may receive less than the target in adverse scenarios.

**This is experimental DeFi.** No false promises.

## Audit Status

| Metric | Value |
|---|---|
| Internal audit passes | 12 cold-read passes on the BullsEth 30-draw implementation (v2.21-v2.32), building on extensive prior internal audit work across the Pick432 and NearestTheETH lineage |
| Findings resolved | 40+ across all severities |
| Open C/H/M at current version | 0 |
| NatSpec coverage | All external and public functions |
| Current version | BullsEth v2.32 / BullsBNB v1.04 / BullsBTC v1.04 |
| Professional audit | Pending |
| Mainnet | Post-audit |

The internal audit methodology follows a build, attack, harden, and document cycle. Every finding is recorded with severity, description, and resolution version in the master changelog. Internal review is not a substitute for professional audit.

## Chainlink Dependencies

| Dependency | Usage |
|---|---|
| `AggregatorV3Interface` | Single price feed per contract. Deterministic draw resolution and winner determination. |
| `AggregatorMinMax` | Circuit-breaker bounds check on the price feed per draw. |
| L2 Sequencer Uptime Feed | Active on BullsEth (Base) and BullsBTC (Arbitrum). Passed as address(0) on BullsBNB, BSC has no sequencer. |
| Chainlink Automation | `checkUpkeep` / `performUpkeep` for keeper-driven draw progression and auto-pick fallback. |

`FEED_STALENESS = 25 hours` (immutable constant). `SEQUENCER_GRACE_PERIOD = 1 hour`. Feed timelock: `TIMELOCK_DELAY = 7 days`, executed by owner. Governance timelocks (breath rails, prize rate): `PRIZE_RATE_TIMELOCK = 48 hours`.

## Security Properties

Owner controls are limited to three categories: treasury withdrawal (capped and gated on pot health), timelocked feed changes (`TIMELOCK_DELAY = 7 days`, owner executes via `executeFeedChange()`), and timelocked governance parameter changes (breath and prize rate adjustments use `PRIZE_RATE_TIMELOCK = 48 hours`).

The owner cannot withdraw from `prizePot`, change immutable constants, stop draws from proceeding, or prevent emergency recovery. `renounceOwnership()` is blocked: a 6-year game cannot become headless.

Key protections throughout: `nonReentrant` on all value-moving functions, CEI ordering, two-step ownership with 7-day expiry on pending transfers, permissionless emergency sweep after 180 days post-settlement, and the dormancy waterfall if the game stalls for any reason.

## Lineage

```
Lettery S1 (2025)
  Eternal Seed primitive, VRF lottery, Aave yield, streak mechanics

Crypto42 1Y (public repo: DYBL-Crypto42)
  42-asset performance prediction, breathing engine, OG layer

Pick432 1Y (Arbitrum, canonical reference)
  Ranked 4-of-32 prediction, Chainlink feeds as arbiter, 52-draw season

NearestTheETH 1Y v1.86 (Base Mainnet, reference only)
  Price proximity mechanic, 52-draw, upfront OG structure

BullsEth v2.32 / BullsBNB v1.04 / BullsBTC v1.04
  30-draw sprint, pure USDC, multi-chain
```

Pick432 1Y carries the full lineage document from Crypto42 through the current release. NearestTheETH 1Y v1.86 is the direct parent of BullsEth and is published as a reference contract, not for deployment.

## Protocol Primitives: Research and Partnership

The DYBL Foundation is actively seeking technical partners and researchers interested in these mechanisms, particularly the breathing engine and OG endgame layer.

The breathing engine is a working implementation with 12 internal audit passes behind it. The DYBL Foundation considers the breathing engine an active area of research and welcomes expert engagement from those with background in geometric solvers, closed-loop control theory, or mechanism design.

Builders interested in the dormancy waterfall, pregame registration system, or other primitives described in this README are welcome to reach out. Licensing arrangements will be considered after external audit acceptance.

- **Breathing Engine:** autonomous geometric solver for prize rate calibration. Under active review.
- **OG Endgame Layer:** committed-capital targeted return with running-average holdback.
- **Dormancy Waterfall:** four-step ordered-priority emergency distribution.
- **Pregame Registration System:** fair FCFS OG slot allocation with commitment signal.

## Protocol Family

| Contract | Chain | Mechanic | Status |
|---|---|---|---|
| Lettery S1 | Base | VRF lottery, Eternal Seed | Internal audit complete |
| Pick432 1Y | Arbitrum | Ranked 4-of-32 prediction | Internal audit complete |
| NearestTheETH 1Y | Base Mainnet | Price proximity, 52-draw | Reference only |
| BullsEth | Base | Price proximity, 30-draw | Internal audit complete, professional audit pending |
| BullsBNB | BSC | Price proximity, 30-draw | Internal audit complete |
| BullsBTC | Arbitrum | Price proximity, 30-draw | Internal audit complete |
| Weather32 | TBD | Temperature prediction, Chainlink Functions | Internal audit complete |
| Lettery 26 | TBD | 26-character alphabet variant | In development |

## Deployment Notes

**BullsEth (Base Mainnet)**
- ETH/USD feed: `0x71041dddad3595F9CEd3dCCFBe3D1F4b0a16Bb70` (confirmed, 8 dec)
- USDC: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` (native Circle)
- Sequencer: `0xBCF85224fc0756B9Fa45aA7892530B47e10b6433`

**BullsBNB (BSC Mainnet)**
- BNB/USD feed: `0x0567F2323251f0Aab15c8dFb1967E4e8A7D42aeE` (confirmed, 8 dec)
- USDC: `0x8AC76a51cc950d9822D68b83fE1Ad97B32Cd580d` (Binance-Peg, not native Circle CCTP)
- Sequencer: address(0), BSC has no L2 sequencer

**BullsBTC (Arbitrum One)**
- BTC/USD feed: address unconfirmed. Two candidates identified. Verify at docs.chain.link before deployment.
- USDC: `0xaf88d065e77c8cC2239327C5EDb3A432268e5831` (native Circle)
- Sequencer: `0xFdB631F5EE196F0ed6FAa767959853A9F217697D`

## Repository Contents

| File | Description |
|---|---|
| `BullsEth_v2.32.sol` | ETH/USD on Base Mainnet |
| `BullsBNB_v1.04.sol` | BNB/USD on BSC Mainnet |
| `BullsBTC_v1.04.sol` | BTC/USD on Arbitrum One |
| `BULLSETH_MASTER_CHANGELOG.md` | Full version history v2.21-v2.32 with all internal audit findings |

## Contact

DYBL Foundation, Scarborough, England

Auditors, grant reviewers, and builders interested in the protocol or its primitives are welcome to reach out. Questions on any aspect of the design, audit history, or development methodology are welcome.

| Channel | Address |
|---|---|
| Email | dybl7@proton.me |
| Discord | dybl777 |
| X | @DYBL77 |
| Farcaster | @dybl |

## License

BUSL-1.1. Change Date: 24 February 2030. On the Change Date, this code becomes available under the MIT License.

*Developed through extensive iterative design, behavioral economics principles, and over 1,000 hours of internal security review between late 2025 and mid-2026. No coin. No VC. No governance. No proxies. No upgrades. No admin key on player funds.*
