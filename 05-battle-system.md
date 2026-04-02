# Battle System — Dragon City-Style Gamification

Async, turn-based battle system for students. Inspired by Dragon City.

---

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    STUDENT WEB APP                          │
│  sprout-lab-web (Next.js 14, Cloudflare Pages)              │
│                                                             │
│  /battle          /squad        /characters    /egg         │
│  BattleArena      SquadPicker   CharacterList  EggOpening   │
│       │                │              │            │        │
│       └────────────────┴──────────────┴────────────┘        │
│                         │ Apollo Client (JWT)                │
└─────────────────────────┼───────────────────────────────────┘
                          │ GraphQL
┌─────────────────────────▼───────────────────────────────────┐
│              sprout-lab-service                             │
│         (Cloudflare Workers + Apollo Server)                │
│                                                             │
│  GraphQL Resolvers                                          │
│  ├── character/   ├── battle/   ├── squad/   ├── egg/       │
│  └── user-health/                                           │
│                                                             │
│  Services (Business Logic)                                  │
│  ├── BattleEngineService   ← core turn resolver             │
│  ├── ActionResolverService ← RPS rule engine                │
│  ├── AiOpponentService     ← weighted-random AI             │
│  ├── CharacterStatsService ← level → stat calc              │
│  ├── EggService            ← random character grant         │
│  ├── SquadService          ← squad validation               │
│  └── UserHealthService     ← global HP management           │
│                                                             │
│  Cloudflare Bindings                                        │
│  ├── DB (D1 SQLite via Drizzle)                             │
│  └── BATTLE_LOCK (KV) ← prevent concurrent battles         │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Database Schema (Drizzle + D1/SQLite)

### character.schema.ts
```ts
// Static definitions — 6 characters, seeded at deploy time
export const character = sqliteTable('character', {
  id: text('id').primaryKey(),          // e.g. 'char_knight'
  name: text('name').notNull(),
  type: text('type').notNull(),         // KNIGHT | NINJA | MAGE | BERSERKER | PALADIN | RANGER
  baseHp: int('base_hp').notNull(),     // e.g. 500
  baseDamage: int('base_damage').notNull(), // e.g. 80
  spriteUrl: text('sprite_url'),
});
```

### user-character.schema.ts
```ts
// Characters owned by students — no duplicates enforced by unique index
export const userCharacter = sqliteTable('user_character', {
  id: text('id').primaryKey(),
  studentId: text('student_id').notNull(),
  characterId: text('character_id').notNull().references(() => character.id),
  level: int('level').notNull().default(1),  // 1–10
  currentHp: int('current_hp').notNull(),    // tracks battle damage (unused in v1, use user_health)
  createdAt: int('created_at', { mode: 'timestamp' }).notNull(),
}, (t) => ({
  uniq: unique().on(t.studentId, t.characterId), // NO DUPLICATES
}));
```

### character-egg.schema.ts
```ts
export const characterEgg = sqliteTable('character_egg', {
  id: text('id').primaryKey(),
  studentId: text('student_id').notNull(),
  isOpened: integer('is_opened', { mode: 'boolean' }).default(false),
  awardedCharacterId: text('awarded_character_id'),  // null until opened
  openedAt: int('opened_at', { mode: 'timestamp' }),
  createdAt: int('created_at', { mode: 'timestamp' }).notNull(),
});
```

### user-squad.schema.ts
```ts
export const userSquad = sqliteTable('user_squad', {
  id: text('id').primaryKey(),
  studentId: text('student_id').notNull().unique(), // one squad per student
  slot1CharacterId: text('slot1_character_id').references(() => userCharacter.id),
  slot2CharacterId: text('slot2_character_id').references(() => userCharacter.id),
  slot3CharacterId: text('slot3_character_id').references(() => userCharacter.id),
  updatedAt: int('updated_at', { mode: 'timestamp' }).notNull(),
});
```

### user-health.schema.ts
```ts
// Global health bar — persists across battles
export const userHealth = sqliteTable('user_health', {
  studentId: text('student_id').primaryKey(),
  currentHp: int('current_hp').notNull().default(1000),
  maxHp: int('max_hp').notNull().default(1000),
  shieldActive: integer('shield_active', { mode: 'boolean' }).default(false),
  shieldPotionExpiresAt: int('shield_potion_expires_at', { mode: 'timestamp' }),
  updatedAt: int('updated_at', { mode: 'timestamp' }).notNull(),
});
```

### user-xp.schema.ts
```ts
// XP is ONLY for leveling — never spendable
// Points use existing transaction_history table
export const userXp = sqliteTable('user_xp', {
  studentId: text('student_id').primaryKey(),
  totalXp: int('total_xp').notNull().default(0),
  updatedAt: int('updated_at', { mode: 'timestamp' }).notNull(),
});
```

### battle.schema.ts
```ts
export const battle = sqliteTable('battle', {
  id: text('id').primaryKey(),
  attackerStudentId: text('attacker_student_id').notNull(),
  defenderStudentId: text('defender_student_id').notNull(), // real student's squad (async)
  status: text('status').notNull().default('IN_PROGRESS'), // IN_PROGRESS | WIN | LOSE | DRAW
  currentTurn: int('current_turn').notNull().default(1),
  maxTurns: int('max_turns').notNull().default(10),
  attackerHpSnapshot: int('attacker_hp_snapshot').notNull(),
  defenderHpSnapshot: int('defender_hp_snapshot').notNull(),
  xpReward: int('xp_reward').notNull().default(0),
  pointReward: int('point_reward').notNull().default(0),
  createdAt: int('created_at', { mode: 'timestamp' }).notNull(),
  endedAt: int('ended_at', { mode: 'timestamp' }),
});
```

### battle-turn.schema.ts
```ts
export const battleTurn = sqliteTable('battle_turn', {
  id: text('id').primaryKey(),
  battleId: text('battle_id').notNull().references(() => battle.id),
  turnNumber: int('turn_number').notNull(),
  attackerAction: text('attacker_action').notNull(), // ATTACK | BLOCK | WEAK_ATTACK
  defenderAction: text('defender_action').notNull(), // AI-generated
  attackerDamageDealt: int('attacker_damage_dealt').notNull(),
  defenderDamageDealt: int('defender_damage_dealt').notNull(),
  isCritical: integer('is_critical', { mode: 'boolean' }).default(false),
  createdAt: int('created_at', { mode: 'timestamp' }).notNull(),
});
```

### Extend shop_item for new item types
```ts
// Add to existing shop_item.schema.ts itemType field:
// SHIELD | SHIELD_POTION | HEALTH_POTION | TOME_LOW | TOME_MID_LOW | TOME_MID | TOME_HIGH
// effect field stores JSON:
//   SHIELD:        { duration: 3600 }         — blocks all damage for N seconds
//   SHIELD_POTION: { duration: 7200 }          — longer shield variant
//   HEALTH_POTION: { hpRestore: 200 }          — restores HP
//   TOME_LOW:      { levelRange: [1, 3] }      — levels up characters at level 1–3
//   TOME_MID_LOW:  { levelRange: [4, 5] }      — levels up characters at level 4–5
//   TOME_MID:      { levelRange: [6, 7] }      — levels up characters at level 6–7
//   TOME_HIGH:     { levelRange: [8, 9] }      — levels up characters at level 8–9
```

---

## 3. Backend Service Structure

```
sprout-lab-service/src/
├── db/
│   ├── character/
│   │   ├── character.schema.ts
│   │   ├── user-character.schema.ts
│   │   └── character.relations.ts
│   ├── battle/
│   │   ├── battle.schema.ts
│   │   ├── battle-turn.schema.ts
│   │   └── battle.relations.ts
│   ├── squad/
│   │   └── user-squad.schema.ts
│   ├── egg/
│   │   └── character-egg.schema.ts
│   └── user-health/
│       ├── user-health.schema.ts
│       └── user-xp.schema.ts
│
├── graphql/
│   ├── schemas/
│   │   ├── character.schema.ts
│   │   ├── battle.schema.ts
│   │   ├── squad.schema.ts
│   │   └── egg.schema.ts
│   └── resolvers/
│       ├── queries/
│       │   ├── character/
│       │   │   ├── get-my-characters.ts
│       │   │   └── get-all-characters.ts
│       │   ├── battle/
│       │   │   ├── get-battle.ts
│       │   │   └── get-battle-history.ts
│       │   ├── squad/
│       │   │   └── get-my-squad.ts
│       │   └── user-health/
│       │       └── get-my-health.ts
│       └── mutations/
│           ├── battle/
│           │   ├── start-battle.ts      ← picks opponent, snapshots HP
│           │   ├── submit-action.ts     ← player picks ATTACK/BLOCK/WEAK_ATTACK
│           │   └── claim-battle-reward.ts
│           ├── egg/
│           │   └── open-egg.ts          ← random character, no-duplicate logic
│           ├── squad/
│           │   └── set-squad.ts         ← validate 3 owned chars, save
│           └── shop/
│               └── use-item.ts          ← apply shield/potion effect
│
└── services/
    ├── battle/
    │   ├── battle-engine.service.ts     ← orchestrates turns
    │   ├── action-resolver.service.ts   ← RPS + crit logic
    │   └── ai-opponent.service.ts       ← weighted-random action picker
    ├── character/
    │   ├── character-stats.service.ts   ← level → HP/damage formula
    │   └── egg.service.ts               ← available chars, random pick
    ├── squad/
    │   └── squad.service.ts
    └── user-health/
        └── user-health.service.ts       ← apply/restore HP, shield check
```

---

## 4. Battle Engine Design

### 4a. Action Resolution — RPS Rule Engine

```
action-resolver.service.ts

ACTIONS = { ATTACK, BLOCK, WEAK_ATTACK }

COUNTER_TABLE = {
  ATTACK     beats  WEAK_ATTACK  → 100% damage to defender
  BLOCK      beats  ATTACK       → 0 damage taken + 30% counter-damage
  WEAK_ATTACK beats BLOCK        → 60% damage (bypasses block)
  ATTACK     vs     ATTACK       → both take 100% damage
  BLOCK      vs     BLOCK        → no damage
  WEAK_ATTACK vs    WEAK_ATTACK  → both take 60% damage
}

CRITICAL_HIT_CHANCE = 0.15  (15%)
CRITICAL_MULTIPLIER = 1.5

resolveTurn(attackerAction, defenderAction, attackerDmg, defenderDmg):
  outcome = COUNTER_TABLE[attackerAction][defenderAction]
  attackerDealt = outcome.attackerDealtRatio * attackerDmg
  defenderDealt = outcome.defenderDealtRatio * defenderDmg
  isCritical = Math.random() < CRITICAL_HIT_CHANCE
  if (isCritical) attackerDealt *= CRITICAL_MULTIPLIER
  return { attackerDealt, defenderDealt, isCritical }
```

### 4b. AI Opponent — Weighted Random

```
ai-opponent.service.ts

Strategy: Simple weighted probability with mild pattern-following.
  Base weights: { ATTACK: 40%, BLOCK: 30%, WEAK_ATTACK: 30% }
  Pattern bias: If player used ATTACK last 2 turns → increase BLOCK weight to 60%
  Cap: No single action > 70% weight

pickAction(turnHistory: BattleTurn[]): BattleAction
  weights = BASE_WEIGHTS
  lastPlayerActions = turnHistory.slice(-2).map(t => t.attackerAction)
  if (lastPlayerActions.every(a => a === 'ATTACK')) weights.BLOCK += 30%
  return weightedRandom(weights)
```

### 4c. Battle Flow

```
start-battle mutation:
  1. Validate: player has a squad (3 chars), not already in active battle
  2. Pick opponent: random student with squad != player (exclude self)
  3. Snapshot: player global HP, opponent squad stats
  4. Create battle record (status: IN_PROGRESS)
  5. Lock: KV BATTLE_LOCK[studentId] = battleId (prevent concurrent)
  6. Return battle state

submit-action mutation (per turn):
  1. Validate: battle IN_PROGRESS, correct student, turn not already submitted
  2. AI picks defenderAction
  3. ActionResolver.resolveTurn(playerAction, defenderAction, ...)
  4. Apply damage:
     - Attacker takes defenderDealt
     - Defender HP (snapshot) takes attackerDealt
  5. Create battle_turn record
  6. Check end condition:
     - Attacker HP ≤ 0 → LOSE
     - Defender HP ≤ 0 → WIN
     - Turn >= maxTurns → DRAW (higher HP wins)
  7. If ended: apply rewards, update global user_health, clear KV lock
  8. Return updated battle state

Damage flows into global user_health (NOT character HP)
Shields in user_health.shieldActive block incoming damage
```

### 4d. Character Stats Formula

```
character-stats.service.ts

// Linear scaling per level (1–10)
HP  = character.baseHp   + (level - 1) * 50   // e.g. 500, 550, 600...
DMG = character.baseDamage + (level - 1) * 10  // e.g. 80, 90, 100...

Squad stat = average of 3 characters' stats
```

---

## 5. GraphQL API Design

### Types
```graphql
type Character { id: ID!, name: String!, type: CharacterType!, baseHp: Int!, baseDamage: Int!, spriteUrl: String }
type UserCharacter { id: ID!, character: Character!, level: Int!, hp: Int!, damage: Int! }
type UserSquad { slot1: UserCharacter, slot2: UserCharacter, slot3: UserCharacter }
type UserHealth { currentHp: Int!, maxHp: Int!, shieldActive: Boolean!, shieldPotionExpiresAt: String }
type Battle { id: ID!, status: BattleStatus!, currentTurn: Int!, maxTurns: Int!, attackerHp: Int!, defenderHp: Int!, turns: [BattleTurn!]! }
type BattleTurn { turnNumber: Int!, attackerAction: BattleAction!, defenderAction: BattleAction!, attackerDamageDealt: Int!, defenderDamageDealt: Int!, isCritical: Boolean! }
type CharacterEgg { id: ID!, isOpened: Boolean!, awardedCharacter: Character }

enum BattleAction { ATTACK, BLOCK, WEAK_ATTACK }
enum BattleStatus { IN_PROGRESS, WIN, LOSE, DRAW }
enum CharacterType { KNIGHT, NINJA, MAGE, BERSERKER, PALADIN, RANGER }
```

### Queries
```graphql
myCharacters: [UserCharacter!]!
allCharacters: [Character!]!    # all 6 for reference
mySquad: UserSquad
myHealth: UserHealth
myBattle: Battle                # active battle or null
battleHistory(limit: Int): [Battle!]!
myEggs: [CharacterEgg!]!
```

### Mutations
```graphql
openEgg(eggId: ID!): UserCharacter!          # returns new character
setSquad(characterIds: [ID!]!): UserSquad!   # exactly 3 IDs
startBattle: Battle!
submitAction(battleId: ID!, action: BattleAction!): Battle!
claimBattleReward(battleId: ID!): Boolean!
useItem(purchaseId: ID!, characterId: ID): UserHealth!  # apply shield/potion; characterId required for tomes
```

---

## 6. Frontend Structure (sprout-lab-web)

```
app/(main)/
├── characters/
│   ├── page.tsx                    # character collection
│   ├── _features/CharacterCard.tsx
│   └── _features/CharacterGrid.tsx
├── egg/
│   ├── page.tsx                    # egg hatching UI
│   └── _features/EggOpening.tsx   # animation + reveal
├── squad/
│   ├── page.tsx                    # squad builder
│   └── _features/
│       ├── SquadSlot.tsx
│       └── CharacterPicker.tsx
└── battle/
    ├── page.tsx                    # battle entry / history
    ├── [battleId]/
    │   └── page.tsx                # active battle arena
    └── _features/
        ├── BattleArena.tsx         # main battle UI
        ├── ActionButtons.tsx       # ATTACK / BLOCK / WEAK_ATTACK
        ├── TurnLog.tsx             # turn-by-turn results
        ├── HealthBar.tsx           # player global HP display
        └── SquadDisplay.tsx        # both squads shown
```

---

## 7. Key Technical Challenges & Solutions

### 7a. Concurrent Battle Prevention
**Problem**: Student submits two actions simultaneously (double-tap).
**Solution**: `BATTLE_LOCK` KV namespace. `start-battle` sets `BATTLE_LOCK[studentId] = battleId`. `submit-action` checks the lock. Clear on battle end.

### 7b. No-Duplicate Character from Eggs
**Problem**: Egg must not award already-owned characters.
**Solution**: `egg.service.ts` queries `user_character` for student's owned `characterId` list. Random pick from `character` table excluding owned. If student owns all 6, award bonus points instead.

### 7c. Async Opponent Squad
**Problem**: Opponent student may change squad mid-battle.
**Solution**: Snapshot opponent squad stats into `battle` table at `start-battle`. Battle uses the snapshot — live changes don't affect ongoing battles.

### 7d. Separate XP vs Points
**Problem**: XP and Points must never mix.
**Solution**:
- **XP**: `user_xp.totalXp` — only incremented (never deducted). Used by `character-stats.service.ts` to determine level thresholds.
- **Points**: `transaction_history` (existing table) — spendable currency. Battle rewards write separate `transaction_history` entries with `type: 'BATTLE_REWARD'`.
- Services are completely separate: `UserXpService` vs existing `AwardPointsToStudentService`.

### 7e. Shield Item Integration
**Problem**: Shop shields must protect against async battle damage.
**Solution**: `user_health.shieldActive = true` when shield is equipped. `UserHealthService.applyDamage()` checks flag before reducing HP. Shield is consumed after battle ends if active.

### 7f. D1 SQLite Limits on Cloudflare Workers
**Problem**: D1 has query limits per request (~1000ms CPU).
**Solution**: Keep `submit-action` to max 5 DB operations (read battle, write turn, update health, check end, write reward). No N+1 queries — use JOIN for battle + turns.

### 7g. Tome Items (Character Leveling)
**Problem**: 4 tomes represent 4 level tiers — must only apply to characters within the correct level range.
**Solution**:
- `shop_item.itemType` = `TOME_LOW` | `TOME_MID_LOW` | `TOME_MID` | `TOME_HIGH`
- Each tome has a `levelRange` in its `effect` JSON: `{ levelRange: [min, max] }`
  - `TOME_LOW` → characters at level 1–3
  - `TOME_MID_LOW` → characters at level 4–5
  - `TOME_MID` → characters at level 6–7
  - `TOME_HIGH` → characters at level 8–9
- `use-item` mutation (`mutations/shop/use-item.ts`):
  1. Load `student_purchase` by `purchaseId`, verify it belongs to the student and is unused.
  2. Load the `shop_item` and parse `effect.levelRange`.
  3. Student picks which `user_character` to apply the tome to (pass `characterId` in mutation args).
  4. Validate: `user_character.level` is within `levelRange`; reject with clear error if not.
  5. Increment `user_character.level` by 1 (max cap: 10).
  6. Mark `student_purchase.usedAt = now()` to prevent reuse.
  7. Award XP via `UserXpService` (tomes grant XP, not points).
  8. Return updated `UserCharacter`.
- `CharacterStatsService.calculate(level)` is pure and testable — no changes needed there.

---

## 8. Reward Configuration

```ts
// config/battle-rewards.ts
export const BATTLE_REWARDS = {
  WIN:  { xp: 100, points: 50 },
  LOSE: { xp: 20,  points: 10 },
  DRAW: { xp: 50,  points: 25 },
} as const;

export const MAX_TURNS = 10;
export const GLOBAL_HP_MAX = 1000;
export const CRIT_CHANCE = 0.15;
export const CRIT_MULTIPLIER = 1.5;
```

---

## 9. Implementation Phases

### Phase 1 — Core Infrastructure
- DB schemas: `character`, `user_character`, `user_health`, `user_xp`
- Seed 6 characters
- `CharacterStatsService`, `UserHealthService`
- GraphQL: `myCharacters`, `myHealth`

### Phase 2 — Egg + Squad
- DB schemas: `character_egg`, `user_squad`
- `EggService` (no-duplicate logic), `SquadService`
- Shop integration: egg as purchasable item
- GraphQL: `openEgg`, `setSquad`, `mySquad`

### Phase 3 — Battle Engine
- DB schemas: `battle`, `battle_turn`
- `ActionResolverService`, `AiOpponentService`, `BattleEngineService`
- GraphQL: `startBattle`, `submitAction`, `claimBattleReward`
- KV lock integration

### Phase 4 — Shop Items
- Extend `shop_item` with new types
- `useItem` mutation: shields, potions, tomes
- Shield integration in `UserHealthService`

### Phase 5 — Frontend
- Characters page, Egg opening animation
- Squad builder
- Battle arena with action buttons + turn log
