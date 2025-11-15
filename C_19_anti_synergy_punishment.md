## 19. Anti-Synergy Punishment (Conflicting Build Elements)

**Psychological Hook:**

Commitment enforcement prevents hedge-betting. Without anti-synergies, optimal = "take everything." Forces build identity: "I'm melee" or "I'm ranged." Mastery demonstration: spotting anti-synergies requires knowledge. New players pick bad combos, suffer, learn. Veterans avoid instinctively.

Build identity creates gameplay coherence. Anti-synergies enforce by punishing mixing. Tension from difficult choices: "This item powerful but ruins build." Taking = sabotage, skipping = FOMO. Cognitive satisfaction from solving puzzle: "How maximize power while avoiding anti-synergies?"

Risk: can feel punishing if hidden. Mitigation: clear communication (display negative stats, warnings).

**Implementation Pattern:**

```python
class AntiSynergySystem:
    def detect_anti_synergies(self, item):
        conflicts = []

        # Character locks
        if item.type == 'ranged' and 'melee_specialist' in self.player_stats:
            conflicts.append({
                'description': 'Brawler reduces ranged damage by 50%',
                'penalty': -0.50
            })

        # Mechanical conflicts
        if item.name == 'Runic Pyramid' and len(self.deck) > 25:
            conflicts.append({
                'description': 'Runic Pyramid + Large Deck = Hand Clogging',
                'penalty': -0.35
            })

        return conflicts
```

**Balancing Guidelines:**

- **Visible penalties:** Display "-50% melee damage" on character screen. Never hide—feels like gotcha.
- **Pure strategies viable:** "All melee" as strong as "all ranged." Anti-synergies enforce specialization, not superiority.
- **Mixed builds penalty:** Hybrid 15-25% weaker than pure. Not unviable, just suboptimal. Allows experimentation.
- **Discoverable through text:** Item descriptions hint: "Reduces ranged damage." Reading comprehension = skill.
- **Teach through failure:** First encounter = survivable lesson, not run-ender. "Oh, that's why mixing is bad."
- **Allow edge cases:** 5-10% successful hybridization (Rainbow Build). Rewards creativity.

**Examples:**

**1. Brotato (Character Locks):** Brawler: -125 range, -50% ranged damage, +15% melee. Ranger: -50% melee, +15% ranged. Anti-synergy: ranged weapon as Brawler = dead slot (50% damage, can't reach). Win rates: pure melee 65%, hybrid 28% = -37%. Rainbow Build exception: 2% via brute-force items.

**2. Slay the Spire (Runic Pyramid):** Don't discard hand. Synergy: small decks (15-20 cards). Win rate: +25%. Anti-synergy: large decks (30+). Win rate: -40%. Hand clogs: Turn 1-4 = 9-13 cards stuck. Can't draw key cards. Solution: exhaust cards, streamline to 10-12. Win rates: <20 cards 72%, >30 cards 38% = -34%.

**3. Hades (Family Favorite):** +10% damage per god (cap 5 = +50%). Synergy: diverse builds (5+ gods). Anti-synergy: focused builds (all Zeus). Getting it focused = 10-20% damage (weak). Strategic conflict: focused enables Duo/Legendary boons. Pick rate: 60% mixed, 15% focused.

**4. Vampire Survivors (Inverse Mode):** Item effects reversed. Amount (projectiles) becomes penalty. Duration (uptime) becomes weakness. Normal: Amount +5 = 5× DPS. Inverse: Amount -5 = 1 projectile (terrible). New priorities: Cooldown reduction = king. Clear rate: 18% first attempt, 62% after learning. Knowledge check.

**5. Risk of Rain 2 (Gesture Conflicts):** Gesture: equipment auto-fires, lose control. Amazing: Capacitor (auto-zap). Deadly: Primordial Cube (sucks enemies to you) = instant death. Auto-casts every 30s = suicide. Solution: scrap Gesture or Cube. Win rates: Gesture + Capacitor 68%, Gesture + Cube 12% = -56%.

**Success Metrics:** Specialization: 60-70% build pure by run 10. Anti-synergy avoidance: 45% pick run 1, 15% run 10 (learning). Win rate gap: pure 55-65%, hybrid 40-50% (+10-20%). Discovery: 80-90% within 5 runs.

**Common Pitfalls:**
- **Hidden penalties:** Not displaying "-50% melee" = don't understand weakness. Always show numbers.
- **No pure path viability:** If mixing required, it's forced diversity (frustrating), not anti-synergy.
- **Too punishing:** -90% power for mixing = "never experiment." -20-30% = "suboptimal but survivable" (better for learning).
- **Obscure interactions:** Gesture + Cube requires deep knowledge. OK advanced, but teach basics first (character locks).
- **No warnings:** Runic Pyramid with 30-card deck should warn: "⚠ May cause hand-clogging."

**Synergies:** Build Variety, Knowledge Checks, Strategic Depth, Forced Trade-Offs (#18)

**Session vs Meta:** 10% Session, 90% Meta (learning prevents future mistakes)
