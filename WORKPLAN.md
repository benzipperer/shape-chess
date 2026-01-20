# Shape Chess - Implementation Workplan

## Overview

Shape Chess (形棋/Xingqi) is a two-player abstract strategy game played on a 12×12 (or larger) grid with black and white stones. Players score points by creating **symmetric shapes** of 6 or more connected stones, which are then removed from the board. The first player to reach 4 points wins.

---

## 1. Game Rules Summary

### 1.1 Core Concepts

- **Shape**: A stone plus all same-colored stones reachable via orthogonal OR diagonal steps (8-connectivity)
- **Symmetric Shape**: A shape that is preserved by reflection along ANY line (not just grid-aligned)
- **Scoring**: A symmetric shape of n ≥ 6 stones scores (n - 5) points and is removed from the board
- **Victory**: First player to reach 4 points wins

### 1.2 Turn Actions

A player must perform exactly ONE of these actions per turn:

| Action | Description | Validation |
|--------|-------------|------------|
| **Drop** | Place own stone on any empty point | Target must be empty |
| **Jump** | Move own stone from current position to any empty point | Source must have own stone; target must be empty |
| **Push** | Push opponent's stone to adjacent empty point, place own stone at origin | Opponent stone must exist at source; destination must be adjacent (8 directions) and empty |

### 1.3 Scoring Sequence

After each turn:
1. Detect all symmetric shapes of 6+ own-colored stones
2. If any exist: remove them, add (n - 5) points per shape
3. Player takes another turn (bonus turn)
4. Repeat until no scoring shapes exist or player wins

### 1.4 Key Rules Details

- **Black moves first**
- Push destinations are limited to 8 adjacent cells (king-move distance)
- Multiple symmetric shapes can score in one evaluation (each grants points, only one bonus turn)
- Symmetry can be along ANY axis (horizontal, vertical, diagonal, or any angle)

---

## 2. Technical Architecture

### 2.1 Technology Stack

```
┌─────────────────────────────────────────────────┐
│                    UI Layer                      │
│  HTML5 Canvas / SVG  |  CSS Grid  |  DOM Events │
├─────────────────────────────────────────────────┤
│                 Game Logic Layer                 │
│   Move Validation | Symmetry Detection | Score  │
├─────────────────────────────────────────────────┤
│                  State Layer                     │
│     Board State | Game History | Undo Stack     │
└─────────────────────────────────────────────────┘
```

### 2.2 File Structure

```
shape-chess/
├── index.html              # Main entry point
├── css/
│   ├── main.css            # Global styles
│   ├── board.css           # Board and stone styling
│   └── ui.css              # Controls, modals, feedback
├── js/
│   ├── main.js             # Entry point, initialization
│   ├── game/
│   │   ├── Game.js         # Main game controller
│   │   ├── Board.js        # Board state management
│   │   ├── Move.js         # Move representation and validation
│   │   └── Player.js       # Player state (score, color)
│   ├── logic/
│   │   ├── ShapeFinder.js  # Connected component detection
│   │   ├── SymmetryChecker.js  # Symmetry detection algorithm
│   │   └── MoveValidator.js    # Move legality checks
│   ├── ui/
│   │   ├── BoardRenderer.js    # Canvas/SVG rendering
│   │   ├── InputHandler.js     # Mouse/touch input
│   │   ├── FeedbackManager.js  # Error messages, highlights
│   │   └── GameUI.js           # Score display, turn indicator
│   └── utils/
│       ├── geometry.js     # Point operations, reflections
│       └── constants.js    # Board size, colors, etc.
└── assets/
    └── sounds/             # Optional: move sounds
```

---

## 3. Data Structures

### 3.1 Board Representation

```javascript
// Board state: 2D array
// Values: null (empty), 'black', 'white'
board[row][col] = null | 'black' | 'white'

// Alternative: flat array with index = row * BOARD_SIZE + col
```

### 3.2 Position/Point

```javascript
class Point {
  constructor(row, col) {
    this.row = row;
    this.col = col;
  }

  equals(other) { ... }
  neighbors8() { ... }  // 8-connectivity neighbors
  isValid(boardSize) { ... }
}
```

### 3.3 Shape Representation

```javascript
class Shape {
  constructor(stones, color) {
    this.stones = Set<Point>;  // All points in shape
    this.color = 'black' | 'white';
    this.size = stones.size;
  }

  isSymmetric() { ... }
  getBoundingBox() { ... }
  getCentroid() { ... }
}
```

### 3.4 Move Representation

```javascript
class Move {
  constructor(type, params) {
    this.type = 'drop' | 'jump' | 'push';
    this.player = 'black' | 'white';

    // For drop: { target: Point }
    // For jump: { source: Point, target: Point }
    // For push: { opponentStone: Point, pushDirection: Point, newStonePos: Point }
    this.params = params;
  }
}
```

### 3.5 Game State

```javascript
class GameState {
  board: Board;
  currentPlayer: 'black' | 'white';
  scores: { black: number, white: number };
  moveHistory: Move[];
  isBonusTurn: boolean;
  gameOver: boolean;
  winner: 'black' | 'white' | null;
}
```

---

## 4. Core Algorithms

### 4.1 Shape Detection (Connected Components)

**Algorithm**: Flood-fill / Union-Find with 8-connectivity

```
FindAllShapes(board, color):
  visited = empty set
  shapes = []

  for each cell (r, c) in board:
    if board[r][c] == color AND (r,c) not in visited:
      shape = BFS/DFS from (r,c) using 8-connectivity
      add all found cells to visited
      shapes.append(new Shape(shape, color))

  return shapes
```

**Complexity**: O(n²) where n = board size

### 4.2 Symmetry Detection

This is the most complex algorithm. A shape is symmetric if it's preserved by reflection along **any** line.

**Approach**: Check multiple potential axes of symmetry

```
IsSymmetric(shape):
  points = shape.stones as array of (x, y) coordinates
  centroid = average of all points

  // Translate points to center at origin
  centered = points.map(p => p - centroid)

  // Check symmetry along many candidate axes
  // 1. Check horizontal axis (y = 0)
  // 2. Check vertical axis (x = 0)
  // 3. Check diagonal axes (y = x, y = -x)
  // 4. Check axes at other angles (0°, 15°, 30°, 45°, ... 180°)

  for angle in [0, 15, 30, 45, 60, 75, 90, 105, 120, 135, 150, 165]:
    if CheckReflectionSymmetry(centered, angle):
      return true

  return false

CheckReflectionSymmetry(points, angle_degrees):
  // Reflect each point across line through origin at given angle
  // Check if reflected point set equals original point set

  reflected = points.map(p => ReflectAcrossLine(p, angle))
  return SetsEqual(points, reflected, tolerance=0.5)
```

**Key Insight**: Since stones are on grid points, the axis of symmetry can pass through:
- Grid points (integer coordinates)
- Midpoints between grid points (half-integer coordinates)

**Reflection Formula** for point (x, y) across line at angle θ through origin:
```
x' = x·cos(2θ) + y·sin(2θ)
y' = x·sin(2θ) - y·cos(2θ)
```

**Optimization**:
- Only check angles that could produce valid grid reflections
- For small shapes, can enumerate all possible symmetry axes
- Cache results for unchanged shapes

### 4.3 Move Validation

```
ValidateDrop(board, player, target):
  if not target.isValid(boardSize): return INVALID_POSITION
  if board[target] != null: return CELL_OCCUPIED
  return VALID

ValidateJump(board, player, source, target):
  if not source.isValid(boardSize): return INVALID_SOURCE
  if not target.isValid(boardSize): return INVALID_TARGET
  if board[source] != player: return NOT_YOUR_STONE
  if board[target] != null: return CELL_OCCUPIED
  if source.equals(target): return SAME_POSITION
  return VALID

ValidatePush(board, player, opponentPos, pushDir):
  opponent = oppositeColor(player)
  if not opponentPos.isValid(boardSize): return INVALID_POSITION
  if board[opponentPos] != opponent: return NOT_OPPONENT_STONE

  destination = opponentPos + pushDir
  if not isAdjacent(opponentPos, destination): return NOT_ADJACENT
  if not destination.isValid(boardSize): return OFF_BOARD
  if board[destination] != null: return DESTINATION_OCCUPIED
  return VALID
```

### 4.4 Scoring Evaluation

```
EvaluateScoring(board, player):
  shapes = FindAllShapes(board, player)
  scoringShapes = shapes.filter(s => s.size >= 6 AND s.isSymmetric())

  if scoringShapes.isEmpty():
    return { scored: false, points: 0, removedStones: [] }

  totalPoints = sum(scoringShapes.map(s => s.size - 5))
  allRemovedStones = union(scoringShapes.map(s => s.stones))

  return {
    scored: true,
    points: totalPoints,
    removedStones: allRemovedStones,
    shapes: scoringShapes  // For highlighting
  }
```

---

## 5. User Interface Design

### 5.1 Layout

```
┌────────────────────────────────────────────────────────┐
│  SHAPE CHESS                          [Rules] [Reset] │
├──────────────────────┬─────────────────────────────────┤
│                      │  Current Turn: ● BLACK          │
│                      │                                 │
│                      │  ┌─────────────────────────┐   │
│                      │  │ Action:                 │   │
│                      │  │ ○ Drop   ○ Jump  ○ Push │   │
│      GAME BOARD      │  └─────────────────────────┘   │
│       12 × 12        │                                 │
│                      │  Score:                         │
│                      │  ● Black: 0 / 4                 │
│                      │  ○ White: 0 / 4                 │
│                      │                                 │
│                      │  ┌─────────────────────────┐   │
│                      │  │ [Undo] [Pass]           │   │
│                      │  └─────────────────────────┘   │
├──────────────────────┴─────────────────────────────────┤
│  Status: Click to place a black stone (Drop mode)      │
└────────────────────────────────────────────────────────┘
```

### 5.2 Board Rendering

- **Grid**: Light background with subtle grid lines
- **Stones**: Circular, with slight gradient/shadow for 3D effect
  - Black stones: Dark gray to black gradient
  - White stones: White with gray border
- **Hover Effect**: Semi-transparent stone preview on valid cells
- **Selection Highlight**: Glowing border for selected stone (Jump/Push)
- **Last Move Indicator**: Small marker on last placed/moved stone

### 5.3 Visual Feedback

| State | Visual Indicator |
|-------|------------------|
| Valid drop target | Green highlight on hover |
| Invalid target | Red highlight + cursor change |
| Selected stone (Jump) | Pulsing blue border |
| Opponent stone (Push) | Yellow highlight when pushable |
| Push direction | Arrows showing valid push directions |
| Scoring shape | Flash animation before removal |
| Symmetry axis | Optional: show detected axis briefly |

### 5.4 Interaction Flow

**Drop Mode:**
1. Player selects "Drop" action (default)
2. Hover over board shows stone preview on empty cells
3. Click empty cell → place stone

**Jump Mode:**
1. Player selects "Jump" action
2. Click own stone to select (highlights it)
3. Click empty cell for destination
4. ESC or click elsewhere to cancel selection

**Push Mode:**
1. Player selects "Push" action
2. Opponent stones highlight as "pushable"
3. Click opponent stone to select
4. Adjacent empty cells highlight with direction arrows
5. Click destination → push executes

### 5.5 Error Messages

Display clear, actionable error messages:

| Error | Message |
|-------|---------|
| Drop on occupied | "That cell is occupied. Choose an empty cell." |
| Jump from empty | "Select one of your stones to jump." |
| Jump wrong color | "That's not your stone." |
| Jump to occupied | "The destination must be empty." |
| Push own stone | "You can only push opponent stones." |
| Push blocked | "Can't push there - the cell is occupied." |
| Push off board | "Can't push stone off the board." |
| Push non-adjacent | "Push destination must be adjacent." |

---

## 6. Game Flow

### 6.1 State Machine

```
                    ┌──────────────┐
                    │  GAME_START  │
                    └──────┬───────┘
                           │
                           ▼
    ┌───────────────────────────────────────────┐
    │              AWAITING_MOVE                │◄────┐
    │  (Display current player, wait for input) │     │
    └───────────────────┬───────────────────────┘     │
                        │ valid move                   │
                        ▼                              │
    ┌───────────────────────────────────────────┐     │
    │            EXECUTING_MOVE                 │     │
    │  (Apply move to board, animate)           │     │
    └───────────────────┬───────────────────────┘     │
                        │                              │
                        ▼                              │
    ┌───────────────────────────────────────────┐     │
    │          EVALUATING_SCORE                 │     │
    │  (Check for symmetric 6+ shapes)          │     │
    └───────────┬───────────────────┬───────────┘     │
                │ no scoring        │ scoring          │
                │                   ▼                  │
                │   ┌───────────────────────────┐     │
                │   │     SCORING_ANIMATION     │     │
                │   │  (Highlight, remove, add  │     │
                │   │   points, check win)      │     │
                │   └─────────────┬─────────────┘     │
                │                 │                    │
                │     ┌───────────┴───────────┐       │
                │     │ not won       │ won           │
                │     │               ▼               │
                │     │   ┌───────────────────┐       │
                │     │   │    GAME_OVER      │       │
                │     │   │ (Display winner)  │       │
                │     │   └───────────────────┘       │
                │     │                               │
                │     │ bonus turn                    │
                │     └───────────────────────────────┘
                │
                │ switch player                        │
                └──────────────────────────────────────┘
```

### 6.2 Turn Sequence Pseudocode

```
ExecuteTurn(move):
  // 1. Validate move
  validationResult = ValidateMove(move)
  if not validationResult.valid:
    ShowError(validationResult.message)
    return

  // 2. Apply move to board
  ApplyMove(move)
  AddToHistory(move)

  // 3. Animate move
  await AnimateMove(move)

  // 4. Check for scoring
  loop:
    scoring = EvaluateScoring(board, currentPlayer)
    if not scoring.scored:
      break

    // 5. Animate and remove scoring shapes
    await AnimateScoring(scoring.shapes)
    RemoveStones(scoring.removedStones)
    AddScore(currentPlayer, scoring.points)

    // 6. Check win condition
    if scores[currentPlayer] >= 4:
      EndGame(currentPlayer)
      return

    // 7. Bonus turn - don't switch player, wait for next move
    ShowMessage("Bonus turn! You scored " + scoring.points + " points.")
    return  // Stay on same player's turn

  // 8. Switch player
  SwitchPlayer()
```

---

## 7. Features Breakdown

### 7.1 MVP (Minimum Viable Product)

1. **Board Display**
   - Render 12×12 grid
   - Display black and white stones
   - Show coordinates (optional)

2. **Basic Moves**
   - Implement Drop action
   - Implement Jump action
   - Implement Push action
   - Move validation with error messages

3. **Shape Detection**
   - Find connected components (8-connectivity)
   - Basic symmetry detection (horizontal, vertical, diagonal axes)

4. **Scoring System**
   - Detect scoring shapes (6+ symmetric)
   - Remove scored shapes
   - Update and display scores
   - Implement bonus turns

5. **Game Flow**
   - Turn management (Black first)
   - Win detection (4 points)
   - Game over screen

6. **Basic UI**
   - Action selection (Drop/Jump/Push)
   - Current player indicator
   - Score display
   - Status messages

### 7.2 Enhanced Features

1. **Improved Symmetry Detection**
   - Full arbitrary-axis symmetry checking
   - Visual indication of symmetry axis

2. **Visual Polish**
   - Smooth animations for moves
   - Scoring shape highlight animation
   - Stone placement animation
   - Responsive design for different screen sizes

3. **Quality of Life**
   - Undo/Redo functionality
   - Move history display
   - Keyboard shortcuts
   - Last move indicator

4. **Game Management**
   - New game / Reset
   - Configurable board size (12×12 to 19×19)
   - Configurable win threshold
   - Rules reference / Help modal

### 7.3 Advanced Features (Future)

1. **Game Analysis**
   - Highlight potential scoring moves
   - Show all current shapes and their sizes
   - Threat detection (opponent can score next turn)

2. **Persistence**
   - Save/Load game state (localStorage)
   - Export game as notation
   - Import game from notation

3. **Multiplayer**
   - Pass-and-play mode (current)
   - Local network play (WebRTC)
   - Online multiplayer (would require backend)

4. **AI Opponent**
   - Basic AI (random legal moves)
   - Intermediate AI (prioritize scoring, block opponent)
   - Advanced AI (search-based)

5. **Tutorials & Puzzles**
   - Interactive tutorial
   - Puzzle mode (find the scoring move)
   - Problem positions from the PDF

---

## 8. Symmetry Detection Deep Dive

### 8.1 Mathematical Foundation

A shape S is **symmetric** if there exists a line L such that reflecting every point in S across L produces S itself.

For a discrete set of grid points, we must:
1. Find the centroid of the shape
2. Test reflection across lines through (or near) the centroid
3. Account for the fact that reflected points may not land exactly on grid points

### 8.2 Candidate Axes

For a shape centered at origin, test these axis angles:
- **0°** (horizontal): reflects (x, y) → (x, -y)
- **90°** (vertical): reflects (x, y) → (-x, y)
- **45°** (diagonal): reflects (x, y) → (y, x)
- **135°** (anti-diagonal): reflects (x, y) → (-y, -x)
- **Additional angles**: 22.5°, 67.5°, 112.5°, 157.5° for half-grid axes

### 8.3 Implementation Strategy

```javascript
function isSymmetric(shape) {
  const points = [...shape.stones];
  const n = points.length;

  // Calculate centroid
  let cx = 0, cy = 0;
  for (const p of points) {
    cx += p.col;
    cy += p.row;
  }
  cx /= n;
  cy /= n;

  // Translate to origin
  const centered = points.map(p => ({
    x: p.col - cx,
    y: p.row - cy
  }));

  // Test multiple axes (in radians)
  const axes = [0, Math.PI/8, Math.PI/4, 3*Math.PI/8,
                Math.PI/2, 5*Math.PI/8, 3*Math.PI/4, 7*Math.PI/8];

  for (const theta of axes) {
    if (checkReflectionSymmetry(centered, theta)) {
      return true;
    }
  }

  return false;
}

function checkReflectionSymmetry(points, theta) {
  const reflected = points.map(p => reflectPoint(p, theta));
  return setsMatchWithTolerance(points, reflected, 0.01);
}

function reflectPoint(p, theta) {
  // Reflect (x, y) across line through origin at angle theta
  const cos2t = Math.cos(2 * theta);
  const sin2t = Math.sin(2 * theta);
  return {
    x: p.x * cos2t + p.y * sin2t,
    y: p.x * sin2t - p.y * cos2t
  };
}

function setsMatchWithTolerance(set1, set2, tolerance) {
  if (set1.length !== set2.length) return false;

  // For each point in set2, find a matching point in set1
  const used = new Set();
  for (const p2 of set2) {
    let found = false;
    for (let i = 0; i < set1.length; i++) {
      if (used.has(i)) continue;
      const p1 = set1[i];
      const dist = Math.hypot(p1.x - p2.x, p1.y - p2.y);
      if (dist < tolerance) {
        used.add(i);
        found = true;
        break;
      }
    }
    if (!found) return false;
  }
  return true;
}
```

### 8.4 Edge Cases

- **Single stone**: Not scorable (size < 6)
- **Two stones**: Always symmetric (line through both)
- **Three+ stones in a line**: Always symmetric
- **Large irregular shapes**: Likely asymmetric
- **Shapes touching board edge**: Handle normally

---

## 9. Move Validation Details

### 9.1 Drop Validation

```javascript
function validateDrop(board, player, target) {
  if (!isOnBoard(target)) {
    return { valid: false, message: "Position is outside the board." };
  }
  if (board.get(target) !== null) {
    return { valid: false, message: "That cell is already occupied." };
  }
  return { valid: true };
}
```

### 9.2 Jump Validation

```javascript
function validateJump(board, player, source, target) {
  if (!isOnBoard(source)) {
    return { valid: false, message: "Invalid source position." };
  }
  if (!isOnBoard(target)) {
    return { valid: false, message: "Invalid target position." };
  }
  if (board.get(source) !== player) {
    return { valid: false, message: "You must select one of your own stones." };
  }
  if (board.get(target) !== null) {
    return { valid: false, message: "The destination cell must be empty." };
  }
  if (source.equals(target)) {
    return { valid: false, message: "Source and destination must be different." };
  }
  return { valid: true };
}
```

### 9.3 Push Validation

```javascript
function validatePush(board, player, opponentPos, destination) {
  const opponent = otherPlayer(player);

  if (!isOnBoard(opponentPos)) {
    return { valid: false, message: "Invalid position." };
  }
  if (board.get(opponentPos) !== opponent) {
    return { valid: false, message: "You must select an opponent's stone to push." };
  }
  if (!isOnBoard(destination)) {
    return { valid: false, message: "Cannot push stone off the board." };
  }
  if (!isAdjacent(opponentPos, destination)) {
    return { valid: false, message: "Push destination must be adjacent (including diagonals)." };
  }
  if (board.get(destination) !== null) {
    return { valid: false, message: "Push destination must be empty." };
  }
  return { valid: true };
}

function isAdjacent(p1, p2) {
  const dr = Math.abs(p1.row - p2.row);
  const dc = Math.abs(p1.col - p2.col);
  return dr <= 1 && dc <= 1 && (dr + dc > 0);
}
```

---

## 10. Testing Strategy

### 10.1 Unit Tests

**Shape Detection:**
- Single stone → 1 shape of size 1
- Two adjacent stones → 1 shape of size 2
- Two separated stones → 2 shapes of size 1
- Diagonal connection counts → verify 8-connectivity
- Complex multi-shape board → correct count and sizes

**Symmetry Detection:**
- Horizontal line of 6 → symmetric
- Vertical line of 6 → symmetric
- 2×3 rectangle → symmetric (multiple axes)
- L-shape → asymmetric
- Plus sign (+) → symmetric
- All 22 shapes of size 4 from PDF → verify symmetric/asymmetric classification

**Move Validation:**
- Drop on empty → valid
- Drop on occupied → invalid
- Jump own stone → valid
- Jump opponent stone → invalid
- Jump to occupied → invalid
- Push opponent to adjacent empty → valid
- Push opponent to non-adjacent → invalid
- Push own stone → invalid
- Push to occupied → invalid

**Scoring:**
- 6-stone symmetric shape → 1 point, removed
- 7-stone symmetric shape → 2 points, removed
- 5-stone symmetric shape → no score
- 6-stone asymmetric shape → no score
- Multiple scoring shapes → all score, all removed
- Score reaches 4 → game over

### 10.2 Integration Tests

- Full game flow: moves → scoring → bonus turns → win
- Undo/redo maintains consistent state
- UI reflects game state accurately

### 10.3 Manual Test Scenarios

Use positions from the PDF:
- Problem 1-3: Find scoring moves
- Problem 4-7: Defensive scenarios
- Problem 8-10: Multi-turn sequences
- Problem 11-13: Mate-in-N problems

---

## 11. Implementation Phases

### Phase 1: Foundation
- Project setup (HTML, CSS, JS structure)
- Board data structure and rendering
- Stone placement (visual only)
- Basic styling

### Phase 2: Core Moves
- Implement Drop with validation
- Implement Jump with validation
- Implement Push with validation
- Action mode selection UI
- Error feedback system

### Phase 3: Shape Logic
- Connected component detection
- Shape class implementation
- Basic symmetry detection (axis-aligned)
- Shape visualization/debugging tools

### Phase 4: Scoring System
- Scoring evaluation after moves
- Stone removal animation
- Score tracking and display
- Bonus turn logic

### Phase 5: Game Management
- Turn switching
- Win detection
- Game over state
- New game / Reset

### Phase 6: Polish
- Full symmetry detection (arbitrary axes)
- Animations and transitions
- Responsive design
- Keyboard shortcuts
- Undo functionality

### Phase 7: Testing & Refinement
- Comprehensive testing
- Bug fixes
- Performance optimization
- User feedback incorporation

---

## 12. Open Questions & Decisions

### 12.1 Design Decisions Needed

1. **Board Size**: Default 12×12, allow configuration?
2. **Coordinate Display**: Show row/column labels?
3. **Stone Style**: Flat circles vs 3D spheres?
4. **Animation Speed**: Fast vs deliberate for clarity?
5. **Mobile Support**: Touch interface priority?

### 12.2 Rule Clarifications

1. **Multiple symmetric shapes**: Rules say they all score with one bonus turn. Confirm this interpretation.
2. **Push off board**: The rules don't explicitly forbid this. Assumed invalid based on standard interpretations.
3. **Pass option**: Not mentioned in rules. Assume no passing allowed.

### 12.3 Technical Decisions

1. **Rendering**: Canvas vs SVG vs CSS Grid
   - Recommendation: SVG for crisp scaling and easy event handling
2. **State Management**: Plain objects vs class instances
   - Recommendation: Class-based for encapsulation
3. **Symmetry Precision**: Float tolerance for coordinate comparison
   - Recommendation: 0.01 tolerance after normalizing to unit grid

---

## Appendix A: Notation System

For recording games:

```
d5      = Drop at column d, row 5
e2-g8   = Jump from e2 to g8
f6:e5   = Push opponent at f6 to e5 (own stone placed at f6)
(2)     = Scored 2 points

Example game fragment:
1. d6       d7
2. e6       e7
3. f6       f7
4. g6:f7    d8
5. c6       g7-e8
6. b6       (diagram) ...
```

---

## Appendix B: Reference Positions

### Known Symmetric Shapes (Size 6)

```
●●●●●●     (horizontal line)

●
●
●          (vertical line)
●
●
●

●●●
●●●        (2×3 rectangle)

  ●
 ●●●       (T-shape, horizontal axis)
  ●

●   ●
 ● ●       (X-shape)
  ●
 ● ●
●   ●
```

### Known Asymmetric Shapes (Size 6)

```
●●●●
    ●●     (L-shape extended)

●●
 ●●
  ●●       (staircase)
```

---

This workplan provides a comprehensive blueprint for implementing Shape Chess. The most challenging aspects are the symmetry detection algorithm and creating an intuitive UI for the three different move types. The phased approach allows for iterative development with a playable game early in the process.
