## 47. Temporary vs Permanent Resources (Session/Meta Split)

**Psychological Hook:** Dual-progression psychology satisfies immediate AND long-term gratification simultaneously. Loss aversion mitigation - "I died but kept my Cells." Operant conditioning with guaranteed minimum rewards. Skill-gating for session resources (good players get more gold) while ensuring meta-progress for all (bad run still earns Darkness). Hope maintenance - even terrible runs contribute to meta. Goal stratification prevents overwhelming complexity.

**Implementation Pattern:**
```
Session Resources (100% Lost on Death/Victory):
├─ Purpose: Tactical power within single run
├─ Earn Rate: High (100-500/run)
├─ Spending: Immediate power (items, upgrades, rerolls)
├─ Risk/Reward: Push luck for more before death
└─ Examples: Gold, Ember, HP, Temporary buffs

Meta Resources (Persistent Across Runs):
├─ Purpose: Strategic unlocks, permanent upgrades
├─ Earn Rate: Moderate (30-150/run)
├─ Spending: Between-run upgrades, unlocks
├─ Protection: Usually safe from loss (auto-bank)
└─ Examples: Darkness, Cells, Titan Blood

Hybrid Resources (Conditional Persistence):
├─ Banking Mechanic: Spend to save
├─ Capacity Limits: Gold Reserves (max 25k)
├─ Risk/Reward: "Extract" safely or gamble
└─ Examples: Dead Cells Gold Reserves, Tarkov insurance
```

Clear visual separation. Session currency top-right (HUD), meta currency post-run screen. Guarantee minimum meta earning even on immediate death. Scale session earning with skill, meta with time.

**Examples:**

1. **Dead Cells (Explicit Split):**
   - **Session: Gold** - 800-2,500/run. Buys weapons (1,500-4,000), stat scrolls, mutation rerolls. Completely lost on death UNLESS Gold Reserves unlocked (costs 250 Cells). Gold Reserves I-V: Preserve 5,000→10,000→15,000→20,000→25,000 gold. Start next run with Dead Man's Bag containing saved gold. Creates strategic choice: "Do I buy this 3,500 weapon knowing I'm preserving gold anyway?"
   - **Meta: Cells** - 30-100/run. Cannot spend during run - only at Collector between biomes. Death before banking = total loss (creates push-luck tension). Survival to checkpoint = secured. Blueprint unlocks permanent. 15,000+ Cells for full completion.
   - **Hybrid Tension:** Carrying 75 Cells approaching Collector. Risk hard path for +25 Cells or safe path and bank now? Loss aversion activates.

2. **Hades (Blended Session→Meta):**
   - **Session: Gold (Obols)** - 100-300/run. Well of Charon shops: 100-200/item. Charon's Shop before boss: 150-300/item. Disappears on death/victory. Pure session resource.
   - **Meta: Darkness** - 50-150/run. Automatically deposited in House. Zero risk. Spent between runs at Mirror. 35,365 total for completion. Even 3-minute death run grants 20-40 Darkness.
   - **Meta-Rare: Titan Blood** - 1-3/run from bounties (first-time-per-weapon clears) or Heat clears. Capped earning = finite without Heat. 326 total needed. Session performance determines earning (must win), but kept permanently once earned.
   - **Perception:** Session feels high-stakes (spend gold NOW), Meta feels safe (Darkness auto-saves). Different psychological modes within same run.

3. **Vampire Survivors (Pure Meta):**
   - **Meta: Gold Coins** - 500-1,500/30-min run. No session spending - all spent in main menu PowerUps. Creates delayed gratification loop. "This run sucked but I earned 750 coins toward Amount PowerUp."
   - **Meta: Eggs** - 1-5/run from map exploration. Permanent +0.5% stat boost each. 200+ eggs possible. Compounds infinitely across runs.
   - **No Session Economy:** Intentional design - all upgrades happen during run through XP/chests, not purchasable. Gold provides ONLY meta-progression. Simplifies in-run decision-making to "survive and level."

4. **Risk of Rain 2 (Session with Meta Unlocks):**
   - **Session: Gold** - 500-3,000/run depending on stage count. Spent on chests (25-400 depending on rarity), shrines, 3D printers, shops. Completely lost on death. High earners (10+ stages) accumulate 5,000+ gold, buying dozens of items.
   - **Meta: Lunar Coins** - 0-5/run from random spawn. ~0.5 per stage average. Persist across runs. Spent on Lunar items (blue portals, 1 coin/item) or Bazaar Between Time (10 coins to enter, shop has 5 items at 1-2 coins each). Slow accumulation (100 hours = 200-300 coins). High-risk high-reward items (Shaped Glass, Gesture).
   - **Meta: Unlocks** - Character/item unlocks from achievements. Permanent expansion of item pools. Session performance (kill Mithrix, loop to Stage 20) earns meta-unlocks.

5. **Balatro (Session with Meta Unlocks):**
   - **Session: Money ($)** - Start $4-8. Earn $3-5/hand played, $1/hand remaining unplayed, $1 per $5 unspent (Interest capped at +$5/round, requires $25 banked). Spend on Jokers ($3-8), Planet/Tarot packs ($3-6), shop rerolls ($5→6→7→8 escalating, resets each shop). "Banking" $25 for Interest vs spending $8 on Joker = strategic tension. Completely lost on game over.
   - **Interest Mechanics:** $25 unspent = +$5/round passive. Over 8 Antes = +$40 total from interest. BUT requires NOT spending on Jokers, weakening deck. High-skill players leverage interest, beginners overspend.
   - **Meta: Unlocks** - Win with deck = unlock Stake 1. Win Stake 1 = unlock Stake 2. 8 Stakes × 15 decks = 120 unlock progression. Also card unlocks, Joker unlocks. Pure achievement-based, no spendable currency.

**Synergies:** Multiple Currencies (#46), Granular Meta-Currency (#2), First Win Bonuses (#8), Push-Your-Luck (#16), Prestige Loops (#5)

**Session vs Meta:** 50% Session (gold/ember/HP management in-run), 50% Meta (Cells/Darkness/Titan Blood progression across runs)
