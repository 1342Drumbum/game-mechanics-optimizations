## 52. Favor Systems (NPC Gift Giving)

**Psychological Hook:** Exploits reciprocity norm (Cialdini, 1984) - receiving gifts creates obligation to reciprocate with loyalty/affection. Progress bar psychology: "5/6 hearts filled" drives completion. Variable ratio reinforcement through gift reactions (Loved > Liked > Neutral creates near-miss engagement). Unlocking personal stories creates parasocial relationships - players care about fictional character problems. Status signaling: romance-able characters become social currency ("Who did you romance?"). Sunk cost fallacy: 10+ hours invested in relationship → must see it through.

**Implementation Pattern:**
```
Gift Value Calculation:
Relationship Points = Base_Value × Quality_Multiplier × Occasion_Multiplier

Base Values:
- Loved Gift: +80 points
- Liked Gift: +45 points
- Neutral Gift: +20 points
- Disliked Gift: -20 points
- Hated Gift: -40 points

Quality Multipliers (if applicable):
- Normal: 1.0×
- Silver/High: 1.25×
- Gold/Iridium: 1.5×

Occasion Multipliers:
- Normal Day: 1.0×
- Birthday: 5-8×
- Festival/Special Event: 3-5×

Relationship Tiers:
- Each heart = 250 points
- Romanceable NPCs: 10-14 hearts (2,500-3,500 points)
- Standard NPCs: 5-10 hearts (1,250-2,500 points)
- Locked tiers require: quest completion, story progress, previous tier

Gift Limits:
- 2 gifts per week (resets weekly)
- +10 bonus points if both weekly gifts given
- Birthday gifts don't count toward weekly limit
- Weekly bonus on relationship milestone days
```

Unlock thresholds:
```
Heart 1-2: Basic dialogue
Heart 3-4: Personal story begins, cutscene
Heart 5-6: Deeper friendship, special requests
Heart 7-8: Romance unlocks (if applicable), requires Bouquet/Pendant
Heart 9-10: Committed relationship, marriage option
Heart 11-14: Post-marriage exclusive content
```

**Examples:**

1. **Hades:** Nectar and Ambrosia economy. 6 Nectar gifts → unlock Favor quest → switch to Ambrosia for hearts 7-10. 230+ total Nectar needed (all NPCs), 151 Ambrosia for completion. Each Nectar: 50-150 Darkness value. First gift → Keepsake (stat bonus item), immediate gameplay benefit. Relationships unlock narrative: Megaera: 10+ encounters + 6 Nectar + defeat her with special dialogue → romance path opens → 6 Ambrosia → max relationship reveals backstory. Thanatos: gift 6 Nectar → Favor quest (beat him in kill competition 3×) → unlock romance. 30-60 hours to max all relationships. Divine boons improve with relationship: Aphrodite at max hearts gives 15% better boon rarity. Strategic gifting creates gameplay advantage + story. Orpheus & Eurydice reunion requires: both at 5 hearts, complete admin chamber contract, witness 10+ dialogue exchanges = 25 hour puzzle.

2. **Stardew Valley:** Most complex gift system. 33 NPCs × 10 hearts = 8,250 total points needed. Each heart = 250 points. Optimal path: 2 Loved gifts per week = 160 points + 10 bonus = 170 points = 1.47 hearts/week. Max relationship in ~7 weeks (in-game). Birthday strategy: Loved birthday gift = 640 points = 2.56 hearts in single day. Quality matters: Iridium Loved Gift on Birthday = 640 × 1.5 = 960 points = 3.84 hearts. Romanceable NPCs lock at 8 hearts until Bouquet given (costs 100g). Marriage candidates unlock at 10 hearts, require Mermaid Pendant (5,000g). Post-marriage: 14 hearts available. Each NPC has 5-12 Loved items, 200+ total items. Example: Abigail loves Amethyst (50g), Blackberry (free forage), Pufferfish (tricky catch). Knowledge check: knowing preferences = efficiency. Community Center completion requires relationships: need 5+ hearts with specific NPCs for cutscenes. 100-200 hours for full social completion.

3. **Fire Emblem Three Houses:** Support Points system. 23+ characters per playthrough. Byleth + Student supports: C → C+ → B → B+ → A → A+ = 6 tiers. Points needed: ~3,000 for full support (varies by character). Gift mechanics: Loved Gift = 60 points + restores 50% Motivation (mechanical benefit). Faculty members +45% bonus points from gifts = 87 points/loved gift. Lost items: return = 20 points. Choir practice: 30 points to all participants. Birthday flowers: 5 points. Tea time: 50-150 points based on performance (topic choice, timing). Strategic optimization: Claude loves Hunting + Riding + History books. Perfect tea time + gift = 150 points = 1/20th of full support. Battle strategy integration: supports unlock combat bonuses (+10 avoid, +3 damage when adjacent). A-Support gives Dual Guard (10-30% nullify damage) + Dual Strike (20-40% follow-up attack). S-Support (romantic): post-timeskip only, requires A-Support + final battle completion + propose. ~80-120 hours to see all supports (3 playthroughs, 23 characters × 6 supports each).

4. **Persona 5 Royal:** Confidant system. 23 Confidants × 10 ranks = 230 total ranks. Point thresholds: Rank 1→2 = 20-40 points, Rank 9→10 = 100-150 points (accelerating requirements). Arcana matching: carry matching Arcana Persona = +50% points. Dialogue points: 0-3 notes, base 5 points/note (15 max). Perfect answers guide available but creates spoiler tension. Gift system: 50+ giftable items, each NPC has 5-10 favored gifts. Gifts give +10-30 points. Time investment: ~8 sessions per confidant (×60 min = 8 hours). Total: 23 × 8 = 184 hours for max all. Fortune confidant (Rank 7) enables "Affinity Boost" = guarantee points on next hangout (5,000 yen). Strategic meta: prioritize Star (Luck boosts), Fortune (efficiency), Moon (EXP) for mechanical advantages. Each Confidant Rank unlocks ability: Kawakami (Rank 10) = free night actions after palace = 20+ extra evening sessions = massive time economy. Romantic lock-in at Rank 9 (choose 1 or face consequences = Valentine's Day confrontation scene if dating multiple).

5. **Dead Cells:** Minimal but effective. 5 NPCs with relationship currency (gold, cells, blueprints). Collector: spend 50,000 cells over 30+ runs → unlocks rare blueprints (15% drop rate items). Blacksmith: invest gold (500/1k/2k/5k increments) → unlock stat reroll service → 50,000g total = guaranteed ++ stat on items. Architect: 30 cells → permanent shortcuts (reduces early grind). Relationship is transactional but persistent across runs. Prisoner's Quarters hub evolves: unlock Training Room (1,000 cells), Fashion Shop (cosmetics), Tailor Room (100,000 gold total investment). Visual progression: empty prison → bustling hub over 50+ hours. Social validation: outfit = skill/time investment signal.

**Synergies:**
- Narrative Progression (#14) - gifts unlock story
- Collection Completion (#4) - gift all NPCs
- Multiple Currencies (#44) - gift items as currency
- Knowledge Checks (#28) - knowing preferences
- Daily Login Streaks (#38) - weekly gift timers
- Mystery Unlocks (#41) - hidden gift requirements
- Milestone Rewards (#8) - relationship tier bonuses

**Session vs Meta:** 30% Session (giving gifts during play), 70% Meta (relationship persists, long-term investment)
