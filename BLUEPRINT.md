# The Chronicles — Implementation Blueprint

## Context
Two-player personality-profiling RPG in a single HTML file. Each player takes a quiz; Claude builds a D&D character sheet from their answers (OCEAN scores hidden, stats visible). Players then play a turn-based cooperative adventure where Claude acts as Dungeon Master and silently updates personality scores each turn. At the end, each player gets a private personality reveal comparing who they were at the start vs. who they became, with a scene-by-scene breakdown designed as a couple's talking-point report.

**Key architectural constraint added:** Cross-device play via share codes. No server. No framework. Single HTML file. Anthropic API called directly from the browser. State persisted in localStorage + share codes passed between devices.

---

## Cross-Device Model

Each device plays as one player. On first launch, a device picks its player identity (P1 or P2) and stores that in localStorage — this is device-local and never travels in share codes.

The **share code** is the full serialized game state (JSON → LZ-compressed → base64). It is the only way state moves between devices. After each player ends their turn, they generate a share code and send it to their partner (via iMessage, etc.). The partner pastes it on their device to continue.

The share-code round-trip is the intended communication ritual — it's what keeps both players engaged between sessions.

---

## State Data Structure

```
STATE (everything in share code except playerIdentity)
├── meta
│   ├── version: string
│   ├── activeTurn: 1 | 2          ← whose turn it is right now
│   └── phase: "quiz" | "forge" | "adventure" | "reveal"
│
├── settings
│   └── apiKey: string             ← stored in localStorage only, never in share code
│
├── players
│   ├── [1] and [2]
│   │   ├── quiz
│   │   │   ├── answers: [{question, answer}]
│   │   │   └── complete: boolean
│   │   │
│   │   ├── profile   ← HIDDEN from all UI during gameplay
│   │   │   ├── openness: 0–100
│   │   │   ├── conscientiousness: 0–100
│   │   │   ├── extraversion: 0–100
│   │   │   ├── agreeableness: 0–100
│   │   │   ├── emotionalStability: 0–100
│   │   │   └── baseline: { same 5 scores }  ← frozen at quiz end
│   │   │
│   │   └── characterSheet   ← VISIBLE
│   │       ├── name, class, backstory
│   │       ├── stats: {STR, DEX, CON, INT, WIS, CHA}
│   │       ├── traits: [string × 3]
│   │       ├── weakness: string
│   │       ├── maxHP, currentHP
│   │       └── actionsRemaining: number  ← resets each turn
│
├── adventure
│   ├── started: boolean
│   ├── complete: boolean
│   ├── currentScene: number
│   │
│   └── scenes: [
│       {
│         sceneNumber, title,
│         openingNarrative: string,
│         encounterSkeleton: string,     ← abstract, for story archive
│         sceneDirective: string,        ← HIDDEN: "joint" | "focus_player1" | "focus_player2"
│         targetedTraits: [string],      ← HIDDEN: which OCEAN traits this scene tests
│         rounds: [
│           {
│             roundNumber: number,
│             player1Turn: TurnRecord | null,
│             player2Turn: TurnRecord | null
│           }
│         ],
│         complete: boolean
│       }
│     ]
│
│   TurnRecord:
│   ├── actions: [
│   │   {
│   │     action: string,               ← player's typed input
│   │     diceRoll: number | null,
│   │     outcome: string,              ← Claude's narrative
│   │     profileDeltas: {OCEAN×5},     ← HIDDEN
│   │     outOfCharacter: boolean       ← HIDDEN
│   │   }
│   │ ]
│   ├── turnSummary: string             ← clear summary for partner
│   ├── vagueSummary: string            ← cryptic summary if OOC occurred
│   └── hasOutOfCharacter: boolean      ← whether to show vague version
│
└── reveals
    ├── player1: { unlocked: boolean, data: RevealObject | null }
    └── player2: { unlocked: boolean, data: RevealObject | null }

RevealObject:
├── profileStart: { openness, conscientiousness, extraversion, agreeableness, emotionalStability }  ← numbers only
├── profileEnd:   { openness, conscientiousness, extraversion, agreeableness, emotionalStability }  ← numbers only
│   NOTE: Quiz answers are NOT surfaced in the reveal UI — only the derived scores are shown.
│         Claude receives quiz answers internally for analysis quality but they are not
│         stored in RevealObject or displayed to the player.
├── summary: string
├── keyShifts: [{trait, direction, magnitude, cause}]
├── sceneBreakdowns: [
│   {
│     sceneNumber, title,
│     yourActions: [string],
│     partnerActions: [string],
│     complementMoments: [string],
│     conflictMoments: [string],
│     outOfCharacterMoments: [{actionText, why}]
│   }
│ ]
└── closingInsight: string
```

---

## Screen Flow

```
[Title Screen]
  "New Game" → [Device Setup]    "Load Game" → [Import Code]
       ↓
[Device Setup]
  "I am Player 1" / "I am Player 2"
  (stored in localStorage, never in share code)
       ↓
[Settings Screen]
  API key input → saved to localStorage only
       ↓
[Quiz Screen]  (each player on their own device)
  Questions one at a time — fixed core, then Claude followups
  Progress shown as neutral steps (no trait names shown)
       ↓  quiz complete
[Waiting Screen — Quiz]
  "Quiz complete. Generate your share code and send it to your partner."
  [SHARE CODE DISPLAY + COPY BUTTON]
  "Paste your partner's code when they're done:"
  [IMPORT FIELD]
  Once both quizzes imported → auto-advance to forge
       ↓
[Forging Screen]
  "Your characters are being forged…" (API call)
       ↓
[Character Reveal Screen]
  Shows YOUR character sheet (class, stats, traits, weakness, backstory)
  Partner's sheet hidden until both have generated → then shows partner card
       ↓  both characters exist in share code
[Adventure Screen — Active Turn]
  ┌─────────────────────────────────┐
  │ Scene title + opening narrative │
  │ Round history (scrollable)      │
  │ Partner summary card ← NEW      │
  │   (clear or vague, see below)   │
  │ Action input (1–2 actions/turn) │
  │ [End Turn] button               │
  └─────────────────────────────────┘
       ↓  player ends turn
[Turn Complete Screen]
  Claude-generated summary of this turn is embedded in share code
  [SHARE CODE DISPLAY + COPY BUTTON]
  "Send this to your partner — it's their turn."
       ↓  (other device)
[Import Screen]
  Paste share code → loads game state
  If not your turn → [Waiting Screen — Adventure]
  If your turn → [Adventure Screen]
       ↓  "End Journey" chosen
[Journey Complete Screen]
  [Generate Reveals] → API calls (one per player)
       ↓
[Reveal Gate — Player N]
  "This is your private reveal. Tap when ready."
       ↓
[Reveal Screen — Player N]
  Profile start vs. end comparison
  Key personality shifts with causes
  Scene-by-scene breakdown
  Closing insight
       ↓  both reveals unlocked
[Comparison View]
  Shared scene analysis — complement and conflict moments
  Designed as talking-point prompts for the couple
```

---

## The Out-of-Character Summary Mechanic

When Player A ends their turn, Claude generates the turn summary. If any of Player A's actions were flagged `outOfCharacter: true`:

1. Claude generates two summaries in the same API call:
   - `turnSummary`: A clear, narrative description of what Player A did
   - `vagueSummary`: A deliberately cryptic/fragmented version — describes *effects* and *reactions* without revealing *intent* or *logic* (e.g., "Something shifted in the air. The merchant looked unsettled. An odd silence followed.")
2. `hasOutOfCharacter: true` is set in the TurnRecord

When Player B loads the share code and it's their turn:
- If `hasOutOfCharacter: false`: Partner summary card shows the clear `turnSummary`
- If `hasOutOfCharacter: true`: Partner summary card shows the `vagueSummary` with a note: *"Your partner's actions are difficult to interpret. You may spend one of your actions to make sense of them."*
  - If Player B spends an action on this: Claude call that returns a clearer (but still flavored) interpretation, uses one of their `actionsRemaining`
  - The spent action is logged as a TurnRecord action type: `{type: "interpret", outcome: string}`

This mechanic:
- Creates in-game mystery around personality deviations
- Costs the out-of-character player indirectly (partner loses an action)
- Creates a natural conversation trigger: "Why did you do that weird thing?"
- Feeds the personality profile: choosing TO interpret vs. ignoring also signals a trait

---

## Action Point System

Actions are split into two categories. Claude classifies every player input as one or the other:

**Free actions (no AP cost)** — exploration, information gathering, no stakes:
- Moving through the world (enter a tavern, walk the market, head to the docks)
- Talking to NPCs casually (ask directions, greet a merchant, eavesdrop on a conversation)
- Browsing a shop (view inventory, ask prices — no purchase yet)
- Examining objects, reading notices, observing the environment

**Committed actions (costs 1 AP)** — consequential, risky, or plot-advancing:
- Combat moves (attack, defend, use an ability)
- Attempting something with meaningful risk (pick a lock, bluff a guard, steal)
- Major social decisions (persuade, intimidate, make a deal)
- Making a purchase
- Any action Claude determines will meaningfully change the state of the world

**Classification is Claude's job.** Each player input returns an `actionType` field:

```json
{
  "actionType": "free" | "committed",
  "narrative": "...",
  "profileDeltas": {...},          ← present on both types, smaller magnitude on free actions
  "pendingConsequence": null,      ← only possible on committed actions
  "outOfCharacter": false,         ← lower sensitivity threshold for free actions
  "diceRoll": null,                ← only on committed actions
  "inCommittedSituation": false    ← reflects scene state AFTER this action resolves
}
```

**Turn flow:**
- Players can take unlimited free actions — no AP cost, no turn limit on these
- Each committed action costs 1 AP
- Claude returns an `inCommittedSituation` boolean on every response, reflecting the current state of the scene:
  - `true` = combat underway, direct confrontation, high-stakes moment in progress
  - `false` = exploration mode, low-stakes conversation, object interaction, shopping
- When AP hits 0:
  - If `inCommittedSituation: true` → turn **auto-ends**, share code generated immediately. The situation demands it.
  - If `inCommittedSituation: false` → turn **continues** with free actions only. Player sees a subtle indicator that their AP is spent but can keep exploring. "End Turn" is always visible.
- "End Turn" is always available at any point regardless of AP remaining

**AP costs for meta-actions:**
- Interpret partner's vague summary → 1 AP
- Intercept partner's pending damage → 1 AP

**Why this works:**
- Natural exploration loop: players gather context freely, then commit to 2 meaningful actions
- Free actions still generate profile deltas (mild) — simply chatting with an NPC can reveal personality
- Shops feel like shops, not resource drains
- AP scarcity only kicks in when something actually matters



## Damage Resolution

Claude decides per-action whether damage is immediate or deferrable based on narrative context — not a fixed rule. Examples:

- A slow melee swing that Player 1 partially dodged → **deferrable** (partner can help)
- A surprise arrow from ambush → **immediate** (no time to react)
- A trap that snapped shut → **immediate**
- A merchant's hired guards closing in → **deferrable** (the threat hasn't landed yet)

When Player 1 takes a risky action, Claude returns a `pendingConsequence` alongside the narrative:

```json
{
  "narrative": "...",
  "pendingConsequence": {
    "target": 1,
    "description": "The guard's axe is already swinging toward you.",
    "damage": 8,
    "canBeIntercepted": true,
    "interceptDifficulty": "medium",
    "interceptWindow": "Your partner may still be able to pull you clear."
  },
  "profileDeltas": {...},
  "outOfCharacter": false
}
```

If `canBeIntercepted: false`, damage is applied immediately and narrated — Player 2 sees the outcome when they load the share code, not an open threat. No intercept option is shown.

If `canBeIntercepted: true`:
- `pendingConsequence` is stored in the current round's state and surfaced to the **partner** (whichever player is not the target) as context
- The partner sees: *"[description]. [interceptWindow]"* with an option to spend one AP intercepting
- Interception is bidirectional — either player can be the target, and either can be the interceptor

**Interception outcomes** (resolved by dice roll + Claude judgment):

| Result | What happens |
|---|---|
| Full failure | Interception attempt fails. Full damage to original target. |
| Partial success | Damage split between both players (e.g. 8 dmg → 4 each). |
| Full success | Damage negated or greatly reduced. No one takes meaningful harm. |
| Self-sacrifice | Interceptor absorbs all damage, protecting partner completely. Rare — Claude triggers this when it fits the narrative or the interceptor rolls exceptionally high and their traits support it (high agreeableness). |

Claude returns the interception resolution:
```json
{
  "interceptNarrative": "string",
  "interceptOutcome": "failure" | "partial" | "success" | "sacrifice",
  "damageToOriginalTarget": 8,
  "damageToInterceptor": 0,
  "profileDeltas": {...}    ← both players get deltas from this exchange
}
```

The choice to intercept — and the outcome type — both generate profile deltas. Sacrificing yourself is a strong agreeableness signal. Failing to intercept and watching your partner take damage can signal other traits depending on what preceded it.

The intercept action:
- Costs 1 of Player 2's actions
- Claude receives: both character sheets, the pending consequence, Player 2's relevant traits
- Claude returns: `{interceptNarrative, success, damageApplied}` — even on failure, the attempt is narratively meaningful
- The intercept attempt itself generates profile deltas (choosing to help = agreeableness signal)

At round-end: any unresolved `pendingConsequence` is automatically applied and narrated.

---

## Scene Crafting System

Before generating each new scene, the engine computes a **deviation score** for each player — the sum of absolute differences between their current OCEAN profile and their baseline across all 5 traits (0–500 max).

```
deviationScore(player) = Σ |current[trait] - baseline[trait]| for each of 5 traits
```

Thresholds:
- `< 40` → tracking with baseline (low deviation)
- `40–80` → moderate drift
- `> 80` → significant deviation

**Scene directive rules:**

| P1 deviation | P2 deviation | Scene focus |
|---|---|---|
| Low | Low | Joint test — challenge designed to probe both profiles simultaneously |
| Moderate/High (one player) | Low | Focus on the drifting player — scenario specifically targets their shifted traits |
| Both Moderate/High | — | Focus on the player with the **higher** deviation score |
| Equal high deviation | — | Scene tests the trait with the greatest drift across both players |

The scene directive is passed to Claude as a hidden instruction alongside both profiles. Claude uses it to shape encounter type, NPC behavior, and moral framing — never mentioning personality terms in the narrative.

**Claude's scene-start prompt receives:**
- Both character sheets
- Both current OCEAN profiles (hidden layer)
- Both baseline profiles (hidden layer)
- Computed deviation scores (hidden)
- Scene directive: `"joint"` | `"focus_player1"` | `"focus_player2"` | `"focus_trait: <traitName>"`
- Previous scene summaries (for narrative continuity)

**Claude's scene-start response schema:**
```json
{
  "title": "string",
  "openingNarrative": "string",
  "encounterSkeleton": "string",   ← abstract description for story archive
  "targetedTraits": ["agreeableness", "conscientiousness"],
  "sceneDirective": "focus_player1"
}
```

`targetedTraits` and `sceneDirective` are stored in the scene record but never shown in the UI during gameplay. They appear in the reveal report to explain *why* certain scenes felt like they were designed for you.

---

## Story Archive (Save System)

Saves store the **adventure scaffold**, not the player profiles.

```
StoryArchive:
├── id: uuid
├── title: string          ← player-named (e.g. "Our First Adventure")
├── savedAt: timestamp
├── scenes: [
│   {
│     sceneNumber, title,
│     openingNarrative,
│     encounterSkeleton: string   ← Claude's encounter setup, no character-specific flavor
│   }
│ ]
└── totalRounds: number
```

What is NOT saved: OCEAN scores, character sheets, dice outcomes, player actions, profile deltas.

**Replaying a story:**
- The same scene structure and encounter types are fed back to Claude
- Claude regenerates each scene using the players' *current* character sheets (re-run through the quiz at replay time, or reuse existing profiles)
- This creates a meaningful comparison: same world, same threats, different people handling them
- The replay's reveal report can optionally compare against the original run

---

## Main Systems & Key Functions

### `shareCode.export(state)` → string
Serialize state → strip API key and deviceIdentity → JSON → LZ-compress → base64. Produces a compact alphanumeric string.

### `shareCode.import(code, deviceState)` → mergedState
Decode share code → merge with device-local data (API key, playerIdentity) → validate schema version → return merged state.

### `api.call(systemPrompt, userMessage)` → parsed JSON
Single wrapper for all Claude calls. All responses are structured JSON. Handles errors and retries.

---

### Quiz Engine
- `CORE_QUESTIONS[10]` — Scenario-based, written in advance, cover all 5 OCEAN traits
- `submitAnswer(player, q, a)` → append answer, call `checkConfidence()` every 3 answers
- `checkConfidence(player)` → API call: `{confident: bool, followUp: string|null}`
- `finalizeProfile(player)` → API call → returns OCEAN scores (0–100 per trait)
- `generateCharacterSheet(player, ocean)` → API call → returns full character sheet object

### Adventure Engine
- `startAdventure()` → API call with both character sheets → opening scene narrative
- `submitAction(player, text)`:
  1. Roll dice if challenge (d20)
  2. API call: both sheets + both profiles + scene history + action + dice
  3. Claude returns `{narrative, pendingConsequence|null, profileDeltas, outOfCharacter}`
  4. Append to current round's TurnRecord; store `pendingConsequence` on round if present
  5. Decrement `actionsRemaining`
- `interceptPendingDamage(player)`:
  1. Costs 1 action
  2. Dice roll vs. `interceptDifficulty`; API call with both sheets + pending consequence + Player 2 traits
  3. Claude returns `{interceptNarrative, success, damageApplied}`
  4. Clears `pendingConsequence` from round state
- `resolveRoundEnd()` → applies any remaining unresolved `pendingConsequence`; narrates automatically
- `computeDeviation(player)` → returns sum of |current - baseline| across 5 traits
- `getSceneDirective()` → compares both players' deviation scores → returns `{directive, targetedTraits}`
- `advanceScene()` → calls `getSceneDirective()`, sends result + both profiles + prior scene summaries to Claude → returns new scene with hidden `targetedTraits` stored in scene record
- `endTurn(player)`:
  1. API call: full TurnRecord → `{turnSummary, vagueSummary, hasOutOfCharacter}`
  2. Mark TurnRecord complete, set `activeTurn` to partner
  3. Call `shareCode.export()`, display to player
- `interpretPartnerTurn(player)`:
  1. Costs 1 action
  2. API call: vague summary + full TurnRecord → clearer narrative interpretation
  3. Append as `{type: "interpret"}` action

### Personality Tracker (hidden)
- `applyDeltas(player, deltas)` — clamp 0–100
- `isOutOfCharacter(player, deltas)` — returns true if any delta direction opposes baseline by threshold

### Reveal Generator
- `generateReveal(player)` → large API call with: quiz answers (for analysis quality only, not shown), baseline profile, final profile, all TurnRecords with hidden fields → returns `RevealObject`
- RevealObject displays OCEAN numbers only — quiz answers are not surfaced in the UI

### State Manager
- `save(state)` → localStorage (includes API key + playerIdentity)
- `load()` → deserialize + derive current screen from phase/activeTurn/playerIdentity
- `reset()` → clear localStorage, confirm first

### Story Archive Manager
- `saveStoryArchive(title)` → extracts scene scaffolds (titles, encounter skeletons, opening narratives) from completed adventure; strips all character/profile data; saves to a named localStorage slot
- `listArchives()` → returns saved story titles + dates
- `replayArchive(id)` → loads scene scaffolds, starts a new adventure using current player profiles but the same encounter structure; Claude regenerates each scene intro with current character sheets layered in

---

## Prompting Strategy

All Claude calls return **structured JSON**. Narrative text is a field inside JSON. Hidden data (profileDeltas, outOfCharacter flags) travels in the same response payload alongside visible content — players never see the raw API response.

Each adventure turn prompt contains:
- Both character sheets (visible layer — drives narrative flavor)
- Both current OCEAN profiles (hidden layer — drives NPC behavior, scene difficulty, tone)
- Full current round history
- Player's action + dice result

Turn response schema:
```json
{
  "narrative": "string",
  "profileDeltas": {"openness": 2, "conscientiousness": -1, "extraversion": 0, "agreeableness": 3, "emotionalStability": -2},
  "outOfCharacter": false,
  "outOfCharacterReason": null,
  "diceUsed": true,
  "sceneComplete": false
}
```

End-of-turn summary schema:
```json
{
  "turnSummary": "string (clear narrative)",
  "vagueSummary": "string (cryptic if hasOOC, else same as turnSummary)",
  "hasOutOfCharacter": false
}
```

---

## Build Order

| # | Phase | What | Why first |
|---|-------|------|-----------|
| 1 | Foundation | State structure + localStorage save/load | Everything depends on this |
| 2 | Foundation | `shareCode.export()` / `shareCode.import()` | Core sync mechanism |
| 3 | Foundation | `api.call()` wrapper + schema validation | All features need this |
| 4 | Foundation | Settings screen + API key storage | Needed before any Claude call |
| 5 | Setup | Device Setup screen (playerIdentity) | Required for cross-device identity |
| 6 | Quiz | Fixed quiz questions flow | Data pipeline starts here |
| 7 | Quiz | `finalizeProfile()` + `generateCharacterSheet()` | Close the quiz pipeline |
| 8 | Quiz | Character reveal screens | Validate output looks right |
| 9 | Quiz | Waiting/share-code screens between quiz phases | Cross-device quiz handoff |
| 10 | Quiz | `checkConfidence()` + adaptive followup questions | Polish quiz layer |
| 11 | Adventure | Adventure screen, basic turn loop (narrative only) | Core gameplay |
| 12 | Adventure | Dice mechanics | Add after basic loop is stable |
| 13 | Adventure | Profile delta tracking per turn | Hidden personality layer |
| 14 | Adventure | `computeDeviation()` + `getSceneDirective()` + scene crafting | Adaptive scene system |
| 14b | Adventure | Context-aware damage + `interceptPendingDamage()` | Co-op damage mechanic |
| 15 | Adventure | `endTurn()` + share code generation | Cross-device adventure handoff |
| 16 | Adventure | Partner summary card (clear + vague logic) | Out-of-character mechanic |
| 17 | Adventure | `interpretPartnerTurn()` mechanic | Costs an action |
| 18 | Reveal | `endAdventure()` + `generateReveal()` | The payoff |
| 19 | Reveal | Private reveal screens (OCEAN numbers only, no quiz answers) | Final presentation |
| 20 | Reveal | Comparison view (scene-by-scene talking points) | Couple's debrief |
| 21 | Archive | `saveStoryArchive()` + `replayArchive()` | Replay with evolved profiles |
| 22 | Polish | `getScreen()` resume logic on load | Handle reloads + imports gracefully |
| 23 | Polish | Loading states, error handling, transitions | Last |

---

## Design Decisions (all resolved)
- Quiz question style: **Scenario-based** ("You find a wallet…") — harder to game, richer answers
- Dice: **Claude-flagged challenges only** — not every action
- HP damage: **Context-driven** — Claude decides per-action if damage is immediate or deferrable based on narrative logic (ambush/traps = immediate; slow threats = deferrable). Player 2 can only intercept when Claude flags it as possible.
- Scene crafting: **Deviation-adaptive** — scenes jointly test both profiles when both are tracking baseline; when one player drifts significantly, scenes target the higher-deviation player's shifted traits specifically
- Saves: **Story archives** — saves scene scaffold only, replayed with current character profiles
- Reveal content: **OCEAN numbers only** — quiz answers used internally for Claude analysis, not shown
