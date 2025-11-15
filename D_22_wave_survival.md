## 22. Wave Survival (Clear/Survive Cycles)

**Psychological Hook:** Creates natural rhythm of tension → relief → tension (emotional rollercoaster). Goal gradient effect accelerates motivation as wave completion nears. Zeigarnik Effect - incomplete waves create cognitive tension. Provides micro-goals (survive wave 7) nested in macro-goal (reach wave 20). "Just one more wave" effect mirrors "just one more turn" addiction. Relief periods allow dopamine restoration, preventing burnout. Completion satisfaction triggers reward circuits.

**Implementation Pattern:**
```
for wave in 1..max_waves:
    combat_phase:
        duration = base_duration + (wave × duration_increase)
        enemies = base_count × difficulty_multiplier^wave
        spawn_until(time_elapsed >= duration OR all_enemies_dead)

    shop_phase:
        duration = 15-30 seconds
        allow_purchases()
        restore_partial_resources()
        display_progress: "Wave 7/20 Complete - Next: Elite Wave"

    if wave == milestone_wave:
        spawn_boss() OR grant_bonus_reward()
```

Wave duration: 20-90 seconds (escalating). Shop/break: 10-30 seconds fixed. Boss waves: every 5-10 waves. Victory condition: survive N waves OR survive infinite with score. Display: "Wave 8/20 (40%)" with progress bar.

**Examples:**

1. **Brotato:** 20 waves = victory. Wave 1 = 20s, +5s each wave, caps at 60s (wave 9). Each wave: combat → shop phase (10-20s) → repeat. Shop offers 4 items from pool matched to build tags (melee weapons if high melee damage). Wave 10: mid-boss (Hordes with 2× enemy count). Wave 20: final boss (Mother or Elite Worm with 500+ HP vs 30 HP normal enemies). Progress display: "Wave 14/20 - Next: Elite Wave." Stats show: enemies killed, damage dealt, items purchased. 10-15 minute run length typical. Endless Mode extends beyond wave 20 with exponential scaling (225% HP increase per wave).

2. **Deep Rock Galactic (main game):** Swarm-based wave system. Point Extraction: 7-10 waves over 20-25 minutes. Egg Hunt: waves trigger when collecting eggs. Each wave: warning → 5s prepare → 30-60s swarm → 20-30s lull → repeat. Swarm composition scales with Hazard Level (1-5) and player count (1-4 players). Hazard 5 with 4 players: 50+ enemies per wave, 3-5 Praetorians (tank enemies), 1-2 Bulk Detonators (mini-bosses). Waves feature enemy types: Grunts (5-10 HP), Praetorians (450 HP), Oppressors (1000 HP). Resupply pods between waves restore 50% ammo. Mission Control voice warns: "Swarm incoming!" creating anticipation.

3. **Vampire Survivors (stage completion):** Not explicit waves but time-based spawn tiers. Every minute = difficulty spike. Minute 10: weapon evolution triggers (chest appears). Minute 15: elite enemies (large versions with 10× HP). Minute 25-28: boss spawns (Death, Reaper). Minute 30: automatic victory if survived. Progress bar at top shows time remaining until evolution/victory. No shop phases - continuous combat with periodic power spikes from levelups (every 30-45s early, every 90-120s late). XP gems auto-collect at 100% magnet range late-game (implicit relief mechanic).

4. **Helldivers 2:** Operation-based wave survival. Typical mission: 15-25 minutes with 4-6 objectives. Each objective triggers: setup phase (20-30s: place equipment, position team) → defense phase (60-90s: waves of 10-20 Terminids or Automatons) → extraction phase (2 minutes: fight to extraction, dropship arrives). Enemy types scale with difficulty (1-9): Difficulty 7+ adds Bile Titans (2000+ HP), Chargers (800 HP charging rams). Limited ammunition creates attrition: primary weapon (200-400 rounds), support weapons (1-4 magazines), Stratagems (cooldown-based airstrikes/resupply). Reinforcements limited (5-20 per mission) creating death pressure.

5. **Call of Duty Zombies (classic mode):** Infinite waves (rounds) with exponential scaling. Round 1: 6 zombies, 150 HP each, slow movement. Round 10: 43 zombies, 1,051 HP each. Round 20: 61 zombies, 3,451 HP each. Round 50: 96 zombies, 47,295 HP each. HP formula: `Round 1-9: 150 × round`, `Round 10+: 150 + ((round - 9) × 100) × 1.1^(round-9)`. Between rounds: 5-10 second break, countdown displayed. Mystery Box (950 points) offers random weapon. Pack-a-Punch machine (5000 points, unlocked round 5+) upgrades weapons +400% damage. Players chase high-round records: round 30 = skilled, round 50 = expert, round 100+ = world-class (10+ hours). Ammunition scarcity forces movement between zones.

**Synergies:**
- Difficulty Ramping (#21) - Waves provide structure for difficulty increases
- Countdown Timers (#23) - Wave duration creates time pressure within waves
- Shop Mechanics (#49) - Between-wave shops create economy decisions
- Resource Management (#47) - Limited resources across waves force conservation
- Milestone Unlocks (#6) - Wave 5/10/15/20 grant special rewards
- Leaderboards (#36) - Wave count or survival time creates competition
- Multiple Currencies (#44) - Combat currency spent in shop phases

**Session vs Meta:** 95% Session (wave progress resets each run, survival is moment-to-moment), 5% Meta (learning optimal strategies for specific waves, enemy spawn pattern knowledge)
