## 40. Screen Effects (Juice/Polish/Feel)

**Psychological Hook:** Leverages multi-sensory integration—visual + audio + haptic creates coherent impact perception. Weber-Fechner Law—effect magnitude must scale logarithmically with power for perceived difference. Vestibular system response to camera shake mimics physical impact, creating embodied cognition. Temporal contiguity—0-100ms delay between action/feedback critical for causality perception. Motion aftereffect—persistent effects (slowdown, color shift) heighten impact memory. Flow state maintenance—juice provides constant feedback loop confirming inputs.

**Implementation Pattern:**
```
Core Juice Components:

1. Screen Shake:
   - Light: offset ±2-5px, 0.1-0.2s duration
   - Medium: offset ±5-10px, 0.2-0.4s duration
   - Heavy: offset ±10-20px, 0.4-0.6s duration
   - Directional: shake perpendicular to impact direction
   - Decay: exponential (starts violent, smooths quickly)

   shake_intensity = base_shake * damage_multiplier
   offset_x = sin(time * frequency) * shake_intensity * decay
   offset_y = cos(time * frequency) * shake_intensity * decay
   decay = 1.0 - (elapsed_time / duration)^2

2. Hit Pause (Hit Stop):
   - Freeze entire game 1-10 frames (16-166ms at 60fps)
   - Light attacks: 2-3 frames (33-50ms)
   - Heavy attacks: 5-8 frames (83-133ms)
   - Critical hits: 8-12 frames (133-200ms)
   - Scale with damage/importance

   if attack_connects():
       freeze_game_state(hitstop_frames)
       spawn_hit_effect(impact_point)
       play_hit_sound()

3. Time Dilation:
   - Slow-motion: 0.1-0.5× speed, 0.5-2s duration
   - Triggers: Critical hits, perfect parries, last enemy killed
   - Audio pitch-shifts with time (low-pass filter)
   - Particle trails extend (motion blur)

4. Color Grading/Filters:
   - Damage taken: Red vignette, 0.5s fade
   - Low health: Desaturate + red pulse (1s cycle)
   - Power-up: Gold/white glow, 2-5s
   - Status effects: Color tint (poison=green, fire=orange)

5. Chromatic Aberration:
   - RGB channel separation on impacts
   - Offset 2-10px based on intensity
   - Duration 0.1-0.3s
   - Creates "energy burst" perception

6. Particle Effects:
   - Hit impacts: 10-30 particles, radial burst
   - Critical hits: 50-100 particles, delayed secondary explosion
   - Death: 100-200 particles, directional based on killing blow
   - Pool particles (reuse instances, never instantiate mid-combat)

7. Damage Numbers:
   - Spawn at impact point
   - Rise upward (50-150px over 1s)
   - Scale with damage: Small (10-100), Medium (100-1000), Large (1000+)
   - Color by type: Physical=white, Crit=yellow, Elemental=colored
   - Font size: base_size * log(damage) for readability
   - Fade out last 0.3s

8. Audio Layering:
   - Base hit: Impact sound (0.1-0.3s)
   - Critical modifier: Add pitch-shifted layer (+12 semitones)
   - Heavy hit: Add bass layer (sub-100Hz rumble)
   - Combo: Pitch increases with combo count (+1 semitone per 5 hits)
   - Maximum 16-32 simultaneous sounds (older sounds dropped)
```

Performance budget: All juice must maintain 60fps. Disable effects on low-end hardware (settings menu).

**Examples:**

1. **Devil May Cry 5:** Hit impacts: 4-frame hitstop (67ms), screen shake (±8px), particle burst (20-40 per hit), damage numbers (white for normal, gold for critical), audio layer (3 simultaneous: weapon impact, enemy grunt, style sound). Exceed attacks (Nero's sword): Extended hitstop (8 frames/133ms), larger shake (±15px), explosion effect (100 particles), camera zoom (1.1× for 0.3s), screen flash (white, 0.2s). Royal Guard perfect block: 12-frame freeze (200ms), blue particle ring, flash, audio "ding" (pitch-perfect A4 note), time dilation (0.25× speed for 0.5s). DT activation: 30-frame freeze (0.5s), full-screen flash, color grading shift (character-specific hue), orchestral flourish, particle wings spawn (500+ particles), slow-motion entry (1.5s). All combined = 60fps maintained via particle pooling, LOD system, effect culling off-screen.

2. **Celeste:** Dash: 2-frame freeze (33ms), motion trail (5 afterimages, 0.3s persistence), dash particles (15-20 along path), screen shake (±3px), audio (synth whoosh). Wall bounce: 3-frame freeze, directional particle burst (10 particles opposite bounce), dust cloud, audio (higher pitch than dash). Death: 5-frame freeze, character explodes (50 particles, red), respawn orb forms (2s animation), sad piano note, camera zoom out (1.2×, 1s), time slows (0.5× for 0.3s before respawn). Feather (flying): Trail particles (100+ over 2s flight), gradient color shift (pink→gold), camera leads slightly (offset 50px in flight direction), audio (ascending arpeggio). Minimalist aesthetic but maximum juice—every action has 3-5 feedback layers.

3. **Enter the Gungeon:** Gun firing: Recoil animation (gun sprite moves back 5-10px, returns in 0.1s), screen shake (±2-5px based on gun size), muzzle flash (3-frame animation), bullet trail effect, audio (varies per gun), shell casing particle (ejects perpendicular, 0.5s lifespan). Heavy weapons (Compressed Air Tank, Makeshift Cannon): 8-frame hitstop, heavy shake (±15px), explosion particles (200+), camera recoil (moves back 20px, returns 0.5s), audio (bass-heavy boom). Dodge roll: 2-frame freeze on button press, roll trail (10 afterimages), invincibility visual (character fades 50% opacity), audio (cymbal crash), bullets deflected emit particle burst (5 per bullet). Blank usage (screen-clear): 15-frame freeze (250ms), expanding shockwave ring (covers entire screen, 0.8s), all bullets convert to particles (500+), time dilation (0.3× for 0.5s), audio (explosive gong), screen flash (white, 0.3s).

4. **Hades:** Attack impacts: 3-frame hitstop, directional shake (±5px perpendicular to attack), hit spark (10-15 particles), damage number (scales with damage), audio layer (weapon + enemy specific). Backstab (Shadow Presence bonus): Extended hitstop (6 frames), red critical indicator, larger particles (30+), damage number in red + 1.5× size, audio modifier (metallic ring added). Doom application (Ares boon): Purple miasma particle cloud (50 particles over 1s), audio (ominous bass note), status icon appears above enemy. Duo boon acquisition: Full screen border flash (god colors, 0.5s), dual-god symbol appears center-screen (3s), unique dialogue, orchestral swell (2s), screen desaturates except boon icon (spotlight effect, 2s). God appearance: Time dilation (0.5× for 1s), god materializes with particle entrance (100+ particles), color grading shifts to god's theme, voice-over starts, boon selection UI fades in with particle framing.

5. **Noita:** Spell impacts: Varies wildly by spell. Basic spark bolt: 1-frame hitstop, small particle burst (5-10), pixel-perfect collision particles. Explosion spells: 10-frame freeze (167ms), screen shake scales with explosion size (±5px to ±30px), fire/smoke particle simulation (1000+ particles, persistent 5-10s, physics-simulated), terrain destruction (permanent), audio (bass rumble, 1s). Black hole spell: Screen warps toward center (radial distortion), particles spiral (500+), entities pulled physically, audio (deep oscillating drone), camera shake (continuous ±10px while active, 3-5s). Polymorphine (sheep transformation): 5-frame freeze, purple particle explosion (100), entity model swaps, audio (magical chime), camera zoom to transformation (1.2×, 0.5s). Emerald Tablet discovery: Screen flash (green, 1s), particle rain (1000+ over 5s), audio (ethereal choir), UI pops up with text, camera shake (±5px, 2s). Physics-based juice—particles interact with world (fire ignites oil, water extinguishes fire) = emergent juice.

**Synergies:**
- Visual Escalation (#36) - effects intensify with power
- Combo Systems (#37) - combo rank-ups trigger heavy effects
- Achievement Pop-Ups (#38) - achievements use effect toolkit
- Rarity Explosions (#39) - legendary drops max-out effects
- All combat mechanics - juice amplifies every action

**Session vs Meta:** 100% Session (effects are per-run, but settings persist)
