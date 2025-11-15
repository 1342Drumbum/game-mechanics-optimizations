## 25. Escape Sequences (Chase/Pursuit Mechanics)

**Psychological Hook:** Fight-or-flight response triggers genuine stress (amygdala activation). Loss aversion amplified - running from threat more motivating than approaching reward. Relief upon escape creates powerful positive emotion (negative reinforcement). Creates memorable narrative moments players retell. Skill expression through spatial awareness and resource management under pressure. Mirror neurons create empathy with avatar's peril. Learned helplessness prevented by providing escape tools/routes.

**Implementation Pattern:**
```
escape_sequence:
    trigger: boss_enraged OR timer_expired OR objective_complete

    phase_1_warning:
        audio_cue("WARNING: EVACUATE IMMEDIATELY")
        visual_cue(screen_shake, red_tint, klaxon)
        duration = 5_seconds
        allow_preparation()

    phase_2_chase:
        spawn_pursuer:
            speed = player_speed × 1.3
            invincible = true OR hp = 9999
            damage = instant_kill OR 90% max_hp
            ai = pathfind_to_player, ignore_obstacles

        spawn_obstacles:
            rate = every_10_seconds
            types = [walls_collapse, fire_spreads, enemies_spawn]

        player_tools:
            speed_boost_pickups(+50% speed, 5s duration)
            temporary_invincibility(iframe_dash)
            environmental_shortcuts(breakable_walls, ziplines)

    phase_3_resolution:
        if player_reaches_exit:
            victory_bonus(+500_xp, rare_item, narrative_progress)
        else:
            death OR heavy_penalty(-50% rewards)
```

Duration: 30-120 seconds (long enough for tension, short enough to replay). Difficulty: 60-70% success rate for average players (close calls create best memories). Always provide: clear exit direction (arrow/marker), visible threat (show pursuer), fair warning (5-10s prepare), skill-based escape tools (not pure RNG).

**Examples:**

1. **Hades (Asterius Bull Rush):** Mini-escape sequence within boss fight. Asterius (Minotaur) telegraph: 1 second wind-up (snort audio + hoof-pawing animation) → charges at 3× movement speed in straight line for 2.5 seconds (60% of arena width). Contact damage: 40-50 (vs player 50-200 HP depending on upgrades). Counter-strategies: (A) Dash perpendicular through Asterius at charge start (iframe abuse, 0.07s invincibility window), (B) Position near wall, bait charge, dash through him → he hits wall (stun for 2s + vulnerable state), (C) Use Athena Divine Dash (reflect damage, Asterius takes 50 damage + interrupts charge). Advanced: force 3-wave shockwave at wall impact (additional 15 damage each). Theseus (partner boss) combines: calls god aid while Asterius charges (dual pressure). Pattern appears 4-6 times per fight (every 20-30 seconds). Speedrunners manipulate pathing: corner Asterius, force charges into walls (6× per fight = 300+ reflected damage = 15% of health bar).

2. **Lethal Company (Monster Chases):** Dynamic chase sequences from various monsters. **Thumper** (indoor): roar audio cue → charges at 1.5× player sprint speed, struggles with corners (loses 40% speed on 90° turns). Escape strategy: break line-of-sight around 2-3 corners → Thumper loses tracking after 8 seconds. Death = instant kill (neck snap). **Bracken**: silent stalker, kills if stares at player back for 10+ seconds uninterrupted. Escape: face Bracken momentarily (resets aggression), take corners (breaks LoS), reach facility entrance (safe zone). **Jester**: 30-60 second music box wind-up → skull phase (10× speed, instant kill, tracks through walls). Only escape: exit facility before transformation completes. 90+ decibel audio warning. Survival rate: <30% if caught inside during skull phase. **Ghost Girl**: haunts 1 player, invisible to others, 5-minute stalk before attack phase. Attack phase: chases at 0.8× player speed (barely outrunnable), follows outdoors, never stops. Survival: loop around objects for 5+ minutes OR reach ship departure. Communication essential: "Thumper aggro, leading to main entrance, open door in 10 seconds!" Teamwork: one player watches Coil-Head (Weeping Angel), others move.

3. **Left 4 Dead (Tank Encounters):** Tank spawns: 250-HP mini-boss (vs 100 HP players), instant-incap hits (3 hits = death), throws cars/rocks (40 damage, knockback), destroys environment. Chase mechanics: Tank moves at 1.1× survivor sprint (inevitably catches), creates 8-second pursuit before catch. Counter-play: team focus-fire (kill in 15-20 seconds optimal), kite in circles (buy time), use Molotov (50 DPS fire), shoot propane tanks (50 instant damage). Environmental factors: outdoor = more kiting space (easier), indoor = corners break LoS (harder but manageable), crescendo events + Tank = overwhelming. AI Director spawns Tanks during: low-health moments (kicks while down), finale holdouts (2-3 Tanks), crescendo events (double pressure). Versus mode: player-controlled Tanks more dangerous (smarter pathing, fake-outs, environmental awareness). Survival: 75% with coordination, 30% without voice comms.

4. **Deep Rock Galactic (Dreadnought Twins):** Escort mission conclusion: trigger Dreadnought Twins boss fight. Arbalest (ranged twin): 1200 HP, fires armor-piercing sonic attacks (25 damage, ignores shields), stationary turret mode. Lacerator (melee twin): 900 HP, charges at players (15 damage, 3× dwarf speed), creates shockwave AoE (30 damage, 8-meter radius). Chase mechanic: Lacerator targets furthest player, charges for 4-second pursuit. Escape: (A) break LoS around stalagmites (Lacerator rams into terrain, stunned 3s), (B) freeze grenade (Cryo Grenade: stuns 5s, +3× damage taken), (C) grappling hook / zipline (vertical escape, Lacerator can't follow). Coordination required: 2 players kite Lacerator, 2 players damage Arbalest (must kill both within 30s or they heal). Enrage phase: when 1 twin dies, survivor gains +50% speed, +100% damage, forces all-team kiting. Duration: 5-8 minutes total. Success rate: Hazard 3 = 85%, Hazard 5 = 45%. Voice comms essential: "Lacerator on me, kiting to gold room, freeze ready in 10 seconds!"

5. **Resident Evil 2 Remake (Mr. X / Tyrant Stalking):** Persistent chase mechanic across 2+ hours. Mr. X: indestructible, 1 hit = 30-40% HP, heavy footsteps audible 2 rooms away (audio telegraphing), walks at 0.8× player walk (jog outruns), sees through walls/doors (x-ray tracking). First encounter (Police Station, 45 minutes in): surprise spawn + 15-second chase to safe room. Persistent phase (60-90 min): patrols station, investigates gunshots (sprints toward noise for 20s), opens doors (no safe rooms except save rooms). Escape strategies: (A) stealth gameplay (walk, don't run = no noise), (B) resource sprint through (ignores zombies, reaches next area), (C) temporary stun (grenade = 10s stun, flashbang = 5s disorient), (D) loop around furniture (can't be grabbed while vaulting). Psychological pressure: footsteps above/below create paranoia, save rooms feel like relief islands, forces inventory management speed (Mr. X hears item boxes opening). Chase encounters: 8-12 throughout campaign, each 30-90 seconds. Final escape: trigger library exit, Mr. X sprints (1.3× player speed), 45-second chase to elevator. No-damage runs: require memorizing patrol patterns (60+ hours practice).

**Synergies:**
- Countdown Timers (#23) - Escape sequences often include time limits
- Difficulty Ramping (#21) - Pursuers intensify pressure over time
- Execution Challenges (#27) - Clean escapes demonstrate skill
- High-Risk Paths (#17) - Optional chases for better rewards
- Environmental Interaction (#42) - Using terrain/objects to escape
- Audio Cues (#36) - Sound design critical for chase tension
- Perfect Information (#26) - Some chases reward enemy pattern knowledge
- Resource Management (#47) - Limited tools (stamina, ammunition) during chase

**Session vs Meta:** 80% Session (chase is immediate, visceral, in-the-moment experience), 20% Meta (learning escape routes, enemy behaviors, optimal counter-strategies)
