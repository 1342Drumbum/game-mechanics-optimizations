## 14. Narrative Persistence (Story Fragments Per Run)

**Psychological Hook:** Zeigarnik Effect drives between-session return. Emotional investment transforms failure into narrative progress. FOMO creates completionist drive. Story provides meaning beyond mechanics. No run feels "wasted."

**Implementation Pattern:**

```python
class NarrativeProgressionSystem:
    """Story progresses across roguelike runs (Hades-style)"""

    def __init__(self):
        self.story_flags = {}
        self.relationship_levels = {}
        self.encounter_counts = {}

    class NPCMemory:
        def __init__(self, npc_id):
            self.npc_id = npc_id
            self.encounter_count = 0
            self.relationship_level = 0
            self.remembered_events = []

        def on_encounter(self, run_context):
            """NPC dialogue adapts to history. Remembers deaths, equipment, conversations."""
            self.encounter_count += 1
            dialogue_pool = []

            # React to previous death
            if run_context.previous_death:
                if run_context.killed_by == self.npc_id:
                    dialogue_pool.append(self.get_victory_gloat())
                else:
                    dialogue_pool.append(self.get_condolences())

            # Comment on equipment
            if run_context.equipped_weapon in self.favorite_weapons:
                dialogue_pool.append(self.get_weapon_comment())

            # Progress relationship
            if self.encounter_count % 5 == 0:
                dialogue_pool.append(self.get_relationship_progression())

            return self.select_unheard_dialogue(dialogue_pool)

    class MetaProgressionTypes:
        """Three-tier progression: Resources, Mechanical, Narrative"""
        def __init__(self):
            # Type 1: Resource Currency (always collected)
            self.darkness = 0  # Minor upgrades
            self.keys = 0      # Unlock weapons
            self.nectar = 0    # Relationship gifts

            # Type 2: Mechanical Unlocks
            self.unlocked_weapons = set()
            self.mirror_upgrades = {}

            # Type 3: Narrative Relationships
            self.npc_relationships = {}
            self.story_completion = 0.0

        def on_run_end(self, run_results):
            """Even shortest runs provide progress"""
            self.darkness += run_results.darkness_collected
            self.keys += run_results.keys_collected

            # Minimum guarantee
            if run_results.rooms_cleared < 5:
                self.darkness += 10  # Consolation rewards

            # Narrative ALWAYS advances
            for npc in run_results.npcs_encountered:
                self.advance_relationship(npc)

    def select_available_content(self, player_progress):
        """Progressive disclosure: gradually reveal story complexity"""
        content = []

        # Act 1: Basic Story (Runs 1-10)
        if player_progress.total_runs >= 1:
            content.extend(['basic_dialogue', 'surface_relationships', 'escape_motivation'])

        # Act 2: Character Depth (Runs 10-30)
        if player_progress.total_runs >= 10:
            content.extend(['backstories', 'relationship_quests', 'family_dynamics'])

        # Act 3: World Lore (Runs 30-60)
        if player_progress.total_runs >= 30:
            content.extend(['codex_entries', 'hidden_chambers', 'mythology_deep_cuts'])

        # Post-Game: True Ending (10+ escapes)
        if player_progress.successful_escapes >= 10:
            content.extend(['epilogue', 'family_reconciliation', 'true_ending'])

        return content
```

**Design Techniques:**

**Death as Story Device:** Traditional roguelike: Death = Failure. Narrative persistence: Death = respawn scene with unique dialogue. Hades comments contextually. NPCs react: "You put up more of a fight this time, Zagreus..." Story flags advance: 1st death = unlock 'first_death_realization'. 50th = 'characters_worry_about_persistence'.

**Nested Story Arcs:** Macro (Main Plot): Escape underworld (10-50 runs) → Reach surface → Find mother → Resolve family (total 40-60 runs). Micro (Character Arcs): Each NPC 5-15 encounters, parallel progression. Meta (Post-Completion): Epilogue (10+ escapes), family reconciliation, true ending reveals.

**Hub World Evolution:** Decorations appear (Meg gift in room at relationship 7). NPCs relocate (reunited Orpheus+Eurydice move to common room). Ambient dialogue updates (NPCs discuss escape after first success). 20-30 visual/NPC changes reflect progress.

**Balancing Guidelines:** Multiple progress vectors (combat + meta upgrades + story + collection). Guaranteed minimum (even 5-room runs grant 50 Darkness + relationship XP). Story pacing: major beats every 3-5 runs, relationship milestones every 5-10 encounters. Contextualize permadeath narratively (Zagreus is immortal god). Variable pace support: some rush story (15 hours), some explore all (80+ hours).

**Examples:**

**1. Hades:** Three progress types: Resources (Darkness 100-300/run, Keys 5-10, Nectar, Ambrosia). Mechanical (Mirror upgrades, weapon aspects, contractor). Narrative (15,000+ dialogue lines, contextual memory, 6-12 Nectar gifts/NPC → Ambrosia → max bond). Contextual dialogue: "You put up more of a fight this time" (remembers death). "Wielding Stygius I see" (notices equipment). Deeper conversations at relationship ≥5. Story pacing: Runs 1-5 surface, 5-20 character depth, 20-40 world lore, 40+ epilogue. Average completion: 40-60 runs main story, 80-120 runs all content. Dialogue variation ensures 100+ hour freshness.

**2. Rogue Legacy: Journal Fragments:** 25+ pages scattered across runs. Each reveals backstory piece. Collected 1-2/run. Complete journal = full narrative. Character traits create emergent stories: heir with 'colorblind' (grayscale) + 'vertigo' (upside-down) = memorable "remember that run" moment.

**3. Loop Hero: Memory Shard Economy:** Loop (combat) → collect shards → return camp (safe) → spend on story → unlock content → repeat. 1 major beat per 2-3 loops (40-60 min). Shard cost escalates. Optional: rush story vs explore mechanics.

**Success Metrics:** Story completion: 60-75% see main conclusion. Relationship engagement: 70-80% pursue ≥1 arc. Dialogue variety: unique dialogues reported 80+ hours. Emotional investment: "Cried at X scene" = narrative resonance. Return motivation: "What happens next" drives 30-40% return sessions.

**Common Pitfalls:**
- Story gating gameplay: must watch cutscenes to continue. Story enhances, never blocks. Skip always available.
- No short-run rewards: 5-min death feels wasted. Minimum guarantee (50 Darkness + relationship XP + knowledge).
- Static hub world: base never changes = stagnant. 20-30 environmental changes reflect progress.
- Overwhelming new players: 15 NPCs, 50 dialogue choices immediately. Progressive disclosure (2-3 NPCs per act).
- Repetitive dialogue: NPCs say same thing 50 times. Dialogue pools scale with encounters (15,000+ lines for Hades).

**Synergies:** First Win Bonuses (#8), Relationship Systems (#47), Mystery Unlocks (#41)

**Session vs Meta:** 20% Session (story beats during run), 80% Meta (overall narrative across runs)

---
