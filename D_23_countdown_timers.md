## 23. Countdown Timers (Creating Urgency)

**Psychological Hook:** Loss aversion (Kahneman & Tversky) - fear of time running out stronger than desire to gain reward. Yerkes-Dodson law - moderate time pressure optimizes performance. Creates artificial scarcity (Cialdini) - "act now or lose opportunity." Reduces analysis paralysis - forces intuitive decisions. Memorable peaks from clutch victories or devastating time-outs. Regret avoidance motivates faster play. Status quo bias overcome by urgency.

**Implementation Pattern:**
```
// Absolute timer (hard deadline)
mission_timer = 300  // 5 minutes
while mission_timer > 0:
    update_gameplay()
    mission_timer -= delta_time
    if objectives_complete:
        grant_bonus(time_remaining × 10)
        break
    if mission_timer <= 0:
        trigger_failure_consequence()

// Soft timer (escalating consequence)
ghost_timer = 150  // 2.5 minutes
while not level_complete:
    if ghost_timer <= 0 AND not ghost_active:
        spawn_invincible_ghost()
    display_warning_at(ghost_timer == 10)  // "A terrible chill..."

// Bonus timer (optional speedrun)
timed_door_threshold = 120  // 2 minutes
if level_completion_time <= timed_door_threshold:
    unlock_bonus_room()
    grant_rewards(rare_item, 200_gold, blueprint)
```

Display prominently with color coding: green (60%+ remaining) → yellow (30-60%) → red (10-30%) → flashing red (<10%). Audio cues at 30s, 10s, 5s. Ensure timer is achievable with optimal play (success rate 40-70% for average players). Allow timer extensions through gameplay (kill X enemies = +30s).

**Examples:**

1. **Spelunky/Spelunky 2:** Ghost timer: 2:30 in Spelunky 1, 3:00 in Spelunky 2. Warning at 2:25: "A terrible chill runs up your spine!" Ghost spawns at 0:00 - invincible, instant death on touch, moves at 1.5× player speed, homes directly toward player. No escape once spawned - must complete level immediately. Forces risk-taking: skip that chest, ignore side-paths, don't bomb walls for gems. Creates skill expression: optimal players speedrun each level in 45-90s, casuals use full 2:30. Ghost can be manipulated (passes through gems, converts to diamonds worth 5000 gold) but extremely risky. Late-game levels (Sunken City, Cosmic Ocean) require 30+ second completion times under ghost pressure.

2. **Dead Cells:** Timed Doors at end of biomes. Promenade: 2:00 door (30% baseline completion rate). Toxic Sewers: 8:00 cumulative. Ossuary: 10:00 cumulative. After Black Bridge: 15:00 cumulative. Default Timer setting excludes: starting room, shops, transition corridors, lore rooms (fair timing). Doors contain: 20 Cells, 200-500 gold, 3-choice item altar (+1 quality tier vs normal drops). Three doors contain blueprints (first completion only): Promenade → Lightning Bolt, Ossuary → Boomerang, Black Bridge → Punishment. Trade-off created: speedrun for door = miss 30-40% of map loot OR thorough explore = miss door. Speedrun strategies: ignore enemies when possible, prioritize mobility mutations (Lightspeed, Disengagement), focus gold->item purchases over Cell banking. Optimal path knowledge essential (2BC+ players regularly hit doors).

3. **FTL: Faster Than Light:** Rebel Fleet advances through sectors. Each sector: 8-12 beacons available, Fleet advances 1 beacon per jump (+ ASB warnings). Caught by Fleet: takes beacon, elite encounter with +150% difficulty, continuous 2-4 hull damage per turn. Creates urgency: explore vs flee tension. Optimal play: visit 6-8 beacons per sector (balance scrap gain vs Fleet), prioritize distress beacons (higher rewards), skip empty beacons when Fleet close. Final sector (Last Stand): reach base before Fleet overwhelms. Resource management affected: spend fuel to skip vs conserve for emergencies. Oxygen system creates secondary timer: breached = 30-60 seconds until crew asphyxiation. Fire creates tertiary timer: spreads to adjacent rooms every 3-5 seconds.

4. **Hades (Tight Deadline modifier):** Pact of Punishment option: adds regional timers. Tartarus: 9 minutes (12-15 chambers). Asphodel: +5 minutes (14-18 chambers). Elysium: +5 minutes (14-20 chambers). Temple of Styx: +5 minutes (5 challenge rooms + tunnels + Satyr Sack search). Total: 24 minutes for full run (vs casual 35-45 minute runs). Each rank of Tight Deadline worth 2 Heat (max 5 ranks = 10 Heat). Timer displayed top-right, turns red under 2 minutes. Failure consequence: warp to next region, forfeit rewards. Forces: skip dialogue, optimize chamber pathing (prefer Pom/gold over story rooms), dash-strike movement tech, memorize optimal chamber clear patterns. Challenge modifier: speedrun community targets 10-15 minute clears (sub-20 minutes = skilled). Tight Deadline 5 achievable but requires: 300+ hour playtime, perfect boon knowledge, frame-perfect dashes.

5. **Into the Breach:** Turn limits per mission. Easy missions: 5 turns. Medium: 6 turns. Hard: 7 turns. Objective: survive turns AND protect buildings (grid power). Each turn: player mechs move → enemies move/attack → Vek spawn. Perfect strategy required: killing Vek inefficient, repositioning Vek to waste attacks optimal, blocking spawns with units/environment. Time traveling (undo turn): 1 per mission, allows mistake correction. Grid power (civilian buildings): 3-5 power at start, lose 1 per destroyed building, 0 power = game over. Creates multi-objective optimization: protect buildings, survive X turns, complete bonus objectives (kill bosses, protect specific buildings). No real-time pressure but puzzle-like constraint. Victory: survive all turns. Bonus rewards for: buildings ≥80% intact, perfect clear (no building damage).

**Synergies:**
- Difficulty Ramping (#21) - Timers can accelerate difficulty (Risk of Rain 2)
- Rush vs Patience (#24) - Creates fundamental speed vs thoroughness trade-off
- High-Risk Paths (#17) - Timed routes for skilled players
- Escape Sequences (#25) - Timers often accompany chase sequences
- Speedrun Potential (#30) - Natural speedrun category creation
- Achievement Systems (#35) - Time-based achievements (sub-10 minute runs)
- Knowledge Checks (#28) - Optimal routing requires game knowledge

**Session vs Meta:** 90% Session (timer pressure is immediate and in-the-moment), 10% Meta (learning optimal routes and timings to beat deadlines consistently)
