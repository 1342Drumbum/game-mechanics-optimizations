## 13. Adaptive Difficulty (AI Director / Comeback Mechanics)

**Psychological Hook:** Flow state when challenge matches skill. Loss aversion: rubber-banding prevents resignation (15-25% quit reduction). Perceived fairness: skill attribution not RNG. Close games more memorable than blowouts.

**Implementation Pattern:**

```python
class AIDirectorSystem:
    """Dynamic tension system (Left 4 Dead). NOT traditional DDA (invisible stat tweaks).
    Instead: controls pacing, spawn timing, dramatic moments."""

    def __init__(self):
        self.survivor_intensities = {}
        self.team_intensity = 0.0
        self.current_phase = 'rest'
        self.phase_timer = 0.0
        self.spawn_pools = {
            'ambient': ['zombie', 'shambler', 'crawler'],  # Low threat
            'common': ['runner', 'spitter', 'bloater'],    # Standard
            'elite': ['hunter', 'smoker', 'charger'],      # High threat
            'boss': ['tank', 'witch']                       # Climax
        }

    def update_player_intensity(self, player, delta_time):
        """Track individual stress level. Range: 0.0 (calm) to 1.0 (max stress)"""
        intensity = self.survivor_intensities.get(player.id, 0.0)

        # Intensity increases
        if player.events.attacked_by_enemy:
            intensity += 0.10
        if player.events.killed_close_enemy:
            intensity += 0.05  # Only close kills (not sniping)
        if player.events.low_health and player.health < 0.3:
            intensity += 0.08
        if player.events.incapacitated:
            intensity = 1.0  # Instant maximum

        # Does NOT increase (important)
        # player.events.friendly_fire: pass  # Player mistakes don't increase pressure
        # player.events.killed_distant_enemy: pass  # Sniping not stressful

        # Natural decay
        intensity -= 0.01 * delta_time
        intensity = max(0.0, min(1.0, intensity))

        self.survivor_intensities[player.id] = intensity
        return intensity

    def update_dramatic_phase(self, delta_time):
        """Create dramatic arcs: Build → Peak → Release → Rest"""
        self.phase_timer += delta_time

        phases = {
            'rest': {'duration': 30.0, 'next': 'build', 'spawn_rate': 0.3},
            'build': {'duration': 60.0, 'next': 'peak', 'spawn_rate': 1.0},
            'peak': {'duration': 15.0, 'next': 'release', 'spawn_rate': 2.0},
            'release': {'duration': 20.0, 'next': 'rest', 'spawn_rate': 0.5}
        }

        current = phases[self.current_phase]

        if self.phase_timer > current['duration']:
            self.current_phase = current['next']
            self.phase_timer = 0.0

        # Override: team intensity critical → force peak
        if self.team_intensity > 0.85 and self.current_phase != 'peak':
            self.current_phase = 'peak'
            self.phase_timer = 0.0

    def make_spawn_decisions(self, player_states):
        """Spawn enemies based on intensity and phase"""
        if self.team_intensity < 0.3:
            self.spawn_from_pool('ambient', rate=0.5)
            self.consider_quiet_period()
        elif self.team_intensity < 0.7:
            self.spawn_from_pool('common', rate=1.0)
            if random.random() < 0.15:
                self.spawn_from_pool('elite', count=1)
        else:  # High intensity
            if self.current_phase == 'peak':
                self.trigger_horde_event()
                if random.random() < 0.25:
                    self.spawn_from_pool('boss', count=1)
            if self.phase_timer > 10.0:
                self.schedule_quiet_period()
```

**Balancing Guidelines:**

**Intensity Calculation:**
- Range: 0.0 (safe) to 1.0 (critical danger)
- Increase: 0.05-0.10 per stressful event (attacked, low HP)
- Decay: 0.01/sec (60-100s from max to zero)
- Instant triggers: incapacitation = 1.0
- Non-factors: friendly fire, distant kills (player skill shouldn't increase difficulty)

**Phase Durations:** Rest: 20-40s (recovery). Build: 45-90s (gradual tension). Peak: 10-20s (intense action). Release: 15-30s (wind-down). Total cycle: 90-180s (1.5-3 min).

**Spawn Control:** `spawn_rates = {'rest': 0.2, 'build': 1.0, 'peak': 2.5, 'release': 0.5}`. Enemy types: low intensity = 70% ambient/25% common/5% elite. High = 10%/50%/30%/10% boss.

**Examples:**

**1. Left 4 Dead: AI Director:** Intensity events: attacked +0.10, killed close +0.05, incapacitated +1.00 (instant max), health<30% +0.08, friendly fire/distant kills +0.00 (intentional). Team intensity = average survivors. Decisions: <0.3 = relax phase (30-45s no spawns). 0.3-0.7 = build (standard + occasional special). >0.7 = peak (horde + Tank 25% chance + intense music). After 10-15s peak, force relax. Spawn intelligence: 60% behind (pressure), 30% ahead (flanking), 10% ambush, never visible. Success: 65-75% campaign completion. 85% "felt fair" (vs 45% traditional DDA).

**2. Hades: God Gauge System (Pity):** Ignored god gauge increases. Chosen god resets. `prob = base_prob + (gauge × 0.05)`. Ignored god after 5 chambers: +25% appearance. Keepsake override: first boon in region guaranteed. Duo appearance: 15-25% when requirements met. Helps struggling players get diversity, enables synergy targeting.

**3. Risk of Rain 2: Hidden Director:** `Difficulty = Base + (Time/60s) × Scaling`. Min 5: Diff 2.0. Min 15: 5.0. Min 30: 10.0. Min 60: 20.0. NOT performance-based (skilled not punished). Creates urgency (can't camp). Hidden spawns: if idling, frequency ×1.5 + spawn elites. If moving fast, frequency ×0.85 (reward momentum). Horde: every 120s, frequency ×3.0 for 30s.

**Success Metrics:** Flow state: 60-75% "in the zone." Completion: skilled 60-75%, average 35-50%. Fairness: >75% "felt fair" even on deaths. Comeback rate: behind 30% should win 20-30%. Dramatic moments: 3-5 peak intensity/session (every 8-15 min).

**Common Pitfalls:**
- Obvious rubber-banding: players notice AI catches up = "cheating." Subtle adjustments (5-15%), attribute to player mistakes.
- Too reactive: difficulty ping-pongs wildly every 30s. Smooth changes, moving averages, phase systems (1.5-3 min cycles).
- Punishing skill: getting better makes game harder = negative feedback. Difficulty based on time/phase, not performance.
- No player control: "Game plays itself." Difficulty sliders + adaptive (both available).
- Hidden manipulation: players discover invisible stat changes = trust broken. Transparent systems (L4D philosophy shared publicly).

**Synergies:** Flow State Design (#29), Variable Rewards (#3), Comeback Rewards (#21)

**Session vs Meta:** 100% Session (resets completely each run)

---
