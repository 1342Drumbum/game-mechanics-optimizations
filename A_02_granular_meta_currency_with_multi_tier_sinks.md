## 2. Granular Meta-Currency with Multi-Tier Sinks

**Psychological Hook:**

Zeigarnik Effect (1927) creates persistent cognitive tension from incomplete goals. "4,523/35,365 Darkness" automatically triggers gap calculation (30,842 remaining), occupying mental bandwidth until resolved. fMRI shows sustained nucleus accumbens activation even when not playing—low-level craving.

Goal gradient (Hull, 1932; Kivetz et al., 2006) accelerates non-linearly at 70% completion. Loyalty program research: 20-35% purchase frequency increase near reward threshold. In games: 30-50% longer sessions when "close" to affording upgrade.

Sunk cost fallacy (Arkes & Blumer, 1985): after 50hrs and 20K currency, abandoning = "wasting" investment. Creates irrational commitment escalation. Designers must balance commitment vs exploitation.

Multi-tier pricing leverages anchoring (Tversky & Kahneman, 1974): costs of 50/500/5,000 make 50 feel "cheap" despite requiring 30min. Endowed progress effect (Nunes & Drèze, 2006): Hades grants 50 Darkness minimum per death, making failed runs feel productive.

Currency granularity creates pseudo-progress: earning 73 Darkness (specific) > "earned resources" (vague).

**Implementation Pattern:**

```python
class MetaProgressionSystem:
    def __init__(self):
        self.base_earnings_per_run = 75  # median
        self.min_earnings = 40  # 53% of average
        self.max_earnings = 200  # 267% of average
        self.variance_coefficient = 0.6  # sweet spot

        self.total_currency_needed = 50000
        self.expected_hours = 150
        self.sessions_to_complete = 450

        # Tier structure
        self.upgrade_tiers = {
            'cheap': (30, 200, 20),    # (min, max, count)
            'medium': (300, 1500, 15),
            'expensive': (2000, 5000, 10),
            'premium': (7000, 15000, 5)
        }

    def calculate_earnings(self, run_performance):
        base = self.base_earnings_per_run
        time_bonus = (run_performance.time_seconds / 60) * 5
        kill_bonus = min(run_performance.kills / 10, 20)
        room_bonus = run_performance.rooms * 3
        boss_bonus = 50 if run_performance.boss_kill else 0

        total = (base + time_bonus + kill_bonus + room_bonus + boss_bonus)
        variance = random.uniform(0.7, 1.3)
        return clamp(total * variance, self.min_earnings, self.max_earnings)
```

**Tiered Pricing:**
- **Tier 1 (50-200):** Fast initial hook. 1-2 runs (15-40min). 20-30 small upgrades. Instant gratification.
- **Tier 2 (300-1,500):** Medium commitment. 5-15 runs (2-4hrs). "I'm invested" psychology. 15-20 upgrades.
- **Tier 3 (2,000-5,000):** Long-term goals. 20-50 runs (8-20hrs). Strong goal gradient. 10-15 upgrades.
- **Tier 4 (7,000-15,000):** 100+hr retention. Status markers. 5-10 premium items.

**Optimization:**
- **Minimum guarantee:** Never <40% of average (prevents "wasted run").
- **Maximum ceiling:** Never >250% of average (some grind is desirable).
- **Variance:** 60-80% CV optimal. <40% boring, >120% frustrating.
- **Session-to-goal:** 3-8 sessions per medium upgrade. <3 trivial, >10 grindy.

**Examples:**

**1. Hades (35,365 Darkness Masterclass):**
- Mirror of Night: 13 paths, 26 skills, 35,365 total Darkness. Calibrated for 80-120hrs.
- Tier 1 (0-3K, 20-40 runs): Shadow Presence (150D, +50% backstab). Death Defiance (1,530D, 3 extra lives—crucial, +50-80% win rate).
- Tier 2 (3-10K): Boiling Blood (1,500D), Olympian Favor (2,750D). Choices emerge.
- Tier 3 (15-35K): Final ranks cost 500-2,000D each. Goal gradient maximizes at 85% (30K/35K).
- Earnings: Tartarus death (35-65D), Asphodel (70-110D), Elysium (100-160D), Victory (150-250D). Average: 105D/35min.
- 35,365÷105 = 337 runs = 196hrs. Actual: 80-150hrs (skill improves).
- Steam: 57.8% escaped once, 9.2% maxed Mirror. Core 10% retain 100+hrs for completion.
- Developer: "75D minimum = 0.2% progress per run—small enough for commitment, large enough for tangibility."

**2. Dead Cells (Dual-Currency):**
- Cells: Abundant, lost on death, spent on upgrades. Blueprints: Rare, permanent, unlock items.
- Cells: 30-80/run (0BC), 100-250/run (5BC). Total needed: 75K. Realistic: 120-180hrs.
- Blueprints: 195 total. Drop rate: 0.4% base, +0.3%/failed kill (pity). After 10 kills: 3.4%.
- Finding blueprint = loss aversion. Must survive to Collector. Creates emergent tension.
- Flawless outfit: Legendary blueprint from no-hit bosses. 15-30 attempts, 30-60hrs.
- Steam: 71.3% unlocked 10, 24.7% unlocked 50, 2.8% all (150-250hrs).
- Developer: "Cells = tension (push for elite or play safe?). Blueprints = permanent (no frustration)."

**3. Vampire Survivors (Gold Incremental Power Creep):**
- 21 PowerUps, 5K each = 105K total.
- Earnings: Mad Forest 500-1,200g, Dairy Plant 1,500-2,500g, Gallo Tower 2,000-3,500g.
- Amount (projectiles): 25K total (most expensive, most impactful). +5 projectiles (Whip shoots 7 vs 2).
- Timeline: 10hrs (12Kg), 30hrs (45Kg), 60hrs (117Kg), 100hrs (270Kg, all maxed).
- Power impact: +25% dmg + 250% coverage + 50% HP + 5 armor + 25% speed + 50% luck = 5-7× baseline.
- 40M sales, D30: 30-40%. Developer: "2% better every run. After 50 purchases, 2× stronger, didn't notice shift."

**4. Deep Rock Galactic (Horizontal Progression):**
- 160+ Overclocks. Acquisition: 3-4/week (Core Hunts, Deep Dives, Machine Events).
- 160÷3.5 = 45.7 weeks = 11 months. Realistic: 12-18 months.
- Fat Boy (Engineer PGL): Tactical nuke, radiation fallout, -50% ammo. Transforms playstyle. 3-6 month grind.
- Psychology: Early (0-3mo, 12-50 OCs) = high novelty. Mid (3-8mo, 50-120) = goal gradient + RNG frustration. Late (8-18mo, 120-160) = completion + sunk cost.
- D30: 35-45%, 1yr: 15-20% (extreme for co-op shooter).
- Developer: "Weekly gate prevents FOMO burnout. Want years of engagement, not week-long grinds."

**5. Slay the Spire (Unlock Expanding Possibility):**
- Currency = experience/victories. 75 cards/character, unlock 1 per 5 victories.
- 4 characters × 75 cards = 300. Required: 80 victories. Per character: 30-50hrs. All: 140hrs.
- Ascension 0-20 per character: 200-400hrs for all.
- Philosophy: Unlocks expand strategy, not power. 0hr runs: simple. 40hr runs: extreme variety.
- Steam: 48.2% unlocked Watcher, 18.7% Asc10, 3.1% Asc20 all.
- Developer: "240 cards day 1 = paralyzing. 50 = manageable. By hr 30, appreciate full complexity."

**Success Metrics:**
- D1: 45-55% (vs 30-40% without)
- D7: 35-45% (vs 15-25%)
- D30: 20-30% (vs 5-10%)
- Session length: 20-35min (optimal)
- Completion: 80-180hrs (sweet spot)
- Sunk cost threshold: 15-20hrs (retention spikes 15-25%)

**Common Pitfalls:**
- Grind >300hrs alienates 95%. Even completionists quit ~200hrs.
- Grind <30hrs removes hook too early. Sweet spot: 80-150hrs.
- Hidden costs breed resentment. Always show "X/Y."
- Meaningless unlocks (+0.5% crit for 5K) feel insulting. Need 5%+ impact.
- Slow starts (<5 unlocks in 5hrs) = churn before investment.

**Synergies:** Incremental Upgrades (#7), Prestige Loops (#5), Collections (#4), First Win (#8), Daily Logins (#38), Achievements (#35)

**Session vs Meta:** 10% Session (earning in-run), 90% Meta (spending between runs, 50-200hr span)

---
