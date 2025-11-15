## 30. Speedrun Potential (Time Attack Modes)

**Psychological Hook:** Competition drives engagement (Social Comparison Theory, Festinger 1954). Personal bests create self-competition loop. Mastery demonstration through optimization. Flow state from execution + routing synergy. Community formation around speedrunning. Status signaling - world record holders become celebrities. YouTube/Twitch content creation opportunities. Replayability multiplied by time optimization. Every second saved = dopamine hit. Numbers going down triggers satisfaction.

**Implementation Pattern:**
```
Speedrun Infrastructure:
- Timer: Always visible, millisecond precision
- Display: Current run vs Personal Best (split comparison)
- Seeded runs: Optional fixed seed for fair competition
- Recording: Built-in replay system or encourage OBS integration
- Leaderboards: Global + Friends + Country rankings
- Categories: Any%, 100%, Low%, specific character/weapon

Timing Standards:
RTA (Real-Time Attack): Total wall clock time including loads
IGT (In-Game Time): Only gameplay, excludes menus/loads/pauses
Segmented: Best time per section added together
Single-segment: One continuous run

Design for Speedrunning:
- Consistent enemy spawns (pattern-learnable)
- Movement tech (dash-cancels, wall-clips, sequence breaks)
- Skip opportunities (intended and unintended)
- Risk/reward routing (dangerous fast vs safe slow)
- Skill expression (execution quality impacts time directly)
- RNG minimization (or RNG manipulation techniques)

Timer Placement & Features:
- Main timer: Top-center, always visible (12-18pt font)
- Split comparisons: +0.34s (behind) or -0.67s (ahead) in real-time
- Personal best ghost (visual representation racing alongside)
- Milestone markers: "Fastest Prisoners' Quarters!" pop-up
- Post-run summary: Time per stage, overall placement
```

Implement pause button that stops timer (prevents abuse). Record ghost data for racing against self. Create official categories to prevent fragmentation. Balance: don't force speedrunning on casual players (optional mode).

**Examples:**

1. **Dead Cells:** Speedrun Mode in settings - displays millisecond timer, generation seed, disables cutscenes/slowmo/lore rooms. Shows per-biome times vs personal best: "Prisoners' Quarters: 1:47 (Best: 1:32, -0:15)". Timing: IGT only counts active gameplay. Movement tech: Assault Shield Roll Cancel (ASRC) = frame-tight dash for 2-3× movement speed. Routing critical: Toxic Sewers route 20-30 seconds faster than Promenade, but riskier. World record any% Warpless Unseeded: 12:54 by Henpaku. Category variety: Any% Warps (10-11 min), 0BC (boss cell difficulty), 5BC (hardest), Fresh File (new save). RNG dependency noted: times vary 30-90 seconds based on layout/drops. Top players: 13-15 minute range. Casual optimized: 18-25 minutes. Community on speedrun.com tracks 8+ categories. Timer persistence creates "just one more attempt" loop.

2. **Hades:** Speedrun categories use IGT (timer only runs during gameplay, pauses during boon selection/shops/loads). Current world record any heat: 5:16 IGT by Croven using Hera Aspect Bow. Heat system creates difficulty tiers: Any Heat, 32 Heat, 40 Heat, 50 Heat. Routing: skip all non-boon chambers, refuse slow gods (Aphrodite dialogue long), target Athena/Artemis/Hermes. Movement optimization: dash-canceling attacks, diagonal dashing. Top 100 runners: <7:00 IGT. God manipulation through keepsakes (guarantee first boon). Extreme Measures 2 (Heat modifier) makes Lernie fight faster by grouping heads. Forced Overtime spawns enemies faster = faster rooms but riskier. RTA vs IGT difference: 15-30% (dialogue/loads). Community achievement: "Breaking 7 minutes places you comfortably in the top 100." Competitive subcommunities: weapon-specific leaderboards (6 weapons × 4 aspects = 24 categories).

3. **Celeste:** Built-in timer in Chapter Select showing Personal Best per chapter. Speedrun timing: files tracked separately (Any% vs 100%). Movement tech: wavedashing (+40% horizontal speed), hyper-dashing (diagonal speed boost), extended hypers, ultras (frame-perfect super-jumps). World record Any% (main story): ~27 minutes (vs 8-12 hour first playthrough). 202 Strawberry 100%: ~50 minutes (vs 25-50 hours casual). Per-chapter optimization: room-by-room routing with pixel-perfect movement. Summit: 9-12 minutes speedrun vs 1-2 hours first time. Dashless category (0 dashes entire game) adds execution challenge. Community-created mods: Everest adds custom chapters with speedrun timers. Competitive players practice individual screens 100-500 times for 0.5-2 second improvements. IGT vs RTA differential: minimal (loads quick). Demo system records inputs for TAS (tool-assisted speedrun) comparison. Top players save 30-120 seconds per chapter through tech.

4. **Spelunky 2:** Timer runs continuously (RTA). Speedrun achievement "Speedlunky": beat game in <10 minutes. World record: <3 minutes using optimal routing + teleporter tech. Time pressure natural: Ghost spawns at 2:30 per level = hard time cap. Movement optimization: hold run continuously, chain whip movements, damage-boost through enemies (lose HP for speed). Critical routing: skip Volcana for Jungle (15-25 seconds faster), skip Black Market unless jetpack offered. Teleporter wall-clipping: 2-5 seconds saved per level × 16 levels = 32-80 seconds total. Speedrun categories: Low%, Any%, Cosmic Ocean (100%), Solo vs Co-op. Community on MossRanking.com tracks times. RNG manipulation: experienced runners recognize generation patterns = predict exit location, saving 5-15 seconds per level. Olmec fight optimization: specific positioning pattern speeds kill from 90 seconds to 30-40 seconds. Speedrun vs casual: 10-12 minute difference on successful runs.

5. **Risk of Rain 2:** Eclipse mode (difficulty) × character creates speedrun categories. Timing: IGT only (RTA includes loads which vary by hardware). Drizzle (easy) speedruns: 8-12 minutes. Monsoon (hard): 15-25 minutes. Movement: bunny-hopping, ability canceling, slide-jumping (character-specific). Routing: skip Stage 4 entirely (go directly to Stage 5 Commencement), rush teleporter ignore monsters (98% speed, 2% killing). Item knowledge critical: prioritize movement (Energy Drink, Wax Quail, Paul's Goat Hoof) over damage for speed. Shrine of Order (rerolls all items to ~3 types) = speedrun gambling - can win/lose run instantly. Artifact combinations create speedrun categories: Artifact of Command (choose items) = 6-10 minute runs with optimal selection. Leaderboards per character × difficulty × artifact. World record tracking on speedrun.com. Community discovered "Vending Machine Loop" (exploit): farm infinite items in 2-3 minutes = unintended category split. RNG mitigation: seed system allows reproducible runs for competition.

**Synergies:**
- Execution Challenges (#27) - speedruns test mechanical skill limits
- Optimal Path Finding (#29) - routing critical for competitive times
- Perfect Information (#26) - determinism enables consistent runs
- Knowledge Checks (#28) - meta-knowledge speeds decision-making
- Leaderboards (#36) - natural pairing with time competition
- Adaptive Difficulty (#13) - Anti-synergy: speedruns need consistent difficulty
- Daily Challenges (#37) - seeded daily speedrun leaderboards

**Session vs Meta:** 50% Session (individual run optimization), 50% Meta (learning tech/routes/strategies over time)
