# CATEGORY 9: ECONOMY & RESOURCE MANAGEMENT

---

## SYNERGY MATRIX: ECONOMY & RESOURCE MANAGEMENT

**High-Impact Combinations:**

| Mechanic Pair | Synergy Effect | Example |
|--------------|----------------|----------|
| #46 Multiple Currencies + #48 Conversion | Creates trading mini-game, mitigates bad RNG | Hades Wretched Broker (6 currencies, complex exchanges) |
| #47 Temp/Perm Split + #50 Crafting | Guarantees meta-progress even on failed sessions | Dead Cells (lose gold, keep blueprints+Cells) |
| #46 Multiple Currencies + #2 Granular Meta | Stratified progression prevents overwhelming complexity | Hades (Darkness = Mirror, Gems = Contractor, separate goals) |
| #49 Shop Rerolls + #3 Variable Rewards | Gambling loop with agency, high engagement | Balatro (reroll for rare Jokers, escalating cost) |
| #48 Conversion + #22 Sacrifice | HP→Power trades create strategic depth | Into the Breach (Grid→Cores), Monster Train (Pyre→Shards) |
| #50 Crafting + #4 Collection | Blueprint hunting extends engagement 100+ hours | Dead Cells (238 blueprints), Hades (326 Titan Blood) |
| #47 Session/Meta + #16 Push-Your-Luck | Risk-reward for banking resources | Dead Cells (carry Cells to Collector or risk death) |
| #49 Shop RNG + #20 Pity System | Prevents frustration from bad shops | Balatro (Luck stat improves reroll odds) |

**Three-Way Synergies:**

1. **#46 Multiple Currencies + #48 Conversion + #47 Session/Meta Split**
   - Example: Hades full economy
   - Effect: Session gold → immediate power, meta Darkness → Mirror, Titan Blood → aspects, Gems ↔ Keys conversion
   - Result: 3D resource management creates 200+ hour engagement

2. **#50 Crafting + #3 Variable Rewards + #4 Collection**
   - Example: Dead Cells blueprint hunting
   - Effect: Random blueprint drops → collection tracking → Cell spending unlocks
   - Result: Dopamine from find + dopamine from unlock = double reward loop

3. **#49 Shop Rerolls + #46 Multiple Currencies + #16 Push-Your-Luck**
   - Example: Balatro reroll economics
   - Effect: Money vs Interest tension + reroll escalation + gambling psychology
   - Result: Every shop = 5-minute strategic mini-game

**Anti-Synergies to Avoid:**

- **Too Many Currencies (>7):** Cognitive overload, analysis paralysis
- **No Conversion Options:** Flooded with useless currency = frustration
- **Session-Only Economy (no meta):** Failed runs feel wasted, high churn
- **Free Rerolls:** Removes strategic tension, players reroll infinitely = decision fatigue
- **Hidden Crafting Costs:** "Blueprint acquired!" then discover need 5,000 Cells you don't have = disappointment

---

## IMPLEMENTATION QUICK REFERENCE

### Economy System Selection Guide

**For 5-10 Minute Sessions (Balatro, Brotato):**
- ✅ Session-only currency (gold/money)
- ✅ Shop rerolls (escalating cost $5→$6→$7)
- ✅ Interest system (reward banking)
- ❌ Avoid complex crafting (too slow)
- ❌ Avoid 5+ currencies (too complex for short session)

**For 15-25 Minute Sessions (Hades, Slay the Spire, Dead Cells):**
- ✅ Session + Meta split (gold + Cells/Darkness)
- ✅ 3-6 currencies with clear purposes
- ✅ Conversion systems (limited, one-way)
- ✅ Crafting (blueprint + currency unlocks)
- ✅ Shop variety (rerolls OR deterministic)

**For 30+ Minute Sessions (Risk of Rain 2, Vampire Survivors):**
- ✅ Complex multi-currency (session + persistent)
- ✅ Deep crafting (Scrapper/Printer conversions)
- ✅ Collection completion (100+ hour goals)
- ✅ Prestige currencies (Lunar Coins)
- ❌ Avoid too-frequent shops (breaks flow)

### Currency Count by Session Length

| Session Length | Optimal Currency Count | Example Split |
|----------------|------------------------|---------------|
| 5-10 min | 1-2 | Money (session) |
| 10-20 min | 2-4 | Gold (session) + Cells (meta) + Keys (rare) |
| 20-30 min | 3-6 | Hades full system (6 currencies) |
| 30+ min | 2-5 | Gold + Lunar Coins + Meta Unlocks |

### Conversion Rate Guidelines

| From → To | Recommended Ratio | Purpose |
|-----------|------------------|----------|
| Common → Uncommon | 10:1 to 50:1 | Frequent small trades |
| Uncommon → Rare | 50:1 to 200:1 | Occasional big trades |
| Rare → Ultra-Rare | 100:1 to 1000:1 | Late-game sinks |
| HP → Power | 10-30% Max HP | Strategic sacrifice |
| Session → Meta | 3-10 runs worth | Prevents rapid completion |

### Shop Pricing Formulas

```
Base Card Price = 50 × Rarity_Multiplier
  Common: 50 (1×)
  Uncommon: 75 (1.5×)
  Rare: 150 (3×)

Relic Price = 150 + (50 × Rarity_Tier)
  Common Relic: 150
  Uncommon: 250
  Rare: 300
  Shop Special: 200

Reroll Cost = Base + (Count × Increment)
  Balatro: $5 + ($1 × rerolls)
  With voucher: $3 + ($1 × rerolls)
  With both: $1 + ($1 × rerolls)

Removal Cost = 75 + (25 × times_used)
  First: 75
  Second: 100
  Third: 125
  With Smiling Mask: Always 50
```

### Crafting Progression Curve

```
Early Unlocks (First 20%): 30-75 currency each
  Purpose: Quick wins, maintain engagement

Mid Unlocks (20-60%): 75-150 currency
  Purpose: Strategic build options

Late Unlocks (60-90%): 150-300 currency
  Purpose: Optimization, prestige

Final Unlocks (90-100%): 500+ currency
  Purpose: Completionist sink, status

Total Collection Time: 50-200 hours recommended
```

---

## BEHAVIORAL PSYCHOLOGY DEEP DIVE

### Multiple Currencies (#46)

**Mental Accounting (Thaler, 1985):**
- Players create separate "budgets" for each currency
- Spending Gems feels different than spending Darkness (even if convertible)
- Violates economic rationality but increases engagement
- "I have 5,000 Gems but only 500 Darkness" creates spending permission for Gems

**Loss Aversion Mitigation:**
- Spreading losses across currencies feels less painful
- "I lost all my gold but kept my Cells" = partial victory
- Single currency death = total loss = higher frustration

**Complexity Mastery:**
- Learning exchange rates = skill expression
- "I know 1 Titan Blood = 1000 Gems = prestige"
- Casual players ignore optimization, hardcore players optimize ruthlessly

### Session vs Meta Split (#47)

**Dual Progression Theory:**
- Short-term dopamine: Session gold → immediate item
- Long-term dopamine: Meta Cells → permanent unlock
- Satisfies both impulsive AND patient player types simultaneously

**Hope Maintenance:**
- Bad session (died immediately) still earns meta currency
- "This run sucked but I got 40 Darkness toward Death Defiance"
- Prevents rage-quit, maintains engagement through failure

**Skill vs Time Balance:**
- Session rewards scale with skill (good players earn more gold)
- Meta rewards scale with time (bad players still progress)
- Inclusive design: accessible to all skill levels

### Resource Conversion (#48)

**Agency Enhancement:**
- "I can convert my surplus Gems to needed Keys" = player control
- Reduces helplessness from bad RNG
- Voluntary trade feels like smart play, not spending

**Cognitive Engagement:**
- Calculating optimal exchange rates = mental challenge
- "Is 10 Nectar worth 1 Diamond? I need 15 Diamonds for Contractor..."
- Creates emergent "trading game" within core loop

**Regret Management:**
- "I overinvested in Gems but can convert some" = mistake mitigation
- Reduces buyer's remorse, increases experimentation

### Shop Rerolls (#49)

**Sunk Cost Escalation:**
- Spent $5 on reroll, saw mediocre item, "already spent $5, $6 more won't hurt"
- Creates spending spiral (intentional design)
- Self-limiting: eventually run out of money (prevents infinite spiral)

**Near-Miss Effect (Clark et al., 2009):**
- Reroll shows "good" item (but not perfect) = "next one might be perfect"
- Activates same brain regions as winning despite not winning
- Reroll 10× hunting Dead Branch = memorable experience even if fails

**Variable Ratio Schedule (Most Addictive):**
- Reroll outcome unpredictable (might get legendary, might get commons)
- Identical to slot machines (most addiction-resistant reinforcement)
- Ethical consideration: single-player game = less predatory than gacha

### Crafting Systems (#50)

**Zeigarnik Effect (Incomplete Tasks):**
- "147/238 blueprints" haunts player's mind
- Brain remembers incomplete tasks 2× better than completed
- Drives collection hunting, extends engagement 100+ hours

**Goal Gradient Effect:**
- Motivation spikes approaching completion
- "10 blueprints left" = intense drive
- "200 blueprints left" = overwhelming, low motivation
- Solution: Sub-collections (weapons, mutations, outfits separate trackers)

**Endowment Effect:**
- "I found this blueprint, it's MINE now" = ownership feeling before unlocking
- Losing blueprint (death before banking in Dead Cells) = intense loss aversion
- Drives careful play when carrying valuable blueprints

---

## CASE STUDY: Hades - The Platinum Standard

**Why Hades' Economy is Perfect:**

1. **Six Currencies, Zero Confusion:**
   - Darkness (Mirror), Gems (Contractor), Keys (unlocks), Nectar (relationships), Diamonds (late-game), Titan Blood (aspects)
   - Each has ONE primary purpose, minimal overlap
   - Conversion via Wretched Broker adds depth without complexity

2. **Perfect Session/Meta Balance:**
   - Session: Obols (gold) = tactical (shops mid-run)
   - Meta: Darkness = strategic (Mirror between runs)
   - Both earned simultaneously, different decision contexts
   - Never feel "wasted run" - always progress on 3+ currencies

3. **Conversion Prevents Bottlenecks:**
   - Flooded with Gems? Convert to Keys via Broker
   - Need 1 more Ambrosia? Convert Diamonds
   - Limited-time deals reward system knowledge (15 Keys → 1 Titan Blood = 85% discount)

4. **Crafting Depth:**
   - 326 Titan Blood for all weapon aspects = 100-150 hours
   - 35,365 Darkness for full Mirror = 200-700 runs
   - Visible progress bars, granular unlocks, always advancing

5. **No Shop Rerolls (Intentional):**
   - Boon RNG uses Fated Persuasion (reroll charges)
   - Well of Charon shops = fixed prices, no reroll
   - Strategic: "Bad shop? Skip it, save Obols for next"
   - Reduces decision fatigue, maintains run flow

**Result:** 93% Steam positive reviews, 500k+ sales, GOTY nominations. Economy design major contributor.

---

**Total Implementation:** These 5 mechanics form the foundation of roguelike economies. Hades combines all 5 perfectly. Dead Cells uses #47/#50 heavily. Balatro masters #49. Slay the Spire balances #46/#48/#49. Monster Train innovates #48 (HP→Power conversions). Together, they create 50-500 hour engagement through resource management alone.
