# PITFALLS — The Chronicles

Hard-won lessons from prior projects that apply directly to this game.
Read before coding. Each entry is a constraint, not a suggestion.

---

## State & Co-op

`[gamedev][state][coop]` Store the seeded RNG's `{seed,count}` inside the saved
world and ban ambient `Math.random()`/`Date.now()` in the sim — one discipline
gives exact save/load, replay, and the lockstep-co-op option. Our share codes
carry full state; dice rolls must be reproducible from the code alone.

`[coop][state]` Clear a relationship at the point you act on it, not only at the
event that ends one side. Applied here: a `pendingConsequence` must be cleared
the instant an intercept resolves OR at round-end — never leave it dangling
waiting for a lifecycle event that might not fire.

`[gamedev][state][process]` A modal overlay reused for multiple roles leaks state:
it showed start-screen class-select mid-game and its close path left `paused` set.
Applied here: we have many screens (title, quiz, adventure, reveal). Tag the mode
explicitly in state, gate all content on it, funnel every dismissal through one
handler that restores ALL global flags. Use the `meta.phase` field as the
single source of truth for which screen is active.

`[gamedev][state][process]` One NaN silently poisons everything downstream of
shared state. Applied here: OCEAN scores are 0–100 integers that feed into every
Claude prompt. `applyDeltas()` must validate with `isFinite` and clamp — a single
undefined delta would corrupt the entire personality model silently.

`[gamedev][state][process]` When a source-of-truth object is replaced at runtime,
rebuild ALL derived/mirrored state through ONE shared clear+respawn path. Applied
here: when importing a share code, don't merge piecemeal — replace the full state
object atomically and re-derive the screen from it.

## Game Design

`[gamedev][state][gamedesign]` Run real-time combat over a turn-based authoritative
reducer by making "retaliation" an opt-out flag plus a separate command. Applied
here: our deferred damage mechanic works the same way — damage is a pending command,
intercept is a separate command that resolves it. Don't bake intercept logic into
the damage path.

`[gamedev][gamedesign][tooling]` Making content data-driven opts you out of
compile-time safety — a typo'd id ships an uncompletable quest with no error. Applied
here: quiz questions reference OCEAN trait names. The `CORE_QUESTIONS` array and all
Claude response schemas should validate trait names against a single canonical list:
`['openness', 'conscientiousness', 'extraversion', 'agreeableness', 'emotionalStability']`.

`[gamedesign]` A multiplier meant to help should be clamped so it can't become a
penalty at the other end of the range. Applied here: profile deltas from Claude
must be clamped so a large negative delta can't push a score below 0 or above 100.
Double-clamp: validate Claude's response AND clamp in `applyDeltas()`.

`[input][coop]` If two systems read the same input edge, one fires when you don't
want it — gate by context or give it its own button. Applied here: the action input
field serves multiple purposes (normal action, interpret partner, intercept damage).
Each should be a distinct UI path, not a magic keyword in the same text box.

## Narrative & Audio

`[narrative][structure]` Events-in-a-row plus polish is a chronicle; a story needs
want/opposition/stakes/choice. Applied here: the adventure engine prompt must
generate scenes with CONFLICT STRUCTURE — both characters' desires in tension with
an external force — not just "here's a room with a thing in it." The scene-start
prompt should explicitly require: what the characters want, what opposes them,
what's at stake, and what choice they face.

`[tts][voice]` Sync visuals to the TTS engine's real speaking state, not an
estimated reading time — estimates drift per device/voice. Applied here: if we ever
add narration audio, never setTimeout-based timing. (Not in scope now, but don't
build a visual timing system that would need to be torn out.)

## Process & Tooling

`[process][harness][git]` If a push to an external remote is refused no matter the
auth, it's the harness/environment blocking it — hand the user a script to push
locally instead of retrying.

`[process][observability]` Instrument the lowest-level primitive first (does the
click even dispatch?). Applied here: if an API call fails, log the raw request/
response before debugging the game logic above it. The `api.call()` wrapper should
have a debug mode that dumps payloads to console.

`[browser][tooling][process]` Build-step-free static ES modules: a "logic bug"
was actually stale module caching. Applied here: this is a single HTML file with
no build step. If we ever split into modules or use importmap, cache-bust with a
version token. For now, single-file avoids this entirely — don't split.

`[process][tooling]` A greedy number lexer + parseFloat silently truncates malformed
numbers. Applied here: when parsing Claude's JSON responses, use strict JSON.parse —
never regex extraction or parseFloat on substrings. If JSON.parse fails, the response
is bad; don't salvage partial data.

## Share Code Specific

These are new constraints specific to our share-code architecture:

- Share codes are compressed full state. A corrupted or truncated paste must fail
  loudly (show "Invalid code" with the parse error), never silently load partial state.
- The API key NEVER enters the share code. Strip it in `export()`, restore from
  localStorage in `import()`. Verify with a test: export → decode → assert no apiKey field.
- Schema version in `meta.version`. If the imported code's version doesn't match the
  running game's version, show a clear error ("Your partner has a newer version of the
  game — refresh your page"). Don't attempt migration in a single-file game.
