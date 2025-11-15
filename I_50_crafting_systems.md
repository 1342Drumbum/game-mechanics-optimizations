## 50. Crafting Systems (Material Sinks)

**Psychological Hook:** Collection completion drive - "143/200 blueprints found." Zeigarnik Effect - incomplete blueprints haunt players. Goal-setting theory (Locke & Latham) - specific unlock targets motivate. Loss aversion - "I have the blueprint and 90% of Cells, can't stop now." Delayed gratification creates anticipation. Mastery demonstration through rare blueprint acquisition. Status signaling ("I have Golden Scarab Outfit" = prestige). Discovery excitement finding new blueprint.

**Implementation Pattern:**
```
Crafting System Design:

Blueprint Acquisition:
├─ Drop Sources: Enemy kills (0.5-5%), bosses (50-100%), events (guaranteed)
├─ Total Pool: 100-300 blueprints
├─ Visibility: Show "New Blueprint!" prominently
├─ Collection Tracker: "Blueprints: 147/238"
└─ Unlock Requirements: Blueprint + Currency

Currency Costs:
├─ Common: 30-75 Cells/Darkness
├─ Uncommon: 75-150 Cells
├─ Rare: 150-300 Cells
├─ Ultra-Rare: 500+ Cells (prestige sinks)
└─ Total Completion: 15,000-35,000 currency

Crafting Categories:
├─ Weapons/Items (directly increase power)
├─ Mutations/Talents (build enablers)
├─ Cosmetics (status/customization)
├─ Permanent Upgrades (meta-power)
└─ Quality-of-Life (convenience)

Strategic Elements:
├─ Priority System: Unlock damage first, cosmetics last
├─ Opportunity Cost: "Unlock weapon or save for expensive mutation?"
├─ Pacing: Early blueprints cheap (30-50), late expensive (200-500)
└─ Diminishing Returns: Last 10% of collection = 40% of grind
```

Display collection progress prominently. Show blueprint sources ("Drops from Inquisitor"). Make early unlocks cheap, late expensive. Ensure blueprints drop before player has currency (creates "almost there" tension).

**Examples:**

1. **Dead Cells (Blueprint + Cell System):**
   - **Blueprint Sources:**
     - Elite enemies: 0.5-4% drop chance (Hand of the King's Rapier from Hand of the King: 100% first kill)
     - Timed doors: Reach in <2min → guaranteed blueprint room
     - Hidden areas: Secret walls → blueprint chambers
     - Curses: Cursed chests → item + blueprint
   - **Collection Stats:** ~200 weapons/mutations total. 238 blueprints including all variants/outfits.
   - **Unlocking Process:**
     1. Kill enemy → blueprint drops → "New Blueprint! Ice Bow"
     2. Survive to Collector (between-biome safe room)
     3. Spend 75 Cells → Ice Bow permanently unlocked
     4. Ice Bow now appears in drop pools
   - **Strategic Costs:**
     - Starter weapons: 30-50 Cells (Magic Missiles: 30)
     - Mid-tier weapons: 75-125 Cells (Flamethrower Turret: 75)
     - Late-game: 200-500 Cells (Giant Whistle: 250)
     - Mutations: 50-300 Cells (Dead Inside: 75)
     - Permanent stat boosts: 100-500 Cells (Health Flask upgrades: 100→200→300)
   - **Total Completion:** ~15,000-20,000 Cells = 150-300 runs.
   - **Psychological Hooks:**
     - Tracker shows "Blueprints: 187/238" = Zeigarnik Effect (51 incomplete)
     - Found blueprint but died before Collector = INTENSE frustration (loss aversion)
     - Carrying 95 Cells approaching Collector, need 75 for blueprint = relief dopamine when banking

2. **Hades (Multi-Currency Crafting):**
   - **Weapon Aspects (Titan Blood):**
     - 6 weapons × ~5 aspects each = 30 aspects
     - Costs per aspect: Rank 1 (1 Blood), Rank 2 (2), Rank 3 (3), Rank 4 (4), Rank 5 (5). Total/aspect: 15 Titan Blood.
     - All aspects max rank: 326 Titan Blood total
     - Earning: 1-2 per bounty (first clear/weapon), 1 per Heat milestone. ~100 hours for full completion.
   - **Mirror Talents (Darkness):**
     - 32 talents, ~35,365 total Darkness
     - Death Defiance (extra lives): 30/500/1000 Darkness = 1,530 total
     - Shadow Presence (+50% backstab): 100 Darkness across 5 ranks
     - Average run: 50-150 Darkness. Full mirror = 250-700 runs = 80-200 hours.
   - **Contractor Projects (Gemstones/Diamonds):**
     - Fountain Renovations: 5,000 Gems (cosmetic)
     - Vanquisher's Keep (administrative chambers): 3,135 Diamonds total
     - Minor Renovations: 50-200 Gems each
     - Extreme Measures Boss Modifiers: 30-50 Gems
   - **Relationship Unlocks (Nectar/Ambrosia):**
     - 6 Nectar/character → max relationship → 1 Ambrosia continues past max
     - ~30 characters × 6 Nectar = 180 Nectar
     - Companion unlocks: 5-6 Ambrosia each. 6 Companions × 6 = 36 Ambrosia.
     - Total needed: ~151 Ambrosia for full completion
   - **Crafting Strategy:** Prioritize Death Defiance (Mirror) → Strong weapon aspect → Other Mirror talents → Contractor cosmetics last. Titan Blood gates power, Darkness gates consistency, Gems purely cosmetic.

3. **Vampire Survivors (Gold → PowerUp Progression):**
   - **PowerUp System:** 11 permanent stat categories (Might, Amount, Cooldown, Duration, Area, Speed, Armor, Max Health, Recovery, Luck, Growth, Greed, Magnet, Curse, Revivals, Reroll, Skip, Banish, Move Speed).
   - **Costs (Escalating per Rank):**
     - Amount (projectiles): Rank 1 (500g), Rank 2 (1000g), Rank 3 (1500g), Rank 4 (2000g), Rank 5 (5000g). Total: 10,000g.
     - Might (damage): Similar scaling. 5 ranks, 5,000-10,000g total.
     - Revival (extra lives): 1000g → 2000g → 3000g → 10,000g → 20,000g. Total: 36,000g.
   - **Total Completion:** 111,450-202,000 gold depending on purchase order. Optimal order (always buy most expensive first) = 111,450. Random order ≈ 160,000. Worst order = 202,000.
   - **Earning Rate:** 500-1,500g per 30-min run. 111,450g ÷ 1,000g avg = 111 runs = 55 hours minimum.
   - **Strategic Order:** Buy Greed (+50% gold gain) FIRST → multiplies all future earnings. 50% more gold over 100 runs = 50,000g bonus = saves 30+ hours.
   - **No Blueprints:** Unlike Dead Cells, all PowerUps available immediately. Pure currency sink. Different psychology: grind gold, not hunt drops.

4. **Monster Train (Shard/Gold Upgrade Paths):**
   - **Unit Upgrades (Gold):**
     - Merchant of Magic: Unit upgrades 75-150g. "Add +10 attack", "Gain Multistrike", "Add Spell Weakness (apply debuff)".
     - Duplicates: Buy same unit = merge upgrade (stats double, abilities combine).
     - Spell upgrades: 75-125g. "Reduce cost by 1", "+20 damage", "Holdover (repeats next turn)".
   - **Concealed Caverns (Pyre Shards):**
     - Enter mini-dungeon, collect Pyre Shards (0-5 depending on performance)
     - Blacksmith: Spend Shards on unit upgrades (better than gold upgrades)
     - Cost: Taking Pyre damage = HP sacrifice. Strong run with 100+ Pyre HP? Spend 30 HP for 5 Shards → massive upgrades. Weak run with 30 HP? Can't afford.
   - **Artifact Crafting:**
     - No blueprint system. Artifacts appear in shops (100-200g) randomly.
     - ~100 artifacts total. Each run sees 8-12 in shops.
     - Strategic: "Skip weak artifact, save gold for Siren Song (artifact that boosts unit cap)."
   - **Caverns vs Shops:** Caverns = deterministic upgrades (HP→power conversion). Shops = RNG pool. Skilled players prioritize Caverns (reliable), weak players avoid (HP too precious).

5. **Into the Breach (Achievement Unlocks, Coins):**
   - **No Crafting Currency:** Instead, achievements unlock content.
   - **Unlock Structure:**
     - Win with squad → unlock 1-2 new squads
     - Rescue 3+ Time Pods → unlock mech variants
     - Perfect Islands (no grid damage) → unlock weapons
     - 30,000 score → unlock pilots
   - **Pseudo-Crafting (Reputation):**
     - Earn 8-12★ Reputation/island
     - Spend on Reactor Cores (3★), Grid Power (1★), or save for next island
     - Not persistent - use-it-or-lose-it per island
   - **Pilot Unlocks (Meta-Crafting):**
     - Pilots persist across runs (like Dead Cells blueprints)
     - "Rescue Pilot Kazaakpleth" → permanent +1 Reactor Core all future runs
     - Finding all pilots = scavenger hunt across events
   - **Different Paradigm:** Achievement-gated unlocks vs currency-gated. Skill check instead of grind. But same psychology: collection completion ("11/13 squads unlocked"), visible progress, Zeigarnik Effect.

**Synergies:** Collection Completion (#4), Multiple Currencies (#46), Granular Meta-Currency (#2), Mystery Unlocks (#41), Achievement Systems (#35), First Win Bonuses (#8)

**Session vs Meta:** 10% Session (blueprint drops during run), 90% Meta (spending Cells/Darkness between runs, permanent unlocks)
