## 10. Effect Stacking (Additive → Exponential Scaling)

**Psychological Hook:** Power fantasy escalation ("number go up"). Skill expression through optimization. Collector psychology satisfies completionist drive. Natural difficulty curve maintains flow state.

**Implementation Pattern:**

```python
class StackingSystem:
    @staticmethod
    def linear_stacking(base_value, stacks, increment):
        """Use for: Flat bonuses (damage, health). Predictable, each stack same value."""
        return base_value + (stacks * increment)
        # Example: 100 base + (5 stacks × 10) = 150

    @staticmethod
    def hyperbolic_stacking(stacks, effect_per_item=1.0, exponent=0.33):
        """
        Use for: Percentage effects that shouldn't reach 100% (crit, dodge, CDR)
        Formula: Effect = 1 - 1 / (1 + a × n)^b
        Never reaches 100%, always valuable
        """
        return 1 - (1 / pow(1 + effect_per_item * stacks, exponent))
        # Example: 1 stack 50%, 2: 62.6%, 5: 77.2%, 10: 84.2%, 20: 89.9%

    @staticmethod
    def exponential_stacking(stacks, base_multiplier=2.0):
        """Use for: High-risk items (Shaped Glass, Lunar items). Glass cannon builds."""
        damage_multiplier = pow(base_multiplier, stacks)
        health_divisor = pow(base_multiplier, stacks)
        return {
            'damage': damage_multiplier,
            'health_reduction': 1.0 / health_divisor
        }
        # Example: 3 stacks = 8× damage, 0.125× health (12.5%)

    @staticmethod
    def additive_vs_multiplicative(bonuses, mode='additive'):
        """Additive: sum then multiply. Multiplicative: each multiplies separately."""
        if mode == 'additive':
            return 1 + sum(bonuses)  # 50% + 50% = 100% total
        elif mode == 'multiplicative':
            total = 1.0
            for bonus in bonuses:
                total *= (1 + bonus)
            return total  # 50% × 50% = (1.5 × 1.5) = 2.25× total
```

**Balancing Guidelines:**

**Linear:** Flat damage/health/armor. Formula: `Effect = Base + (Stacks × Increment)`. Scaling: +5-15% per stack. No cap (infinite scaling).

**Hyperbolic:** Crit chance/dodge/CDR. Formula: `Effect = 1 - 1/(1 + a×n)^b`. Exponent: b=0.33 standard (Risk of Rain 2). First stack: 40-60% effective. Stack 10: 80-90%. Stack 50: 95-99% (soft cap).

**Exponential:** High-risk items only. Base: 1.5-2.0 (2.0 most common). Practical limit: 3-5 stacks before unplayable.

**Multiplicative vs Additive:** Additive (simple, predictable) within categories. Multiplicative (natural diminishing, encourages diversity) between categories.

**Examples:**

**1. Risk of Rain 2: Soldier's Syringe (Linear):** `Attack Speed = Base × (1 + 0.15 × Stacks)`. 1 syringe: +15% (1.15×). 10: +150% (2.5×). 50: +750% (8.5×). 100: +1500% (16×). Common rarity, drops frequently. Speedrunners prioritize early (30-40% win rate boost).

**2. Risk of Rain 2: Shaped Glass (Exponential):** `Damage = 2^Stacks, Health = 1/2^Stacks`. Stacks: 1=2× dmg/50% HP, 2=4×/25%, 3=8×/12.5%, 4=16×/6.25%. Pick rate by skill: New 15%, Intermediate 35%, Skilled 60%, Speedrun 85%. Synergies: Transcendence (shields replace HP), Personal Shield Generator, Aegis.

**3. Hades: Pom of Power (Diminishing):** Level 1→2: +35%. Level 2→3: +25%. Level 5→10: +15% avg. Level 15→16: +5-8%. Example (Lightning Strike): Lv1: 20 dmg. Lv2: 27 (+35%). Lv5: 45 (+125%). Lv10: 65 (+225%). Lv20: 95 (+375%). Strategy: Level 1→2 most efficient. Late: spread poms across boons.

**Success Metrics:** Stacking engagement 80-90% by run 5. No single stat >40% total investment (diversity). Average distribution: 3-5 heavily stacked (10+), 10-15 moderately stacked (3-6), 20+ single stacks. Power increase 5-10× early to late. Diminishing return awareness by run 10.

**Common Pitfalls:**
- All linear stacking: optimal = "stack one stat to infinity." Mix linear/hyperbolic/exponential.
- Hidden formulas: display exact numbers. "Next stack: +12.3% (current: 67.8%)"
- No soft caps: hyperbolic formula prevents 100% invulnerability
- Multiplicative confusion: tooltip clarity "Multiplicative with other sources" + examples
- No visual feedback: screen shake/VFX scale with damage/stacks

**Synergies:** Build-Around Items (#9), Item Tier Scaling (#17), Combo Systems (#31)

**Session vs Meta:** 100% Session (stacks reset, knowledge persists)

---
