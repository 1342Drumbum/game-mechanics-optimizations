## 48. Resource Conversion (Exchange Systems)

**Psychological Hook:** Agency through flexible optimization - "I have excess Gems, convert to Keys." Cognitive engagement from exchange rate calculation. Exploits mental accounting - conversion feels like "smart trading" not spending. Mitigates bad RNG - flooded with Nectar but need Diamonds? Convert. Mastery expression through optimal exchange paths. Creates "trading game" mini-loop. Endowment effect reversal - willingness to trade surplus resources.

**Implementation Pattern:**
```
Conversion Design Patterns:

1. One-Way Hierarchy (Hades Wretched Broker):
   Gems → Keys → Nectar → Diamonds → Ambrosia → Titan Blood
   └─ Prevents reverse gaming (no infinite loops)
   └─ Exchange rates: 100:1 to 1000:1 (exponential)

2. Two-Way Market (Risk of Rain 2 Printers):
   Item A ↔ Item B (1:1 via Scrapper + Printer)
   └─ Scrap any item → generic Scrap token
   └─ Printer consumes Scrap → specific item
   └─ Allows flexible rebalancing mid-run

3. Limited-Time Deals (Flash Sales):
   Normal: 100 Gems → 1 Key
   Special: 10 Gems → 1 Key (90% discount)
   └─ Creates urgency, rewards checking shop

4. Crafting Conversion (Dead Cells):
   Blueprint + Cells → Permanent Unlock
   └─ Non-reversible sink
   └─ Blueprint (0-5% drop) + 50-300 Cells

5. Sacrifice Conversion (Health → Power):
   Max HP → Damage boost
   Grid Power → Reactor Core (Into the Breach)
   └─ Strategic permanent trade-offs
```

Display exchange rates clearly. Allow preview before conversion. Prevent infinite loops. Offer 2-5 conversion types. Make some rates favorable (limited-time deals).

**Examples:**

1. **Hades Wretched Broker (Hierarchical Exchange):**
   - **Standard Repeatable Trades (One-Way Only):**
     - 10 Gemstones → 1 Chthonic Key
     - 5 Chthonic Keys → 1 Nectar
     - 10 Nectar → 1 Diamond
     - 1 Diamond → 1 Ambrosia
     - 5 Ambrosia → 1 Titan Blood
   - **Combined Rate:** 1 Titan Blood = 1,000 Gemstones = 100 Keys = 20 Nectar = 2 Diamonds. Establishes value hierarchy.
   - **Limited-Time Offers (Much Better Rates):**
     - 10 Gems → 1 Nectar (normally 50 Gems via Keys)
     - 15 Keys → 1 Titan Blood (normally 100 Keys) = 85% discount
     - 25 Darkness → 3 Keys (unlocks Darkness trading, normally impossible)
   - **Strategic Play:** Hoard 500+ Gems, wait for Limited Offer, convert at 5× better rate. Reward patience and system knowledge.
   - **Infinite Loop (Patched):** 1 Titan Blood → 200 Gems → 20 Keys, then later trade 15 Keys → 1 Titan Blood. Net +5 Keys/cycle. Developers left this intentionally as late-game relief valve.

2. **Risk of Rain 2 (Scrapper + Printer System):**
   - **Scrapper:** Convert any item → Item Scrap (same rarity). Destroys item permanently, generates Common/Uncommon/Legendary Scrap token.
   - **3D Printer:** Trade 1 item (same rarity) → 1 copy of displayed item. Prioritizes consuming Scrap over other items.
   - **Workflow:**
     1. Find 5 weak items (e.g., 5× Monster Tooth = heal 2/kill, useless late-game)
     2. Scrap all 5 → 5× Common Scrap
     3. Find Soldier's Syringe Printer (+15% attack speed)
     4. Trade 5 Scrap → 5× Soldier's Syringe = +75% attack speed
   - **Strategic Depth:** Scrapper spawns randomly. Good players identify weak items early, scrap immediately, hunt printers for power items. Converts bad RNG (weak item drops) into deterministic power (targeted item stacking).
   - **Lunar Items:** CANNOT be scrapped (by design). Prevents reversing Shaped Glass (-50% HP). Commitment enforcement.

3. **Slay the Spire (Card Removal as Conversion):**
   - **Card Removal Service:** Gold → Deck Improvement (indirect conversion). Remove Strikes/Defends (weak starter cards). 75→100→125→150 gold escalating per removal.
   - **Event Conversions:**
     - Bonfire Spirits: Remove 1 card → gain max HP
     - Augmenter: Lose 25% max HP → upgrade all Strikes
     - Falling: Lose 1 rare relic → gain 3 random relics
   - **Forge Event:** Upgrade card, lose 50-100 HP. HP→Power conversion.
   - **Strategic Calculus:** Is removing 1 Strike worth 100 gold? In 15-card deck, 100g removes 6.7% of draws. Worth ~300g value in consistency. Expert players remove 4-6 cards/run.

4. **Into the Breach (Reputation → Grid/Cores):**
   - **Reputation Earning:** 1-2★ per objective (defend building, protect train). 8-12★ per island. Total ~40-50★ per full run.
   - **Conversion Options:**
     - 1★ Reputation → +1 Grid Power (defensive HP)
     - 3★ Reputation → +1 Reactor Core (powers mech abilities)
   - **Strategic Decision:** Grid protects from loss (health), Cores increase power (offense). Optimal: Heavy Reactor investment (7-12 cores) + minimal Grid (2-4). Strong mechs prevent damage > absorbing damage. Reputation spent on Grid is "insurance" but weak players need it.
   - **Time Pods:** Rescue = 1-2 Reactor Cores + reputation. Priority target - cores can't be "bought" with reputation efficiently (3★ = very expensive).
   - **Session vs Meta Tradeoff:** Reputation earned THIS run, spent THIS run. Pilots/achievements carry over but resources don't. Creates clean session resource cycle.

5. **Monster Train (Gold → Ember Economy):**
   - **Ember Generation Cards:**
     - Excavated Ember: 0-cost spell, gain 2 Ember + draw 1. Converts deck slot → 2 Ember/play.
     - Calcified Ember: 1-cost spell, -10 damage. Survives 2 battles → transforms into Excavated Ember. Early weak → late strong.
   - **Gold → Unit Upgrades:** Gold doesn't directly convert to Ember, but buying strong cheap units = more Ember-efficient deck. Fledgling Imp: 0-cost, 5 attack. Free play = saves Ember for bigger spells.
   - **Artifact Conversions:**
     - Perils of Production: Gain 50 gold, take 10 Pyre damage. HP→Gold conversion.
     - Sap Pylon: -1 starting Ember, +25 HP pyre. Ember→HP conversion (permanent tradeoff).
   - **Strategic Layering:** Converting gold → cheap units converts future Ember cost → damage output. Indirect resource conversion creates complex economy.

**Synergies:** Multiple Currencies (#46), Shop Mechanics (#49), Sacrifice Mechanics (#22), Push-Your-Luck (#16), High-Risk Paths (#17)

**Session vs Meta:** 70% Session (in-run conversions like Scrapper/Printer), 30% Meta (Hades Broker converting meta-currencies)
