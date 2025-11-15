## 20. Pity/Mercy Systems (Bad Luck Mitigation)

**Psychological Hook:**

Loss aversion mitigation prevents frustration churn. "70 pulls, no legendary" = quit. "70/90, guaranteed in 20" = sustained engagement. Hope maintenance: "Pity: 74/90" creates goal gradient (Hull, 1932). Final 20%: 30-50% increased effort when "close."

Goal gradient: "Only 15 more!" = concrete. Without pity: "Maybe next?" = infinite uncertainty = demotivation. Fairness perception: worst-case bounded = tolerable. Unbounded = "rigged." Spending encouragement (gacha): "74/90, 16 from SSR" = urgency (ethical concern: exploitative).

Zeigarnik Effect: incomplete goals occupy mental bandwidth. "54/90" persists, creates itch to complete.

**Implementation Pattern:**

```python
class PitySystem:
    def __init__(self, hard_pity=90, soft_pity_start=74):
        self.hard_pity = hard_pity
        self.soft_pity_start = soft_pity_start
        self.base_rate = 0.006  # 0.6%
        self.rolls_without_legendary = 0

    def roll_gacha(self):
        self.rolls_without_legendary += 1

        # Hard pity: absolute guarantee
        if self.rolls_without_legendary >= self.hard_pity:
            return self.grant_legendary(guaranteed=True)

        # Soft pity: escalating rates
        if self.rolls_without_legendary >= self.soft_pity_start:
            pulls_into_soft = self.rolls_without_legendary - self.soft_pity_start
            current_rate = self.base_rate + (pulls_into_soft * 0.06)  # +6%/pull
        else:
            current_rate = self.base_rate

        return current_rate
```

**Balancing Guidelines:**

- **Hard pity threshold:** 80-100 rolls typical. Lower (50) = too generous (removes RNG excitement). Higher (150+) = too punishing.
- **Soft pity start:** 75-85% of hard pity. Genshin: 74/90 (82%). Smooth probability curve, not cliff.
- **Visible counter:** Display "54/90 pulls" clearly. Hidden pity feels manipulative (distrust).
- **Pity carries over:** Never reset between sessions/banners. Resetting = "wasted progress" = rage.
- **50/50 system (gacha):** First legendary 50% featured. Lost = next guaranteed. Bounds worst-case at 2× hard pity.
- **Transparent odds:** Display all rates. "0.6% base, increases after 74" builds trust. Hidden rates = regulatory issues.

**Formulas:**

```
Hard Pity: N = 80-100
Soft Pity: M = 0.75-0.85 × N

Rate(pull) = Base + (pull - M) × Increase
Genshin: 0.6% + (pull - 74) × 6% = up to 60.6% at 89

Expected pulls to legendary (with pity): ~72 average
F2P time to guarantee: 180 / 56 monthly = 3.2 months
```

**Examples:**

**1. Genshin Impact (Gold Standard):** 0.6% base, hard pity 90, soft 74. Soft: +6%/pull (74: 6.6%, 89: 60.6%). 50/50 system: worst-case 180 pulls (lose 50/50 at 90, hit 90 again). F2P: 9,000 primos/month (56 pulls) = 3.2 months worst-case. Average: 62.5 pulls. Pity carries between banners. Controversy: $200-360 for guarantee = predatory.

**2. Balatro (Implicit Pity):** No counter. Luck stat improves odds. +10 Luck = 2× rare appearance. Reroll escalation: $5, $10, $15. Legendary removal (no duplicates) thins pool = invisible pity. Average: Ante 4-6 with Luck, 8-10 without.

**3. Hades (Fated Authority):** 1-2 reroll charges/run. Reroll god boon for different god. Strategic: save for critical chambers (5-10). Combine with keepsakes: force specific god + reroll = near-guaranteed. Win rate: 52% with, 47% without = +5%.

**4. Slay the Spire (Weighted Pools):** Anti-duplicate bias. Obtaining Dead Branch reduces second appearance chance. Declining Strike 3× = appears less. Shop pool: 3 attack last time = biased toward skill/power next. "Unlucky" streaks (10+ shops no rare) <5% of runs. Invisible pity = feels naturally fair.

**5. Deep Rock Galactic (Time-Gated):** 1 Overclock/week minimum (Core Hunt + Deep Dive). 2/week max. Machine Events: 0-5/week. 160 Overclocks = 160 weeks (3 years) worst-case. Average: 1.5-2 years. Target acquisition: specific from 160 pool = 80 weeks (1.5 years). Philosophy: "years of engagement, not week grinds." 2-3 year retention for hardcore.

**Success Metrics:** <10% hit hard pity (8-10% healthy). Frustration reduction: churn -15-25% with pity. Spending (gacha): 60-70% pity-triggered pulls = real money (ethical concern). Fairness perception: 75-85% "fair RNG" with pity vs 40-50% without.

**Common Pitfalls:**
- **Hidden pity:** Not displaying counter = conspiracy theories. "Game is rigged!" Always show progress.
- **Pity resets:** Resetting between banners/sessions = "wasted progress" rage. Never reset.
- **Too generous:** Hard pity at 20 rolls = no RNG excitement. Sweet spot: 80-100 (long enough for RNG, short enough to tolerate).
- **No soft pity:** Cliff at hard pity (0.6% until 90, then 100%) feels bad. Smooth escalation better UX.
- **Pay-to-complete:** $200 to complete = predatory. F2P completion in 1-3 months reasonable.

**Synergies:** Variable Rewards (#3), Collection Completion (#4), Meta-Currency

**Session vs Meta:** 20% Session, 80% Meta (pity across runs/banners)
