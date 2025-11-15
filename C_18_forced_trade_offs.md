## 18. Forced Trade-Offs (Sacrifice HP/Economy for Power)

**Psychological Hook:**

Meaningful choices (Sid Meier): eliminate dominant strategies. Opportunity cost creates tension: choosing A = foregoing B = cognitive dissonance. Irreversible choices create commitment. Regret/satisfaction from consequences = learning + story.

Strategic thinking demonstrates intelligence: evaluating "+1 Energy, no campfire healing" requires future projection. Engages executive function. Commitment escalation: trading HP = committed to making it work. Loss aversion: trading HP/currency feels like losing even while gaining power. Frame as "investment," display gains prominently.

**Implementation Pattern:**

```python
class TradeOffSystem:
    def offer_trade_off(self, trade_type):
        trades = {
            'devil_deal': {
                'cost': {'max_hp': -40},
                'gain': {'damage_multiplier': 2.5},
                'irreversible': True
            },
            'energy_relic': {
                'cost': {'rest_healing': 0},
                'gain': {'energy': +1},
                'irreversible': True
            }
        }

        # Display costs/gains clearly
        # Calculate expected value
        ev = (gain_value - cost_value) / cost_value
        return ev
```

**Balancing Guidelines:**

- **Visible consequences:** Display exactly what you're trading. "+1 Energy, can't heal" = clear. "+1 Energy, ???" = distrust.
- **Both options viable:** Neither >70% dominant. If always best, it's IQ test, not trade-off.
- **Irreversibility creates weight:** Permanent = significant. Temporary = less tension, more forgiving.
- **Frequency:** 1-3 major trade-offs per run. Too many (5+) = decision fatigue. Too few (0-1) = lacks depth.
- **Context dependency:** "+1 Energy" at 3 energy = +33%. At 6 = +17%. Same trade, different value.
- **Decision points:** Present between levels/shops/boss rewards. Not mid-combat.

**Examples:**

**1. Binding of Isaac (Devil Deals):** Trade 2 heart containers for Brimstone. Start: 3 containers (6 HP). Trade: 1 container (2 HP) remaining. Power: 4.3× damage. At 2 HP: enemy contact = death. Win rate: 45% with Brimstone at 1 HP vs 25% baseline. Pick rate: 85% floors 1-3, 30% floors 7+. Soul Heart loophole: 1 red + 6 soul = safe deals.

**2. Slay the Spire (Boss Relics):** Coffee Dripper: +1 Energy, no rest healing. Energy +30-50% damage, lost healing = -40 HP potential. Runic Dome: +1 Energy, no enemy intent. Expert-only (requires memorizing 50+ patterns). Skilled: +40% win rate. New: -30%. Velvet Choker: +1 Energy, 6 cards/turn max. Build-defining. Win rates: Coffee 58%, Runic 54% skilled/38% novice, Velvet 49%.

**3. Hades (Heat System):** Voluntary difficulty modifiers for rewards. Extreme Measures (2 Heat): bosses +30-60% harder. Forced Overtime (2 Heat): enemies 20% faster. Stacking: 32 Heat = 8-10 modifiers = 5-10× harder. 6 weapons × 32 Heat = 192 milestones = 300+ hours. Clear rates: 32 Heat <5%, 16 Heat ~25%, 8 Heat ~60%.

**4. Risk of Rain 2 (Beads of Fealty):** Hidden effect: transforms final boss to Mithrix (5× harder). Trade item slot + Lunar Coin for harder fight + cosmetic. Brittle Crown: gain gold on hit, lose on damage taken. Good players: +200% gold. Bad: -50%. Shaped Glass stacking: 3 = 8× damage, 12.5% HP. Gesture + bad equipment = suicide.

**5. Dead Cells (Mutations):** Choose 3 from 70+. Massive opportunity cost. Necromancy: heal 5 HP per elite (worthless avoiding elites). Dead Inside: 75% damage reduction cursed (enables cursed rushing). Slot constraints: choosing one = not choosing 67 others. ~15 viable builds from 70+ options. Re-spec: infrequent upgrade stations (soft-lock 3-5 biomes).

**Success Metrics:** No single trade >40% pick rate. Win rate parity: ±10% accepting vs declining. Regret/satisfaction: 30-40% regret, 40-50% satisfied. Build identity: 70-80% report trade-offs shaped playstyle.

**Common Pitfalls:**
- **Hidden consequences:** "???" costs = feel tricked. Always telegraph: "+1 Energy, can't heal."
- **Trap options:** 20% win rate vs 60% baseline = unfun. Both paths should be 45-60%.
- **Reversibility without cost:** Easily undoable = lacks weight. Commitment creates investment.
- **Too many simultaneous:** 5+ in 2 minutes = paralysis. 1-3 per run maintains engagement.
- **No-brainer choices:** "+50% damage, -5% movespeed" = obviously take. Need comparable costs/benefits.

**Synergies:** Push-Your-Luck (#16), High-Risk Paths (#17), Anti-Synergies (#19)

**Session vs Meta:** 100% Session
