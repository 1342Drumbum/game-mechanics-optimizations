## 37. Combo/Multiplier Systems (Chain Rewards)

**Psychological Hook:** Operant conditioning via interval reinforcement—every N hits triggers dopamine. Flow state creation (Csikszentmihalyi) where attention narrows to maintaining combo. Loss aversion once combo starts—"Don't break the chain!" Zeigarnik Effect—interrupted combos create tension (47/50 combo memory lingers). Goal gradient accelerates effort approaching milestones (98 → 100 hit combo). Status signaling—high combo screenshots = social currency.

**Implementation Pattern:**
```
Combo Counter:
hits = 0
multiplier = 1.0
time_since_last_hit = 0

on_hit_enemy():
    hits += 1
    time_since_last_hit = 0

    # Tier-based multiplier
    if hits >= 100: multiplier = 3.0, rank = "SSS"
    elif hits >= 50: multiplier = 2.5, rank = "SS"
    elif hits >= 25: multiplier = 2.0, rank = "S"
    elif hits >= 10: multiplier = 1.5, rank = "A"
    elif hits >= 5: multiplier = 1.2, rank = "B"
    else: multiplier = 1.0, rank = "C"

    score += base_damage * multiplier
    display_combo_counter(hits, multiplier, rank)

on_update(dt):
    time_since_last_hit += dt
    if time_since_last_hit > COMBO_TIMEOUT:
        reset_combo()

on_take_damage():
    reset_combo()  # Harsh penalty maintains tension
```

Visual feedback: Counter enlarges at milestones, color shifts (white → yellow → orange → red → rainbow), screen flash at rank-ups, rank text appears (D/C/B/A/S/SS/SSS), audio cue per hit (pitch increases with combo), multiplier text pulses.

Timeout tuning: 1-2 seconds (tight, skill-based), 3-5 seconds (forgiving, casual), no timeout (progress-based only).

**Examples:**

1. **Devil May Cry 5:** Stylish Rank system—D/C/B/A/S/SS/SSS. Starts at D (×1.0). Each hit grants Style Points based on move variety and difficulty. A-rank: ~4,000 points (×1.5 multiplier), S-rank: 5,000+ points (×2.0+). SSS requires: 8,000+ points, no repeated moves, 10+ hit combo, perfect timing, enemy step usage. SSS multiplier: ×3.0 to score. Mission score = (Kills × Damage) × Style Multiplier × Time Bonus. Perfect SSS mission: 6,000+ base → 18,000+ with multiplier. Taking damage drops rank 1-2 levels instantly. Audio: "Stylish!" (C), "Badass!" (A), "Savage!" (S), "SSadistic!/SSensational!/SSmokin' Sexy Style!" (SSS). Visual: Text explosions, screen border pulses, rank icon enlarges 3×, combat music intensifies.

2. **Hades:** Combo Counter shows consecutive hits (no timeout while attacking). Combo milestones unlock bonuses: 12-hit: Hermes Flurry Strike (+15% attack speed), 15-hit: Artemis Pressure Points (+150% crit to 16th hit), 20-hit: Dionysus After Party (3 seconds invulnerability). Combo displayed top-right, enlarges every 5 hits. No multiplier but boon-based rewards. Breaking combo (enemy dodges, miss swing) requires restarting. Speedrun strats maintain 50+ hit combos through entire Tartarus via dash-strike canceling. Visual feedback: Red glowing numbers, screen shake intensifies with combo length, "BOILING BLOOD" text at 25+ combo activates % bonus damage buff.

3. **Guitar Hero / Rock Band:** Note streak counter = combo system. Multiplier tiers: 10 notes (×2), 20 notes (×3), 30 notes (×4, maximum). Base note: 50 points. At ×4: 200 points/note. Song: 1,000 notes at ×4 = 200,000 vs ×1 = 50,000 (4× difference). Missing 1 note resets to ×1 (brutal). Star Power at 50+ combo doubles multiplier (×4 → ×8). Visual: Multiplier flames grow at bottom screen, crowd cheers intensify, stage lights sync to combo level. Combo preservation creates edge-of-seat tension—100+ streak, difficult section approaching, sweaty hands. Physiological stress response = engagement.

4. **Call of Duty: Modern Warfare 2 (Multiplayer):** Killstreak system (kills without dying). 3 kills: UAV (radar sweep), 5 kills: Predator Missile (player-controlled), 7 kills: Harrier Strike (AI jet), 11 kills: AC-130 (gunship), 25 kills: Tactical Nuke (instant win). Each threshold triggers audio cue + screen notification. Killstreak rewards help achieve next tier (snowballing). Average player: 3-7 kills before death. Reaching 11+ requires camping/tactical play. 25 killstreak (Nuke): <0.1% of matches, generates social media clips. Combo broken by single death = tension sustained 5-10 minutes. Visual: Kill feed text enlarges, double/triple kill medals appear, screen borders flash gold at milestones.

5. **Tetris Effect:** Combo = consecutive line clears. Each back-to-back clear increases Zone meter (+10% per combo). Zone meter reaches 100% → Zone Mode activates (time slows, lines bank, clear all at once for 10× multiplier). Standard clear: 100 points/line. 5-combo clear: 100 × 1.5 × Zone bonus (3×) = 450/line. Perfect 40-line combo in Zone: 100 × 40 × 10× = 40,000 points vs standard 4,000 (10× difference). Visual escalation: Background particles intensify (10 → 500), music layers add per combo level (starts minimal, builds to orchestral), screen pulses sync to beat, cleared lines dissolve into particle explosions. Audio feedback: Piano notes pitch up chromatically per combo level, dopamine-triggering resolution chord at combo break.

**Synergies:**
- Visual Escalation (#36) - combo triggers screen-fill growth
- Achievement Pop-Ups (#38) - milestone combos trigger celebrations
- Skill Expression (#26-30) - combo maintenance proves mastery
- Screen Effects (#40) - combo rank-ups trigger shake/flash

**Session vs Meta:** 95% Session (combo resets constantly), 5% Meta (learning combo thresholds)
