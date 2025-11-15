## 38. Achievement Pop-Ups (Celebration Moments)

**Psychological Hook:** Variable ratio reinforcement—achievements trigger unpredictably, maximizing dopamine (Schultz reward prediction error). Endowment effect—unlocking achievement increases its perceived value. Social proof desire—screenshots prove skill. Collection completion (Zeigarnik)—"47/50 achievements" creates itch. Self-determination theory—achievements provide competence validation. Peak-end rule (Kahneman)—memorable celebration moments define experience more than average gameplay.

**Implementation Pattern:**
```
Achievement System:
- Toast notification: 3-5 seconds, top-right corner
- Animation: Slide in from right (0.3s) + hold (2-3s) + slide out (0.3s)
- Components: Icon (64-128px), Title (bold), Description (subtitle)
- Sound: Unique audio signature (0.5-2s, triumphant chord)
- Backdrop: Darkened background behind toast (50% opacity)
- Particle effects: Confetti/stars burst from icon (30-50 particles, 1s duration)
- Progress display: "10/100 kills" → "100/100 COMPLETE" (color shift)

Rarity tiers:
- Common (70%): Bronze icon, simple sound, 3s display
- Rare (25%): Silver icon, fuller sound, 4s display, light particle effect
- Epic (4%): Gold icon, orchestral sound, 5s display, heavy particles
- Legendary (1%): Rainbow/platinum icon, full audio fanfare (3-5s), 6s display, screen flash, persistent badge

Timing considerations:
- Never during critical gameplay (deaths, boss fights)
- Queue if multiple achievements (display sequentially)
- Allow manual dismissal (ESC/click)
- Persist in achievement menu (permanent record)
```

**Examples:**

1. **World of Warcraft:** Achievement toast appears top-right with "ACHIEVEMENT!" text + unique sound (triumphant horn fanfare, 2.5s). Displays: Icon, title, point value (10-50). Rare achievements (Feats of Strength): Purple text, longer display (5s), guild-wide announcement ("[Player] earned Insane in the Membrane!"). Sound file: "UI_Alert_AchievementGained" (iconic, instantly recognizable). Server-first achievements broadcast realm-wide with unique sound. Social pressure drives achievement hunting—players inspect achievement points (5,000 = casual, 15,000 = dedicated, 25,000+ = obsessive). Achievement mount rewards (310% speed) require 100-200 achievements = 50-100 hour investment.

2. **Hades:** Prophecy fulfillment appears as purple scroll icon + text ("Prophecy Fulfilled: Chthonic Colleague"). Awards: Titan Blood, Diamond, Ambrosia (meta-currencies). No fanfare but narrative payoff—Zagreus comments, NPCs acknowledge. Work Orders display progress: "Kill 50 Numbskulls: 47/50" creates tension approaching completion. First Clear achievements: "Is There No Escape?" (first Hades kill) triggers extended narrative sequence (5+ minutes), unique music, character conversations. Subtle but meaningful—fits narrative-first design. Completion rate: First clear 40%, Full prophecy <5% = long-tail engagement.

3. **Steam Platform (Universal System):** Toast bottom-right, gray background, 5s display. Sound: Single piano note (E4, 0.8s). Components: Game icon, achievement name, unlock timestamp. Global achievement stats visible: "Only 2.7% of players unlocked this." Social sharing—automatically posts to activity feed if enabled. Rare achievements (>90% completion) vs Legendary (<5%) creates hierarchy. Showcasing: Players select 6 achievements for profile display = identity signaling. Trading card drops on achievement unlock (3 random per game) = monetary incentive. Achievement hunters: 50-100 hours per game for 100% completion, dedicated community forums.

4. **Balatro:** Achievements displayed as sticker-style icons in collection menu. Pop-up during run: Sticker slides onto screen (0.5s), jiggles (1s), fades (0.5s). Examples: "Ace in the Hole" (win with Flush only), "High Card Hero" (win with High Card only), "Completionist" (discover all jokers). No audio fanfare (minimalist design philosophy) but visual polish: Sticker quality increases (common → foil → holographic → animated). Challenge-specific achievements: Ante 8 completion shows golden Ante badge. Unlock requirements visible: "Win 5 runs with Checkered Deck: 3/5" = goal gradient. Completion rate: Base victory 40%, All achievements <1% (months of play).

5. **Risk of Rain 2:** Achievement unlocks CHARACTER/ITEM immediately available. Toast: Top-center, 4s, character art/item icon. Sound: Triumphant synth chord (1.2s). Text: "[Character Name] Unlocked!" + flavor text. Examples: "Loader: Complete 5 stages" (20 min unlock), "Mercenary: Obliterate yourself at the Obelisk" (secret ending, 40+ min), "Acrid: Complete Void Fields" (difficult challenge, 15% player completion). Each unlock immediately affects gameplay—new strategies available. Mastery achievements (kill final boss as each character): Unique skin unlocked, displayed in lobby = prestige signaling. Eclipse 8 achievements: <1% completion, community recognition. Achievement integration directly into progression loop = extrinsic + intrinsic motivation combined.

**Synergies:**
- Collection Completion (#4) - achievements track collections
- Milestone Unlocks (#6) - achievements gate content
- First Win Bonuses (#8) - achievements celebrate victories
- Combo Systems (#37) - combo milestones trigger achievements
- Social Competition (#31-35) - achievement comparison drives engagement

**Session vs Meta:** 20% Session (achievement earned during run), 80% Meta (permanent unlock, long-term goals)
