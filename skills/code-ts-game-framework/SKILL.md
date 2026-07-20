---
name: code-ts-game-framework
description: Build or extend simple browser-based TypeScript games for a trusted group of friends using the current Phantom Ink architecture. Use when creating a game in code/, or implementing DEBUG_ID-isolated localStorage identities, player profiles, rooms, lobbies, rejoining, server persistence and message passing, consensus voting, shared reducers, responsive Svelte UI, XState game flows, or live debug helpers and routes.
---

# Build TypeScript Multiplayer Games

Use Phantom Ink as the canonical starting point for every pattern in this skill, including the Svelte UI. Copy its proven flow before inventing abstractions.

The links in this skill pin the inspected revision so line references remain stable:

- [Phantom Ink `1398bd4`](https://github.com/smirea/phantom-ink/tree/1398bd43841fd9e7533f0106041169cb4cab9e8b) — canonical SvelteKit, oRPC, SQLite action log, rooms, profiles, voting, and XState.

Before copying, inspect the repository's current default branch for improvements made after this revision. Read the linked files and their imports rather than relying only on the excerpts here.

## Keep it simple

Optimize for a trusted friend group, one Bun server, one SQLite file, and easy local development.

- Send the full game state to every client. Hide secret information in the UI for gameplay; do not build per-user redaction.
- Use the local `userId` as identity. Do not add authentication, permissions, signed sessions, or account recovery.
- Keep one shared reducer, one command path, and one event stream.
- Avoid brokers, queues, microservices, generic framework layers, and multi-process support until a real need appears.
- Prefer a small duplicated helper over a premature abstraction.

## Default architecture

For a room-based game, begin with Phantom Ink's workspace shape:

```text
apps/ui              SvelteKit SPA and route-level orchestration
apps/server          Bun HTTP server, persistence, and streaming
packages/shared      serializable types, reducers, rules, RPC contract
```

See the [workspace scripts and package layout](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/package.json#L1-L38) and [shared package exports](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/package.json#L1-L22).

Keep these boundaries:

- Put authoritative, framework-free room and game transitions in `packages/shared` so the server, optimistic client, replay, and tests use the same code.
- Let the server check game rules, append accepted actions, derive the next state, and broadcast snapshots.
- Let the UI render the shared snapshot and send semantic actions. Do not let components mutate authoritative state directly.
- Store only identity pointers and UI preferences in browser storage. Do not store the authoritative room or game snapshot there.

## Copy the localStorage and DEBUG_ID flow

Copy these Phantom Ink files as one unit, then rename only the game namespace and storage shape:

1. [`createLocalStorage.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/createLocalStorage.ts#L1-L166) — typed JSON values, defaults, corrupt-value cleanup, memory fallback, namespacing, reset, and change subscriptions.
2. [`debugId.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/debugId.ts#L1-L30) — accepts `DEBUG_ID` and `debug_id`, including a debug ID nested in a safe `returnTo` URL.
3. [`storage.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/storage.ts#L1-L16) — creates the one application storage instance.
4. [Root-layout debug navigation](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L340-L378) — preserves the debug ID through links, redirects, setup, and `returnTo`.

Use the same construction:

```ts
const DEBUG_ID = typeof location === 'undefined' ? null : debugIdFromSearch(location.search);
const namespace = `my-game${DEBUG_ID ? `-DEBUG_ID=${DEBUG_ID}` : ''}`;

export const LS = new LocalStorage<{
	userId: number | null;
	darkMode: boolean;
}>({
	namespace,
	getDefaults: () => ({ userId: null, darkMode: true }),
});
```

`?DEBUG_ID=alice` and `?DEBUG_ID=bob` produce separate keys such as `my-game-DEBUG_ID=alice:userId` and `my-game-DEBUG_ID=bob:userId`. This is how two tabs in the same browser become two independent users. The query parameter must survive every internal navigation; otherwise a tab silently falls back to the unscoped identity.

Use local storage like this:

```ts
const userId = LS.get('userId');
LS.set({ userId: savedUser.id });
LS.delete('userId');
```

Do not read or write `window.localStorage` elsewhere. Do not use `BroadcastChannel`, `postMessage`, or the browser `storage` event for multiplayer synchronization. DEBUG_ID separates local identities; the server stream synchronizes players.

Use `reset()` to clear only this game's namespace. Reserve `clearAll()` for explicit debug tooling because it clears the entire backing store, including unrelated namespaces.

Verify the flow by completing account setup in two tabs, one with `?DEBUG_ID=alice` and one with `?DEBUG_ID=bob`; navigate each tab through setup, lobby, and a room; reload both; confirm that each retains a different server user ID.

## Implement account setup

Model the account as a small server record with `id`, `name`, `color`, and `icon`. Treat it as a convenient player identity, not authentication.

Follow Phantom Ink's flow:

- Define color and icon IDs as closed preset unions in [`playerProfile.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/playerProfile.ts#L1-L35). Store preset IDs, not CSS colors or component names.
- Define the persisted user schema and public columns in [`db/schema.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/db/schema.ts#L1-L18).
- Keep shared sanitizers and player/member shapes in [`onlineGame.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L31-L175). Sanitize again on the server.
- Create an unsaved local profile with a random icon and color, save it through the API, then persist only the returned ID. See the [root app context and save flow](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L29-L76) and [server save logic](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L282-L325).
- Route users without a saved profile through setup while preserving the intended destination. See the [profile gate](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L187-L248).
- Present name, icon choices, color choices, and randomization on one focused screen. Copy the behavior from [`setup/+page.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/setup/%2Bpage.svelte#L1-L204).
- Render identity everywhere through one [`Avatar`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/Avatar.svelte#L1-L47) and one icon/color mapping layer rather than duplicating presentation logic.

Keep the in-memory app user reactive so profile edits update the UI immediately. Debounce background saves, but explicitly await the first save before entering routes that need a profile.

## Build the room and lobby lifecycle

Use four-letter uppercase room codes unless the game has a reason to need something else. Parse them in the client with [`roomCodes.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/roomCodes.ts#L1-L7) and validate them again on the server.

Represent a room as serializable state plus semantic actions. Phantom Ink's canonical definitions are in [`onlineGame.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L37-L153):

```ts
interface OnlineRoomState {
	v: number;
	phase: 'lobby' | 'playing';
	members: RoomMember[];
	votes: RoomVote[];
	gameState: GameState | null;
}

type OnlineRoomAction =
	| { type: 'join'; actorId: string; userId: number; name: string; color?: string; icon?: string }
	| { type: 'leave'; actorId: string }
	| { type: 'set-seat'; actorId: string; team: Team; role: PlayerRole }
	| { type: 'vote'; actorId: string; vote: RoomVoteAction }
	| { type: 'game-action'; actorId: string; action: GameEvent };
```

Drive this lifecycle:

1. Require a saved profile before lobby or room routes.
2. List open rooms and online users in the lobby. Phantom Ink polls the directory and presence every 2.5 seconds in [`lobby/+page.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/lobby/%2Bpage.svelte#L37-L111).
3. Navigate room creation through `/room/new`; create on mount; replace the temporary URL with the returned code. Join existing rooms through `/room/[code]`. See the [room connection flow](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L72-L148).
4. Render an optimistic room immediately using [`optimisticRoom.svelte.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/optimisticRoom.svelte.ts#L1-L32), then replace it with the server snapshot.
5. Let lobby members select seats, teams, roles, or settings. Clear readiness when a change invalidates prior consensus.
6. Start only when shared validation passes and the ready vote reaches consensus. Add players who join an active game as spectators.
7. Keep one current room per account by leaving other rooms on join. Use presence pings to prune abandoned lobby rooms; do not destroy an active game merely because a client briefly disconnects. See [server presence and room creation/joining](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L57-L156).

Return a compact `RoomViewState`, not raw persistence rows. Include connection status, the viewer's player ID, snapshot version, members, vote summaries, phase, the full game state, and any `startProblem`. Derive it with a shared selector as in [`selectRoomViewState`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L229-L253).

### Rejoin the current room

Intend a returning player to resume their existing room and game without re-entering a code.

1. Load the stored `userId`.
2. Call `users.currentRoom`; keep the server as the source of truth instead of storing a room snapshot locally. See the [RPC contract](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/rpc.ts#L24-L41) and [server lookup](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L66-L70).
3. If it returns a code, navigate to `/room/[code]` while preserving `DEBUG_ID`.
4. Let the room route call `rooms.join` again and reopen `rooms.events`. Join is idempotent for an existing member, and the returned snapshot restores the current lobby or game phase.
5. If no room remains, continue to the lobby. An explicit abandon action sends `leave` before navigating away, as in the [root layout](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L275-L291).

A refresh on `/room/[code]` follows the same join-and-stream path directly. Joining a different room automatically leaves the previous one through [`leaveOtherRooms`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L490-L509). Active games remain available for rejoining; presence cleanup only removes inactive lobby membership.

## Use server storage and message passing

Use this flow for a single-process hobby deployment:

```text
UI event
  -> typed OnlineRoomAction RPC
  -> server actor check and shared reducer
  -> append accepted action to SQLite
  -> update in-memory snapshot cache
  -> broadcast room snapshots
  -> streamed event updates every connected UI
```

The canonical pieces are:

- Define one typed oRPC contract for user, presence, room command, and room event endpoints in [`rpc.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/rpc.ts#L1-L46).
- Create the browser client in [`api.ts`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/api.ts#L1-L9).
- Send mutations through `rooms.action`; identify both the server user and the expected `actorId`:

```ts
await api.rooms.action({
	code,
	userId: user.id,
	action: { type: 'leave', actorId: playerIdForUser(user.id) },
});
```

- Consume `rooms.events` as a long-lived event iterator and replace the local snapshot on every event. See the [room client stream](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L82-L133).
- Persist `users`, `rooms`, and append-only `room_actions` in SQLite through Drizzle. See the [schema](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/db/schema.ts#L1-L47).
- Rebuild a room by replaying actions through `applyOnlineRoomAction`; cache the result; clone and reduce before accepting a new action; append only changed actions; then broadcast. Copy [`loadRoomState` and `appendRoomAction`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L432-L488).
- Keep per-room client queues and heartbeats in memory, as in [`streamRoom` and `broadcastRoom`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/server/src/index.ts#L518-L582).

Treat the shared reducer as the consistency boundary. Keep only the checks needed to prevent accidental illegal moves: wrong actor, phase, role, or game state. The server response and stream are authoritative; optimistic client reduction is only a latency improvement, as shown by Phantom Ink's [pending game-action replay](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L279-L316).

This in-memory broadcast map assumes one server process. SQLite makes state durable, but it does not fan out messages across replicas. Add a shared pub/sub layer only if the deployment actually becomes multi-process.

## Implement voting at two levels

Keep lobby consensus separate from in-game decisions even if both reuse the same UI.

### Room votes

Represent each vote as `{ playerId, action }`. Define voter eligibility, threshold, and whether the namespace is a single toggle or an exclusive choice in shared code. Phantom Ink uses:

- unanimous non-spectator votes for `ready`;
- a simple majority for `word-mode:standard` versus `word-mode:custom`;
- toggle-to-remove behavior;
- one vote per player in a choice namespace;
- derived summaries containing eligible, voting, missing, required, and consensus values.

Copy the mechanics from [`applyVote`, `finalizeVotes`, and vote summaries](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L367-L532). Send a vote as a normal room action:

```ts
await api.rooms.action({
	code,
	userId: user.id,
	action: {
		type: 'vote',
		actorId: player.id,
		vote: { type: 'ready' },
	},
});
```

Never let a client-supplied count or `consensus` boolean decide the result. Recalculate eligibility and thresholds from authoritative room state. Clear ready votes when membership, seating, or a setting that affects the game changes.

### Game votes

Use a declarative configuration keyed by semantic vote type. Each entry specifies the active machine state, eligible role, selection count, valid choices, and event produced by consensus. See Phantom Ink's [`votingConfig`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L72-L149).

```ts
const votingConfig = {
	mediumAction: {
		activeState: 'mediumsTurn',
		mode: 'activeMedium',
		count: 1,
		choices: () => ['ask', 'guess'],
		event: (_context, [choice]) =>
			choice === 'ask' || choice === 'guess' ? { type: choice } : null,
	},
} satisfies Record<VoteType, VoteConfig>;
```

Record a vote inside the XState machine, remove it on a second click, and raise the resulting semantic game event only when every eligible player has selected the same required option set. Copy [`recordVote`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L220-L236) and the [`canVote`, `toggleVote`, and consensus helpers](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L416-L580).

Expose `{ voted, eligible, required }` to UI controls. Reuse [`VoteBadge.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/VoteBadge.svelte#L1-L259), the voting props in [`InkButton.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/InkButton.svelte#L1-L92), and the selected/voted states in [`GameCard.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/lib/GameCard.svelte#L1-L205). Show who voted and who is missing, not only a number.

## Model game rules with XState

Use XState for the deterministic game flow, not for HTTP, room membership, or component-local animation state.

Define:

- a string-literal `GameState` union;
- a fully serializable `GameContext`;
- a discriminated `GameEvent` union;
- named actions for context changes;
- guards for rule constraints;
- `entry` actions and `always` transitions for automatic progress;
- terminal states with no outgoing gameplay transitions.

Follow [`gameMachine`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L151-L342). Keep random choices deterministic by storing a seed or the chosen result in the accepted action/state.

Persist only a plain `{ state, context }`, not a live actor. For every accepted game action, reconstruct an actor from that persisted snapshot, send one event, copy the next snapshot, and stop the actor. Phantom Ink's wrapper is the canonical implementation:

```ts
export function applyGameAction(state: PersistedGameState, event: GameEvent): boolean {
	const before = JSON.stringify(state);
	const actor = createActor(gameMachine, { snapshot: persistedSnapshot(state) });
	actor.start();
	actor.send(event);
	const next = actor.getSnapshot();
	state.state = next.value as GameState;
	state.context = structuredClone(next.context);
	actor.stop();
	return before !== JSON.stringify(state);
}
```

Copy the actual [`createInitialGameState`, apply wrapper, and persisted snapshot adapter](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L635-L690). This makes the XState machine compatible with SQLite action replay, optimistic client replay, and framework-free tests.

Wrap game events inside room actions. Check the actor's role and event identity before handing the event to XState, as in [`canApplyGameAction`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L327-L345). XState guards enforce game rules; the room reducer enforces which network user may attempt them.

## Add two debug surfaces

Keep debugging small: one global live-room API and one local debug route.

### Expose live-room helpers

Expose `window.DEBUG` while the room route is mounted. Follow the console convention in Phantom Ink's [README](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/README.md#L40-L51) and include:

```ts
window.DEBUG.getState();          // cloned game state
window.DEBUG.saveState();         // save to a dedicated debug localStorage key
await window.DEBUG.loadState();   // load the saved state
await window.DEBUG.loadState(x);  // load a state, { state }, or JSON
await window.DEBUG.setState(x);   // alias for loadState
window.DEBUG.getRoom();           // cloned room view
await window.DEBUG.resetRoom();   // return the room to its lobby
```

Treat the saved debug snapshot as the one exception to the rule against browser-stored game state. Keep it in the existing namespaced `LocalStorage`, so each `DEBUG_ID` gets its own snapshot.

Load state through a debug room action rather than changing one client's state:

```ts
| { type: 'load-state'; actorId: PlayerId; state: PhantomInkGameState }
```

Parse a plain state, `{ state }`, or JSON; clone it into `gameState`; set the phase to `playing`; append the action; and broadcast the room snapshot. Implement reset with the existing [`reset-room` action](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/onlineGame.ts#L327-L337). Remove the global functions when the room route unmounts so they never retain stale room closures.

### Build `/debug/game`

Use a local debug route to exercise the real game UI without creating accounts, rooms, or server state. Copy Phantom Ink's [`debug/game/+page.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/debug/game/%2Bpage.svelte#L1-L145):

- Run the real `gameMachine` in a local actor and render the real `GameScreen`.
- Provide baked-in mock players with distinct names, icons, and colors.
- Switch `viewerId` to act as each player and inspect every player's perspective.
- Select any machine state and active team through debug-only events. See the machine's [`debugSetState` and `debugSetTeam`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/packages/shared/src/game.ts#L240-L258).
- Add a few named baked scenarios by resetting the actor and sending deterministic setup events. Cover the states that otherwise require several players or many turns to reach.
- Keep the route local and disposable. Use real `DEBUG_ID` tabs and `window.DEBUG` when testing networking or shared live-room behavior.

## Reuse the UI skeleton

Build the product flow in this order:

```text
/                 branded start screen
/setup            name, icon, and color
/lobby            create room, room directory, online players
/room/new         optimistic room while the server allocates a code
/room/[code]      room setup while phase=lobby; game UI while phase=playing
```

Use the root layout as the application shell. Phantom Ink's [`+layout.svelte`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/%2Blayout.svelte#L1-L137) owns account context, theme, route gating, presence, persistent DEBUG_ID navigation, room code/header, profile controls, and the content shell. Keep those cross-route concerns out of individual screens.

In the room route, switch on the server phase:

```svelte
{#if room?.phase === 'playing' && gameState}
	<GameScreen
		game={gameState.context}
		state={gameState.state}
		players={players}
		viewerId={user.id}
		send={sendGameEvent}
	/>
{:else}
	<RoomSetup {members} {votes} />
{/if}
```

See the actual [room lobby/game split](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/room/%5Bcode%5D/%2Bpage.svelte#L319-L420). Keep `GameScreen` driven by `state`, `context`, `viewerId`, player presentation data, and one `send` callback. Derive what the UI displays and enables from those inputs; the full context may remain client-side.

Reuse component roles, not Phantom Ink's theme:

- `Avatar` for player identity.
- `InkButton` or an equivalent button primitive for loading, disabled, primary, and vote states.
- `GameCard` for selectable/revealed/redacted game objects.
- `VoteBadge` for voter avatars and missing-voter details.
- `ErrorBox` for request/stream errors.
- A tokenized theme file and one responsive shell. See [`theme.css`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/theme.css#L1-L72) and the core [`layout.css`](https://github.com/smirea/phantom-ink/blob/1398bd43841fd9e7533f0106041169cb4cab9e8b/apps/ui/src/routes/layout.css#L20-L138).

Keep mobile-height constraints, reduced-motion behavior, keyboard focus, `aria-pressed`/`aria-checked`, and disabled/loading states from the beginning. Multiplayer UI must distinguish waiting on the server, waiting on other players, and being ineligible to act.

## Implementation order

1. Read project instructions and inspect any existing game code before introducing this structure.
2. Copy the local storage, DEBUG_ID, profile, and navigation flow; prove two same-browser identities work.
3. Define serializable shared user, room, action, viewer-state, and game types.
4. Implement and test pure room reduction before wiring HTTP.
5. Add SQLite action persistence, typed commands, streaming snapshots, and basic actor checks.
6. Build start, setup, lobby, and room routes, including room rejoining.
7. Implement the XState game machine and its plain persisted wrapper.
8. Add room votes and game votes in shared code, then attach vote-aware UI primitives.
9. Add `window.DEBUG` and `/debug/game`; bake in representative states and player perspectives.
10. Test create, join, rejoin, leave, invalid action, spectator join, consensus reset, game start, state load/reset, and refresh with multiple DEBUG_ID tabs.
11. Run the repository's typecheck, tests, lint, and production build. Prefer meaningful reducer and state-machine tests over trivial component tests.
