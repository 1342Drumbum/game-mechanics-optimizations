## 46. Multiple Currency Types (Specialized Economies)

**Psychological Hook:** Segregates progress streams preventing "solved" optimization. Loss aversion across currencies - can't hoard everything. Complexity mastery rewards system knowledge. Scarcity creates strategic tension - "do I need Darkness or Gems more?" Mental accounting (Thaler, 1985) - separate budgets feel less painful than unified spending. Collection completion across currencies (endowment effect). Status signaling through rare currency accumulation.

**Implementation Pattern:**
```
Currency Tier Design:
Session-Only: Earned 100-500/run, spent in-run only
   └─ Gold, Ember, Temporary Coins

Session→Meta Bridge: Earned 30-100/run, spent between runs
   └─ Cells, Darkness, Primary progression currency

Meta-Rare: Earned 1-5/run, gates major unlocks
   └─ Titan Blood, Diamonds, Ambrosia, Keys

Ultra-Rare: Earned 0.1-1/run, prestige unlocks
   └─ Special tokens, event-only currencies

Conversion Ratios:
Common → Rare: 100:1 to 1000:1
Rare → Ultra-Rare: 10:1 to 100:1
Ensure one-way conversion (prevents gaming)
```

Design 3-7 distinct currencies with clear purposes. Display all currencies prominently. Gate different systems behind different currencies. Allow limited conversion (see #48). Ensure earning multiple types per session.

**Examples:**

1. **Hades (6 Currencies):**
   - **Darkness:** 35,365 total for all Mirror upgrades. Earn 50-150/run. Death Defiance costs 1,530 (10-15 runs). Session→Meta bridge currency.
   - **Gemstones:** House Contractor upgrades. Earn 15-40/run. Fountain costs 5,000 (125+ runs). Cosmetic/convenience focus.
   - **Chthonic Keys:** Unlock Mirror talents (50 total needed). Earn 3-6/run initially, tapering to 0-2/run. Front-loaded currency.
   - **Nectar:** Relationship building (150+ total for all NPCs). Earn 2-5/run from chambers/fishing. Social progression gate.
   - **Diamonds:** Late-game Contractor items. Earn 0-2/run from Styx chambers. Ultra-rare tier.
   - **Titan Blood:** Weapon aspects (326 total for all aspects). Earn 1-3/run from bounties/Heat clears. Mastery currency. Exchange: 1 Titan Blood = 1000 Gems = 100 Keys = 20 Nectar via Wretched Broker.

2. **Dead Cells (2 Currencies):**
   - **Gold (Session-Only):** Earn 800-2,500/run. Spent on weapons (1,500-4,000), mutations reroll (1,000→2,000→4,000→8,000), stat scrolls (varies). Lost on death except Gold Reserves (max 25,000 preserved).
   - **Cells (Meta):** Earn 30-100/run. ~200 blueprints × 75 avg = 15,000 Cells for full unlock collection. Mutation unlocks: 50-300 Cells. Quality-of-life upgrades: 100-500 Cells. Recycling Tube (refund items for gold): 250 Cells. Lost if die before banking at transition.

3. **Slay the Spire (1 Primary):**
   - **Gold (Session-Only):** Earn 200-600/run. Card removal: 75→100→125→150 escalating. Shop cards: Common 50, Uncommon 75, Rare 150. Shop relics: 150-300. Potions ~50. Membership Card halves all prices. The Courier gives -20% AND restocks. Combined = 60% discount (0.5 × 0.8 = 0.4). Strategic spending - early removal vs late relics.

4. **Vampire Survivors (1 Meta Currency):**
   - **Gold Coins (Meta):** Earn 500-1,500/30-min run. PowerUp total: 111,450-202,000 depending on purchase order. Single PowerUp costs: Amount (projectiles) 5,000, Might (damage) 5× ranks starting 500. Greed PowerUp increases gold gain +50% max - early purchase multiplies future earnings. Strategic order optimization saves 90,000+ coins.

5. **Monster Train (2+ Currencies):**
   - **Gold (Session-Only):** Earn 200-500/run. Unit upgrades: 75-150, Spell upgrades: 75-125, Artifact shop: 100-200. Events offer gold modifiers (+50% gold but harder fight). Spent on shops between battles.
   - **Ember (Battle Resource):** Regenerates to 3 each turn. Card costs: 0-4 Ember. Resource management within battle - "banking" Ember for next turn not possible. Excavated Ember card: 0-cost, gain 2 Ember + draw 1.
   - **Pyre Shards (Event Currency):** Concealed Caverns event. Trade Pyre health (damage) for permanent upgrades. Non-renewable resource conversion.

**Synergies:** Granular Meta-Currency (#2), Resource Conversion (#48), Shop Mechanics (#49), Prestige Loops (#5), Crafting Systems (#50)

**Session vs Meta:** 40% Session (gold/ember spent in-run), 60% Meta (permanent currencies like Darkness/Cells/Titan Blood)
