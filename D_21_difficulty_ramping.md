## 21. Difficulty Ramping (Escalating Spawn Rates)

**Psychological Hook:** Flow state maintenance through dynamic challenge scaling (Csikszentmihalyi, 1990). Creates "just barely surviving" sensation that maximizes engagement. Leverages arousal theory - moderate stress optimizes performance. Progressive difficulty creates narrative arc (weak → struggling → triumph OR death). The "one more try" effect emerges when players feel they can conquer the next tier.

**Implementation Pattern:**
```
enemy_health = base_health × (1 + (time_elapsed / 60) × 0.15)
spawn_rate = base_rate × (1 + (time_elapsed / 120) × 0.25)
enemy_count = base_count + floor(time_elapsed / 180)

// Exponential scaling option
difficulty_coefficient = 0.0506 × difficulty_value × player_count^0.2 × time_factor

// Wave-based scaling
wave_health = base_health + (wave_number × health_per_wave)
wave_duration = min(20 + (wave_number × 5), 60)  // caps at 60s
```

Start gentle (minutes 0-3: tutorial pace), escalate linearly (min 3-10: steady increase), exponential endgame (min 10+: overwhelming). Display difficulty visually (enemy density, elite frequency). Ensure player power scales parallel to enemy difficulty for first 60-70% of run.

**Examples:**

1. **Vampire Survivors:** Enemy HP scales with player level: `HP × Level` applied at spawn. At level 50, basic bat (10 HP) becomes 500 HP. Spawn frequency increases every minute: minute 1 = 5 enemies/wave, minute 10 = 30-40 enemies/wave, minute 25 = 200+ simultaneous enemies (screen-filling). Curse stat amplifies: each point increases spawn frequency +5% and enemy HP +1%. At 100 Curse, enemies spawn 2× faster with 2× health. Endless Mode adds +100% base HP per cycle, +50% spawn frequency, +25% damage per loop.

2. **Risk of Rain 2:** Difficulty coefficient increases continuously: `0.0506 × difficulty_value × time_elapsed × player_count^0.2`. Monsoon difficulty (value 3) means enemies gain 15.18% stats per minute. At 20 minutes: enemies have +303% HP/damage. Stage completion spikes difficulty +50% instantly (jumping to next tier). Multiplayer accelerates scaling: 4 players = 1.32× faster difficulty increase. Director system spawns: minute 5 = Lemurians, minute 15 = Lesser Wisps + Golems, minute 25 = Elder Lemurians + Brass Contraptions (elites). Credits spent on spawns increase exponentially.

3. **Hades:** Heat system provides manual difficulty ramping. Hard Labor (5 ranks): +20%/40%/60%/80%/100% enemy HP per rank. Damage Control (4 ranks): enemies gain 1/2/3/4 shields per rank requiring multiple hits to damage. Extreme Measures (4 ranks): bosses gain new attack patterns +40% damage. At 32 Heat with all modifiers: enemies have +100% HP, +100% damage, +60% armor, +50% speed, Hydra has 5 heads instead of 1. God Gauge increases boon rarity over time (mitigating difficulty increase slightly). Late-stage enemies (Elysium): base 150 HP, at 32 Heat = 450 HP with 4 armor shields.

4. **Brotato:** 20-wave structure with linear scaling. Wave 1 = 20 seconds, increasing +5s/wave until 60s (wave 9+). Enemy HP: `base_health + (wave_number × hp_per_wave)`. Basic enemy (Alien): Wave 1 = 3 HP, Wave 10 = 30 HP, Wave 20 = 60 HP. Damage scales identically. Danger 5 difficulty adds: Elite waves (waves 11/12, 14/15, 17/18) with +100% HP/damage, 2× boss spawns at wave 20. Endless Mode (wave 21+): Endless Factor = (wave - 20), enemy max HP increases 225% × Endless Factor. Wave 30 = +2250% HP (23.5× multiplier).

5. **Dead Cells:** Boss Cell system provides meta-difficulty ramping. 0BC baseline → 5BC ultimate. Each BC: +40% enemy HP, -25% healing from flask, +1 elite per biome, enemies gain new abilities. 5BC: no flask refills between biomes, -60% food healing, +20% enemy damage, Malaise builds over time (30 stacks = instant death). Time-based difficulty within runs: enemies gain +5% stats per minute after 10-minute mark. Cursed Chest mechanic: 10-hit instadeath but +100% loot quality (player-controlled difficulty spike).

**Synergies:**
- Exponential Power Curves (#1) - Player power must scale alongside enemy difficulty
- Adaptive Difficulty (#13) - Can adjust ramping based on player performance
- Wave Survival (#22) - Wave breaks allow recovery between difficulty spikes
- Pity Systems (#20) - Mitigate frustration from overwhelming difficulty
- Prestige Loops (#5) - Heat/BC systems extend difficulty ceiling
- Rush vs Patience (#24) - Faster play avoids late-game difficulty spikes

**Session vs Meta:** 85% Session (difficulty scales within individual run, resets completely), 15% Meta (learning enemy patterns at higher difficulties, understanding optimal power curve timing)
