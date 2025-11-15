## 33. Asynchronous Competition (Ghost Runs)

**Psychological Hook:** Competition without confrontation - avoids multiplayer anxiety while preserving rivalry. Temporal flexibility - compete on your schedule. Perfect information - see opponent's exact performance (no RNG excuses). Self-efficacy building through observing slightly-better performers (Bandura, 1977). Flow state preservation - no waiting for matchmaking or dealing with griefing. Parasocial relationship with top players - "racing against world record holder" creates imagined relationship. Immediate feedback loop - delta timing shows real-time performance gap.

**Implementation Pattern:**
```
Ghost Recording:
- Sample player input every frame (60fps = 3600 samples/min)
- Compress redundant data (no change = skip sample)
- Store: position, rotation, actions, timestamps
- File size: ~50KB for 5min run (highly compressed)

Ghost Playback:
- Interpolate between samples for smooth movement
- Render as semi-transparent (alpha 0.4-0.6)
- Display nameplate + delta timer above ghost
- Color-code: Green (ahead), Red (behind), White (tied)

Delta Timing Display:
current_delta = player_time - ghost_time_at_same_position
display: "+0.347s" (behind) or "-0.124s" (ahead)
rolling_average = last_10_checkpoints.average()

Ghost Selection:
- Personal Best (beat yourself)
- Friend Best (social comparison)
- Top 100 Global (aspiration)
- "Next Rank Up" (player 1 rank above you)
- AI Pacing Ghost (slightly faster than your average)
```

**Examples:**

1. **Trackmania (2020):** Ghost racing is **core multiplayer mode**. Race against **up to 100 simultaneous ghosts** with **zero collision** (drive through each other). **World Record ghost** auto-loads on official tracks. **Delta timing** shows **+/-seconds vs selected ghost** in real-time. Medal ghosts: **Bronze (AI), Silver, Gold, Author (developer), Champion (top 10%)**. Openplanet plugin **"Any Ghost"** loads any player by username. Track Central hosts **30 million+ user tracks** with individual leaderboards. Ghost file size: **~30KB for 1min track**. **TrackMania.io integration** - paste track link to load top ghost. Community "Hunters" focus exclusively on **world record hunting** (beating WR ghosts). **60% of playtime** spent in solo time attack vs ghosts vs **40% live multiplayer**.

2. **Mario Kart 8 Deluxe:** Time Trial mode with **Staff Ghosts** (Nintendo developer times) as baseline. **Download ghosts from online rankings** - **top 10 global** or **friends list**. Race against **up to 7 ghosts simultaneously**. **Ghost data saved automatically** on personal best. Staff Ghosts typically **5-10 seconds slower** than world records, providing **intermediate target**. World record ghosts show **ultra-optimized lines** impossible for casuals (mushroom shortcuts, mini-turbos). **Sleep mode downloads** random ghosts passively. Ghost file: **~15KB per 2-3min race**. **Mario Kart World integration** (fan site) hosts **150,000+ ghosts**. **Time Trial engagement: 40% of online players**, **average 6.5 ghosts raced per session**.

3. **Hades (Community Mods):** No native ghost system, but **speedrun community uses video overlays**. Tool: **"LiveSplit"** shows **side-by-side comparison** with previous PB or WR. Split timer displays **+/- time** per chamber. Example: Chamber 15: **-2.34s (ahead), Chamber 23: +1.87s (behind)**. Community **"Race Against Shade"** mod (unofficial) creates **in-game ghost replays**. Stores **ability usage + position data** = **~200KB per 10min run**. Popular races: **PB vs PB (self-improvement)**, **friend vs friend**, **top speedrunner ghosts**. **Discord bot** allows uploading runs for ghost generation. **15% of speedrunning community** uses ghost comparison tools regularly.

4. **Gran Turismo 7:** **B-Spec mode ghost racing** where AI Bob drives your car (asynchronous). **Daily Races** use **qualifying ghost** (your fastest lap replays visually). **Sport Mode** shows **top 10 regional ghosts** for each track. **Delta widget** (can toggle): **+0.5s sector 1, -0.3s sector 2**. **Driving Line Ghost** (colored line showing racing line + braking zones). **Downloadable ghosts from top players** - **GT World Tour champions** share ghosts for learning. Community feature: **"Beat the Developer"** - Polyphony Digital staff ghosts. **License tests** require beating **Gold Ghost times** (developer). **60% of single-player time** spent in **Time Trial vs ghosts**.

5. **Nioh 2:** **Benevolent Graves** - when players die, gravestone appears in other players' worlds as **ghost summon**. Interact to see **"Bloody Grave"** challenge - fight **AI recreation** of that player's build. **Stats, equipment, skills copied exactly**. Rewards: **Ochoko Cups** (multiplayer currency). **Revenant trading** - community shares **grave locations** for specific builds (e.g., "Level 150 Switchglaive build at Region 3-2 bonfire"). **20 million+ graves generated**. **Average player encounters 3-5 graves per mission**. **15% of players** seek specific graves for loot farming. Not traditional ghost racing but **asynchronous PvE competition**.

**Synergies:** Leaderboards (#31), Speedrun Potential (#30), Daily Challenges (#32), Mastery Expression (#26)

**Session vs Meta:** 85% Session (racing ghosts this session), 15% Meta (improving to create better ghost for others)
