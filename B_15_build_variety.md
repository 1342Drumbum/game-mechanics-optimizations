## 15. Build Variety (Multiple Viable Strategies)

**Psychological Hook:** Autonomy creates ownership. Mastery through diverse execution validates competence. Novelty maintains engagement. Discovery rewards exploration. Prevents optimization fatigue from single "correct" build.

**Implementation Pattern:**

```python
class BuildVarietySystem:
    """Ensure multiple strategies remain viable from normal to extreme difficulties"""

    def __init__(self):
        self.archetypes = {}
        self.viability_by_difficulty = {}

    def validate_balance(self, weapon, difficulty_level):
        """All weapons should perform similarly at same difficulty"""
        test_runs = self.simulate_runs(weapon=weapon, difficulty=difficulty_level, iterations=1000)
        average_score = mean(run.score for run in test_runs)
        benchmark = self.get_benchmark_score(difficulty_level)

        # Within 85-115% of benchmark
        if average_score < benchmark * 0.85:
            log_warning(f"{weapon} underperforming")
            return False
        if average_score > benchmark * 1.15:
            log_warning(f"{weapon} overperforming")
            return False
        return True

    class ItemScaling:
        """Items remain relevant throughout progression"""
        def calculate_value(self, item, current_stage):
            if item.scaling_type == 'multiplicative':
                return item.base_value * (1 + current_stage * 0.2)  # Grows with power
            elif item.scaling_type == 'percentage':
                return player.total_stats * item.percentage  # Scales with stats
            elif item.scaling_type == 'flat':
                return item.base_value  # Fixed (diminishes relatively)

    def determine_best_strategy(self, context):
        """No universal best. Best depends on context."""
        if context.enemy_type == 'swarm':
            return {'weapon': 'aoe', 'stat': 'area_damage'}
        elif context.enemy_type == 'boss':
            return {'weapon': 'single_target', 'stat': 'burst_damage'}
        elif context.player_health < 0.30:
            return {'priority': 'defensive_items'}
        else:
            return {'priority': 'offensive_items'}

    class ArchetypeSystem:
        """Define distinct playstyles (Dead Cells color system)"""
        archetypes = {
            'brutality': {  # Red
                'focus': 'melee, critical hits, speed',
                'health': 'moderate',
                'playstyle': 'aggressive, high mobility',
                'win_condition': 'burst damage, dodge perfect'
            },
            'tactics': {  # Purple
                'focus': 'ranged, turrets, traps',
                'health': 'low (glass cannon)',
                'playstyle': 'tactical positioning',
                'win_condition': 'sustained dps, keep distance'
            },
            'survival': {  # Green
                'focus': 'shields, heavy weapons, dots',
                'health': 'high',
                'playstyle': 'tank, methodical',
                'win_condition': 'attrition, outlast enemies'
            }
        }

        def apply_scroll(self, scroll_color, player):
            """Scrolls boost color-matched gear. Rewards commitment."""
            player.stats[scroll_color] += 1
            for item in player.equipment:
                if item.color == scroll_color:
                    item.power = base_power * (1 + player.stats[scroll_color] * 0.15)
```

**Balancing Guidelines:**

**Viability Across Difficulty:**
- Normal (Diff 1): 80-90% builds viable. Suboptimal still win. Creativity encouraged.
- Hard (Diff 10): 50-70% viable. Optimization helpful. Hybrid builds weaker.
- Extreme (Diff 20): 20-40% viable. Optimization required. Pure builds dominate.

**Design Principle:** "Early choices shouldn't be drowned out by later ones. An item picked at start should matter by final boss." (multiplicative/percentage scaling keeps early items relevant)

**Preventing Build Staleness:** Forced adaptation: if last 3 runs = poison_build, poison_items_weight ×0.5, alternatives ×1.5. Build challenges: melee_only (1.5× rewards), no_healing (2.0×), glass_cannon (max_health=1, damage=3.0×). Counter dominant: if strategy >40% play rate AND >65% win rate, nerf 10%, buff alternatives 15%.

**Build Diversity Metrics:**

```python
# Shannon diversity index: 0.0 (one strategy) to 1.0 (perfectly balanced)
diversity = -sum(p × log(p) for p in strategy_distribution) / log(num_strategies)

# Target Metrics:
# - Diversity index: >0.7 (healthy variety)
# - Viable strategies: >70% have 45%+ win rate
# - Dominant strategy: <30% pick rate
# - Worst strategy: >35% win rate
```

**Examples:**

**1. Slay the Spire: Ironclad Archetypes (6+ viable):** Strength Scaling (Demon Form, Limit Break, Heavy Blade): 58% win rate, medium difficulty. Exhaust Synergy (Dead Branch, Corruption, Feel No Pain): 72% win rate when assembled, hard difficulty (rare). Block Tank (Barricade, Entrench, Body Slam): 52% win rate, easy. Perfected Strike (meme build): 41% win rate, easy. Even "meme" builds win at base difficulty. High difficulty rewards optimization.

**2. Hades: 6 Weapons × 9 Gods × 4 Aspects = 200+ Combinations:** All weapons: 45-55% win rate at 0 Heat (balanced). All aspects: 40-60% (player preference). God variety: 60-80% runs use 4-5 gods. Examples: Divine Dash (Poseidon), Hunting Blades (Artemis), Merciful End (Ares+Athena), Chiron Bow (Artemis Special).

**3. Risk of Rain 2: On-Hit Proc vs Tank vs Crit vs One-Shot:** On-Hit (Ukulele, AtG, Sticky Bomb): 65% win, exponential scaling. Tank (Aegis, Infusion, Tougher Times): 58% win, linear/safer. Crit (Glasses, Scythe, Predatory Instincts): 62% win, exponential after 50% crit. One-Shot (Crowbar, Focus Crystal, Shaped Glass): 55% win, high variance, glass cannon. Meta health: no build >35% pick rate. Win rates 45-65%. Hybrid builds possible (proc+crit).

**Success Metrics:** Build diversity index >0.7 (Shannon). Viable strategies >70% have 45%+ win rate. Dominant strategy <30% pick rate. Player reports: 80%+ "every run feels different." Repeat experimentation: 80%+ players try different builds in first 10 runs.

**Common Pitfalls:**
- Single dominant strategy: 80% converge on same build. Nerf dominant 10%, buff alternatives 15%.
- Trap options: items seem good but <35% win rate. Buff or clearly communicate weakness.
- Required items: "must have X to win at high difficulty." Ensure multiple paths to victory.
- Ignoring low-level balance: only balance extreme difficulty. All difficulties should offer variety.
- Power silos (no hybrids): builds completely incompatible. Allow transitioning, hybrids 15-20% suboptimal (not unviable).
- Early choices irrelevant: first 5 items don't matter endgame. Multiplicative scaling keeps early items relevant.

**Synergies:** Build-Around Items (#9), Anti-Synergy Punishment (#19), Draft Choice (#40)

**Session vs Meta:** 95% Session (build unique to run), 5% Meta (learning viable builds)

---
