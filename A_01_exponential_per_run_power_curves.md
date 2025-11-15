## 1. Exponential Per-Run Power Curves

**Psychological Hook:**

Exponential curves create peak engagement through goal gradient (Hull, 1932) and flow states (Csikszentmihalyi, 1990). As damage escalates from 50 → 500 → 5,000, dopamine reward prediction errors fire—unexpected multiplicative jumps (finding perfect synergy) trigger 2-3× stronger dopamine surges than predicted rewards (Schultz, 1998).

Power curves create compressed narrative arcs: vulnerability (min 0-3) → competence (min 3-8) → mastery (min 8-15) → transcendence (min 15+). Campbell's hero's journey in 15-25 minutes.

Behavioral economics: exponential curves create hyperbolic discounting in reverse. Players underestimate multiplicative stacking (1.3^4 = 2.86× exceeds intuitive 1.3+1.3+1.3+1.3 = 5.2 additive), creating persistent positive surprise.

**Implementation Pattern:**

```python
# Multiplicative scaling
base_damage = 10
damage_multiplier = 1.0

for item in player_inventory:
    damage_multiplier *= (1 + item.bonus_percent)

final_damage = base_damage * damage_multiplier

# Example: 5 items with +15% each
# 1.15^5 = 2.01× multiplier (10 → 20.1 damage)
# 10 items: 1.15^10 = 4.05× (10 → 40.5)
# 15 items: 1.15^15 = 8.14× (10 → 81.4)
```

**Progression Timeline (Vampire Survivors baseline):**
- Min 0-1: 10 dmg, 5-10 hits to kill
- Min 3: 25 dmg (2.5×), 2-4 hits
- Min 5: 50 dmg (5×), 1-2 hits
- Min 10: 200+ dmg, weapon evolutions trigger
- Min 15: 500-1000 dmg, screen-clearing
- Min 20: 2000+ dmg, pre-emptive elimination
- Min 25-30: 5000+ dmg (mandatory for final spike)

**Formulas:**

```
Multiplicative Stacking: Final = Base × ∏(1 + bonus_i)
Additive + Multiplicative: Final = (Base + Σ flat) × ∏(1 + percent)
Diminishing Returns: Effective = bonus / (1 + bonus/cap)
Exponential Target: Power(t) = Base × e^(growth_rate × t)
For 100× over 20min: growth_rate = ln(100)/20 = 0.23/min
```

**Balancing:**
- **Early (0-30%):** Linear/slight exponential. 2-3× power. Baseline viability required.
- **Mid (30-70%):** Exponential acceleration. 5-10× power. Synergies emerge, build identity forms.
- **Late (70-100%):** Full exponential/super-exponential. 20-100× power. Screen-clearing, god-mode payoff.
- **Soft Caps:** Diminishing returns at 80-90% theoretical max prevent invincibility.
- **Anti-Synergy:** 10-15% trade-off items (glass cannon: 2× dmg, 0.5× HP).

**Examples:**

**1. Vampire Survivors:**
- Min 0: Whip (10 dmg). Min 3: Whip+Bible+Spinach (60 DPS). Min 10: Evolutions unlock (600-800 DPS). Min 20: All evolved, 10,000-15,000 DPS. Min 30: Death or victory.
- Psychology: Min 0-5 tense survival → 5-10 mastery → 10-20 transcendence → 20-30 climax.
- Sales: 40M copies, 35-45% D7 retention. Developer: "Weak 20min, god 10min."

**2. Risk of Rain 2:**
- Parallel exponential scaling: player AND enemy power. Difficulty = 0.0506 × time. Enemy HP = base × (1 + Difficulty).
- Min 0: 110 HP, 12 dmg. Min 10: 8-12 items, 5.76× effective power. Min 20: 35-50 items, 20-30× power.
- Shaped Glass: 3 stacks = 8× dmg, 0.125× HP (13.75 HP). One-shot from anything = high-risk/reward.
- Retention: 97% positive Steam reviews, D30: 25-35%.

**3. Hades:**
- Boon stacking: additive-then-multiplicative. Backstab build: 20 base → 1500/rotation by endgame = 10.9× DPS.
- Chamber 2: Artemis +50% dmg. Chamber 14: Legendary boon. Chamber 40: 1200+ dmg/rotation = 1000 DPS.
- No-boon run: 92 DPS. Boss takes 140sec. With boons: 1000 DPS, 26sec.
- 1M copies in 4 days, 5M+ by 2024. D30: 30-40%. "Every boon matters, synergies = 10× spikes."

**4. Balatro:**
- Score = (Chips + Bonuses) × (Mult + Bonuses). Jokers apply multiplicative.
- Ante 1: Pair = 60pts. Ante 3: Four Kings + jokers = 3,080pts (one hand wins). Ante 8: 250K+ required.
- God-tier: Blueprint+Brainstorm+Baron+Duo+Hologram = X291.4 mult. Four Kings = 349,680pts.
- Endless mode: X2,560 mult builds, 583M point hands.
- 1M copies in month 1, D7: 40-50%. "Excel optimization disguised as cards."

**5. Slay the Spire:**
- Synergy discovery: mediocre cards + relics = 10× multiplicative damage.
- Linear build: 36 dmg/turn, boss takes 8.3 turns (likely death). Exponential build: Heavy Blade (18 base + 5×Strength). Turn 1: 38 dmg. Turn 3: 128 dmg. Turn 4: 276 dmg. Turn 5: 414 dmg. Boss dead.
- Infinite combo: Dropkick+Thunderclap loop = infinite damage.
- 50K concurrent peak, 95% positive. D30: 20-30%. "Linear additions create exponential power."

**Success Metrics:**
- Completion rates: 60-75%
- "One more run": 70%+ immediate restart
- Power fantasy satisfaction: 85-95%
- D7 retention: 35-45% (vs 15-25% without exponential)

**Synergies:** Variable Rewards (#3), Build Discovery (#11), Milestone Unlocks (#6), Adaptive Difficulty (#16), Rush vs Patience (#24)

**Session vs Meta:** 90% Session (power resets), 10% Meta (learning synergies)

---
