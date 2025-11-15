## 17. High-Risk High-Reward Paths (Elites/Curses)

**Psychological Hook:**

Voluntary challenge demonstrates competence (Deci & Ryan, 2000). Success attributed to skill = pride. Risk assessment showcases knowledge: "This elite is dangerous, I'll skip" = mastery. Bragging rights = social capital. Leaderboards, achievements, Twitch highlights amplify this.

Loss aversion creates careful play: high stakes = peak attention. Players use hoarded consumables, optimize skills, employ defensive strategies. Reward justification: 2-3× normal to justify risk. Behavioral economics: humans require 2.5× return to accept significant risk.

**Implementation Pattern:**

```python
class HighRiskPathSystem:
    def __init__(self):
        self.elite_difficulty_multiplier = 2.5
        self.elite_hp_multiplier = 3.0
        self.elite_damage_multiplier = 2.0
        self.elite_currency_reward = 35  # vs 10-15 normal
        self.optimal_elites_per_run = 5

    def calculate_elite_rewards(self, elites_defeated):
        # First 5: full value. 6-7: 75%. 8+: 50% (unsustainable)
        if elites_defeated < 5:
            return base_currency
        elif elites_defeated < 7:
            return base_currency * 0.75
        else:
            return base_currency * 0.5
```

**Balancing Guidelines:**

- **Always optional:** Never force elites. Ability to decline = agency.
- **Visible before commitment:** "Elite ahead" warning. No surprise elites.
- **2-3× reward multiplier:** Currency 2-3× normal. Guaranteed rare drops. Unique drops (legendary relics).
- **Skill mitigation:** Perfect play = 80-90% win rate. Sloppy = 30-40%. Rewards skilled risk-taking.
- **Diminishing returns:** First 3-5 elites full value. 6-7 reduced. 8+ unsustainable.
- **HP attrition:** 30-50% max HP cost per elite. 8 elites = 300% HP = requires healing. Self-limiting.
- **Optimal frequency:** 3-7 encounters per run. <3 = insufficient rewards. >7 = death by exhaustion.

**Examples:**

**1. Slay the Spire (Elite Pathing):** 2-3 elites per act, 5-7 total optimal. Rewards: guaranteed rare relic, 25-40 gold, 15-25% HP damage. Strategic routing: Elite → Campfire → Elite = sustainable. Elite → Elite → Elite = death. Relic impact: Dead Branch +15-25% win rate. Average elites: 4.2 winning runs, 2.8 losing runs. Speedrunners: 8-10 elites (high risk/reward).

**2. Hades (Chaos Gates):** Curse lasts 3-5 chambers for powerful Chaos boon. Severity: mild (-20% attack speed), moderate (+35% damage taken), severe (lose 2 HP/sec). Boon rewards: +40-80% damage, build-defining. Curse synergies: +40% damage taken mitigated by Aphrodite Weak. Acceptance: 60-70% early, 10-20% late. Chaos-heavy builds: 50-100% more damage.

**3. Dead Cells (Challenge Rifts):** Kill 20 enemies in 45s, earn blueprint. Difficulty scales with BC. Reward: guaranteed blueprint (10-30 runs of RNG). Build dependency: ranged trivializes, melee struggles. No death penalty = encourages experimentation. Blueprint uniqueness drives collection.

**4. Risk of Rain 2 (Lunar Items):** Shaped Glass: +100% dmg, -50% HP per stack. 3 stacks: 8× damage, 12.5% HP (one-shot). Gesture of Drowned: auto-fires equipment (lose control). Lunar Coin scarcity (5-10/run). Pick rate Shaped Glass: 45% easy, 15% hard, 60% speedruns. Build transformation: commits to dodge-everything playstyle.

**5. Vampire Survivors (Curse Stat):** +Curse = +enemy count = +XP. 100 curse = 2× enemies = 2× XP = half time to cap. Risk curve: 0-30 manageable, 80-100+ bullet-hell. Build dependency: evolved weapons handle 100, single-target dies at 50. Speedrun meta: reach level 100 by min 12 vs 18. 60% curse deaths: min 25-30 (final spike).

**Success Metrics:** 40-60% risk-taking rate. Win rate 60-75% (skilled high-risk). Rewards worth risk: 70-80% "yes." Skill gap: 70% skilled vs 30% unskilled. Low-risk path viable: 40-50% win rate.

**Common Pitfalls:**
- **Mandatory high-risk:** Forcing elites removes agency. Optional path required.
- **Insufficient rewards:** 1.3× for 2× difficulty = mathematically bad. Need 2-3× minimum.
- **Hidden difficulty:** Surprise elites feel unfair. Always telegraph.
- **Pure stat-check:** Unkillable without X DPS removes skill. Allow outplay potential.
- **No healing options:** Unsustainable elite pathing = death spiral. Balance healing availability.

**Synergies:** Push-Your-Luck (#16), Forced Trade-Offs (#18), Adaptive Difficulty

**Session vs Meta:** 100% Session
