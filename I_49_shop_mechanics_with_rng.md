## 49. Shop Mechanics with RNG (Reroll Systems)

**Psychological Hook:** Variable reward schedule (Skinner) in controlled environment. Agency through reroll choice - "bad shop? spend to improve." Gambling psychology - "next reroll might have Dead Branch." Sunk cost escalation - "$5→$6→$7 spent rerolling, can't stop now." Near-miss effect - seeing good-but-not-perfect item triggers "one more." Mastery demonstration through optimal reroll timing. Strategic resource management (gold vs items).

**Implementation Pattern:**
```
Shop Reroll Design:

Base Shop Structure:
├─ 5-8 items from pool (60-200 total items)
├─ Rarity distribution: Common 60%, Uncommon 30%, Rare 10%
├─ 1 guaranteed discount (25-50%)
└─ Items removed when bought (no restock unless relic)

Reroll Mechanics:
├─ Cost: Starting $5, +$1 each reroll, reset per shop
├─ Alternative: Flat $2-3 with voucher discount
├─ Restock: 2-5 items replaced with new RNG
├─ Limitations: Doesn't restock special items (vouchers)
└─ Strategic: "Spend $5-10 rerolling or $50 on suboptimal item?"

Reroll Modifiers:
├─ Voucher: "Reroll Surplus" (-$2 reroll cost)
├─ Voucher: "Reroll Glut" (-$2 more, unlocks after 100 rerolls)
├─ Luck Stat: Increases rare item odds per reroll
└─ Interest: Banking cash vs spending on rerolls

Psychological Hooks:
├─ Display reroll count (commitment escalation)
├─ Show "New!" on rerolled items (novelty)
├─ Guarantee improvement after N rerolls (pity)
└─ Limit rerolls (10 max = scarcity) OR unlimited (spending limit)
```

Reset reroll cost between shops. Increase cost per reroll (prevents spam). Allow viewing before committing. Display total spent. Consider pity system after 5-10 rerolls.

**Examples:**

1. **Balatro (Escalating Reroll Economy):**
   - **Shop Structure:** 2 Jokers ($3-8), 2 Booster Packs ($3-6), 1 Voucher ($10, appears randomly). Items bought → removed. Packs/Vouchers don't restock.
   - **Reroll Cost:** $5 base, +$1 each reroll ($5→$6→$7→$8), resets when entering next shop. Unlimited rerolls if you have money.
   - **Voucher Discounts:**
     - Reroll Surplus: Rerolls cost $3 (normally $5)
     - Reroll Glut (upgraded): Rerolls cost $1 (unlocked after 100 total rerolls across all runs)
     - Combined: Both vouchers stack → $1 rerolls
   - **Strategic Decisions:**
     - Ante 1: $15 budget. Spend $12 on Joker or reroll 2× ($5+$6=$11) hoping for better?
     - Ante 4: $60 budget. See "Blueprint" Joker ($8) which copies adjacent. Worth $20+ in rerolls to find Baron/Multiplication Joker to combo?
   - **Interest Conflict:** Every $5 unspent = $1 interest/round. Spending $25 rerolling loses 5 rounds of $5 interest = $25 opportunity cost. Experienced players calculate: "This reroll costs $6 now + $1.20 future interest = $7.20 real cost."
   - **Luck Scaling:** Luck stat (from Jokers) increases rare item appearance. Rerolling with +10 Luck = ~15% rare (vs 5% base). Incentivizes reroll-heavy strategies.

2. **Slay the Spire (Limited Reroll via Relics):**
   - **Shop Structure:** 5 cards (2 Attack, 2 Skill, 1 Power), 3 relics, 3 potions, card removal service. Fixed layout per shop seed.
   - **No Base Reroll:** Cannot reroll by default. Shop determined by seed. BUT:
     - **Prismatic Shard (Rare Relic):** Colorless cards replaced with cards from ANY class. Changes shop pool dramatically.
     - **Membership Card (Shop Relic):** 50% discount on ALL items, including removal. Doesn't reroll but makes "bad shop" affordable ($150 rare → $75).
     - **The Courier (Shop Relic):** -20% prices AND restock items when bought. Buy Offering → disappears → 2 new relics appear. Pseudo-reroll via purchase.
   - **Gambling via Events:**
     - "Wheel of Fortune" event: Spin for random card/relic/curse/gold. Gamble runs on RNG manipulation.
     - Library Event: Choose 1 of 20 cards. Reroll choice via "skip."
   - **Strategic Scarcity:** No reroll = must adapt to RNG. "Bad shop?" Skip it, save gold for next. Creates acceptance vs chase decision. Different psychology than Balatro's agency.

3. **Hades (God Pool Manipulation):**
   - **Shop Structure:** Well of Charon appears every 3-5 chambers. 3 items: Boons (god RNG), health/gold, attack/cast upgrades. Fixed prices (100-200).
   - **Reroll Mechanics (Boon Pool, not shop):**
     - Fated Persuasion (Mirror talent): 1-10 charges. Reroll single boon choice (3 options → new 3 options). Reroll cost: 1 charge (non-renewable per run).
     - God Gauge System: Ignored gods increase appearance rate. If you avoid Ares for 10 chambers, Ares +60% spawn chance. "Soft reroll" by avoidance.
     - Keepsakes: Force specific god first 4 chambers. Athena Keepsake → guaranteed Athena chamber 1-4. Deterministic manipulation vs RNG.
   - **Strategic Meta-Gaming:**
     - Want Dionysus + Ares duo boon "Curse of Vengeance"? Equip Ares keepsake → chambers 1-4 Ares guaranteed. Switch to Dionysus keepsake chamber 5 → Dionysus appears. Both in pool → duo available.
     - Used all 10 Fated Persuasions? Stuck with bad boons. Scarcity enforcement.
   - **Different from Shop Reroll:** Can't reroll shop items. But boon rerolling = 80% of run power. Chamber rewards matter more than shops.

4. **Dead Cells (Gear Reroll via Loot Pool):**
   - **Shop Structure:** Appears every biome. 2-3 weapons (1,500-4,000 gold), 1 mutation respec (1k→2k→4k→8k escalating), 1 skill/deployable.
   - **No Direct Reroll:** Cannot reroll shop inventory. BUT mutation reroll costs escalate like Balatro (1,000→2,000→4,000→8,000 cap).
   - **Gear Reroll via Drops:** Every enemy drops random weapon. Cursed chests → random item. "Reroll" by playing more, finding more drops. Different paradigm: time = rerolls, not gold.
   - **Mutation Strategy:** Found powerful Tactics-scaling weapon (Electric Whip) but have Brutality mutations? Pay 4,000g to respec mutations → Tactics mutations. Gold converts to build flexibility.
   - **Recycling Tube (Unlock):** Refund weapons/skills for gold. Picked bad weapon? Recycle for 50% gold back. Pseudo-reroll by converting unwanted → currency → shop purchase.

5. **Monster Train (Shop Floors with Duplicates):**
   - **Shop Structure:** Merchant of Magic/Steel appears every 2 rings. 5-8 units, 3-5 spells, 2-4 artifacts. Fixed prices (75-200 gold).
   - **No Reroll Button:** Cannot reroll shop. BUT multiple shops per run (4-6 total). Bad shop 1? Skip, wait for shop 2.
   - **Duplicate Unit System:** Buying duplicate units = merge upgrade. First Shroud Spike (2 cost, 10 attack) → Second Shroud Spike in shop → merge = 2 cost, 20 attack, +abilities. "Reroll" less important when duplicates amplify.
   - **Artifact Shops:** Separate shop type with only artifacts. Guarantees quality. "Bad RNG on unit shop? Artifact shop compensates."
   - **Strategic Resource:** Gold scarce (200-500/run). Can't afford rerolls anyway. Binary decisions: "Buy this or skip?" No reroll agonizing like Balatro.
   - **Concealed Caverns Event:** Mini-dungeon with Blacksmith. Spend Pyre Shards (HP damage) to upgrade units. Converts HP → deterministic upgrades. Alternative to shop RNG.

**Synergies:** Multiple Currencies (#46), Variable Rewards (#3), Push-Your-Luck (#16), Pity Systems (#20), Strategic Depth (#40)

**Session vs Meta:** 95% Session (reroll decisions within run, gold spent), 5% Meta (learning optimal reroll strategies, unlocking reroll discounts)
