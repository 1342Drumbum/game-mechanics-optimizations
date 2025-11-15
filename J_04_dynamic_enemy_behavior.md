## 54. Dynamic Enemy Behavior (Adaptive AI)

**Psychological Hook:** Mastery pursuit through adapting to intelligent opposition. Flow state maintenance - AI difficulty scales to player skill (Csikszentmihalyi). Attribution to skill not RNG - "AI outsmarted me" feels fair vs "RNG screwed me." Unpredictability creates novelty, preventing boredom from pattern recognition. Learning from failure: "The AI countered my strategy, I need to adapt" creates growth mindset. Social comparison: "I beat the adaptive AI" = bragging rights.

**Implementation Pattern:**
```
Utility-Based Decision Making:
Enemy evaluates all possible actions and chooses highest utility score

Action Utility = Base_Value + Position_Bonus + Target_Bonus + State_Bonus

Position Scoring:
- Full Cover: +50 utility
- Half Cover: +25 utility
- Flanking Position: +40 utility
- High Ground: +30 utility
- Near Allies: +20 utility per ally (max +60)
- Exposed: -40 utility

Target Prioritization:
1. Easiest to hit (out of cover, flanked)
2. Lowest HP (wounded = high priority)
3. High threat (damage dealers > tanks)
4. Class priority (medics > snipers > assault > tanks)

Adaptive Difficulty Tracking:
player_performance_score = (kills - deaths) + (accuracy × 10) + (objectives_completed × 50)

if performance_score > threshold_high:
    enemy_aggression += 15%
    enemy_spawn_rate += 20%
    elite_enemy_chance += 10%
elif performance_score < threshold_low:
    enemy_aggression -= 10%
    health_drop_chance += 25%
    spawn_rate -= 15%

Behavior Personality Types (per enemy class):
- Aggressive: prioritize damage over safety (+30 utility to attack actions)
- Defensive: prioritize cover and positioning (+40 utility to defensive actions)
- Tactical: balanced, uses abilities optimally (standard utilities)
- Flanker: seeks flanking positions (+50 utility to flanking movement)
- Supporter: stays near allies, uses buffs (+40 utility to support actions)
```

Learning patterns:
```
Track player behavior over 10+ encounters:
- Most used weapon type → adapt resistances
- Preferred tactics (stealth, rush, camping) → counter-tactics
- Ability usage patterns → interrupt timing
- Movement patterns → prediction and pre-firing

Example:
if player_uses_stealth > 60%:
    increase_perception_radius(+50%)
    deploy_detection_enemies(+30% spawn)
    reduce_hiding_spots(-20%)
```

**Examples:**

1. **XCOM 2:** Utility-based AI with 17+ behavior sets. Each alien type has personality: Sectoid (defensive, uses cover), Viper (flanker, tongue-pull priority), Berserker (aggressive, charges), Sectopod (area denial, maximizes multi-target attacks). Targeting priority: (1) Easiest hit - out of cover soldier gets 90% target priority, (2) Distance - closer = higher priority, (3) Class - prioritizes low defense units. Flanking AI: if no flank available, 40% chance to move toward flank position (even without shot), 60% take safe overwatch. Pod activation: enemies get free "scramble" move to cover on reveal, preventing player ambush. Reinforcement scaling: time-based + performance-based. Turn 8: 70% reinforcement chance. Player doing well (no wounds, high kills): +15% reinforcement chance, +1 enemy per drop. Player struggling (2+ wounded): -10% reinforcement chance. Muton Elite behavior: 50% chance to use grenade if 2+ soldiers in range, prioritizes destroying player cover over direct damage. Lost (zombie faction): follows noise, 100% attacks closest target (no utility calculation), creates emergent strategy (use noisy weapons to redirect). 40-60 hour campaign teaches AI patterns: "Sectopods always AOE, spread out" = learned counterplay.

2. **Left 4 Dead 2:** AI Director is foundational example. Monitors player "stress" (intensity score): Takes damage (+10 intensity), Ally downed (+20), Horde survived (-15), Saferoom reached (reset to 0). Target intensity: 50-70 (optimal flow state). Below 40 = Build-Up phase (spawn common, special infected, tension). 70-90 = Peak Intensity (crescendo event, tank spawn, overwhelming force). Above 90 = Relax phase (30-60s no spawns, health items spawn). Special Infected spawn slots: 4 slots for Hunter, Smoker, Boomer, Spitter. Each has 20-45s spawn timer. Spawn location rules: Boomers ahead of survivors (ambush), Smokers above/behind (isolated pull), Hunters in dark areas (jump scare). Tank spawn: guaranteed 1-2 per chapter, spawns when team at 40-60 intensity (not too weak, not too strong = most dramatic). Item spawns convert based on team HP: Team at 80%+ HP = Pain Pills spawn. Team at 40% HP = Medkit spawn. Team at 20% HP = Defibrillator spawn (revive dead). 4-player co-op scales: AI tracks EACH player's intensity separately, spawns target lowest performer (prevents snowball loss). Versus mode: human-controlled infected see "ghost mode" spawn suggestions based on Director's utility calculations.

3. **Hades:** Subtle adaptivity through God Gauge and pacing. God Gauge: tracks god boon offers. Ignore Aphrodite for 15+ chambers → 90% chance she appears next. Ensures build viability (always offered solution to weakness). Boss patterns adapt: Lernie (Hydra) spawns heads based on damage speed. Slow DPS → 3 heads. Fast burst → 5 heads. Theseus calls Olympian aid at 50% HP, god chosen based on player's boon weaknesses (has Athena deflect → calls Ares for curse damage). Asterius (bull) becomes more aggressive as HP drops: 100-50% HP: 5s between charges. 50-25%: 3s between charges. <25%: 2s + rage mode (permanent +50% speed). Hades (final boss) has 4 phases, each with 10+ attack patterns. Randomly mixes patterns per attempt (no two fights identical). Spin attack telegraphs 0.5s before activation. Skilled players dodge at 0.45s (perfect dodge), AI learns and occasionally delays 0.1s to punish timing memorization. Community data: 50+ attempts to first clear (average), 150+ for 32-heat clear = mastery journey.

4. **Darkest Dungeon:** Stress-based adaptive storytelling. Enemies target stress systematically: Cultist Witch has 55% chance to cast "Eldritch Push" (5 stress) if hero is already at 50+ stress (finish them off). Madman (rare enemy) only uses stress attacks, ignores HP damage entirely = test of stress management. Boss AI priorities: Hag targets hero with best trinkets (steals gear = massive penalty). Collector spawns adds matching party composition (have 2 melee? Spawns 2 melee heads to counter). Shuffling enemies (Bone Arbalist) move player party positions, disrupting strategy = forces adaptation mid-combat. Affliction targeting: Afflicted party member gets -20% target priority (they're already broken, break others). Light level affects enemy behavior: Torchless (0 light) = +30% enemy crit, +20% damage, stress attacks prioritized. Veteran/Champion difficulty: enemies gain +15% HP, +20% damage, +25% stress damage per tier. By Champion, stress becomes primary threat (not HP damage). 80-120 hours teaches: "Always kill stress dealers first" = learned priority targeting.

5. **Monster Train:** Covenant system as difficulty adaptation. Covenant 0 (baseline) → Covenant 25 = +450% enemy HP, +150% damage, new mechanics. Each covenant adds 1-2 modifiers: C1 = +20% HP, C5 = +50% HP + enemies gain armor, C10 = +100% HP + armor + regeneration, C15 = +150% HP + armor + regen + damage shields, C25 = all modifiers + boss gets permanent +25 damage per turn. Enemy targeting AI: prioritizes Pyre (player's HP), ignores units unless threatened. If unit deals >30 damage previous turn → becomes priority target (+80% chance to attack that unit). Boss patterns scale: Seraph the Chaste (final boss) baseline: 10 × 3 attacks (30 damage). Covenant 10: 15 × 3 attacks (45 damage). Covenant 25: 25 × 4 attacks (100 damage). Requires perfect unit positioning, scaling, and adaptation. Divinity scaling: player units scale +5/+5 per floor (8 floors = +40/+40). Enemy scaling matches: floor 6 enemies are 2× baseline stats. Floor 8 (boss): 3× baseline. Forces multiplicative scaling strategies (additive math fails). 100+ hours to Covenant 25 clear (per clan) = mastery expression.

**Synergies:**
- Adaptive Difficulty (#13) - both adjust to player skill
- Nemesis Systems (#51) - enemies remember tactics
- Flow State Design (#29) - maintaining challenge/skill balance
- Knowledge Checks (#28) - learning enemy patterns
- Execution Challenges (#27) - reacting to AI behaviors
- High-Risk Paths (#17) - elite AI provides challenge
- Prestige Systems (#5) - harder AI at higher heat/covenant

**Session vs Meta:** 70% Session (AI adapts during play), 30% Meta (learning AI patterns over time)
