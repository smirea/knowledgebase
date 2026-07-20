---
name: code-ts-game-framework
description: Build or extend simple browser-based TypeScript games for a trusted group of friends using the current Phantom Ink architecture. Use when implementing shared game rules and data, XState flows, a Svelte game UI, DEBUG_ID-isolated localStorage identities, player profiles, rooms, lobbies, rejoining, server-owned state and message passing, voting, or game debug tooling.
---

# Build TypeScript Multiplayer Games

To avoid re-solving project structure and integration details, use [Phantom Ink `1398bd4`](https://github.com/smirea/phantom-ink/tree/1398bd43841fd9e7533f0106041169cb4cab9e8b) as the canonical implementation, including for Svelte. Inspect the current default branch for later improvements, then read the linked files and their imports before copying a pattern.

## Favor simplicity

Optimize for a known group of friends, one Bun server, one SQLite file, and easy local development.

- Send the full game state to every client. Hide secret information in the UI for gameplay; do not build per-user state redaction.
- Use a local `userId` as identity. Do not add authentication, permissions, signed sessions, or account recovery.
- Use one action path, one shared reducer, and one room event stream.
- Keep basic checks for accidental invalid actions, but do not design for hostile clients.
- Avoid brokers, microservices, generic framework layers, and multi-process support until a real need appears.

Start with Phantom Ink's [workspace layout](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/package.json#L1-L38):

```text
apps/ui              SvelteKit SPA and route orchestration
apps/server          Bun HTTP server, persistence, and room streams
packages/shared      serializable game rules, actions, types, and data
```

## Keep every client in sync

To make clients converge after actions, refreshes, and reconnects, let the server own and distribute authoritative room and game state. Let clients render the latest snapshot and send semantic actions such as `join`, `vote`, `pickWord`, or `guessLetter`. Never make a UI component the authority for whether an action is legal.

To reuse the same decisions in the server, optimistic UI, replay, debug route, and tests, keep all framework-free, deterministic logic in `packages/shared`:

- Define game flow, context, events, guards, and voting rules in [`game.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L72-L342).
- Define rooms, members, room actions, room voting, and the game-action wrapper in [`onlineGame.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L31-L175).
- Define game content and configuration in files like [`data.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/data.ts#L1-L193). This is an example of domain data, not a requirement for cards or assets. A different game may share prompts, boards, questions, lookup tables, or configuration instead. Keep visual and audio assets in the UI when they are needed.

Keep transient concerns such as open menus, animations, and pending form input in the UI. Make shared state serializable and avoid browser, Svelte, database, or server APIs in shared game logic.

## Isolate gameplay from networking

To iterate on rules and UI without multiplayer setup, implement the full game locally in `/debug/game` before adding accounts, rooms, persistence, or networking. Still design for networking from the start: use serializable state and events, deterministic transitions, explicit viewer identity, and a single `send(event)` boundary.

Follow [`debug/game/+page.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/debug/game/%2Bpage.svelte#L1-L145):

- Run the real XState machine and render the real `GameScreen`.
- Provide baked-in players with distinct names, icons, and colors.
- Switch the viewer to verify every player's perspective.
- Jump to hard-to-reach phases with debug-only events such as [`debugSetState` and `debugSetTeam`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L240-L258).
- Add a few named, deterministic scenarios for states that otherwise require several players or turns.

Do not begin networking until a full game can be played start-to-finish in this route and every role, major phase, and end state can be inspected.

## Make game flow explicit with XState

To keep a multi-phase game understandable and replayable, use XState for game flow, not for HTTP, room membership, or component-local UI state. Define a serializable context, a discriminated event union, named actions, guards, automatic transitions, and terminal states in one shared machine. See [`gameMachine`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L151-L342).

Persist plain `{ state, context }`, never a live actor. Reconstruct an actor from the persisted snapshot, send one accepted event, copy out the next snapshot, and stop it. Copy the [`createInitialGameState` and persisted-state wrapper](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L635-L690). This keeps server replay, optimistic client replay, debug scenarios, and tests on the same rules.

Store random results or a seed in accepted state so replay stays deterministic.

## Support multiple local users with DEBUG_ID

To test multiple players in one browser while preserving each identity across refreshes, namespace localStorage with `DEBUG_ID`. Copy these files together, changing only the application namespace and storage shape:

- [`createLocalStorage.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/createLocalStorage.ts#L1-L166) provides typed JSON values, defaults, cleanup, fallback, namespacing, reset, and subscriptions.
- [`debugId.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/debugId.ts#L1-L30) reads `DEBUG_ID`, `debug_id`, and a debug ID nested in `returnTo`.
- [`storage.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/storage.ts#L1-L16) creates the single application storage instance.
- [Root-layout navigation helpers](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L340-L378) preserve the debug ID through links, redirects, setup, and `returnTo`.

```ts
const DEBUG_ID = typeof location === 'undefined' ? null : debugIdFromSearch(location.search);
const namespace = `my-game${DEBUG_ID ? `-DEBUG_ID=${DEBUG_ID}` : ''}`;

export const LS = new LocalStorage({ namespace, getDefaults });
```

Use `?DEBUG_ID=alice` and `?DEBUG_ID=bob` to create independent users in two tabs of the same browser. Preserve that query parameter through every internal navigation. If it disappears, the tab silently falls back to the unscoped identity.

Store only identity pointers and UI preferences, plus an explicit debug snapshot. Never use raw `window.localStorage` elsewhere, and do not use browser storage events for multiplayer synchronization. Use `reset()` for this game's namespace; reserve `clearAll()` for explicit debug tooling.

## Make players recognizable

To make players easy to recognize with minimal setup, represent an account as a server record with `id`, `name`, `color`, and `icon`. Treat it as player presentation, not authentication.

- Define closed color and icon presets in [`playerProfile.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/playerProfile.ts#L1-L35).
- Randomize an unsaved local profile, save it through the API, then persist only the returned user ID. Follow the [root app/profile flow](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L29-L76).
- Gate room routes until the first save completes, preserving the intended destination. See the [profile gate](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L187-L248).
- Keep setup to one screen for name, icon, color, and randomization. See [`setup/+page.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/setup/%2Bpage.svelte#L1-L204).
- Render identity consistently through one [`Avatar`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/Avatar.svelte#L1-L47).

## Keep networking simple

For a trusted game with few players and infrequent actions, use one server process with this state flow:

```text
client action -> typed RPC -> server validation + shared reducer
              -> SQLite action log -> broadcast snapshot -> clients
```

Define the contract in [`rpc.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/rpc.ts#L1-L46), create the UI client in [`api.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/api.ts#L1-L9), and copy the server's [action replay and append path](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L432-L488) plus [per-room stream and broadcast](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L518-L582).

Let clients send actions, not replacement state. Treat server snapshots as authoritative. Optionally replay pending actions locally for responsiveness, as in the [room route](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L279-L316), but always converge on the next server snapshot.

Keep the room lifecycle small:

- Create or join by a short room code; keep one current room per user.
- Use a lobby phase for membership, seats or roles, settings, and readiness.
- Start only when shared validation and voting pass; add late joiners as spectators when appropriate.
- Render a compact shared `RoomViewState`, not persistence rows. See [`selectRoomViewState`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L229-L253).
- Use the [lobby route](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/lobby/%2Bpage.svelte#L37-L111) for room discovery and presence, and the [room route](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L72-L148) for join and streaming.

Assume one server process. SQLite makes actions durable; the in-memory room stream distributes snapshots only inside that process. Add shared pub/sub only if multi-process deployment becomes real.

### Resume rooms after refresh

To let a returning player continue without re-entering a code, load `userId` on startup, ask the server for `users.currentRoom`, and navigate to `/room/[code]` while preserving `DEBUG_ID`. Let the room route idempotently join again and reopen the event stream. A direct refresh follows the same path. If no room remains, continue to the lobby; send `leave` only for an explicit abandon action.

Use the [current-room RPC](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/rpc.ts#L24-L41), [server room lookup](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L66-L70), and [`leaveOtherRooms`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L490-L509). Do not store the current room snapshot in localStorage.

## Make group decisions consistent and visible

To keep every client aligned on who can vote and when consensus is reached, derive voting from shared authoritative state. Keep lobby consensus separate from in-game decisions, even when both use the same presentation components.

- For room votes, derive eligible voters and thresholds from authoritative room state. Use unanimous readiness and majority settings only as defaults; clear stale readiness when membership or relevant settings change. Copy the shared mechanics in [`onlineGame.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L367-L532).
- For game votes, define active machine state, eligible role, allowed choices, required selections, and the semantic event raised on consensus in one shared configuration. See [`votingConfig`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L72-L149) and the [vote/consensus helpers](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L416-L580).
- Show who voted and who is missing through a shared component such as [`VoteBadge`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/VoteBadge.svelte#L1-L259).

Derive vote counts and consensus from current shared state rather than accepting a client's result.

## Reproduce live room state quickly

To inspect, save, restore, or reset a live multiplayer game without replaying turns, expose a small `window.DEBUG` API while the room route is mounted:

```ts
window.DEBUG.getState();
window.DEBUG.saveState();
await window.DEBUG.loadState();
await window.DEBUG.loadState(stateOrJson);
await window.DEBUG.setState(stateOrJson);
window.DEBUG.getRoom();
await window.DEBUG.resetRoom();
```

Follow the [documented console convention](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/README.md#L40-L51). Save snapshots in the existing DEBUG_ID-namespaced storage. Load state through a debug room action so the server appends and broadcasts it; never mutate only one client. Reset through the existing [`reset-room` action](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L327-L337). Remove the globals when leaving the route.

Use `/debug/game` for fast local gameplay and baked states. Use multiple live room tabs with distinct `DEBUG_ID` values plus `window.DEBUG` for networking, rejoin, voting, and synchronization checks.

## Keep route responsibilities clear

To keep setup, room orchestration, and gameplay easy to reason about, use this route sequence:

```text
/ -> /setup -> /lobby -> /room/new or /room/[code]
                              lobby phase -> playing phase
```

Let the [root layout](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L1-L137) own account context, route gating, presence, DEBUG_ID-aware navigation, room header, theme, and the content shell. Let the [room route](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L319-L420) switch between setup and the game screen from the server phase.

Keep `GameScreen` driven by plain state, context, players, `viewerId`, and one `send` callback. Reuse component roles rather than Phantom Ink's visual theme: player avatar, button, selectable game object, vote status, error display, and responsive shell. Clearly distinguish waiting for the server, waiting for other players, and being ineligible to act.

## Build in dependency order

1. Define shared serializable game state, events, rules, and domain data.
2. Build the real game UI and complete game loop in `/debug/game`; add viewer switching and representative baked states.
3. Add XState persistence and verify deterministic replay.
4. Copy profile storage and DEBUG_ID-preserving navigation; prove two same-browser identities survive reloads.
5. Add room state, lobby flow, SQLite action persistence, server snapshots, and rejoining.
6. Add room and game voting through shared actions.
7. Add live-room debug helpers and test create, join, refresh, rejoin, leave, invalid actions, spectators, consensus reset, state load, and room reset with multiple DEBUG_ID tabs.
8. Finish responsive and accessible UI states, then run typecheck, meaningful reducer/machine tests, lint, and a production build.
