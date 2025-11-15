## 53. Faction Reputation (Multiple Allegiances)

**Psychological Hook:** Identity formation through group affiliation (Social Identity Theory, Tajfel, 1979). Creates "us vs them" mentality driving engagement. Cognitive dissonance when serving opposing factions creates tension ("Do I betray the Brotherhood?"). Status signaling through faction rank ("I'm Idolized by NCR"). Branching consequences create FOMO - choosing one faction locks out others, replay motivation. Exclusive rewards tied to reputation create goal-directed grinding. Visible progress bars (Neutral → Friendly → Honored → Exalted) exploit goal gradient effect.

**Implementation Pattern:**
```
Reputation Tiers (Standard RPG):
- Hated: -3000 to -750 (hostile on sight)
- Hostile: -750 to -375 (attack on sight)
- Unfriendly: -375 to -125 (refused service)
- Neutral: -125 to +125 (standard prices)
- Friendly: +125 to +375 (5% discount)
- Honored: +375 to +750 (10% discount, quests unlock)
- Revered: +750 to +1500 (15% discount, special items)
- Exalted: +1500+ (20% discount, unique rewards, titles)

Reputation Gain/Loss:
Quest completion: +25 to +500 (scales with difficulty)
Kill faction enemy: +5 to +25
Kill faction member: -50 to -250
Betray faction quest: -500 to -1000
Wear faction tabard: +10% rep gains (if applicable)

Opposing Faction Mechanics:
Gain +100 with Faction A → Lose -50 with opposing Faction B
Some factions are mutually exclusive (can only max one)
Point of no return: committing to main faction locks others

Mechanical Unlocks by Tier:
Friendly: Faction merchant access, basic quests
Honored: Rare items, unique quest line opens, 10% discount
Revered: Epic items, faction-specific ability/mount
Exalted: Legendary items, exclusive title, faction ending unlocked
```

Faction conflict matrix:
```
                NCR    Legion    Strip    Khans    BOS
NCR           [100]    [-50]    [0]      [-25]    [+25]
Legion        [-50]    [100]    [0]      [+25]    [-50]
Strip         [0]      [0]      [100]    [-10]    [0]
Great Khans   [-25]    [+25]    [-10]    [100]    [0]
Brotherhood   [+25]    [-50]    [0]      [0]      [100]
```

**Examples:**

1. **Fallout New Vegas:** 15 factions with Fame/Infamy dual tracking. Fame 0-100 scale (Neutral → Idolized at 100). Infamy 0-100 scale (Neutral → Vilified at 100). Both tracked independently: can be Famous (hero reputation) AND Infamous (villain reputation) = "Mixed" reputation (unique dialogue). Major factions: NCR, Caesar's Legion, Mr. House, Boomers, Brotherhood, Khans, White Glove Society. Each faction unlocks: Safehouse (free rest + storage + faction armor), Unique companion (some faction-gated), Quest lines (20+ hours exclusive content), Ending slide (different endings based on final reputation). Example path: Idolized by NCR requires ~1,200 reputation = 12 major quests + 40 minor tasks. Wearing NCR armor while doing quests = +10% rep. Opposing consequences: Help NCR → Lose Legion rep (50:1 ratio). Point of no return: "Hoover Dam" quest locks you into faction, remaining 8-15 hours determined by choice. 4 major factions × unique ending = 4 playthroughs for completionists = 120-160 hours.

2. **World of Warcraft Classic:** 60+ factions across 1-2 years of content. Tiers: Hated (-3000 to -42000), Hostile (-3000 to 0), Unfriendly (-3000 to 0), Neutral (0 to 3000), Friendly (3000 to 9000), Honored (9000 to 21000), Revered (21000 to 42000), Exalted (42000). Total points to Exalted from Neutral: 42,000 reputation. Methods: Quest turnins (250-500 rep), Repeatable quests (10-25 rep), Dungeon kills (5-10 rep per mob), Cloth turnins (50-75 rep per stack). Time investment: 40-80 hours per Exalted faction. Benefits: Epic mounts (Exalted = unique mount, 100g discount), Enchant recipes (head/leg enchants locked to Argent Dawn Revered), Class quests (Paladin/Warlock epic mount requires multiple Exalted factions), PvP ranks (tied to battleground reputation: WSG, AB, AV). Example: Argent Dawn Exalted requires 840 Scourgestones (elite mob drops) = 200+ Stratholme runs = 100 hours. Goblin cartel reputation: 4 interconnected factions (Booty Bay, Everlook, Gadgetzan, Ratchet). Gain +100 with one = +25 with all three others. Lose -2500 with one = lose -1250 with all. Kill enough guards = Hated with all goblins = 20% of vendors locked = massive consequences.

3. **Divinity Original Sin 2:** Attitude system (0-100 per NPC). Vendors: 100 attitude = 20% discount. Each point of Persuasion skill = +5 attitude with all NPCs. Gold bribery: 350 gold = +100 attitude with merchant (scaled by vendor level). Main faction system: Divine Order, Black Ring, Voidwoken, Magisters. Choices create cascading reputation changes. Example: Kill Magisters in Act 1 → -50 reputation → guards hostile in Act 2 → need disguise armor or fight through cities. Pet Pal talent unlocks animal faction: +25 reputation for feeding, +50 for quest help. Undead race: -25 reputation with living, +25 with undead. 4 playable races × reputation modifiers = 4 different social games. God faction system: 7 gods × personal alignment. Choose god in character creator → +50 starting reputation with god's faction. Praying at shrines: +10 reputation. Temple quests: +100. At max reputation: god grants permanent blessing (+2 to attribute, +10% crit, +15% damage vs race). 60-80 hours to optimize reputation for build (critical for Honour difficulty).

4. **Skyrim:** 9 major faction questlines but simplistic reputation. Joining faction = automatic progression (no grinding). Join Thieves Guild → complete 9 story quests → become Guildmaster = 15 hours. No numerical reputation. Hold-based bounty system creates temporary faction hostility: 1000g bounty = guards attack on sight until paid. Civil War questline: Imperials vs Stormcloaks (mutually exclusive). Choice locks 50% of faction content per playthrough = replay motivation. College of Winterhold: 5 quests = Archmage title + chest with 100g + 200+ magicka regen item. Dark Brotherhood: 13 quests = Blade of Woe + 20,000g contract payment + Spectral Assassin summon. Simplistic but effective: clear goal (become leader) + immediate rewards. 60+ hours for all faction questlines. Modding community adds reputation systems: "Skyrim Reputation" mod creates fame/infamy tracking across 10,000+ interactions.

5. **Monster Hunter World:** Research Levels (faction reputation for endemic life). 6 research level tiers per monster (50 large monsters). Each tier: +10% drop rate for rare materials, unlock special investigations, reduce hunt difficulty (-5% HP per tier). Gain research: Track monster (10-25 points), Capture (100 points), Slay (50 points), Break parts (25 points per part), Special interactions (mount, environmental trap = 50). Level 1→6 requires 1,000 research points = 20-40 hunts per monster. Total completion: 50 monsters × 30 hunts = 1,500 hunts = 500+ hours. Endemic Life research: capture rare creatures for Research Center = unlock canteen ingredients (+20 HP, +30 stamina buffs). Palico gadgets locked behind region research: Coral Highlands = Coral Orchestra (attack buffs), Rotten Vale = Plunderblade (steal materials). 10+ hours per region to max research. Faction integration: Research unlocks are mechanical power increases (faster hunts = more efficient farming = endgame optimization).

**Synergies:**
- Narrative Progression (#14) - faction quests tell stories
- Milestone Unlocks (#6) - reputation tier rewards
- Branching Paths (#40) - faction exclusivity creates choice
- Achievement Systems (#35) - max all factions achievement
- Multiple Currencies (#44) - faction-specific currencies
- Collection Completion (#4) - collecting all faction rewards
- Knowledge Checks (#28) - optimal faction strategies

**Session vs Meta:** 35% Session (reputation gain during play), 65% Meta (working toward long-term faction goals)
