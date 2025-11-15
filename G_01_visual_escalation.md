## 36. Visual Escalation (Screen-Fill Growth)

**Psychological Hook:** Visual representation of power progression creates tangible power fantasy. Leverages peripheral vision satisfaction—player sees entire screen responding to their power. Mimics biological threat response (screen-filling = dominance). Novelty escalation prevents adaptation—early game (sparse) vs late game (chaos) feels dramatically different. Anchoring effect makes early game feel weak, amplifying late-game satisfaction.

**Implementation Pattern:**
```
Early Game (0-5 min):
- Single projectile, small radius (50-100px)
- 2-5 enemies on screen
- Clear space between entities
- Particle count: 5-10 per second

Mid Game (5-10 min):
- 3-6 simultaneous effects
- 15-25 enemies on screen
- Screen coverage: 30-50%
- Particle count: 50-100 per second

Late Game (10-20 min):
- 10-20+ simultaneous effects
- 50-100+ enemies on screen
- Screen coverage: 80-100%
- Particle count: 300-500 per second
- Performance optimization required (particle pooling)
```

Ensure visibility: character outline/highlight persists, minimap shows player position, audio cues for damage taken.

**Examples:**

1. **Vampire Survivors:** Minute 1: Single whip (200px range), 5-10 enemies. Minute 10: 6 evolved weapons (Vandalier spirals screen-wide, Holy Water covers 50% screen, Garlic creates 300px aura). Minute 20: Screen completely filled with projectiles—player character obscured by effects. Particle count escalates 50× from start to finish. Performance mode exists because effects literally overwhelm hardware.

2. **Hades:** Early Tartarus: Single attack (Stygian Blade 150 range), 3-5 enemies per chamber, clean combat. Mid-run Elysium: 3 boons stacking (Ares Doom AoE + Dionysus Festive Fog + Zeus chain lightning), 8-12 enemies, 30% screen coverage. Final Styx: Privileged Status + Hunting Blades + Triple Casts = 6 bouncing projectiles clearing entire rooms automatically, 60-80% screen visual saturation.

3. **Risk of Rain 2:** Stage 1 (3 min): Single weapon (Commando pistol, 10 shots/sec), 5-8 enemies visible. Stage 5 (25 min): 40+ item stacks create: Ukulele chains (3-jump lightning every 3rd hit), Missiles (10% chance), Will-o-Wisp explosions (chain reactions), Gasoline fire trails. Screen fills with: Damage numbers (50+/sec), Electric arcs, Explosions (15-20 simultaneous), Enemy death effects. Particle count exceeds 1000/sec. GPU usage spikes 60% → 95%.

4. **Devil May Cry 5:** Mission start: Basic combo (5-hit sword), 2-3 enemies, clean arena. SSS Rank moments: Cavaliere Angelo motorcycle dash (screen-wide trail), Balrog meteor punches (camera shake + fire pillars), Royal Guard perfect releases (explosion effects), Devil Trigger activation (wings + aura + color filter). Screen fills 70% with attack trails, particle effects, style rank indicators, damage numbers. Frame rate maintains 60fps via aggressive culling.

5. **Balatro:** Ante 1: Playing 5 cards shows simple highlight animation, score ticks up linearly (100 → 500). Ante 8 with multiplier build: Blueprint + Baron + Sock & Buskin combo. Playing 4 Kings: Each King triggers (+16 chips), Baron multiplies (×1.5 per King = ×5.06), Blueprint doubles Baron effect (×25.6), Sock & Buskin retriggers Kings. Score explosions cascade: 100 → 2,500 → 63,750 → 1,593,750. Numbers enlarge exponentially, screen flashes per multiplier application, sound effects layer (11 simultaneous), animation duration extends from 0.5s to 3.5s. Visual feedback matches mathematical escalation.

**Synergies:**
- Exponential Power Curves (#1) - visual matches power escalation
- Effect Stacking (#10) - each stack adds visual layer
- Combo Systems (#37) - escalation rewards sustained play
- Screen Effects (#40) - combines with camera/shake effects

**Session vs Meta:** 100% Session (visual escalation resets each run, no carryover)
