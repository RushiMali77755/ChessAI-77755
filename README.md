# ChessAI (Desktop-ready MVP)

## Project Structure

- `backend/` Spring Boot (Socket.IO server, Ehcache session state, Stockfish integration)
- `frontend/` Angular (chessboard.js UI + chess.js validation + Socket.IO client)

## Prerequisites

- Java 17+
- Node.js 18+
- Stockfish engine installed
  - Windows: download `stockfish.exe` and either:
    - add it to PATH as `stockfish`, or
    - set `stockfish.path` in `backend/src/main/resources/application.yml`
- Maven
  - Maven Wrapper is included under `backend/`.

## Run Backend (Spring Boot)

1. Configure Stockfish path (if needed)

   - Edit `backend/src/main/resources/application.yml`
   - Set:
     - `stockfish.path: C:/path/to/stockfish.exe`

2. Start the Socket.IO backend

   - From `backend/`:
     - `mvnw.cmd spring-boot:run`

3. Ports

- Socket.IO server: `http://localhost:9092`

## Run Frontend (Angular)

1. From `frontend/`:

- `npm install`
- `npm start`

2. Open:

- `http://localhost:4200`

## Socket.IO Events

### Client -> Server

#### `start_game`
**Purpose**: Initialize a new chess game session with specified difficulty  
**Payload**:
```json
{
  "sessionId": "uuid-string",
  "difficultyLevel": 1
}
```
**Behavior**:
- Creates new `GameState` in Ehcache with `sessionId` as key
- Resets chess board to starting position
- Sets difficulty level (1-5) for AI opponent
- Emits `board_update` with initial FEN position

---

#### `make_move`
**Purpose**: Submit a player move for validation and processing  
**Payload**:
```json
{
  "sessionId": "uuid-string",
  "move": "e2e4"
}
```
**Behavior**:
- Validates UCI format move against current board state using chesslib
- Rejects illegal moves with `error` event
- On valid move: updates game state, emits `board_update`, triggers AI thinking
- AI response comes via `ai_move` event after Stockfish calculation

---

#### `reset_game`
**Purpose**: Reset current session to initial state  
**Payload**:
```json
{
  "sessionId": "uuid-string"
}
```
**Behavior**:
- Clears move history
- Resets board to starting position
- Emits `board_update` with initial FEN

---

### Server -> Client

#### `board_update`
**Purpose**: Broadcast current game state to clients  
**Payload**:
```json
{
  "sessionId": "uuid-string",
  "fen": "rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1",
  "moves": ["e2e4"],
  "status": "ONGOING"
}
```
**Behavior**:
- Sent after any successful move (player or AI)
- Client updates chessboard.js position with provided FEN
- `status` can be: `ONGOING`, `CHECKMATE`, `STALEMATE`, `DRAW`

---

#### `ai_move`
**Purpose**: Notify client of AI's chosen move  
**Payload**:
```json
{
  "sessionId": "uuid-string",
  "move": "e7e5"
}
```
**Behavior**:
- Sent after Stockfish completes calculation
- Client should wait for subsequent `board_update` with new FEN

---

#### `game_over`
**Purpose**: Notify that game has ended  
**Payload**:
```json
{
  "sessionId": "uuid-string",
  "status": "CHECKMATE"
}
```
**Behavior**:
- Sent when chesslib detects game-ending condition
- Disables further moves on client-side

---

#### `error`
**Purpose**: Report validation or processing failures  
**Payload**:
```json
{
  "where": "make_move",
  "message": "Illegal move: e2e9"
}
```
**Behavior**:
- Client displays error message and may snap back illegal moves

---

## Problems / Known Issues : Not Yet any

**Investigation steps attempted**:
1. Added explicit height/min-height to `.board` CSS (520px)
2. Copied piece images from `node_modules/chessboardjs` to `src/assets/img/chesspieces/wikipedia/`
3. Added `pieceTheme` config pointing to `/assets/img/chesspieces/wikipedia/{piece}.png`
4. Added console logging to `ngAfterViewInit` and `initBoard`

**Possible causes**:
- `ViewChild('board')` element reference may be undefined at initialization time
- Chessboard.js may fail silently if jQuery is not loaded when component initializes
- Piece image paths may not resolve correctly in dev server
- Angular's change detection may not trigger after board creation

**Next debug steps**:
1. Open browser console (F12) and check for:
   - `ngAfterViewInit - boardEl:` log output
   - Any 404 errors for `.png` piece images
   - JavaScript errors from chessboard.js
2. Check Elements tab for `.board` div - should contain chessboard HTML after load
3. Verify piece images accessible at: `http://localhost:7070/assets/img/chesspieces/wikipedia/wP.png`

---

## Validation Scenarios

- Valid move -> backend accepts -> Stockfish returns best move -> UI updates
- Invalid move -> frontend snaps back; backend also rejects if sent
- Reset -> game state resets in Ehcache for the same `sessionId`
- Difficulty 1..5 -> maps to Stockfish `Skill Level` + `go movetime` as specified
