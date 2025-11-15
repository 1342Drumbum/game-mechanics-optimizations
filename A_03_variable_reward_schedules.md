## 3. Variable Reward Schedules (Loot Rarity Tiers)

**Psychological Hook:**

Variable-ratio reinforcement (Skinner, 1953) creates addiction-resistant behavior. Unpredictable reward timing produces more persistent patterns than fixed schedules. Skinner compared his apparatus to slot machines—both create compulsive behavior through unpredictability.

Dopamine fires strongest for *unexpected* rewards (Schultz et al., 1997). Common→Legendary surprise creates 2-3× larger dopamine surge than predicted rewards. Unpredictability maintains sensitivity; predictable rewards create tolerance.

Near-miss effect (Clark et al., 2009): receiving Rare when hoping for Epic activates 60-70% of win-response brain regions (insula, ACC). "Almost Epic, next chest!" Slot machine research: near-misses increase persistence 30-40% vs pure losses.

Loss aversion (Kahneman & Tversky, 1979): receiving Common when Epic possible feels like *losing* Epic, not neutrally receiving Common. Emotional range (disappointment→elation) wider than binary (got/didn't).

Ludic Loop: Action → Reward evaluation → Anticipation → Action. Variable rewards keep "evaluation" exciting for 1,000+ repetitions. Fixed rewards bore after 10-20.

Ethically borders gambling. Loot box research: 20-30% exhibit problematic behavior (Drummond & Sauer, 2018). Responsible implementation: (1) no real-money RNG, (2) pity systems, (3) transparent odds, (4) skill-based alternatives.

**Implementation Pattern:**

```python
class RaritySystem:
    def __init__(self):
        self.rarities = {
            'Common': {'weight': 600, 'power_mult': 1.0, 'color': '#FFF'},
            'Uncommon': {'weight': 250, 'power_mult': 1.3, 'color': '#0F0'},
            'Rare': {'weight': 120, 'power_mult': 1.7, 'color': '#0080FF'},
            'Epic': {'weight': 25, 'power_mult': 2.3, 'color': '#A020F0'},
            'Legendary': {'weight': 5, 'power_mult': 3.5, 'color': '#FFD700'}
        }
        # Probabilities: 60%, 25%, 12%, 2.5%, 0.5%

        self.pity_thresholds = {'Epic': 40, 'Legendary': 100}
        self.pity_counter = {'Epic': 0, 'Legendary': 0}

    def roll_rarity(self, luck_modifier=0):
        # Check pity first
        if self.pity_counter['Legendary'] >= 100:
            self.pity_counter = {'Epic': 0, 'Legendary': 0}
            return 'Legendary'

        if self.pity_counter['Epic'] >= 40:
            self.pity_counter['Epic'] = 0
            return 'Epic'

        # Apply luck to rare weights
        weights = self.rarities.copy()
        weights['Rare']['weight'] *= (1 + luck * 0.5)
        weights['Epic']['weight'] *= (1 + luck * 0.8)
        weights['Legendary']['weight'] *= (1 + luck * 1.2)

        result = weighted_random(weights)

        # Update pity
        if result != 'Legendary': self.pity_counter['Legendary'] += 1
        if result not in ['Epic', 'Legendary']: self.pity_counter['Epic'] += 1

        return result
```

**Probability Tiers:**
- **Common (50-70%):** Baseline. Frequent, expected.
- **Uncommon (20-30%):** Common enough for consistency (every 3-4 drops).
- **Rare (8-15%):** Noticeable upgrade. Every 7-12 drops.
- **Epic (1-5%):** Excitement. Every 20-100 drops.
- **Legendary (0.1-1%):** Euphoria. Every 100-1000 drops.

**Pity Systems:**
- Hard pity: Guaranteed at threshold (100 chests = Legendary).
- Soft pity: Increasing odds (1% base, +0.5%/chest after 50).
- Smart pity: Track across sessions, prevent "unlucky" 200-chest droughts.

**Visual/Audio Feedback:**
- Common: No effect, white text
- Rare: Pulsing glow, distinct sound
- Epic: Particle aura, musical flourish
- Legendary: Beam of light, epic fanfare, screen shake

**Examples:**

**1. Diablo 3 (Loot 2.0 Revolution):**
- Pre-2.0 (2012): 0.02% Legendary rate. Players farmed 100hrs without drops. Auction House > playing.
- Post-2.0 (2014): 5-10% Legendary rate. Smart loot (70% class-appropriate). Pity timer (6-8hrs = guaranteed).
- Psychology shift: "When will I get Legendary?" → "Which Legendary this session?"
- Retention: Loot 2.0 patch increased D7 by 40%. Player count tripled.
- Ancient/Primal tiers (2015): Ancient (10% of Legendaries, 1.3× stats). Primal (1/400 Legendaries, perfect stats).
- Endgame: Chasing Primal ancient weapons. 400+ hour grinds for perfect rolls.

**2. Borderlands 3 (Billion Guns):**
- 1 billion+ weapon combinations via procedural parts. Rarity tiers determine part quality.
- White (50%), Green (30%), Blue (12%), Purple (6%), Orange (1.5%), Pearlescent (0.1%).
- Legendary: 1.5% base. Boss-specific: 10% (targeted farming).
- Dedicated farms: Farm Graveward 50× for specific Legendary (10% × desired drop from loot pool = 2-5% effective).
- Community: "Loot the universe" events (temporary 5× Legendary rates). FOMO spikes engagement.
- D7: 30-35%. 100+ hour players: 25%.

**3. Risk of Rain 2 (Rarity = Power Tiers):**
- White (Common, 79%): +15-25% power. Green (Uncommon, 20%): +25-50% power. Red (Legendary, 1%): +100-300% power.
- 57 Leaf Clover (Legendary): All RNG rolls twice, take better. Doubles crit chance, proc rates, chest rarity.
- Finding Clover minute 10 vs minute 25 = 3× power difference in endgame.
- Legendary items define runs. "Got Clover min 8" = likely victory. "No Clover by min 20" = struggle.
- Scrapper stations: Convert 5 Common → 1 Uncommon, 5 Uncommon → 1 Legendary. Control variance.

**4. Hades (Boon Rarity Tiers):**
- Common (70%): Base effect. Rare (25%): 1.3-1.5× effect. Epic (4.8%): 1.8-2.2× effect. Heroic (0.2%): 2.5-3.0× effect.
- Olympian Favor (Mirror upgrade): +10-40% Epic/Heroic chance. Meta-progression affecting RNG.
- God's Legacy keepsakes: Force specific god first 3/4/∞ rooms. Control randomness via strategic choice.
- Eurydice nectar: "Refreshing Nectar" upgrades 3 random boons +1 rarity. Strategic reroll.
- Psychology: Flexibility. Tailor variance tolerance (force gods vs full random). Casuals force, hardcore embrace chaos.

**5. Monster Hunter (Desire Sensor Meme):**
- Rarity tiers: 1-3% for rare drops (gems, plates, mantles). Requires 3-5 of same rare for crafting.
- Rathalos Ruby: 3% carve, 2% capture, 5% tail cut. Need 3 for armor set. Expected: 50-100 hunts.
- Community: "Desire sensor" meme (game "knows" what you want, denies it). Confirmation bias + RNG frustration.
- Kulve Taroth siege: Farm 20-30 runs for specific weapon (0.5% specific drop in loot pool of 300+).
- Psychology: Social validation. Hunt in groups, shared suffering. "200 hunts, no gem!" becomes badge.
- D30: 20-25%. 500+ hour players: 15% (exceptional).

**Success Metrics:**
- Drop frequency: 1 rare item per 15-30min optimal. Too frequent (every 5min) = devaluation. Too rare (>1hr) = frustration.
- Pity timing: Guarantee rare drop every 6-10hrs worst-case. Prevents "unlucky" quit spiral.
- Near-miss display: Show "almost" (blue chest instead of purple) increases persistence 20-30%.
- Session length: Variable rewards increase avg session by 10-20min (vs fixed rewards).

**Common Pitfalls:**
- Pure RNG, no pity: 5% of players experience 3× expected drought, quit.
- Hidden odds: Breeds mistrust. Always display percentages.
- Too many tiers: 8+ rarity levels = diluted excitement. Optimal: 4-5.
- Pay-to-roll: Ethical disaster. Keep real money out of RNG.
- Meaningless rarity: Legendary needs 2×+ power vs Common, not 10% better.

**A/B Testing:**
- Test Legendary rates: 0.5% vs 1.0% vs 2.0%. Measure frustration (quit rate) vs satisfaction (session length).
- Test pity timing: 50 drops vs 100 drops. Balance safety net vs earned achievement.
- Test visual feedback intensity. Epic fanfare increases perceived value 30-40% even with identical stats.

**Synergies:** Exponential Power (#1), Collection Completion (#4), Crafting Systems (#13), Achievement Hunting (#35), Boss Rush (#23)

**Session vs Meta:** 80% Session (drops occur in-run, immediate impact), 20% Meta (chasing specific rares across runs, collection)

---
