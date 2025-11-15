## 9. Build-Around Items (Single-Item Run Definers)

**Psychological Hook:** Eureka moments trigger 30-50% stronger dopamine. Player agency creates pride attribution. Mastery expression through recognizing synergies. Social currency from memorable run stories.

**Implementation Pattern:**

```python
class BuildAroundItem:
    def __init__(self, name, rarity, transformative_power):
        self.name = name
        self.rarity = rarity  # 'rare' or 'legendary'
        self.transformative_power = transformative_power  # 2.0-5.0x
        self.synergy_count = 0  # Items that combo

    def evaluate_impact(self, player_build):
        baseline_power = calculate_baseline_dps(player_build)
        with_buildaround = calculate_dps_with_item(player_build, self)
        power_multiplier = with_buildaround / baseline_power

        # Should be 2-5× baseline power
        # Should enable 10-15+ synergies
        compatible_items = find_synergies(self, item_pool)

        return {
            'power_multiplier': power_multiplier,
            'viable': power_multiplier >= 2.0 and len(compatible_items) >= 10,
            'synergy_depth': len(compatible_items)
        }
```

**Balancing Guidelines:**

- **Frequency:** 5-15% of total item pool. <5% too rare, >20% dilutes identity
- **Power level:** 2-5× baseline with proper support. <2× not transformative, >5× trivializes difficulty
- **Synergy web:** 10-15 strong combos, 20-30 moderate. Creates discovery space
- **Independent viability:** 60-70% effectiveness alone. Prevents "dead" picks
- **Rarity appropriateness:** Match power to rarity (Dead Branch = Rare, achievable)

**Formulas:**

```
Power Multiplier = (DPS_with_item × Synergy_Factor) / DPS_baseline
Target: 2.0-5.0×

Synergy_Factor = 1 + (Synergy_Items_Obtained × 0.15-0.25)

Pick Rate vs Win Rate (Slay the Spire methodology):
- Optimal: 60-75% pick rate, +15-25% win rate
- Overpowered: >80% pick, +30% win
- Underpowered: <40% pick, +5% win
```

**Examples:**

**1. Slay the Spire: Dead Branch + Corruption:** Exhaust card → add random card. Corruption: skills cost 0, exhaust. Synergy: drawback becomes engine. Normal turn: 5-10 cards, 40-60 damage. Dead Branch turn: 30-50 cards, 200-400 damage. Pick rate: 85% with exhaust synergy. Win rate: +25-30%. Defended as "fair broken" due to scarcity (Rare + Skill card required).

**2. Risk of Rain 2: Gesture of Drowned:** Equipment auto-fires at 50% cooldown (lose manual control). Lunar item (scarce currency, limited to 2/run). Synergy: Royal Capacitor = auto orbital strikes every 6s (+200% DPS). Anti-synergy: Primordial Cube = automatic suicide. Pick rate: 45% with good equipment, 5% without.

**Success Metrics:** Pick rate 70-85% with combo pieces, 40-60% independently. Win rate +20-30%. 5-10 distinct archetypes created. Community posts: 15-25% strategy discussions reference build-arounds. Synergy discovery: 60-75% organic within 20 runs.

**Common Pitfalls:**
- Too many build-arounds (>20% pool): dilutes identity
- Insufficient supporting items: requires 15-20 synergy items per build-around
- Power without commitment: require 3-5 synergy items for peak effectiveness
- Binary viability: ensure 2× alone, 5× with support (gradual scaling)
- No counter-play: situational weaknesses (strong vs swarms, weak vs bosses)

**Synergies:** Synergy Discovery (#11), Build Variety (#15), Item Tier Scaling (#17)

**Session vs Meta:** 95% Session, 5% Meta (knowledge persists)

---
