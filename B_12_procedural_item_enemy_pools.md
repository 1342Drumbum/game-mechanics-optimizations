## 12. Procedural Item/Enemy Pools (Seeded Variation)

**Psychological Hook:** Novelty maintains dopamine sensitivity and prevents tolerance. Variable ratio reinforcement resists extinction. Mastery through adaptation. Seeded runs enable competition and bug reproduction.

**Implementation Pattern:**

```python
import random
import hashlib

class ProceduralPoolSystem:
    def __init__(self, master_seed=None):
        self.master_seed = master_seed or random.randint(1000000, 9999999)
        self.master_rng = random.Random(self.master_seed)
        self.floor_rngs = {}

    def get_floor_rng(self, floor_number):
        """Hierarchical seeding: each floor deterministic RNG"""
        if floor_number not in self.floor_rngs:
            floor_seed = int(hashlib.md5(
                f"{self.master_seed}:floor:{floor_number}".encode()
            ).hexdigest(), 16) % (10 ** 8)
            self.floor_rngs[floor_number] = random.Random(floor_seed)
        return self.floor_rngs[floor_number]

    def generate_item_pool(self, floor_number, total_items=150, shown_per_run=20):
        """Select subset of total pool. Maintains variety, prevents overwhelming choice."""
        floor_rng = self.get_floor_rng(floor_number)
        items_by_rarity = self.categorize_items_by_rarity()
        available_items = []

        for rarity, items in items_by_rarity.items():
            ratio = shown_per_run / total_items
            count = max(1, int(len(items) * ratio))
            available_items.extend(floor_rng.sample(items, count))

        return available_items

    def weighted_pool_by_synergy(self, player_items, item_pool, synergy_weight=2.0):
        """Smart pooling: bias toward synergies without removing randomness"""
        synergy_items = self.find_synergies(player_items, item_pool)
        random_items = [i for i in item_pool if i not in synergy_items]

        weighted_pool = [(item, synergy_weight) for item in synergy_items]
        weighted_pool.extend([(item, 1.0) for item in random_items])

        items, weights = zip(*weighted_pool)
        return self.master_rng.choices(items, weights=weights, k=3)

    def validate_run_quality(self, generated_items, generated_enemies):
        """Quality assurance: reject unwinnable/trivial runs"""
        damage_items = [i for i in generated_items if i.type == 'damage']
        if len(damage_items) < 3:
            return False  # Unwinnable

        difficulty = [e.power_level for e in generated_enemies]
        if max(difficulty) < 5 or min(difficulty) > 8:
            return False  # Too easy or too hard

        return True
```

**Balancing Guidelines:**

**Pool Sizing:**
- Total pool: 60-150 items for depth without overwhelming
- Items per run: 10-20% of total (maintains scarcity)
- Enemy pool: 30-60 types, encounter 40-60% per run
- Guaranteed minimums: 3 damage items, 2 defense, 2 utility

**Seeding Best Practices:**
- Master seed: single seed defines entire run (reproducibility)
- Hierarchical seeds: floor, room, enemy spawns use derived seeds
- Shareable format: human-readable ("TREE-FROG-MOUNTAIN" vs "7842936")
- Separate RNG streams: player actions don't affect generation

**Pool Dilution Solutions:**
- Problem: 100 items, 1 synergy = 1%. 200 items = 0.5% (50% reduction)
- Solution 1: Scale choice count. `choices = 3 + (pool_size // 50)`. 100 items: 5 choices. 200: 7 choices.
- Solution 2: Smart pooling. Synergy probability = base × (1 + synergy_factor). 10% base, 2.0 factor = 30% for synergies.
- Solution 3: Unlockable pools. Start 50 items (focused), unlock to 150 (veteran depth).
- Solution 4: Item orthogonality. 150 items × 3 synergies = 450 interactions. Better than 300 items × 1 synergy = 300.

**Examples:**

**1. Slay the Spire:** Total: 240 relics + ~350 cards. Seen/run: 12-20 relics (5-8%), 25-40 cards (7-11%). Seed format: 6 chars (A-Z, 0-9), e.g., "1AB7C9F". Same seed = same offers. Quality: starter relic always damage-focused, Act 1 = 80% attack card in first 3 rewards, guaranteed campfire before elite. Data: 75M runs analyzed (Oct 2018-Nov 2020). Community tools: SpireStars, spirelogs.com.

**2. Vampire Survivors:** Total weapons: ~80 base + evolutions. Total passives: ~120. Available/run: 6 weapons + 6 passives. Constraint: 12/200 = 6% encounter rate. Evolution probability (specific weapon+passive): (6/80) × (6/120) = 0.38%. Strategy: target evolution early, lock in base + passive. Character starting weapon influences build (soft determinism).

**3. Binding of Isaac:** Total: 732 items (Repentance). Items/run: 15-30 (2-4% encounter rate). Total combinations: ~10^70 possibilities. Pool segmentation: treasure, boss, shop, devil, angel, secret, golden. Pool exhaustion: items removed after obtained (no duplicates), thins as run progresses = late game higher probability remaining high-tier. Seeded runs: 8-char alphanumeric. Daily challenge: same seed for all players.

**Success Metrics:** Variety perception: 80%+ "every run feels different." Build diversity: no strategy >40% pick rate. Optimal pool: 60-150 items (balances variety/familiarity). <60 repetitive after 20-30 runs. >150 dilution problem. Seeded engagement: 15-25% try seeded/daily challenges. Repeat playability: 20-30% reach 100+ hours (vs <10% without procedural).

**Common Pitfalls:**
- System time as seed: not shareable/reproducible. Generate integer seed (1M-10M) + share mechanism.
- Shared RNG state: player actions change generation. Separate RNG for generation vs gameplay.
- No quality guarantees: pure random = unwinnable (0 damage items) or trivial (all legendary). Validate generation, enforce minimums.
- Opaque seeds: "184729364" not memorable. Word-based ("TREE-FROG-MOUNTAIN") or short alphanumeric ("A7C9F").
- Pool dilution ignored: adding 100 items without adjusting. Scale choice count, smart pooling, or unlockable progression.

**Synergies:** Variable Rewards (#3), Build Variety (#15), Knowledge Checks (#28), Synergy Discovery (#11)

**Session vs Meta:** 90% Session (variety within run), 10% Meta (learning pool enables decisions)

---
