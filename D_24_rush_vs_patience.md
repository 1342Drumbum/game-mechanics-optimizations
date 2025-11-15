## 24. Rush vs Patience (Speed vs Thoroughness Trade-Offs)

**Psychological Hook:** Approach-avoidance conflict (Miller, 1944) creates cognitive tension. Regret theory - players agonize over "what if I explored more?" or "what if I rushed?" Satisficing vs maximizing personality expression. Risk tolerance demonstration. FOMO (fear of missing out) pulls toward exploration, time pressure pushes toward speed. Creates diverse playstyle validity - both viable but require different skills.

**Implementation Pattern:**
```
// Route A: Speed (high risk, time bonus)
speedrun_route:
    time_limit = tight
    loot_multiplier = 0.5
    enemy_encounters = 0.7
    bonus_reward = legendary_item OR +200% gold
    success_rate_requirement = high_skill

// Route B: Thorough (low risk, full loot)
exploration_route:
    time_limit = generous OR none
    loot_multiplier = 1.5
    enemy_encounters = 1.3
    bonus_reward = steady_accumulation
    success_rate_requirement = medium_skill

// Hybrid approach
adaptive_strategy:
    if player_power > threshold:
        switch_to_speed_route()
    elif player_health < 30%:
        switch_to_safe_route()
    evaluate_every_checkpoint()
```

Balance rewards: speed gives 1 premium reward, thorough gives 3-5 good rewards (roughly equal value). Signal choice clearly: "Timed door (2 min): Legendary" vs "Secret path: 3 chests, 5 encounters." Allow strategy switching mid-run based on situation. Ensure both strategies viable at high difficulty (no trap options).

**Examples:**

1. **Dead Cells:** Fundamental tension between timed doors and thorough exploration. Speed path: ignore 50-70% of enemies, skip biome branches, hit all timed doors (3 blueprints + 1000+ gold total). Thorough path: explore all branches, break all doors, kill all enemies (90+ kills for 4/5 kill doors), accumulate 15-20 weapons for upgrading. Doors offer: +1 quality tier items (S-tier vs A-tier) but sacrifice 200-400 Cells and 1000-2000 gold from unexplored zones. Meta strategy: early-game speedrun (weak build needs door items) → mid-game thorough (strong build farms Cells for meta upgrades) → late-game speedrun (5BC punishes prolonged combat). Cursed Chests accelerate tension: take 10-curse for immediate power OR wait for safe shop item. Scrolls (stat boosts) distributed evenly, so thorough path gets +5-7 scrolls (significant power difference).

2. **Slay the Spire:** Path choice creates thoroughness vs risk spectrum. Act 1 optimal: 5-7 elite fights (rare relics + 150-200 gold) balanced against HP attrition. Too few elites (0-3) = weak deck, too many (8-10) = dead from damage. Every floor: choose next chamber from 2-3 options: Combat (guaranteed, 10-20 gold + card), Elite (risky, rare relic + 25-40 gold), Event (random outcome), Shop (spend gold), Campfire (heal OR upgrade card). Hallway battles: 20-30s, 12-20 gold. Elites: 60-120s, 25-40 gold, rare relic, 20-30 HP damage typically. Route calculation: "Can I afford 2 more elites before Act 1 boss?" considers current HP (60+? safe, 30-? risky, <20? skip), current deck strength, remaining campfires (heal vs upgrade opportunity cost), gold for shop visits. Late Act 3: skip combats when deck strong (reduce draw RNG risk), take events (narrative variety), beeline to boss. Time investment: thorough Act 1 = 25-30 min, rushed = 15-18 min.

3. **Hades:** Chamber path choice creates moment-to-moment decisions. Each chamber: choose 1 of 2-3 doors showing rewards. Speed priorities: Boon chambers (god power), Pom chambers (upgrade boons), avoid Story chambers (0 combat power), minimize Trial chambers (2 consecutive fights for 1 reward). Thorough priorities: Erebus gates (Chaos gates: +1 chamber, strong curse for 3-4 rooms, powerful boon after), Infernal Troves (locked chests, enemy key-holders, 100-200 gold + boon), all Charon shops (reroll Well: 200 gold, key items). Satyr Sack (Temple of Styx): found in 1 of 5 tunnel branches, forces searching 1-5 paths (average 3). Speedrunners memorize: optimal boon combos (target 12-15 specific boons for build), skip 20-30% of chambers, avoid Erebus/Troves (time investment), complete in 12-18 minutes (vs casual 35-45 min). Tradeoff: speedrun risks weak build from limited boon choices, thorough exploration guarantees power but extends run time (death opportunity increase).

4. **Risk of Rain 2:** Time = difficulty curve creates natural rush vs patience tension. Difficulty coefficient increases +15% per minute (Monsoon). Thorough exploration: loot all 8-12 chests per stage (150 gold × 8 = 1200 gold spent), kill all Newt altars (10 Lunar coins), use all shrines. Time cost: 8-12 minutes per stage. Difficulty at stage completion: minute 24 = Very Hard (2.4 difficulty coefficient). Speed strategy: loot 2-3 chests only, rush to teleporter, complete stage in 4-5 minutes. Difficulty at stage completion: minute 15 = Hard (1.5 difficulty coefficient). Power difference: thorough = 10 items by stage 3, speed = 4 items by stage 3, BUT enemies have 60% less HP/damage in speed strategy. Community consensus: optimal = 5 minutes per stage (3-5 chests, balance power vs difficulty), sub-5 minute stages = racing difficulty curve (skill expression), 10+ minute stages = falling behind (exponential difficulty overwhelms linear power). Decision point at every chest: "Worth 2 minutes extra difficulty for 1 item?" Scrapper machine: convert 5 white items → 1 green (thorough-path payoff). 3D Printer: convert all items to copies of 1 (speed-path consistency).

5. **Vampire Survivors:** No explicit choice system but implicit strategy split. Speed strategy: rush weapon evolution (maximize 1-2 weapons to level 8, ignore most items, hit 10-minute chest ASAP), creates focused build (Vandalier + Peachone + Ebony Wings = fast evolution = 1000+ DPS by minute 12). Thorough strategy: balanced levelup (6 weapons + 6 items, all to level 5+), creates rainbow build (moderate power across board = 600 DPS spread across 6 weapons). Speed pros: evolution spike carries to minute 30, fewer weapons = more focused damage, less screen clutter. Thorough pros: better coverage (flying + ground + penetrating), more forgiving to mistakes, more synergies discovered. Stage influences: Library (small) favors speed, Dairy Plant (huge) favors thorough. Character influences: Imelda (starting weapon) favors speed evolution, Krochi (lots of items) favors thorough. Arcana system: can force thorough (require 3+ evolutions for scaling bonuses). Gold gain: speed = 500-800 gold (10-min rush), thorough = 1500-2500 gold (full exploration). Meta upgrades: Amount (+projectiles) makes thorough weaker (caps reached faster).

**Synergies:**
- Countdown Timers (#23) - Timers force rush vs patience decisions
- High-Risk Paths (#17) - Speed routes often high-risk, high-reward
- Knowledge Checks (#28) - Optimal routing requires game knowledge
- Adaptive Difficulty (#13) - Performance affects optimal strategy
- Path Choice (#40) - Multiple routes enable strategy diversity
- Speedrun Potential (#30) - Rush strategy naturally creates speedrun categories
- Resource Management (#47) - Thorough exploration accumulates resources
- Economy Systems (#44) - Speed sacrifices gold/currency for time

**Session vs Meta:** 70% Session (strategic choice made within run based on current situation), 30% Meta (learning which strategy suits playstyle and which situations demand specific approaches)
