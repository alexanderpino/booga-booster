# RollerCoaster Tycoon 1/2 gameplay mechanics - research for a tycoon on BoogaBooster

Purpose: give the roadmap changes 6 `park-world-grid`, 7 `ride-catalog`, 8 `guests-and-needs`, 9 `park-economy` and 10 `coaster-editor` a sourced, number-backed description of how RollerCoaster Tycoon (RCT) 1/2 behaves, so specs can copy mechanics or deliberately simplify them.

## 0. Method, sources, reading guide

Primary source: the OpenRCT2 source code, pinned to commit `44de4bf07ee70a4cd668d23919529e095211c857` (the head of `develop` on 2026-10-14). OpenRCT2 is a faithful re-implementation of RCT2; its code comments carry the original RCT2 function addresses (for example `rct2: 0x0069545B` at `Guest::onEnterRide`), which is the evidence that the logic mirrors the original binary. Every code link below is a permalink to that commit, `L` anchors are line numbers at that commit.

- Base for links: `https://github.com/OpenRCT2/OpenRCT2/blob/44de4bf07ee70a4cd668d23919529e095211c857/src/openrct2/`. In this document it is abbreviated `OR:`. So `OR:world/Park.cpp#L376` means that base plus `world/Park.cpp#L376`.
- Secondary sources, used only as cross-checks: the OpenRCT2 wiki pages [Ride rating calculation](https://github.com/OpenRCT2/OpenRCT2/wiki/Ride-rating-calculation) (self-described as incomplete; its G-force, drop and inversion formulas match the code), [Peep AI](https://github.com/OpenRCT2/OpenRCT2/wiki/Peep-AI) (pathfinding only) and [Track types](https://github.com/OpenRCT2/OpenRCT2/wiki/Track-types) (a list of ride type ids, no geometry).
- NOT consulted successfully: the original RCT1/RCT2 manuals and the community RCT wikis. Where this document says "original game" it relies only on OpenRCT2 code comments. Any claim about RCT1-specific behaviour is UNKNOWN unless a code flag names it (for example `ParkFlag::rct1Interest`).
- Licence note: OpenRCT2 is GPL-3.0. This document records numbers and rules (facts). Do not copy its code into this repo; re-derive from the rules and document derivations in `docs/` as the repo conventions require.

Labels used on every non-trivial claim:

- **VERIFIED**: the linked source states it directly (code or comment).
- **INFERRED**: derived by the author of this document from verified facts (arithmetic or reading across functions); the derivation is given.
- **UNKNOWN**: not found in the consulted sources, or not checked. No number is invented.

### 0.1 Units and clocks (read before using any number)

| Fact | Value | Label / source |
|---|---|---|
| Game update rate | 40 ticks per second | VERIFIED: `kGameUpdateFPS = 40` in `Game.h` (`https://github.com/OpenRCT2/OpenRCT2/blob/44de4bf07ee70a4cd668d23919529e095211c857/src/openrct2/Game.h#L21`) |
| Month length | 16384 ticks (month ticks run 0..65535 in steps of 4), i.e. 409.6 s at 1x | VERIFIED steps: `OR:Date.cpp#L18`; INFERRED seconds: 65536 / 4 / 40 |
| Days per month | 8 months per year with 31, 30, 31, 30, 31, 31, 30, 31 days (RCT years have 8 months) | VERIFIED: `OR:Date.cpp#L24` |
| Week | every 4096 ticks (102.4 s); wages, interest and marketing run weekly | VERIFIED mask: `OR:Date.cpp#L19` and `OR:scenario/Scenario.cpp#L330-L345`; INFERRED seconds |
| Park rating / guest probability refresh | every 512 ticks (12.8 s) | VERIFIED: `OR:world/Park.cpp#L334-L341` |
| Ride rating scale | integers in hundredths: `RideRating::make(3, 00) == 300` means 3.00 | VERIFIED: `OR:ride/RideRatings.h#L29` |
| Guest stat scale | 0..255 for happiness, hunger, thirst, toilet, nausea; energy about 32..128 | VERIFIED: `OR:entity/Guest.h#L34-L38`, energy clamps at `OR:entity/Guest.cpp#L781` |
| Guest hunger/thirst direction | LOW value means hungry/thirsty; HIGH toilet means needs toilet; HIGH nausea means sick | VERIFIED by thought triggers (`hunger <= 10` hungry, `toilet >= 160`, `nausea >= 140`): `OR:entity/Guest.cpp#L1002-L1020` |
| Money | code uses an internal money type with `_GBP` literals; amounts below are in that unit ("currency units") | VERIFIED literals; real-world unit irrelevant for the tycoon |

## 1. Core loop

The scenario loop (VERIFIED from `OR:scenario/Scenario.cpp#L330-L345`, `OR:world/Park.cpp#L321-L353`):

1. Player places paths, rides, stalls, scenery, staff and sets prices, entrance fee, research funding, marketing.
2. Every tick the park may spawn one guest (section 3).
3. Guests wander paths, pick rides/shops from needs and preferences (sections 4, 5, 9), pay entrance/ride/item prices, leave litter and vomit.
4. Rides break down; mechanics inspect and repair (section 7).
5. Every 512 ticks the park rating, company value and spawn probability are recomputed; every week wages, research cost, loan interest and marketing run (section 14).
6. Scenario objectives (guests plus rating, park value, and so on) are checked daily/monthly (UNKNOWN in detail; `OR:scenario/Scenario.cpp#L299-L310` shows an objective switch with `guestsAndRating`).

INFERRED feedback loops: rating up -> more spawns -> more income -> more/better rides -> rating up; counter-loops are queues (unhappiness), breakdowns (downtime lowers rating), litter (lowers rating), overpriced entrance (spawn probability divided by 4 or 16).

## 2. Park rating

VERIFIED, `OR:world/Park.cpp#L376-L470` (`CalculateParkRating`), recomputed every 512 ticks. All terms are added to a running integer; the final value is clamped to 0..999.

| Term | Formula | Range |
|---|---|---|
| Base | 1150 (1050 with `difficultParkRating`) | |
| Guest count | `-(150 - min(2000, guests) / 13)` | -150 (0 guests) to +3 (2000 guests) |
| Happy guests | `-500 + 2 * min(250, happyGuests * 300 / guests)`; "happy" means `happiness > 128` | -500 to 0 |
| Lost guests | `-(lost - 25) * 7` when more than 25 guests are leaving the park and `guestIsLostCountdown < 90` | 0 downward |
| Ride uptime | `-200 + (sum(100 - ride.downtime) / rideCount) * 2` | -200 to 0 |
| Ride quality | `-100 + (100 - min(50, abs(avgExcitement/8 - 46) / 2) - min(50, abs(avgIntensity/8 - 65) / 2))`, averaged over rated rides | -100 to 0 |
| Ride totals | `-(200 - (min(1000, sumExcitement/8) + min(1000, sumIntensity/8)) / 10)` | -200 to 0 |
| Litter | `-(600 - 4 * (150 - min(150, oldLitter)))` where old litter is aged at least 7680 ticks (about 3.2 minutes at 1x) | 0 (no old litter) to -600 (150+) |
| Casualty penalty | subtract `ratingCasualtyPenalty`: +25 per drowned guest (cap 1000), decays 7 per day | 0 upward |

Reading it (INFERRED): maximum is reached with zero old litter, nearly all guests happy, no downtime, rides whose average excitement is near 3.68 (46*8/100) and intensity near 5.20 (65*8/100), and many exciting rides. The "ideal" rating shape therefore rewards moderate intensity, not maximum intensity.

Casualty penalty sources: `OR:entity/Peep.cpp#L793` (drowning, +25) and `OR:scenario/Scenario.cpp#L322-L324` (decay, 7 per day, 40 with `noMoney`). VERIFIED.

Awards: 17 award types, each positive or negative (`OR:management/Award.cpp#L44-L63`), each current award shifts spawn probability by plus/minus 25 percent (section 3). Which awards are granted is decided in `AwardIsDeserved` (not read in detail: UNKNOWN thresholds; the random draw of the candidate award is at `OR:management/Award.cpp#L674-L688`).

## 3. Guest generation (spawn rate)

VERIFIED, `OR:world/Park.cpp#L163-L229`.

Every game tick: draw a 16-bit random number; if it is less than `guestGenerationProbability`, spawn a guest (so the per-tick chance is `probability / 65536`). `probability` is recomputed every 512 ticks:

1. Start: `50 + clamp(rating - 200, 0, 650)`. Range 50..700.
2. If `guestsInPark + guestsHeadingForPark > suggestedGuestMaximum`: divide by 4 (and by another 4 in difficult-generation parks).
3. If more than 52000 guests: divide by 4.
4. If `entranceFee > totalRideValueForMoney`: divide by 4; if additionally `entranceFee / 2 > totalRideValueForMoney`: divide by 4 again.
5. For each current award: add `probability / 4` if positive, subtract `probability / 4` if negative. (The in-code comment says 0.25 percent; the arithmetic is 25 percent.)

Derived (INFERRED, arithmetic from the above at 40 ticks/s): rating 999 gives 700/65536 per tick = 0.43 guests per second (about 26 per minute at 1x); rating 200 gives 50/65536 = 0.031 guests per second (about 1.8 per minute). Each of the divide-by-4 penalties cuts those rates to a quarter.

Supporting values:

- `suggestedGuestMaximum` (the park's "guest cap") is the sum over open, non-broken rides of a per-ride-type `BonusValue` (`OR:world/Park.cpp#L103-L145`, VERIFIED). Examples: looping coaster 95 (`OR:ride/rtd/coaster/LoopingRollerCoaster.h` line with `.BonusValue = 95`), merry-go-round 45, swinging ship 35, food stall 15, information kiosk 15, toilets 5 (VERIFIED from the `rtd` headers listed in section 12).
- `totalRideValueForMoney` = sum over open, non-broken rides of `2 * (ride.value - ridePrice)`, only when positive (`OR:world/Park.cpp#L75-L101`, VERIFIED). So pricing rides high lowers the entrance fee the park can charge before the spawn rate collapses.
- Default entrance fee in the code: 10.00 currency units (VERIFIED by the first-pass read of `Park.cpp`; line not re-pinned: treat as INFERRED for default value).
- Marketing campaigns add extra independent spawn rolls each tick (section 14.2).
- Spawn point: random "peep spawn" (the park entrance path); guests start walking in as `heading for park` (UNKNOWN detail beyond `OR:entity/Guest.cpp#L7152`).

## 4. Guest stats, needs and thresholds

Initial stats, VERIFIED `OR:entity/Guest.cpp#L7152-L7320` (`Guest::generate`):

| Stat | Initial |
|---|---|
| Happiness | scenario option (default 128 if 0) plus a random delta of -15..+16 |
| Hunger, thirst | scenario option plus random delta -15..+16 (values UNKNOWN: the scenario defaults live in scenario data) |
| Toilet, nausea | 0 |
| Energy | uniform 65..128 |
| Cash | `(rand & 3) * 100 - 100 + scenarioInitialCash`; 500 (units: smallest money unit) if the option is 0; 0 in no-money parks |
| Intensity preference | highest = `(rand & 7) + 3` (3..10); lowest = `min(highest, 7) - 3`; if highest >= 7 it becomes 15. Resulting ranges: highest 3 -> [0, 3], 4 -> [1, 4], 5 -> [2, 5], 6 -> [3, 6], 7..10 -> [4, 15] |
| Nausea tolerance | pick uniformly from a 12-slot table: 1 none, 2 low, 3 average, 6 high (so none 8%, low 17%, average 25%, high 50%); the park flag "prefer more intense rides" adds 4 to the index |

Per-update motive changes, VERIFIED `OR:entity/Guest.cpp#L678-L830`:

- Idle drift (each update): `happinessTarget` moves 1 toward 127; minus 2 if energy <= 50; minus 1 each if hunger < 10, thirst < 10, or toilet >= 195. `nauseaTarget` falls 2 per update. A walking guest with `nauseaTarget >= 128` may vomit with chance `(nausea - 128) / 2` out of 256.
- Happiness and nausea (the visible values) move 4 per update toward their targets; energy moves 2 down or 4 up toward `energyTarget`.
- Hunger: `GuestUpdateHunger` subtracts 2 from hunger (while hunger >= 3), adds 2 to `energyTarget` and 1 to toilet (`OR:entity/Guest.cpp#L2993-L3002`). It runs once per 512-ish update window while walking, sitting, queuing (`OR:entity/Guest.cpp#L1080-L1160`).
- Thirst: falls 1 per update when temperature >= 21 degrees and thirst >= 5 (`OR:entity/Guest.cpp#L3017`); sitting: thirst -4, toilet +3.
- Eating/drinking: each consumption step takes `timeToConsume` down by 3; a drink adds 7 to thirst; food adds 7 to hunger, subtracts 3 from thirst, adds 2 to toilet (`OR:entity/Guest.cpp#L724-L760`).
- Sitting also raises `energyTarget` by 5 (while <= 135) and lowers `nauseaTarget` by 6 (while >= 50).
- Toilet visit: toilet reset sequence, then `happinessTarget +30`; first aid works only when nausea <= 35 afterwards (`OR:entity/Guest.cpp` toilet/first-aid section, summarised; exact lines UNKNOWN, treat as INFERRED from the first-pass read).

Thought triggers (checked in a rotating window, about once per 1024 ticks per guest), VERIFIED `OR:entity/Guest.cpp#L1000-L1060`:

| Thought | Condition | Action |
|---|---|---|
| Tired | energy <= 70 and happiness < 128 | none |
| Hungry | hunger <= 10 and no food/drink held | head for nearest food stall |
| Thirsty | thirst <= 25 and no food/drink held | head for nearest drink stall |
| Toilet | toilet >= 160 | head for nearest toilet |
| Running out of money | cash <= 9.00 units, happiness >= 105, energy >= 70, not no-money | head for nearest cash machine |
| Sick | nausea >= 140 | thought only |
| Very sick | nausea >= 200 | head for first aid |

Other happiness events (VERIFIED, same file): surroundings check every 18 windows adds +45 to `happinessTarget` when the area within 160 units (5 tiles) shows "fountains" (at least 5 fountains and fewer than 20 rubbish), "scenery" (at least 40 scenery items and fewer than 8 rubbish), "music" or "very clean" (fewer than 2 rubbish) (`OR:entity/Guest.cpp#L2872-L2990`, `#L943`); more than 15 ticks on a ride lowers `happinessTarget` 5 per window (`OR:entity/Guest.cpp#L960-L985`); lost guests get -30 (`OR:entity/Guest.cpp#L1395-L1400`).

Leaving the park, VERIFIED `OR:entity/Guest.cpp#L3010-L3060`: guests stay if energy >= 55, happiness >= 45 and cash >= 5.00 (no-money parks: energy >= 70 and happiness >= 60). Otherwise 5 percent chance per check (`(rand & 0xFFFF) <= 3276`). A guest who has not found a ride after about 5 x 2048 ticks (about 4.3 minutes) loses 128 happiness target and leaves (`OR:entity/Guest.cpp#L936-L952`).

Cash machines: a guest withdraws only with at most 20.00 units on hand and `115 + rand%128 <= happiness`, energy >= 80 (summarised from first-pass read; exact lines UNKNOWN).

## 5. Guest thoughts and the decision to ride

### 5.1 Choosing a ride

VERIFIED, `OR:entity/Guest.cpp#L1762-L1883`.

- While walking, each update window the guest rolls `rand16 <= 2184` (3.3 percent) to pick a ride, or `<= 8192` (12.5 percent) if it holds a park map. It does not pick a ride while holding food/drink or while leaving the park.
- Candidate rides: rides within 10 tiles of the guest (radius `10 * 32` units), or all rides when the guest has a map. Rides already ridden are skipped; rides with the "queue full" flag are skipped.
- Among candidates that pass `shouldGoOnRide` and have ratings, the guest picks the highest excitement. So excitement drives choice; intensity and nausea only gate it.
- Rides are also chosen opportunistically at the path next to the entrance (guest "thinking" vs "at ride" flag in `shouldGoOnRide`).

### 5.2 Gates in `shouldGoOnRide`

VERIFIED, `OR:entity/Guest.cpp#L1884-L2170`. The gates, in the order evaluated for an open, non-broken ride:

1. Leaving-park guests refuse everything except free transport rides.
2. Shops/facilities go to `GuestShouldGoToShop` (5.4).
3. Queue admission: a ride without a queue accepts one waiting guest at a time; with a queue, the guest cannot join if within 8 units of the last guest in the queue, or within 13 units of a last guest who has already waited more than 10 ticks.
4. Not the previous ride (the guest never rides the same ride twice in a row, except via the ride-again rule).
5. Price: if price > cash the guest refuses (thought "can't afford"); at zero cash "spent money".
6. Recent crash: refused when happiness < 225 (thought "not safe", `happinessTarget -8`).
7. Weather: when precipitating, refused unless the ride is sheltered (at least 3 sheltered eighths) or the guest has an umbrella and the ride allows it (50 percent); "not while raining" costs `happinessTarget -8`.
8. Intensity window (skipped if the guest already decided to ride and arrives; then only intensity > 10.00 is refused): `maxIntensity = min(guest.max * 100, 1000) + happiness`; `minIntensity = guest.min * 100 - happiness`. Ride intensity below min -> "more thrilling" refusal; above max -> "too intense" refusal. Happiness is on 0..255, so a fully happy guest widens each side by up to 2.55 rating points.
9. Nausea gate: refused if `ride.nausea > NauseaMax[tolerance] + happiness` with `NauseaMax = {300, 600, 800, 1000}` for none, low, average, high (`OR:entity/Guest.cpp#L188-L193`, VERIFIED). Guests with `nausea > 160` also refuse rides with nausea >= 1.40.
10. Unrated rides that can have G forces: 90 percent are ignored; others are refused if max vertical G > 5.00, max negative G < -4.00 or max lateral G > 4.00.
11. Value: if the ride has a computed value, the acceptable price is `value` (divided by 4 if the guest paid to enter the park). Price > 2 x that value -> "bad value" refusal (`happinessTarget -16`); price <= half of it -> "good value" thought.

Free-ride vouchers bypass the price gates. Thoughts about rides ("more thrilling", "too intense", "sickening", "bad value", "not while raining", "not safe", "can't afford") are VERIFIED enum names inserted in those branches.

### 5.3 What riding does to the guest

VERIFIED, `OR:entity/Guest.cpp#L1676-L1700` (`onEnterRide`) and `#L2620-L2860`.

On entering, the guest computes `satisfaction` (range -140..+105 per the code comment) and adds it to `happinessTarget`:

- Value term: ride value >= price -> -5; `value * (1 + happiness/256) >= price` -> -30; otherwise 0; -30 in no-money parks or if value is undefined.
- Intensity/nausea term, from the guest's windows widened in three steps (exact fit, then widened by happiness, then widened again): the number of widening steps needed gives an intensity score 0..3 and a nausea score 0..3. Lookup on (highest, lowest) score: (0) 70; (1,0) 50; (1,1) 35; (2,0) 35; (2,1) 20; (2,2) 10; (3,0) -35; (3,1) -50; (3,2) -60; (3,3) -60. An unrated ride returns 70.
- Queue term: waited at least 4500 queue ticks -35; at least 2250 -10; at most 750 +10.
- Experience term: +10 if the guest has ridden this ride type before, another +10 if this very ride.
- Ride satisfaction statistic: 3 if satisfaction >= 40, 2 if >= 20, 1 if >= 0, else 0.

Nausea growth on entering: `nauseaTarget += ((ride.nausea * clamp(256 - happinessTarget, 64, 200) / 512) * max(128, hunger) / 128 * 2) >> toleranceIndex` where toleranceIndex is 0, 1, 2, 3 for none, low, average, high (so nausea builds 100, 50, 25, 12.5 percent as fast). The code comment lists the same factors (`OR:entity/Guest.cpp#L2796-L2810`). VERIFIED.

On exit: happiness and nausea snap to their targets; a favourite ride is recorded if `excitement/4 + satisfaction` (capped 255) beats the guest's previous best and both happiness and its target are >= 160; the guest may ride again (needs ride type flag, intensity <= 10.00, happiness >= 180, energy >= 100, nausea <= 160, hunger >= 30, thirst >= 20, toilet <= 170, then a random gate; at most 7 rides unless a 25 percent roll passes); with happiness >= 200 and `rand8 >= intensity` the guest's maximum preferred intensity grows by 1 up to 15; "was great" thought if happiness >= 215 and nausea <= 120 (`OR:entity/Guest.cpp#L1700-L1740`, `#L2770-L2860`). VERIFIED.

### 5.4 Shops, stalls, toilets, kiosks as "rides"

VERIFIED `OR:entity/Guest.cpp#L2172-L2260` (`GuestShouldGoToShop`) and `#L1438-L1560` (`GuestDecideAndBuyItem`):

- No visiting the same shop twice in a row. Price > cash is refused.
- Toilets: ignored if toilet < 70; the guest also refuses if `price * 40 > toilet` (comment: effective range 0.10 to 0.60 units).
- First aid: ignored if nausea < 128.
- Food: refused if hunger > 75 ("not hungry") or if the guest still holds food; drink refused if thirst > 75; both refused if nausea >= 145. Balloons, ice cream, candyfloss, sunglasses are refused in precipitation; ice cream and sunglasses below 12 degrees. Souvenirs need `rand(0..127) + 0x73 <= happiness` and at least 3 rides.
- Price-value for items: an item has a base, hot and cold value chosen from the temperature thresholds of the climate object (`OR:entity/Guest.cpp#L1407-L1420`; the numeric thresholds live in the climate object data: UNKNOWN). If the item value is below the price, the guest accepts with a probability that falls with the overshoot and rises with happiness (>= 128 halves the effective overshoot, >= 180 halves it again); if the value is above the price, `happiness` and `happinessTarget` rise by `4 * (value - price)` (at least 0.80 units).

### 5.5 Queue behaviour and wait time

VERIFIED `OR:entity/Guest.cpp#L1080-L1160`, `#L5631-L5740`, `#L2620-L2640`:

- `timeInQueue` counts guest update steps while queuing (INFERRED: one per game tick, so 2000 is about 50 s, 3500 about 87 s, 4300 about 108 s at 1x; the exact stepping was not traced).
- At 2000: without a queue TV monitor on the path tile `happinessTarget -4` per window; with one, target is raised to at least 90 and then +2 per window up to 165. At 2000 the guest may eat (0.18 percent chance per check). At 3500: "queuing for ages" thought. At 4300: if happiness <= 65, 3.3 percent chance per check to leave the queue. If the ride is closed or broken, guests are removed from the queue.
- Entertainers within 3 tiles cut `timeInQueue` by 200 and add 3 to `happinessTarget` (`OR:entity/Staff.cpp#L889-L940`).
- Reported queue time for the ride sign: derived from `daysInQueue` of the last guest to enter: `(daysInQueue + 3) / 2` (`OR:entity/Guest.cpp#L3836-L3841`); `daysInQueue` increments once per game day for guests in queue (`OR:entity/Peep.cpp#L1275-L1284`). The display unit is UNKNOWN; INFERRED one day is 16384/31 ticks, about 13 s.

## 6. Ride metrics: excitement, intensity, nausea

### 6.1 Pipeline

VERIFIED `OR:ride/RideRatings.cpp#L1020-L1060` and `OR:ride/RideData.h#L233-L260`. Each ride type carries a `RideRatingsDescriptor` with: calculation type (`normal` needs a completed test run, `flatRide` is automatic, `stall` has no ratings), base ratings (excitement, intensity, nausea), a base unreliability, a shelter value, a flag that relaxes drop/negative-G requirements when the ride has inversions, and up to 32 modifiers (type, threshold, excitement coefficient, intensity coefficient, nausea coefficient). The final steps are:

1. For each modifier in list order add `metric * coefficient >> 16` per rating (bonuses), or divide the three ratings by the three coefficients if the requirement fails (requirements).
2. Intensity penalty: for each bound in {10.00, 11.00, 12.00, 13.20, 14.50}, if intensity >= bound, excitement loses a quarter of its current value (so excitement can fall to about 24 percent). VERIFIED by first-pass read of `RideRatingsApplyIntensityPenalty` (`OR:ride/RideRatings.cpp#L1051` call site; the bounds were read in the function body).
3. Entry adjustments and air time: each rating gains `rating * multiplier / 128` where the multiplier is the ride's "ratings multipliers" (for example looping coaster 50, 30, 10 = +39 percent, +23 percent, +8 percent); standard rides add `min(airtime, 200)/8` excitement and `airtime/16` nausea (`OR:ride/RideRatings.cpp#L1259-L1310`).
4. Compute upkeep and value (section 6.4, section 7).

### 6.2 Measured quantities (how a "ride test" works)

VERIFIED `OR:ride/Vehicle.cpp#L470-L530`: during a test run every tick the first car updates `maxSpeed`, a running `averageSpeed` accumulator, `segmentLength` (distance integrated as `((velocity + acceleration) >> 10) * 42`), `maxPositiveVerticalG`, `maxNegativeVerticalG`, `maxLateralG` (each G reading is averaged with the previous reading), and counts `totalAirTime` ticks where vertical G <= 0. It also counts drops, inversions, helices, turns, sheltered sections and reversals while walking track pieces (`OR:ride/RideRatings.cpp` track-walking state; details UNKNOWN beyond the formulas below).

### 6.3 Sub-rating formulas (raw units = hundredths)

All VERIFIED in `OR:ride/RideRatings.cpp`; the coefficients are fixed-point (divide by 65536 after multiplying).

| Component | Excitement | Intensity | Nausea | Source |
|---|---|---|---|---|
| Max positive vertical G (G in hundredths) | `G * 5242 >> 16` (about 0.08 G) | `G * 52428 >> 16` (0.8 G) | `G * 17039 >> 16` (0.26 G) | `#L1689-L1691` |
| Max negative vertical G, clamp [-2.50, 0] | `G * -15728 >> 16` (0.24 per G) | `(G - 1.00) * -52428 >> 16` | `(G - 1.00) * -14563 >> 16` | `#L1694-L1699` |
| Max lateral G | `min(1.50, G) * 26214 >> 16` (0.4) | `+ G` | `G * 21845 >> 16` (0.33) | `#L1701-L1703` |
| Excess lateral G | G > 2.80: intensity +3.75, nausea +2.00. G > 3.10: excitement becomes negative half of G excitement, intensity 12.25, nausea 6.00 (a "death" ride) | | | `#L1707+`, summarised from first-pass read |
| Drops | `min(9, n) * 11.1` + `2*h*0.244` | `n * 14.2` + `2*h*0.488` | `n * 10` + `2*h*0.156` | `#L1727-L1745`, `h` highest drop height in height units |
| Inversions | `min(6, n) * 26.7` | `n * 50` | `n * 21.3` | `#L1522-L1530` |
| Turns, flat | 3+ element turns 2.5 each, 2-element 3.0, 1-element 0.97 | 1.25, 0.75, 0.32 | 5.0, 3.1, 0.65 | `#L1447-L1470` |
| Turns, banked | 3.75, 3.75, 1.13 | 1.25, 0.75, 0.32 | 5.0, 3.1, 0.74 | `#L1477-L1500` (summarised) |
| Sheltered length and sections | `min(L,1000)*0.14` + `min(sections, 11)*11.8` (+20 if banking sheltered, +20 if rotating) | `min(L,2000)*0.15` | `min(L,1000)*0.25` (+15 per sheltered banking/rotating) | `#L1646-L1678` |
| Scenery | `min(count, 47) * 5` items counted in the 11x11 tile area around the station; 40 if the station is underground | | | `#L1760+` (summarised) |
| Proximity (track near track/station/water/path/loops) | list of up to 27 weighted terms | | | `#L1560+` (summarised) |

Per-ride modifier coefficients then scale these (example below). Other bonuses: length (`min(length, threshold) * coeff`), train length (`(cars - 1) * coeff`), duration, max speed (`(maxSpeed >> 16) * coeff` for all three ratings), average speed (excitement and intensity), synchronisation, number of trains, operation option, rotations (`OR:ride/RideRatings.cpp#L1832-L1890`, `#L2096`).

Requirement checks (divide all three ratings by a coefficient, usually 2, when the condition fails; `OR:ride/RideRatings.cpp#L2101-L2250`): minimum station-segment length, minimum highest drop, minimum max speed, minimum drop count, minimum (least negative) vertical G, maximum lateral G, maximum inversions, minimum sheltered eighths, maximum reversers/holes, station count. These are why very short or flat coasters rate poorly.

### 6.4 Worked data: coaster and flat rides

VERIFIED `OR:ride/rtd/coaster/LoopingRollerCoaster.h` lines 65-92:

- Base ratings 3.00 / 0.50 / 0.20 (excitement / intensity / nausea), base unreliability 15, relaxed requirements with inversions.
- Modifiers (threshold; excitement, intensity, nausea coefficient): length (6000; 764), train length (187245), max speed (44281, 88562, 35424), average speed (291271, 436906), duration (150; 26214), G-forces (24576, 35746, 49648), turns (26749, 34767, 45749), drops (29127, 46811, 49152), sheltered (15420, 32768, 35108), proximity (20130), scenery (6693), requirements: drop height >= 14, max speed >= 0xA0000 (10 mph fixed-point units), negative-G requirement threshold stored as `MakeFixed16_2dp(0, 10)` (sign convention not traced: UNKNOWN), at least 2 drops; lateral-G penalty on.

Flat rides (VERIFIED, `OR:ride/rtd/gentle/MerryGoRound.h`, `.../thrill/SwingingShip.h`, `.../thrill/Twist.h`, `.../thrill/Enterprise.h` `RatingsData` blocks):

| Ride | Type | Base (E / I / N) | Modifiers | Base unreliability |
|---|---|---|---|---|
| Merry-go-round | gentle | 0.60 / 0.15 / 0.30 | bonusRotations (5,5,5 per rotation), scenery 19521 | 16 |
| Swinging ship | thrill | 1.50 / 1.90 / 1.41 | bonusOperationOption (5,5,10 per option step), scenery 16732 | 10 |
| Twist | thrill | 1.13 / 0.97 / 1.90 | bonusRotations (20,20,20), scenery 13943 | 16 |
| Enterprise | thrill | 3.60 / 4.55 / 5.72 | bonusOperationOption (1,16,16), scenery 19521 | 22 |

Fixed values for a few flat rides are in the modifier code: motion simulator 2.90/3.50/3.00 (3.25/4.10/3.30 for film-thrill mode); top spin variants beginners 2.00/4.80/5.74, intense 3.00/5.75/6.64, berserk 3.20/6.80/7.94; 3D cinema variants 3.50/2.40/1.40, 4.00/2.65/1.55, 4.20/2.60/1.48 (summarised from `OR:ride/RideRatings.cpp` first-pass read; INFERRED exact lines).

Design consequence (INFERRED): flat rides are essentially base ratings plus one operator-chosen knob and a scenery bonus; only coasters need a track simulation.

### 6.5 Ride value (what guests think it is worth)

VERIFIED `OR:ride/RideRatings.cpp` `RideRatingsCalculateValue` (call at `#L444`; body read in the first pass): `value = (excitement * mult_e + intensity * mult_i + nausea * mult_n) * 32 >> 15` using the ride-type `RatingsMultipliers` (looping coaster 50/30/10); times an age factor: +30 under 5 months, +10 under 13 months, then x1 until 40 months, x0.75 until 64, x0.56 until 88, x0.42 until 104, x0.32 until 120, x0.16 until 128, x0.08 until 200 (the table differs at 200+: UNKNOWN reason); minus 25 percent when another open ride of the same type exists. Ride value drives how high the player can price (5.2 gate 11), the spawn probability (section 3) and park value.

## 7. Vehicle physics and track validity (for coaster-editor)

### 7.1 Vehicle motion model

VERIFIED `OR:ride/Vehicle.TrackMotion.cpp#L1541-L1600` and `OR:ride/Vehicle.cpp#L72-L90`. The simulation is a simple scalar model along the track, not rigid-body physics:

- Each car gets `acceleration = getAccelerationFromPitch(pitch)` from a lookup by track pitch (gravity component).
- For a train: `a = (sum(car accelerations) / numCars) * 21 / 512`.
- Rolling resistance: `a -= velocity / 4096`.
- Drag: `a -= ((velocity >> 8)^2 >> 4) / totalMass` (sign follows velocity). Heavier trains lose less to drag.
- Powered cars (lift hills, launches, boosters) add `(speedSetting << 14 - velocity) * poweredAcceleration * 2 / (speedSetting * mass / 4)` with special cases (`OR:ride/Vehicle.TrackMotion.cpp#L1313-L1385`).
- Creep boost: on flat track, if the car is slower than about 0.5 mph and acceleration is in [-500, 0], add +400 so trains do not stall at exactly zero.
- Position advances by `(velocity >> 10) * 42` distance units per tick.

### 7.2 G-forces

VERIFIED `OR:ride/Vehicle.cpp#L1158-L1185` (`GetGForces`): vertical G = pitch/roll gravity term (0x280000 scaled by pitch and roll vectors) plus `|velocity| * 98 / verticalFactor`, where `verticalFactor` and `lateralFactor` are per-track-piece, per-progress lookup tables (`ted.verticalFactor(track_progress)`); lateral G = `|velocity| * 98 / lateralFactor`; both times 10, shifted right 16. So G is `v * piece_constant`: pieces with tight geometry have small factors. The per-piece tables live in `OR:ride/TrackData.cpp` and `OR:ride/TrackData.h`-adjacent files (not read: UNKNOWN values).

Hard limits seen in code: guests refuse unrated G-capable rides above vertical 5.00 G, below -4.00 G, or lateral 4.00 G (`OR:entity/Guest.cpp#L2095-L2110`); lateral G above 2.80 and 3.10 triggers the intensity/nausea penalties of 6.3; ratings beyond intensity 10.00 are refused for repeat rides.

### 7.3 Lift hills, brakes, banking, track groups, modes

- Lift speed range per coaster type: `LiftData = { sound, min, max }`, looping coaster `{ liftClassic, 4, 6 }` (units UNKNOWN; mph in the UI) (`OR:ride/rtd/coaster/LoopingRollerCoaster.h`). Unreliability grows with lift speed: `+ (liftSpeed - minimum) * 2` (`OR:ride/RideRatings.cpp` `SetUnreliabilityFactor`, first-pass read).
- Enabled track groups for a looping coaster (VERIFIED, same header): straight, station end, lift hill, flat-to-banked roll, vertical loop, slopes (25 degree and steep), slope-curve, S-bend, small/standard/large curves, banked half helix, brakes, on-ride photo, block brakes, booster, diagonal pieces and quarter helices; "extra" groups unlocked by research or flags: vertical slope, twist, corkscrew, quarter loop, barrel roll, half loops, zero-G roll. Each coaster type has its own list, so the *piece set is data, per ride type*.
- Ride modes for a looping coaster: continuous circuit, block-sectioned circuit, reverse-incline launched shuttle, powered launch (pass-through and stop) (`rideModes`, same header). Operating setting range `{10, 27}`, booster `{18, 18}` (meaning UNKNOWN).
- Brakes add 20 upkeep each, boosters 80, reversers 10 (`OR:ride/RideRatings.cpp` `RideComputeUpkeep`, first-pass read).
- Coaster tester ("ride testing"): the ride must complete a test circuit once (`RideStatus::testing`, `tested` flag) before ratings exist; flat rides skip this (`calculation type flatRide`). Statuses: closed, open, simulating, testing (`OR:ride/Ride.h` `RideStatus`, via `Ride.cpp` read).
- Track validity: UNKNOWN in detail. OpenRCT2 enforces it in track placement actions (not read). Observable rules from ratings: a ride is unrated until it has a station, a lift or launch that gets the train round, and a closed circuit unless the mode is a shuttle.

### 7.4 Coaster construction costs

VERIFIED struct `RideBuildCost { TrackPrice, SupportPrice, PriceEstimateMultiplier }` (`OR:ride/RideData.h#L175-L180`). Looping coaster `{ 45.00, 2.50, 50 }` per straight track piece and per support (INFERRED reading of field names), merry-go-round `{ 57.50, 1.00, 1 }`, swinging ship `{ 77.50, 1.00, 1 }`, toilets `{ 225.00, 0, 1 }`, food stall `{ 300.00, 0, 1 }`, information kiosk `{ 250.00, 0, 1 }`. Whole-ride cost for flat rides is a single piece price. Real placement cost multiplies by track piece count (formula UNKNOWN).

## 8. Breakdowns, inspection and mechanics

VERIFIED `OR:ride/Ride.cpp#L1010-L1160`.

- Runs every 256 ticks per ride (6.4 s).
- Reliability starts at `kRideInitialReliability = (100 << 8) | 0xFF = 25855` (`OR:ride/Ride.h#L792`). Each check subtracts `unreliabilityFactor + agePenalty` (floored at 0). Age penalty by ride age in years: 0 -> 0; 1 -> factor/8; 2 -> factor/4; 3-4 -> factor/2; 5-7 -> factor; 8+ -> 2 x factor.
- A breakdown happens when `reliability == 0` or `(rand & 0x2FFFFF) <= 1 + 25855 - reliability`. INFERRED probability per check: `(25856 - reliability) / 3145728` (the 22-bit range is 0x2FFFFF + 1). The code comment says "(25000 - reliability) / 3 000 000".
- Breakdown kinds and weights (when a breakdown fires, one kind is drawn; kinds not available for the ride type are excluded): safety cut-out 25, restraints stuck closed 12, restraints stuck open 10, doors stuck closed 13, doors stuck open 10, vehicle malfunction 6, control failure 3, brakes failure 3 (20 in precipitation; excluded for rides under 16 months old or with reliability above 50 percent). Which kinds apply is a per-ride-type `availableBreakdowns` list (merry-go-round: safety cut-out and control failure; swinging ship: safety cut-out only; looping coaster: safety cut-out, restraints closed/open, vehicle malfunction, brakes failure).
- Downtime: every 256 ticks while broken, `downtimeHistory[0]++`; every 8192 ticks `downtime = min(sum(history) / 2, 100)` and the history shifts. `downtime` feeds the park rating uptime term (section 2).
- Inspection: interval options in minutes `{10, 20, 30, 45, 60, 120, never, never}`; `lastInspection` increments every 2048 ticks and a mechanic is requested when the interval is reached. A mechanic's inspection restores reliability by `((100 - reliabilityPercent) / 4) * (rand & 0xFF)` (`OR:entity/Staff.cpp#L2638`, VERIFIED).
- Dispatch: the closest patrolling mechanic by Manhattan distance, with orders "fix rides" and "inspect rides"; a mechanic call times out after 2500 ticks (`OR:entity/Staff.cpp#L1295`).
- Fix duration has no constant; it comes from animation frames (about 37 to 101 frames) (summarised from `Staff.cpp`; INFERRED).
- Rides that crash (as opposed to break down) set a crash type that makes guests with happiness below 225 refuse the ride (5.2 gate 6). Crash probability: UNKNOWN.

## 9. Staff

VERIFIED `OR:entity/Staff.cpp#L2645-L2659` for wages; behaviour from the same file (first-pass summary, lines noted where re-read).

| Staff | Wage per month | Behaviour |
|---|---|---|
| Handyman | 50.00 | Sweeps litter within a 3-tile radius, mows grass, waters plants, empties bins. Speed numbers: UNKNOWN (driven by animation steps). |
| Mechanic | 80.00 | Inspects and repairs rides (section 8). |
| Security guard | 60.00 | Counts crowding (a path tile with at least 20 walking guests within 3 tiles makes the guard switch animation, `kSecurityPathCrowdedThreshold = 20`, `OR:entity/Staff.cpp#L1680`). Prevents vandalism within 224 units (7 tiles) by absorbing the vandalism attempt and counting a stopped vandal (`OR:entity/Guest.cpp#L6244-L6300`). |
| Entertainer | 55.00 | Within 3 tiles of a guest: walking guests `happinessTarget +4`, queuing guests `timeInQueue -200` and `happinessTarget +3`; acts on 25 percent of checks (`OR:entity/Staff.cpp#L889-L945`). |

Wages are paid weekly at a quarter of the monthly figure (`OR:management/Finance.cpp#L106-L122`: `GetStaffWage / 4`, week = 4096 ticks). Staff patrol areas: no area means anywhere in owned land. Hiring cost and staff-count limits: UNKNOWN (not in `Staff.cpp`).

## 10. Litter, vandalism, path and scenery rules

- Litter source: VERIFIED `OR:entity/Guest.cpp#L5418-L5460`. Normal guests do NOT throw litter at random. A guest holding an EMPTY food/drink container, while on a path tile (not surface), drops it with chance 4096/65536 per check, on average about once per 512 ticks (the check uses the guest id masked with the tick). If no container is found, the dropped object is vomit. The randomly-littering flag (`PeepFlag::litter`) is set only for an easter-egg name (`OR:entity/Guest.cpp#L639`). So litter volume is driven by food/drink sales plus nausea (vomit).
- Litter age: litter aged >= 7680 ticks counts toward the park rating litter term (section 2); handymen and bins are the counter (bin emptying frame logic only; capacity UNKNOWN).
- Vandalism: VERIFIED `OR:entity/Guest.cpp#L6244-L6300`. A walking guest with happiness < 48, energy >= 85, and a high count of litter/disgust seen (both top bits set) vandalises a broken-able path addition (bench, bin, lamp) with chance 3276/65536 per check; skipped if a security guard is within 224 units or someone sits on that bench. Angry guests (just vandalised) bypass the mood test. A broken path addition counts as "rubbish" in the surroundings check (section 4).
- Path rules: wiki says guests reject path areas wider than 3x3 tiles, never step backwards, and avoid queue paths unless seeking them (Peep AI wiki, secondary). Search budget 15000 tiles and 6 junctions for guests; score uses distance/16 + 2 x height difference (same page; "(i expect)" flags the height multiplier as a guess). Whole-path validity rules (slopes, connection edges): UNKNOWN.
- Scenery: only affects (a) the ride rating scenery bonus (section 6.3: up to 47 items x 5 within the 11x11 tile area around the station), (b) the "scenery" surroundings thought (>= 40 items nearby and under 8 rubbish gives happiness +45), (c) "beautiful" award (not read in detail).
- Information kiosk: sells park maps and umbrellas (INFERRED from shop item names). A map raises a guest's ride-picking roll from 3.3 percent to 12.5 percent and extends candidates from 10 tiles to all rides (section 5.1). Toilets: use only when toilet >= 70 and priced reasonably. First aid: nausea >= 128.
- Placement effects of stalls/toilets: hunger/thirst/toilet thoughts send guests to the NEAREST matching facility (section 4), so placement matters by walking distance; the exact search radius is UNKNOWN.
- Queue TV monitors raise queuing guests' happiness (section 5.5).

## 11. Weather

VERIFIED `OR:world/Weather.cpp#L36-L60`, `#L335-L372`.

- Nine weather types: sunny, partially cloudy, cloudy, rain, heavy rain, thunderstorm, snow, heavy snow, blizzard, each with a temperature offset added to the month's base temperature: +10, +5, 0, -2, -4, +2, -10, -15, -20.
- The next weather is drawn from a per-month, per-climate distribution table (the `ClimateObject`); the table values are data: UNKNOWN. Transition timer is 1920 ticks (48 s at 1x).
- Gameplay effects found: guests refuse unsheltered rides while it is raining or snowing heavily (5.2 gate 7); some shop items are refused in precipitation or cold (5.4); brake failure weight rises from 3 to 20 during precipitation (section 8); thirst falls faster at 21 degrees or more (section 4); ride-again and shop item values depend on temperature thresholds.
- Not found: any direct weather term in the park rating or in spawn probability. Marketing and entrance fee are the only spawn inputs besides rating (section 3).

## 12. Ride catalog categories

VERIFIED `OR:ride/Ride.h#L698-L708` (`RideCategory`): transport, gentle, rollerCoaster, thrill, water, shop (plus none). Ride type descriptors live under `OR:ride/rtd/` (directory listing at the pinned commit):

- coaster: 43 types, including looping, corkscrew, twister, wooden, junior, mini, stand-up, inverted, hyper, giga, hybrid, vertical drop, wild mouse (wooden/steel/spinning), water coaster, reverser, bobsleigh, LIM/LSM launched, flying, alpine, heartline, air-powered vertical.
- gentle (17): car ride, circus, crooked house, dodgems, ferris wheel, flying saucers, ghost train, haunted house, maze, merry-go-round, mini golf, mini helicopters, monorail cycles, monster trucks, observation tower, space rings, spiral slide.
- thrill (11): 3D cinema, enterprise, go karts, launched freefall, magic carpet, motion simulator, roto-drop, swinging inverter ship, swinging ship, top spin, twist.
- transport (5): chairlift, lift, miniature railway, monorail, suspended monorail.
- water (7): boat hire, dinghy slide, log flume, river rafts, river rapids, splash boats, submarine ride.
- shop (7): cash machine, drink stall, first aid, food stall, information kiosk, shop (souvenirs), toilets.

Each type descriptor defines: category, enabled track groups, ride modes, operating settings, breakdown list, heights, max train mass, lift speed range, `RatingsMultipliers`, `UpkeepCosts`, `BuildCosts`, `DefaultPrices`, `BonusValue` (guest-cap contribution), and `RatingsData` (`OR:ride/RideData.h#L500-L545`, VERIFIED). Default prices from the data: merry-go-round 10, swinging ship 15, looping coaster 20 (units: tenths of the currency in the original, UNKNOWN: treat as relative).

Upkeep (VERIFIED `OR:ride/RideRatings.cpp` `RideComputeUpkeep`, first-pass read): base cost plus per-track-piece, per-train, per-car and per-station terms, +40 for an on-ride photo, +80 per booster, launch modes +30 to +320, scaled by 0.625. Running cost enters ride profit as `incomePerHour - upkeepCost * 16` (`OR:ride/Ride.cpp` income section, first-pass read).

## 13. Pacing cheat sheet (real time at 1x, all INFERRED from section 0.1 arithmetic)

| Event | Interval |
|---|---|
| Park rating, spawn probability refresh | 12.8 s |
| Breakdown check per ride | 6.4 s |
| Inspection counter step | 51.2 s |
| Day (about 1/31 of a month) | about 13 s |
| Week (wages, interest, marketing) | 102.4 s |
| Month | 409.6 s (6.8 min); year = 8 months = 54.6 min |
| Weather transition timer | 48 s |

## 14. Finance, research and marketing

### 14.1 Loans, interest, company value

VERIFIED `OR:management/Finance.cpp#L141-L162`, `#L240-L245`, `OR:world/Park.cpp#L470-L505`.

- Starting loan 10000.00, maximum loan 20000.00, interest rate field 10 in the default setup (scenarios override).
- Interest per week: `loan * 5 * rate >> 14`, i.e. `loan * rate * 5 / 16384`. INFERRED: with rate 10 that is 0.305 percent per week; at 32 weeks per year about 9.8 percent per year (so "10 percent per annum"). A flag `rct1Interest` switches to `loan / 2400` per week (about 1.33 percent per year of 32 weeks).
- Park value: 7.00 per guest in the park plus the sum of ride values, each `(ride.value * 10) * (customersInLast5Minutes + BonusValue * 4)`. Company value = park value - loan + cash. Both refreshed every 512 ticks.
- Profit estimate (`Finance.cpp#L271-L283`) subtracts staff wages, research cost and `loan / 600`.

### 14.2 Marketing

VERIFIED `OR:management/Marketing.cpp#L24-L60`.

Six campaign types with weekly cost and extra spawn probability (per tick, out of 65536 draw space, each campaign rolls separately): free park entry vouchers 50.00 (400), free ride voucher 50.00 (300), half-price entry vouchers 50.00 (200), free food/drink vouchers 50.00 (200), park advertising 350.00 (250), ride advertising 200.00 (200). The probability is divided by 8 when the offer is pointless: free entry with an entrance fee under 4.00, half-price entry with a fee under 6.00, free ride when the ride price is under 0.30. A campaign lasts `weeks` (1..255) and is billed `weeks * weekly price` up front (`OR:actions/park/ParkMarketingAction.cpp#L50-L97`). INFERRED: park advertising at 250/65536 per tick is about +0.15 guests per second, a third of the max natural rate.

### 14.3 Research and development

VERIFIED `OR:management/Finance.cpp#L31-L37`, `OR:management/Research.cpp#L42-L48`, `#L330-L345`.

- Monthly cost by funding level: none 0, minimum 100, normal 200, maximum 400 (paid weekly at a quarter).
- Progress per update: 0, 160, 250, 400 (out of 65536 per stage, with a longer stage first: the expected-date code uses 0x20000 for designing and 0x10000 for completing a design). INFERRED: one new ride/scenery item takes roughly three stages' worth of progress units at the funding rate; absolute time UNKNOWN because the update frequency was not traced.
- What is unlocked is a scenario-defined ordered list of items (`researchItemsUninvented`), categories matching the ride categories plus scenery themes (UNKNOWN full list).

### 14.4 Income and pricing

- Entrance fee, ride price and shop price are per-guest charges; income per ride is `customersPerHour * (price - cost)` and updated about every 960 ticks (30 s) (`OR:ride/Ride.cpp`, first-pass read).
- Pricing consequences are derived from verified gates: a guest refuses a ride if price > 2 x ride value (4x less after paying entry: value/4 then x2 = price > value/2), enjoys "good value" if price <= value/2, and the satisfaction value term costs 5 or 30 happiness-target points when the price is at or above the (happiness-scaled) value. Entrance fee above the total ride value cuts the spawn rate to a quarter or a sixteenth (section 3).

## 15. Mapping to this repo's roadmap

Ground truth for this repo (from the shared brief and the code): carnival Octopus digital twin, deterministic 1/120 s physics, `Queue` and `Weather` modules, `Person` entity with name and weight (`src/Queue/FourDotnet.BoogaBooster.Queue/Domain/Person.cs`), ECS simulation core planned (changes 2 to 5). The roadmap changes 6 to 10 are scaffolded but empty (`openspec/changes/park-world-grid`, `ride-catalog`, `guests-and-needs`, `park-economy`, `coaster-editor` contain no artifacts yet at the time of writing).

### 15.1 Change 6 `park-world-grid`

Findings that matter:

- RCT is a tile map with paths, queue paths, height (z in 3-unit steps in OpenRCT2 code constants: UNKNOWN exact), ownership/"rights" of land (staff with no patrol area walk anywhere owned: `OR:entity/Staff.cpp` patrol notes), and per-tile elements (path, track, scenery). Guest AI reads the map locally: 5-tile radius for surroundings, 10-tile radius for ride candidates, 11x11 area for ride scenery, 3-tile radius for staff and entertainers, 7-tile radius for security (section 4, 5, 6, 9, 10).
- Path rules that gameplay depends on: queue path vs normal path (guests only enter queue paths when going to a ride), no backwards steps, wide-area rejection, slopes must connect.
- A path graph (not just a grid) is the right search structure: guests choose goals (ride, shop, exit) and pathfind with a node budget (15000) and a junction budget (6) (Peep AI wiki).
Suggested contract (INFERRED, to match the earlier brief's "intent not code" style): `ParkGrid` of tiles with `TileKind` (empty, path, queuePath, rideFootprint, scenery), integer height, owned flag; `PathGraph` derived view; range queries "tiles within N", "scenery count in NxN area", "nearest facility of kind K by walking distance". Radii above become named constants in the new module with source links in `docs/`.

### 15.2 Change 7 `ride-catalog`

- Mirror the data-driven ride type descriptor: category (6 values, section 12), build cost, upkeep, default price, `BonusValue` (guest-cap contribution), ratings base and modifiers, breakdown list, unreliability, footprint. All in data, not code branches. This is exactly what RCT does (`RideTypeDescriptor`, VERIFIED).
- Flat rides = formula over base ratings + one knob (section 6.4). The existing Octopus (Great Mill) is a flat ride whose operation option is speed; nearest RCT analogues by mechanics are Twist (rotations bonus) and Enterprise (operation-option bonus), INFERRED, not claimed as the original design. Excitement/intensity/nausea for the Octopus can come from the digital twin instead of a table: peak G from `docs/05-forces-and-g-forces.md` could feed the G-force sub-ratings in 6.3 (INFERRED).
- Stalls and facilities are catalog entries with no ratings and guest-need hooks (food, drink, toilet, first aid, cash machine, information kiosk).
- The catalog must expose per-ride: excitement, intensity, nausea (hundredths), reliability, price, value, status (closed/open/testing/broken), queue length and queue time. Section 6.5 gives the value formula.

### 15.3 Change 8 `guests-and-needs`

- Needs: happiness, hunger, thirst, toilet, nausea, energy, cash, plus intensity window and nausea tolerance (section 4). The simplest faithful core is: three need meters with thresholds (hungry <= 10, thirsty <= 25, toilet >= 160, nausea >= 140, tired energy <= 70), a happiness target that moves 1 per step toward the midpoint and is shifted by events, and a visible happiness that follows the target at 4 per step.
- Ride choice: highest excitement among rides in range passing intensity and nausea windows, price-vs-value and weather gates (section 5.1-5.2).
- Ride response: satisfaction lookup (section 5.3) is a table and is the key to a teachable, testable guest model.
- Determinism: use injected `Random`/sampler seeded per park; per-guest update stride (the RCT 512-tick window, section 4) is a natural job-system batch granularity for change 4.
- The `Person` type in the Queue module is the seed of the guest; it must grow `Happiness`, `PreferredIntensity`, `Nausea` first (see 16), then hunger/thirst/toilet/energy/cash as the guests module takes ownership (ECS component candidates: `Needs`, `Preferences`, `Wallet`).

### 15.4 Change 9 `park-economy`

- Revenue: entrance fee, ride price, shop price. Cost: construction, ride upkeep, staff wages, research, marketing, loan interest (sections 8, 9, 14).
- Spawn model: `P(spawn per step) = (50 + clamp(rating - 200, 0, 650)) / 65536` with the divide-by-4 penalties for crowding and entrance fee above total ride value, +/-25 percent per award (section 3). For a 1/120 s tick the per-step probability must be rescaled (RCT tick is 1/40 s): INFERRED, divide by 3 to keep the per-second rate.
- Park rating formula (section 2) is the economy's health signal; it can be implemented verbatim with fewer terms (section 17).
- The existing Queue `NiceWeather` multiplier (section 16) is a stand-in for the whole spawn model; change 9 should replace it by the rating-driven probability and keep weather as a modifier.
- Loans: simple weekly interest `loan * rate / 3276.8` with rate 10 gives about 10 percent per 32-week year (INFERRED), max loan 20000 in the default start.

### 15.5 Change 10 `coaster-editor`

- Coaster stats are measured from a simulated test run of the actual track, not hand-entered (section 6.2). The repo's deterministic fixed-step physics fits: run the train along the spline, record max speed, G (vertical, lateral), air time, drops, inversions, turns, length, then evaluate section 6.3.
- Track is a sequence of typed pieces, each piece having a pitch/roll profile and G-factors; the set of allowed pieces is per ride type (section 7.3). RCT's vehicle dynamics are a one-dimensional energy model with linear and quadratic drag and a slope term (section 7.1); this can be replaced by proper symplectic integration in the repo's physics style while keeping the same observable rule (energy from lift hill, losses from drag, mass reduces drag).
- Validity checks: closed circuit, a station, a lift or launch able to complete the circuit, minimum length/drop/speed requirements (those divide ratings by 2, not forbid), G limits that make guests refuse the ride (section 7.2).
- The existing 3D view (Three.js) and glTF pipeline (change 11) are the display layer; this research gives no rendering numbers.

## 16. Existing feature request, Queue and Weather modules

### 16.1 `docs/feature-requests/passenger-happiness.md` against RCT mechanics

The request: per-person Happiness (initial 65 to 85), Preferred ride intensity (50 to 100), Nausea (0), all on a 0..100 scale; queue wait over 5 minutes decays happiness exponentially; happiness peaks when ride intensity equals preference (50 percent or 100 percent of the max allowed G); nausea rises exponentially if preferred intensity is low but the ride is much more intense (difference at least 30 percent); each second at max G adds 25 percent nausea; a Rider Experience panel with four averages.

| Request | RCT counterpart (VERIFIED unless noted) | Difference |
|---|---|---|
| Happiness 0..100, start 65..85 | 0..255, start about 128 +/- 15..16 (section 4) | Rescale; RCT starts near the midpoint, request starts happy. Scale 0..100 vs 0..255 is a divide by 2.55 |
| Preferred intensity 50..100 single value | A range [min, max] with min 0..4 and max 3..10 or 15 on a 0..15 scale, widened by happiness (section 4, 5.2) | Request is a point preference; RCT is a window plus a nausea tolerance class. A single point can be modelled as the window center |
| Nausea starts 0 | nausea starts 0, target-following, grows on rides by `ride.nausea * f(happiness, hunger) >> tolerance` (section 5.3) | RCT nausea comes from the RIDE'S nausea rating times a tolerance divisor, not from a preference gap; the request ties it to intensity gap. Both are plausible; the request's rule is a game design choice |
| Queue wait over 5 minutes -> exponential happiness decay | RCT: linear -4 per window after 2000 ticks, leaves the queue at 4300 ticks if happiness <= 65, ride satisfaction penalty -10 at 2250 and -35 at 4500 (section 5.5) | RCT is piecewise-linear with thresholds; the request is exponential and 5 minutes long. Entertainers and queue TVs are RCT's mitigations (not in the request) |
| Happiness rises near preferred G | Satisfaction lookup on windows (section 5.3): in-window gives +70, one widening +35 to +50, outside up to -60 | Same idea (match = happy). RCT uses a table of 4 levels per axis rather than a continuous bump |
| Max G for over 1 s -> nausea +25 percent | RCT: only the whole-ride maximum G enters ratings; G over 2.80 lateral raises ride intensity/nausea ratings; over 4.00 (lateral) / 5.00 / -4.00 guests refuse the ride | The request's time-at-max-G rule has no RCT equivalent (UNKNOWN in RCT sources; it is a per-instant real-time model the digital twin can support) |
| Average bars (queue happiness, rider happiness/intensity/nausea) | RCT shows per-guest windows and aggregates only as park rating happy-guest share and ride satisfaction (stat 0..3, averaged over 20 samples, `OR:ride/Ride.cpp` income section) | A good fit to expose per-ride aggregates |

Recommendation (INFERRED): keep the request's 0..100 and Octopus-specific G-based rules for the live ride (the twin already has G-force), and add the RCT-derived tables as the layer on top: (a) map the request's single preference to the window center, (b) map nausea tolerance to an extra random per-guest trait, (c) feed the ride's measured peak G/intensity through the same sub-rating formulas to get ride-level excitement/intensity/nausea numbers that the guests module consumes in changes 7 and 8.

### 16.2 Queue module (existing code, VERIFIED by reading)

`src/Queue/.../Domain/Person.cs` (number, name, weight 30..150 kg), `Filling/ArrivalPlanner.cs` (headcount uniform 4..8 per cycle, scaled by weather `Ceiling * NiceWeather^Exponent`, floored to 0 when NiceWeather is 0), `QueueModuleOptions.cs` (cycle every 10 to 30 s, 30 to 60 s in bad weather, `BadWeatherThreshold` 0.4, groups of 1 to 5, `MaxQueueLength` 500, boardable group up to 32). Archived specs `openspec/specs/weather-driven-queue-fill`, `weather-driven-queue-multiplier`, `ride-queue`, `gondola-boarding`.

Connections to RCT:

- RCT's spawn is global at the park entrance with a rating-driven probability; the Queue module spawns per ride queue with a weather-driven count. For the tycoon, per-ride arrival should become "guests choose this ride" (section 5.1) rather than "guests appear in this queue". The existing `QueueFillerService` is a good test harness but not the final model (INFERRED).
- RCT's queue has a maximum length and a "queue full" flag that removes the ride from candidate lists (section 5.1: `RideFlag::queueFull`). `MaxQueueLength` 500 is the analogue; the ride needs a configurable per-ride queue limit (RCT has the ride option; exact name UNKNOWN).
- RCT's boarding is by vehicle capacity with minimum wait/load options (UNKNOWN details); the repo's "pairs stay together, groups are never split across boardings unless over 32" rule is a deliberate departure.
- The group concept (1 to 5 people) has no RCT counterpart: RCT guests are individuals who, in effect, queue independently.

### 16.3 Weather module

`WeatherConditionDto` carries temperature (Celsius), Beaufort wind, sunshine percent, precipitation type, regime and a `NiceWeather` indicator in [0, 1] (`src/Weather/FourDotnet.BoogaBooster.Weather.Abstractions/WeatherConditionDto.cs`). It is richer than RCT's discrete 9-state weather. RCT weather uses (section 11): temperature for item values and thirst, precipitation for ride refusal, shop-item refusal and brake-failure weight, month for the distribution. Mapping candidates (INFERRED): `Precipitation != None` -> "precipitating" gate; `TemperatureCelsius >= 21` -> thirst drain; `< 12` -> no ice cream/sunglasses; `NiceWeather` retained as the aggregate input for spawn modulation until change 9 introduces rating-driven spawn.

## 17. What to simplify for a workshop

Keep (high teaching value, small code):

1. Guest needs as five meters on 0..100: happiness, hunger, thirst, toilet, nausea; energy optional. Thresholds from section 4 rescaled by 100/255.
2. The happiness-target mechanic (visible value follows a target at a fixed rate; events move the target). This single trick gives smooth, testable behaviour.
3. Ride choice = best excitement among rides in range that pass intensity window, nausea gate, price gate (section 5.2 gates 5, 8, 9, 11).
4. Ride satisfaction as a lookup table over (intensity fit, nausea fit) plus queue penalty (section 5.3).
5. Park rating with 4 terms: happy-guest share, uptime, ride quality (distance from ideal average excitement/intensity), old-litter count. Spawn probability from rating with the entrance-fee-vs-ride-value penalty.
6. Flat rides = base ratings + one knob; one hand-tuned coaster archetype instead of the full 27-term proximity system.
7. Breakdown: per-check reliability decay and a single breakdown kind; one mechanic with an inspection interval.
8. Finance: loan, weekly interest, wages, ride upkeep, entrance fee, ride price, marketing as an optional extra spawn roll.

Drop or stub for the workshop: awards, proximity score, per-piece G tables (use physics from the repo instead), research tree (unlock by cash), vandalism and security guards, guest cash machines, souvenirs and temperature item values, 8-month calendar (use days or minutes), park-size and rights/land purchase, shelter/umbrella rules, RCT1 interest variant, guest favourite ride, ride-again probability tables, pathfinding budget (use A* with a fixed goal).

Rules of thumb (INFERRED): replace every `rand & mask` with an injected sampler; express timers in seconds with a `TimeProvider`; keep RCT's fixed constants in a single `Tuning` class per module with the OpenRCT2 permalink in a comment; keep RCT's rating units (hundredths) at the ride-catalog boundary so data from OpenRCT2 stays comparable.

## 18. Gaps and open questions

- Original RCT1/RCT2 manuals and RCT wikis not consulted: no independent confirmation that OpenRCT2 numbers equal the original (only code comments with original addresses).
- Climate weather probability tables, scenario default hunger/thirst/happiness/cash values, per-piece G-factor tables (`TrackData`), track placement validity rules, awards thresholds, hiring cost, exact research item count, ride crash probability, bin capacity and handyman speeds were not read: UNKNOWN.
- `timeInQueue` and `queueTime` units were inferred, not traced to the call frequency of `updateQueuing`.
- Some formulas marked "first-pass read" (intensity penalty bounds, scenery/proximity details, value age table, upkeep, flat-ride fixed values, cash machine, toilet/first-aid details) were read from fetched source text but the exact line anchors were not re-pinned; re-verify against the pinned commit before using them in a spec.
- Third-party library or tool names: none introduced here. Any future library choice (physics, pathfinding) must pass the currency check described in the shared brief at apply time.
