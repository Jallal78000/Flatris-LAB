# Copilot Instructions for Flatris-LAB

## Big Picture

**Flatris** is a real-time multiplayer Tetris game with client-side React/Redux UI and a Node.js/Express server using WebSocket (Socket.IO) for live game synchronization.

### Architecture

- **`web/`** — React + Redux frontend (Next.js), served server-side rendered; handles game UI, player input, and local state
- **`server/`** — Express backend with Socket.IO; manages game sessions, player connections, action broadcasting, and state snapshots
- **`shared/`** — Isomorphic game logic and data structures (actions, reducers, Tetromino pieces); shared between client and server
- **`startup/`** — Server entry points and initialization
- **`scripts/`** — Build and utility scripts

### Key Data Flow

1. **Initial Load**: Client requests game → server returns snapshot + server-rendered HTML
2. **Hydration**: Client fetches JS, hydrates static HTML, opens WebSocket connection
3. **Action Sync**: Player input → local Redux action → emit to server → server broadcasts to opponent
4. **State Reconciliation**: Server maintains short-term action pool for backfill if client reconnects mid-game

### Critical Pattern: Action-Based Sync

- **Actions** are deterministic events (e.g., `MOVE_LEFT`, `DROP_PIECE`)
- Broadcast **intent**, not outcome (e.g., "move left" not "new position")
- Actions link to previous action (`prevActionId`) to validate consistency and prevent gaps
- See `multiplayer.md` for action queuing, backfill, and conflict resolution logic

## Developer Workflows

### Setup & Local Dev

```bash
# Install dependencies
yarn install

# Start dev server + client
yarn dev:server &
yarn dev:client

# Or watch mode (hot reload)
yarn dev:server:watch
```

### Build & Test

```bash
# Run all tests (Flow type-check → lint → Jest unit tests)
yarn test

# Watch mode for tests
yarn test:watch

# Lint only
yarn lint

# Format code
yarn format

# Build for production (Next.js build)
yarn build

# Start production server
yarn start
```

### Debugging

- **React DevTools**: Client-side React state inspection
- **Redux DevTools**: View dispatched actions and state diffs (enabled in dev via `redux-devtools-extension`)
- **WebSocket**: Monitor Socket.IO events in browser DevTools under Network
- **Server logs**: Check terminal output from `yarn dev:server:watch`

## Project Conventions & Patterns

### State & Actions

- Actions live in `shared/` and are dispatched to Redux store
- Reducers are pure functions; actions contain only serializable data
- Socket.IO emits actions via `dispatch` events on client and server listeners
- Game state is reduced from initial snapshot + all subsequent actions

### File Structure

- Game logic: `shared/logic/` (move, drop, line clearing, block transfer)
- React components: `web/components/`, `web/pages/`
- Server handlers: `server/routes/`, `server/socket/`
- Styles: co-located CSS modules or `web/styles/`

### Styling

- CSS Modules (`.css` files next to components)
- Global styles in `web/styles/`
- Follow Prettier for formatting

### Error Handling

- Flow types (static analysis) catch type errors at build time
- ESLint enforces code quality (React hooks, import order, unused variables)
- Jest tests validate reducer logic and component rendering
- Socket.IO errors logged to server; client-side errors sent to Rollbar (error reporting service)

## Integration Points

### Socket.IO Communication

- Client connects on hydration: `socket.emit('JOIN_GAME', { playerId, gameId })`
- Server listens for `dispatch` events (actions from client)
- Server broadcasts actions to all players: `io.to(gameId).emit('dispatch', action)`
- **Backfill**: On reconnect, client may request missing actions via `socket.emit('BACKFILL', { lastActionId })`

### Firebase Admin

- Used for authentication/user management (check `server/auth/` or references in startup)
- Credentials loaded from environment

### Deployment

- **PM2** (see `ecosystem.config.js`): Process manager for production
- **Travis CI** (`.travis.yml`): Run tests on push
- Game state is ephemeral; no persistent DB (stateless game sessions)

## Patterns to Copy

### Adding a New Game Action

1. Define action type in `shared/actions/types.js`
2. Create action creator in `shared/actions/index.js` (e.g., `export const moveLeft = () => ({ type: MOVE_LEFT })`)
3. Update reducer in `shared/reducers/game.js` to handle the action
4. Emit from client in `web/components/GameBoard.js` or input handler (dispatch → socket.emit)
5. Add tests in `shared/__tests__/` and `web/__tests__/`

### Adding a Server Route

1. Create handler in `server/routes/api.js` or new module
2. Register route in `server/index.js` (Express setup)
3. Return JSON; handle errors with appropriate status codes
4. Test via `curl` or Jest integration tests

### Multiplayer Logic: Block Transfer

- When a player clears lines → action: `TRANSFER_BLOCKS_TO_OPPONENT`
- Reducer marks transferred block rows, excludes the Tetromino that just fell
- Both clients reduce the same action; deterministic outcome prevents de-sync
- Reference: `shared/logic/transferBlocks.js` and `multiplayer.md` section on "Transferring partial lines"

## External Dependencies & Credentials

- **Socket.IO v2.2**: Real-time bidirectional communication
- **Next.js v7**: Server-side rendering (SSR) and React framework
- **Redux**: State container; synced with server actions
- **Babel**: Transpile ES6+ (Flow types, JSX)
- **Flow v0.89**: Optional static typing; run `yarn test` to type-check

## What Not to Change

- Do not modify game constants (block shapes, grid size) without updating both client and server logic
- Do not add server-side state mutation without understanding action history and backfill logic
- Do not change action payload structure without a migration strategy (old clients may send old formats)
- Do not disable Socket.IO heartbeat or timeout without testing reconnection scenarios

## PR Checklist for AI Agents

- [ ] Run `yarn test` and confirm all checks pass (Flow, Lint, Jest)
- [ ] If adding an action: Update `shared/`, add reducer logic, add Jest test in `shared/__tests__/`
- [ ] If modifying game balance: Test multiplayer scenarios (block transfer, line clearing, grid overflow)
- [ ] If touching server: Test Socket.IO connection, backfill on reconnect, multiple concurrent games
- [ ] Keep action payloads minimal (determinism + network efficiency)
- [ ] Update comments in `multiplayer.md` if game mechanics change

---

**Questions for clarification?** Ask about action IDs, backfill logic, or how to test WebSocket scenarios.
