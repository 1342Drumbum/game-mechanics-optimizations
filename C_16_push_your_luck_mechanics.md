## 16. Push-Your-Luck Mechanics (When to Stop)

**Psychological Hook:**

Optimal stopping problem creates cognitive tension. Gambling psychology: "one more" compulsion stems from near-miss activation. Reward prediction error (Schultz, 1998) fires strongest when outcome timing unpredictable—maintains dopamine sensitivity.

Regret aversion creates dual pressure: stopping early = "I could have won more!" Pushing too far = "I lost everything!" Peak-end rule (Kahneman, 1999): memories weighted by emotional peaks and final moments. Creates stories: "I had 2,000 gold, pushed to 3,500, lost everything!"

Loss aversion: losses feel 2-2.5× worse than gains. Losing 75% must be balanced by 2-3× reward potential. If 5 draws yield 500, draw 6 must offer 300+ (total 800) to maintain positive EV despite loss aversion.

**Implementation Pattern:**

```python
class PushYourLuckSystem:
    def __init__(self):
        self.base_reward = 100
        self.reward_multiplier = 1.2  # 20% increase per round
        self.base_failure_chance = 0.05
        self.failure_increment = 0.08  # +8% per round
        self.loss_on_failure = 0.75  # lose 75%

    def draw_reward(self):
        round_reward = self.base_reward * (self.reward_multiplier ** self.current_round)
        failure_chance = min(0.95, self.base_failure_chance +
                            (self.failure_increment * self.current_round))

        # Calculate EV
        success_ev = (1 - failure_chance) * (self.accumulated_reward + round_reward)
        failure_ev = failure_chance * (self.accumulated_reward * (1 - self.loss_on_failure))
        expected_value = success_ev + failure_ev - self.accumulated_reward

        return expected_value
```

**Balancing Guidelines:**

- **Escalation rate:** 5-10% failure increase per round. <5% = no tension, >15% = too punishing.
- **Loss percentage:** 50-75% on failure. 100% loss = feel-bad. <50% = no stakes.
- **Reward scaling:** 1.15-1.25× per round (multiplicative). Must exceed risk increase to maintain positive EV for 3-6 rounds.
- **Checkpoints:** Every 3-5 rounds allows "bank and continue" strategy.
- **Display information:** Show probabilities. Transparency reduces "cheated" feeling.
- **Optimal stopping:** Design sweet spot at rounds 4-6. Early = safe but suboptimal. Late = greedy and punished.

**Formulas:**

```
Expected Value = (Success_Rate × Total_if_Continue) - (Failure_Rate × Loss_Amount) - Current_Total

Optimal Stopping: When EV < 0
Solve: (1-p_fail) × (Current + Reward) + p_fail × Current × (1 - loss_rate) - Current < 0

Risk Tolerance:
Risk-Averse: Stop when EV < +10% current
Balanced: Stop when EV < 0
Risk-Seeking: Continue until EV < -20%

Reward Curve: Reward(n) = Base × Multiplier^n
Failure Curve: Failure(n) = min(0.95, Base_Chance + Increment × n)
```

**Examples:**

**1. Binding of Isaac (Devil Deals):** Trade 1-2 heart containers for items. Start: 3 containers (6 HP). Trading 2 for Brimstone = 1 container (2 HP) remaining. Power: 15 DPS vs 3.5 base = 4.3×. At 2 HP, one mistake = death. Win rate 1 HP + Brimstone: 45% vs 25% baseline. Pick rate: 78% floors 1-3, 34% floors 5+ (context-dependent).

**2. Slay the Spire (Knowing Skull):** Spin for rewards, 3rd curse kills. First: 25% curse. Second: 33%. Third: 50%. Expected value 2 spins: 1.42 rewards, 17% curse risk. Three spins: 50% death (trap). Players learn: 2 spins max. Death rate: 8% first, 12% second, 42% third. Design elegance: third option exists as greed trap.

**3. Risk of Rain 2 (Shrine of Chance):** First use: 5g, 65% item. Each failure: cost 1.2×, odds drop. Use 1-4: +91% ROI. Five uses: +12%. Six: -15%. Optimal: 3-4 uses. Gold time-limited (difficulty scales). Tension: waste time AND gold.

**4. Hades (Chaos Gates):** Accept curse for 3-5 chambers, receive powerful boon. Curse: "+40% damage taken." Boon: +50-100% damage. Risk assessment: Elysium at 150 HP = survivable. Styx at 50 HP = death. Acceptance: 75% Tartarus, 40% Elysium, 15% Styx. Strategic timing crucial.

**5. Dead Cells (Cursed Chests):** Powerful item, 10 curse (any hit = death). Clear: 10 kills no-hit. Success rate: 85% at 0BC (skilled), 35% at 4BC. Take early (biome 1-2, slow enemies). Avoid late (biome 4-5, swarms). Optional: risk-averse skip 100%, clear game 20% slower.

**Success Metrics:** 50-70% engagement (experienced players). Regret stories shared. Optimal stopping learned (10-20 sessions, convergence ±1 round). Emotional range: ecstatic wins, crushing losses, tense decisions. 20-30% viable path avoiding entirely.

**Common Pitfalls:**
- **Hidden probabilities:** Not displaying odds = "rigged" feeling. Always show percentages.
- **100% loss on failure:** Losing everything = "wasted time" resentment. 50-75% loss maintains stakes with partial rewards.
- **Impossible odds:** Round 3 = 95% failure = players feel trapped. Cap at 70-80% to maintain agency.
- **No skill mitigation:** Pure RNG removes skill expression. Add elements players control.
- **Mandatory participation:** Optional = pride. Forced = resentment.

**Synergies:** High-Risk Paths (#17), Sacrifice Mechanics (#18), Variable Rewards (#3)

**Session vs Meta:** 100% Session (resets between runs)
