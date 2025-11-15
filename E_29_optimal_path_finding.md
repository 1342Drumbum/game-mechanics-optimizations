## 29. Optimal Path Finding (Route Optimization)

**Psychological Hook:** Optimization puzzle appeals to analytical thinking (Kirton, 1976 - Adaption-Innovation theory). Efficiency gains create satisfaction - "I saved 2 minutes!" Decision-making tension at every fork. Regret/satisfaction from path consequences creates emotional investment. Mastery demonstration through optimization. Appeals to min-maxing psychology. Speedrunning community forms around optimization. Creates replayability through "what if I'd gone left?"

**Implementation Pattern:**
```
Path Choice System:
Decision Frequency: Every 1-3 minutes
Options per Decision: 2-4 visible paths
Information Shown: Partial (fog of war) to Complete (full map)
Commitment: Irreversible (can't backtrack) vs Exploratory (can return)

Path Differentiation:
Risk Axis: Safe route (low reward) ↔ Dangerous route (high reward)
Speed Axis: Long route (thorough) ↔ Short route (efficient)
Resource Axis: Gold route ↔ Power route ↔ HP route
Challenge Axis: Many easy enemies ↔ Few hard enemies

Optimal Path Factors:
- Time efficiency (speedruns): minimize minutes/stage
- Resource efficiency: maximize gold/XP per minute
- Risk-adjusted: minimize damage taken × maximize rewards
- Build-specific: melee character prefers different path than ranged
- Procedural adaptation: optimal path changes based on RNG layout

Metrics to Display:
- Stage completion time: "Prisoners' Quarters: 3:47 (Best: 2:21)"
- Treasure found: "7/12 chests opened"
- Enemies defeated: "47/89 killed (efficiency: 53%)"
- Risk taken: "2 elite fights, 5 curse events"
```

Generate procedurally but with patterns (speedrunners can learn generation rules). Provide path preview (show reward types ahead). Balance paths so multiple routes are viable. Track player statistics to show optimization potential. Allow experimentation without excessive punishment.

**Examples:**

1. **Spelunky / Spelunky 2:** Every level randomly generated but follows rules enabling pattern recognition. Speedrunners "see patterns in the way the levels are generated to make good guesses about which direction they should go." Pathing rules: go down wherever possible, avoid idol/altar rooms unless required, spot exit from above. Skilled players traverse holding run button continuously (optimal movement). Each level: 15-45 seconds for speedrunners vs 2-5 minutes for casuals. Critical decisions: shortcut vs full exploration (10-30 second differences), ghost trigger timing (forces 2:30 time limit), shop theft (risk 2 HP for $40k-60k gold). World record Spelunky HD: 1:38 (vs 3-8 hour first playthrough). Jetpack/Cape acquisition: saves 20-40 seconds rest of run. Mattock from snake pit: 10-15 second long-term savings. Teleporter wall-clipping: skips 30-60% level geometry = 2-5 second per-level savings.

2. **Slay the Spire:** Each Act: 15 rooms in branching map, player chooses path node-by-node. Optimal pathing: 5-7 elite fights across Acts 1-3 (too many = HP attrition death, too few = weak deck). Path decisions every 30-90 seconds. Information: can see 2-3 nodes ahead + entire floor layout. Critical junctions: campfire vs shop vs treasure (resource allocation), elite vs avoid (risk/reward), event rooms (~40 events, some beneficial, some punishing). Build-specific routing: deck needs cards = hit every shop, needs relics = fight elites, needs HP = prioritize campfires. Act 1 optimal: exactly 2 elites + 1-2 shops + 9-11 fights + 1-2 campfires = strongest Act 2 entry. Time differential: optimal path 8-12 minutes/Act vs greedy path 15-20 minutes. A20 high win-rate players (~65-75%) demonstrate consistent optimal pathing vs A20 average players (20-40%).

3. **Dead Cells:** Biome branching with ~12 total biomes, 8-10 visited per run. Each biome offers 2-3 exit paths. Timed doors force efficiency: Prisoners' Quarters door at 2:00 (vs 3-5 minutes average). Route optimization: Toxic Sewers (2:00) vs Promenade (2:30) = 30-second differential × shorter/longer path. Scroll optimization: some paths offer more Scrolls of Power (+15% damage/health per scroll, 25-35 per run). Gear optimization: different biomes drop different weapon colors (Brutality/Tactics/Survival). Blueprint hunting: specific enemies in specific biomes for unlocks. Speedrun Mode shows per-biome times: "Prisoners' Quarters: 1:47 (Best: 1:32)". World record optimization: 12:54 full run requires frame-perfect routing + execution. Casual route: 30-45 minutes, Optimized: 15-25 minutes, Speedrun: <15 minutes. Route knowledge differential: 20-40% time savings.

4. **Hades:** 4 biomes × 3-4 chambers each = 15-20 chambers/run at 30-90 seconds each. Chamber choice: Combat vs Treasure vs Boss vs Shop vs Fountain (healing). Information: next chamber reward shown (Darkness, Gems, Boon, Hearts, Gold, Centaur Heart, Pom of Power). Optimal routing: prioritize boons early (build foundation), gold mid-game (buy key items), Poms late-game (scale existing boons). God choice routing: 9 gods, want 2-4 god focused build = refusing certain gods. Chaos Gates: enter curse (3-4 chambers), gain +50-100% stats. Erebus Gates: extra challenge chamber, requires >80% HP, grants unique reward. Speedrun routing: skip all non-boon chambers = 7-10 minute runs. World record: 5:16 IGT (Hera Bow), top 100 breaks 7:00. Casual run: 20-35 minutes, Optimized: 12-18 minutes. Room choice adds 30-90 seconds decision-making every 1-2 minutes. God Pool manipulation: holding keepsakes guarantees specific god first chamber, then remove for organic randomness.

5. **FTL: Faster Than Light:** 8 sectors with 20-25 beacons each. Each sector: visit 5-8 beacons before Rebel Fleet catches up. Branching map with incomplete information (can see adjacent only). Optimal routing: hit 2-3 stores per sector (30-40 scrap profit each), maximize events before exits, avoid empty beacons when possible, balance between thoroughness (more encounters = more scrap) vs speed (stay ahead of fleet). Fleet advancement: 1 beacon/turn (2 at later sectors). Path prediction: experienced players recognize "Rebel-Heavy" vs "Safe" sectors = routing adjusts. Critical decisions: pursue nebula (cloak enemy sensors, slows fleet) or standard (faster fights), Distress Beacon events (50% good, 50% combat). Scrap efficiency: optimal routing = 120-180 scrap/sector vs casual 80-120. Hard mode + optimal routing = ~55-65% win rate vs 30-40% casual routing. Each path decision: 10-30 seconds thinking, impacts next 3-6 minutes.

**Synergies:**
- Perfect Information (#26) - complete information enables optimal routing
- Knowledge Checks (#28) - requires knowing what paths offer
- Speedrun Potential (#30) - optimal routes critical for times
- High-Risk Paths (#17) - risk/reward influences routing
- Push-Your-Luck (#16) - "one more room" routing decisions
- Procedural Pools (#12) - randomization makes each routing unique
- Strategic Depth (#40) - routing is core strategic decision

**Session vs Meta:** 70% Session (routing decisions each run), 30% Meta (learning optimal patterns)
