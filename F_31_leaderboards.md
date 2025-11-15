## 31. Leaderboards (Global/Friend Rankings)

**Psychological Hook:** Social comparison theory (Festinger, 1954) - humans compulsively evaluate themselves against others. Status seeking drives upward comparison with top players (aspiration) and downward comparison with friends (superiority). Loss aversion motivates defending rank. Mere measurement effect - tracking performance improves it. Rank volatility near percentile boundaries (99th → 98th) creates disproportionate emotional response. Public visibility triggers self-presentation theory - performing identity through gameplay skill.

**Implementation Pattern:**
```
Display tiers:
- Global Top 10 (Prestige/Celebrity Status)
- Global Top 100 (Elite Recognition)
- Global Top 1% (Expert Tier)
- Friends Only (Immediate Social Circle)
- Personal Best (Self-Improvement Track)

Update frequency:
- Real-time for friends (immediate feedback)
- Hourly for global (prevents server spam)
- Historical tracking (show improvement: "Up 347 ranks this week")

Score normalization:
normalized_score = (raw_score - min) / (max - min) × 10000
percentile_rank = (players_below_you / total_players) × 100
```

Categories prevent discouragement:
- Separate by difficulty mode (Ascension 20, 5BC, Heat 32)
- Time-limited seasons (weekly/monthly resets)
- Skill-based brackets (new players isolated from veterans)
- Multiple scoring vectors (speed, score, kills, etc.)

**Examples:**

1. **Spelunky 2:** Daily Challenge leaderboards track both **score** and **depth** separately (2024 update). Each daily seed generates identical level layout for global competition. Top players reach **$500,000+ scores** with **World 7-99 depths**. Reaching World 7 guarantees **top 50 placement** regardless of money collected. Leaderboard updates within **2-3 minutes** of completion. Friend leaderboards show top 10 from Steam friends list, creating intimate competition. **75,000+ daily participants** on peak days. Historical tracking shows "Best Daily: Rank #47 / 68,234 players."

2. **Dead Cells:** Daily Challenge leaderboard separates into **global rankings** (top 1000 displayed) and **friend rankings** (all friends shown). Scores range **50-500 points** for average runs, **800-1,200+ for top players**. Point sources: enemy kills (**1-5pts base**, **5× for elites**), cursed chest completion (**+25pts**), bonus stars (**+5pts/kill for 15s**, stacking to **+10pts/kill**). No Assist Mode runs permitted (excluded from leaderboards). **4 minutes 30 seconds time limit** enforced. Top 100 players typically score **1,000+**, top 10 exceed **1,400+**. Friend competition drives **60% higher daily engagement** vs non-competitive modes.

3. **Slay the Spire:** Daily Climb uses complex scoring: **255pts for floors climbed**, **150pts for bosses killed**, multipliers for constraints (Pauper: **+50pts** no rares, Highlander: **+100pts** no duplicates, Light Speed: **+50pts** sub-45min). Top scores reach **1,700-2,000+ points**. Average completion: **800-1,200pts**. Leaderboard shows **global top 100** + **friend rankings** + **personal best history**. Seeded runs enable reproducibility - players share seeds to compete on identical runs. **Daily participation: ~15,000 players**, **weekly unique players: ~40,000**. Percentile display: "Top 12% (Rank 1,847/15,234)."

4. **Trackmania (2020):** Ghost racing creates asynchronous leaderboards where **100 players race simultaneously** against recorded ghosts. Track leaderboards show **world record** (often **sub-40 seconds** for 60s tracks), **top 100 times**, **friends list**, **personal best**. Delta timing shows **real-time comparison**: "+0.347s vs WR", "-0.124s vs Friend." Player medals: **Bronze (95th percentile)**, **Silver (85th)**, **Gold (50th)**, **Author (25th)**, **Champion (top 10%)**. Openplanet plugin "Any Ghost" allows racing **specific users by name**. Ranked matchmaking uses **TrueSkill system** (Microsoft, 2005) for skill-based brackets. **3+ million monthly active players**, **500,000+ daily tracks played**.

5. **Hades:** No native leaderboards, but **Speedrun.com integration** creates community rankings. Categories: **Any Heat (most popular)**, **32 Heat**, **Unseeded/Seeded**, **Fresh File**. Top speedrunners complete runs in **sub-8 minutes** (world record: **7m 31s**). Community Discord runs **"Aspect of the Week"** unofficial leaderboards for casual competition. Lack of in-game leaderboards criticized by competitive community - external tools like **haelian.com/zeus** track unofficial heat records. **60% of speedrunners** report leaderboards as primary motivation for continued play **beyond 100 hours**.

**Synergies:** Daily Challenges (#32), Achievement Systems (#35), Speedrun Potential (#30), Prestige Systems (#5)

**Session vs Meta:** 40% Session (performing for leaderboard), 60% Meta (rank maintenance, historical tracking)
