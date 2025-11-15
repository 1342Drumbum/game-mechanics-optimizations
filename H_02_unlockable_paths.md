## 42. Unlockable Paths (Branching Routes)

**Psychological Hook:** Exploits **autonomy need** from Self-Determination Theory (Deci & Ryan, 2000) - choosing paths creates ownership. **Opportunity cost** creates meaningful decisions and potential regret/satisfaction. **FOMO** (fear of missing out) motivates replaying to experience alternate routes. **Exploration satisfaction** rewards surveying all options. Path choices demonstrate **mastery** - skilled players access harder routes for better rewards. **Novelty seeking** prevents boredom from repetitive content.

**Implementation Pattern:**
```
Path Generation:
Standard Path:
- Difficulty: Baseline (100%)
- Rewards: 100% (standard progression)
- Time: 100% (intended duration)

Alternate Path A (Risk Path):
- Difficulty: +40-60% enemy HP/damage
- Rewards: +50-100% currency, +1 rarity tier
- Time: +20-30% duration (more encounters)
- Unlock: Complete specific objective or have item

Alternate Path B (Speed Path):
- Difficulty: -20% enemy count
- Rewards: -25% total loot
- Time: -40% duration (skip encounters)
- Unlock: Find hidden shortcut or sacrifice resources

Path Selection:
- Display before commitment: "Hard Path (Elites) vs Normal"
- Make visible on map: branching corridors, colored doors
- Allow informed decisions: show rewards/risks
- Ensure all paths viable (no traps)
- Lock commitment: can't backtrack after choosing

Reward Balance:
Risk Path Value = Base × 1.75 × Skill Factor
Speed Path Value = Base × 0.6 (time savings justifies)
Secret Path Value = Base × 2.5 (one-time discovery bonus)
```

**Examples:**

1. **Slay the Spire:** Act 1-3 branching paths visible on map. Path types: (1) **Elite Path** - 5-7 elite fights (vs 2-3 standard). Elites have 2-4× HP and deal 150% damage. Reward: Guaranteed Rare relic (17% pool) + 25-40 gold (vs normal 10-20). Risk: -40-60 HP over act (6-8 hits taken). Optimal: ~5 elites total across 3 acts for max power. (2) **? Path** - 50% chance beneficial (treasure/shrine), 30% neutral (merchant), 20% punishing (curse/damage). Average value 115% of normal fight. (3) **Shop Path** - Costs 75-120 gold entry + item purchase. Break-even if buying $250+ value. (4) **Campfire Path** - Heal 30% HP or upgrade card. Value depends on HP% and deck quality. Optimal routing requires 8-12 campfires across full run. Path decisions at 40+ nodes/run. Speedrun routing saves 3-5 minutes by skipping shops/campfires.

2. **Dead Cells:** Biome routing with 6+ branches per run. **Normal Route** (easy): Promenade → Ramparts → Black Bridge → Stilt Village → Clock Tower → High Peak Castle. Scroll average: 24 scrolls. **Cursed Route** (hard): Toxic Sewers → Corrupted Prison → Ancient Sewers → Graveyard → Cavern → Guardian's Haven. Scroll average: 32 scrolls (+33% power). Cursed Chests: +1 equipment, take 10 curse (one-hit death until 10 killstreak no-damage). **Timed Doors** appear 2-4× per run: 2:00, 8:00, 15:00 timers. Require speed routing, skip combat. Reward: Guaranteed +1 scroll + blueprint. Missing = permanent loss. Hit 60% of doors = optimal (sacrificing routing for all 100% loses HP). **Boss Cells** lock routes: 4BC+ adds Spoiler Boss route for exclusive weapon unlocks. Route complexity: 8 starting biomes × 4 mid-game paths × 3 late-game options = 96 possible routes.

3. **Enter the Gungeon:** Secret floor routing extends runs +30-50% duration. **Oubliette** (floor 1.5): Requires extinguishing fireplace in Keep (1/4 rooms) with Water barrel or Potion. Entry costs 2 keys. Reward: +1 heart container + shop + 3 extra rooms = ~4 chests value ($400-600). Always profitable if found early. **Abbey of the True Gun** (floor 2.5): Requires visiting Oubliette + finding Old Crest + carrying to Gungeon Proper + taking zero damage + placing on altar. Spawn rate: 20% (altar room). Reward: Lies weapon (top-tier DPS) + 2 item pedestals. **Resourceful Rat's Lair**: Requires 6 cheese wedges → fight Rat (2-phase boss) → punch-out minigame. Reward: 20-40 items from stolen chest pool (Rat steals 1 item per floor). RNG-heavy minigame: 60% success rate. **Bullet Hell** (floor 6): Requires defeating 4 character pasts (15-25 hours each). Final true ending floor. Path decision points: 15-20 per run (key usage, resource trading for shortcuts).

4. **Risk of Rain 2:** Stage selection branches (8 environments × 4 stages + alternates). **Standard Loop**: Balanced enemy density (100%), time scaling normal. **Hidden Realms**: (1) **Void Fields** - Enter via Bazaar Between Time (purple portal in pillar shaft). Challenge: 9 waves with -50% healing while inside + time continues = difficulty spike. Reward: Guaranteed Legendary item (0.5% normal drop rate → 100%). Time cost: +5-7 minutes. (2) **Gilded Coast** - Gold Portal spawn (4% chance) or Halcyon Seed use. Contains Aurelionite boss. Reward: guaranteed Red item + Halcyon Seed (deployable ally). **Bazaar Between Time** - Blue Portal (25% spawn). Shop with Lunar items (8 available, costs 1 Lunar Coin = 1.5% drop rate). Risk/reward: Shaped Glass (+100% damage, -50% HP per stack). **Commencement** (Stage 6): Standard final boss. **A Moment, Fractured** (obliteration): Alternative ending via Celestial Portal. Beads of Fealty: Access **A Moment, Whole** (4 Twisted Scavengers fight, 10 Lunar Coins reward). **Void Locus**: 5% portal spawn post-DLC. Leads to alternate final boss (Voidling). Routing decisions: 5-10 stage portals per run affecting items/time/difficulty.

5. **Noita:** Open-world vertical exploration with ~20 major path branches. **Standard Descent**: Holy Mountain after each biome (7 total) for perks + healing + shop. Linear: Mines → Coal Pits → Snowy Depths → Hiisi Base → Underground Jungle → The Vault → Temple → The Laboratory → final boss. **Parallel Worlds**: Dig through 30,000-pixel-wide rock barriers. East/West worlds: identical layout, no Holy Mountains after first 6, only 10/11 orbs (Lava Lake missing). Cursed health pickups (+44% max HP + continuous damage). **Secret Areas**: (1) **Tower** - West of spawn, requires flight + digging. Contains 16 floors of challenge gauntlets. (2) **Pyramid** - East of spawn, branching puzzle dungeon. (3) **Lake** - South, requires water immunity exploration. (4) **Abandoned Alchemy Lab** - Hidden in Coal Pits' right side. (5) **Fungal Caverns** - Secret biome parallel to Hiisi Base. **Orb Quest**: 33 Orbs for secret ending (11 main world + 11 East + 11 West parallel). Requires 100+ hours of exploration. NG+ randomizes 22/33 orb locations. ~500+ distinct paths through world (most games see <20% of content).

**Synergies:**
- High-Risk Paths (#17) - Difficult routes offer proportional rewards
- Path Choice (#40) - Decision-making creates strategic depth
- Secret Finding (#41) - Hidden routes reward exploration
- Build Variety (#15) - Different builds enable different routes
- Time Pressure (#21) - Timed doors force routing decisions
- Procedural Generation (#12) - Randomized layouts ensure variety

**Session vs Meta:** 85% Session (path choices made and resolved within single run), 15% Meta (learning optimal routing strategies and which paths suit which builds)
