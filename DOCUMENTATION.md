# Maze Game - Complete Source Code Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Enumerations](#enumerations)
3. [Data Models](#data-models)
4. [Tile System](#tile-system)
5. [Game Logic](#game-logic)
6. [Database Layer](#database-layer)
7. [User Interface](#user-interface)
8. [Class Relationships](#class-relationships)

---

## Architecture Overview

This is a **two-player cooperative maze game** built using Java Swing. The architecture follows a **Model-View-Controller (MVC)** pattern:

- **Model**: `MazeModel`, `Player`, `PlayerState`, `Team`, `Position`, `Tile` and its subclasses
- **View**: `MazeGameGUI`, `MazePanel`
- **Controller**: `GameController`, `RoundManager`

The game also implements:
- **Service Layer Pattern** for database operations (`ScoreService`, `ScoreServiceImpl`)
- **Strategy Pattern** for tile behavior (each tile type determines passability)
- **State Pattern** for game states (`GameState` enum)

### Game Flow
1. Players enter a team name
2. Play through 3 difficulty levels (Easy → Medium → Hard)
3. Each round generates a new maze
4. Both players must reach the goal before time runs out
5. Scores are saved to database if all rounds are completed

---

## Enumerations

### 1. **DifficultyLevel.java**
**Purpose**: Represents the three difficulty levels of the game.

```java
public enum DifficultyLevel {
    EASY,    // 10x10 maze
    MEDIUM,  // 15x15 maze
    HARD     // 25x25 maze
}
```

**Usage**: 
- Determines maze size in `MazeModel.setDifficulty()`
- Controls game progression in `GameController`
- Displayed in UI labels

---

### 2. **Direction.java**
**Purpose**: Represents the four possible movement directions for players.

```java
public enum Direction {
    UP,
    DOWN,
    LEFT,
    RIGHT
}
```

**Usage**:
- Passed from `MazeGameGUI` keyboard input to `GameController.onPlayerInput()`
- Used in `GameController.calculateNewPosition()` to compute new coordinates

---

### 3. **GameState.java**
**Purpose**: Represents the current state of the game application.

```java
public enum GameState {
    MENU,            // Main menu screen
    PLAYING,         // Active gameplay
    ROUND_COMPLETE,  // Round finished successfully
    GAME_OVER,       // Game ended (win or lose)
    SCOREBOARD       // Viewing high scores
}
```

**Usage**:
- Managed by `GameController.gameState`
- Checked in `GameController.onPlayerInput()` to prevent moves during non-playing states
- Determines which UI screen to show in `MazeGameGUI`

---

### 4. **PlayerId.java**
**Purpose**: Identifies which player (Player 1 or Player 2) is performing an action.

```java
public enum PlayerId {
    PLAYER_1,  // Blue player (WASD controls)
    PLAYER_2   // Red player (Arrow keys)
}
```

**Usage**:
- Keys in `Map<PlayerId, PlayerState>` in `RoundManager`
- Parameter in `Tile.canPlayerPass()` to determine if a player can pass through a tile
- Determines which colored walls each player can pass through

---

### 5. **TileType.java**
**Purpose**: Categorizes the different types of tiles in the maze.

```java
public enum TileType {
    FLOOR,          // Walkable space
    REGULAR_WALL,   // Impassable wall
    BLUE_WALL,      // Only Player 1 can pass
    RED_WALL,       // Only Player 2 can pass
    GOAL            // Win condition tile
}
```

**Usage**:
- Stored in `Tile.type`
- Determines rendering color in `MazePanel.drawTile()`
- Used in `Tile.canPlayerPass()` switch statement

---

### 6. **WallColor.java**
**Purpose**: Represents which colored wall a player is allowed to pass through.

```java
public enum WallColor {
    BLUE,  // Player 1's allowed wall color
    RED,   // Player 2's allowed wall color
    NONE   // No special wall permissions
}
```

**Usage**:
- Field in `Player.allowedWallColor`
- Used in `Player.canPassWall()` logic (though this method isn't actively used in current implementation)

---

## Data Models

### 7. **Position.java**
**Purpose**: Immutable value object representing a 2D coordinate in the maze.

#### Fields:
- `int x`: X-coordinate (column)
- `int y`: Y-coordinate (row)

#### Methods:

##### Constructor
```java
public Position(int x, int y)
```
Creates a position at the specified coordinates.

##### Getters/Setters
```java
public int getX()
public void setX(int x)
public int getY()
public void setY(int y)
```
Standard accessors for x and y coordinates.

##### `equals(Object obj)` - **CRITICAL FOR HASH MAP**
```java
@Override
public boolean equals(Object obj)
```
- Compares two positions for equality based on x and y coordinates
- **Why it matters**: `Position` is used as a key in `Map<Position, Tile>` in `MazeModel`
- Without proper `equals()`, the HashMap couldn't find tiles by position

##### `hashCode()` - **REQUIRED WITH EQUALS**
```java
@Override
public int hashCode()
```
- Returns: `31 * x + y`
- **Why it matters**: Java requires that objects used as HashMap keys have consistent `hashCode()` and `equals()` implementations
- The formula `31 * x + y` provides reasonable distribution of hash values

##### `toString()`
```java
@Override
public String toString()
```
- Returns: `"Position(x, y)"`
- Used for debugging and logging

**Connections**:
- Used in: `Tile`, `PlayerState`, `MazeModel`, `RoundManager`, `GameController`
- Key type in: `Map<Position, Tile>` in `MazeModel`

---

### 8. **Player.java**
**Purpose**: Represents a player entity with their identification and wall-passing abilities.

#### Fields:
- `PlayerId playerId`: Unique identifier (PLAYER_1 or PLAYER_2)
- `WallColor allowedWallColor`: Which colored walls this player can pass through

#### Methods:

##### Constructor
```java
public Player(PlayerId playerId, WallColor allowedWallColor)
```
Creates a player with the specified ID and wall permissions.

**Example usage** (from RoundManager):
```java
Player player1 = new Player(PlayerId.PLAYER_1, WallColor.BLUE);
Player player2 = new Player(PlayerId.PLAYER_2, WallColor.RED);
```

##### Getters/Setters
```java
public PlayerId getPlayerId()
public void setPlayerId(PlayerId playerId)
public WallColor getAllowedWallColor()
public void setAllowedWallColor(WallColor allowedWallColor)
```
Standard accessors.

##### `canPassWall(WallColor wall)`
```java
public boolean canPassWall(WallColor wall)
```
- **Parameters**: `wall` - The color of wall being checked
- **Returns**: `true` if player can pass this wall type
- **Logic**: Returns true if wall is NONE or matches player's allowed color
- **Note**: This method exists but isn't actively used in the current implementation. Wall-passing logic is handled in `Tile.canPlayerPass()` instead.

**Connections**:
- Contained in: `PlayerState`
- Created in: `RoundManager.startRound()` and `Team` constructor
- References: `PlayerId`, `WallColor`

---

### 9. **PlayerState.java**
**Purpose**: Tracks the current state of a player during a round (position, goal status).

#### Fields:
- `Player player`: The player entity
- `Position position`: Current location in the maze
- `boolean reachedGoal`: Whether this player has reached the goal

#### Methods:

##### Constructor
```java
public PlayerState(Player player, Position position)
```
- Initializes a player state at a starting position
- Sets `reachedGoal` to `false`

##### Getters/Setters
```java
public Player getPlayer()
public void setPlayer(Player player)
public Position getPosition()
public void setPosition(Position position)
public boolean isReachedGoal()
public void setReachedGoal(boolean reachedGoal)
```

##### `hasReachedGoal()` - **Convenience method**
```java
public boolean hasReachedGoal()
```
- Same as `isReachedGoal()`, provided for better readability
- Used in: `RoundManager.isRoundComplete()` to check if both players reached the goal

**Connections**:
- Value type in: `Map<PlayerId, PlayerState>` in `RoundManager`
- Updated by: `RoundManager.handleMove()`
- Read by: `MazePanel` for rendering player positions

---

### 10. **Team.java**
**Purpose**: Represents a team of two players playing together, tracking their total time across rounds.

#### Fields:
- `String teamName`: Name of the team (max 20 characters, must be unique)
- `Player player1`: Player 1 instance
- `Player player2`: Player 2 instance
- `long totalTime`: Accumulated time across all rounds (in milliseconds)

#### Methods:

##### Constructor
```java
public Team(String teamName)
```
- Creates a team with the given name
- Automatically creates Player 1 (BLUE wall permission) and Player 2 (RED wall permission)
- Initializes `totalTime` to 0

##### Getters/Setters
```java
public String getTeamName()
public void setTeamName(String teamName)
public Player getPlayer1()
public void setPlayer1(Player player1)
public Player getPlayer2()
public void setPlayer2(Player player2)
public long getTotalTime()
public void setTotalTime(long totalTime)
```

##### `addRoundTime(long millis)` - **Score Accumulation**
```java
public void addRoundTime(long millis)
```
- **Purpose**: Adds the time from a completed round to the team's total
- **Parameters**: `millis` - Time elapsed in the round (milliseconds)
- **Called by**: `GameController.onRoundComplete()` after each round
- **Example**: If Round 1 took 15 seconds and Round 2 took 20 seconds, `totalTime` becomes 35000ms

**Connections**:
- Created by: `GameController.startGame()`
- Used in: `GameController` to track team scores
- Name saved to database via `ScoreService.saveScore()`

---

### 11. **ScoreRecord.java**
**Purpose**: Data Transfer Object (DTO) representing a single high score entry.

#### Fields:
- `String teamName`: Name of the team
- `long totalTime`: Total completion time in milliseconds

#### Methods:

##### Constructor
```java
public ScoreRecord(String teamName, long totalTime)
```
Simple value object constructor.

##### Getters/Setters
```java
public String getTeamName()
public void setTeamName(String teamName)
public long getTotalTime()
public void setTotalTime(long totalTime)
```

**Connections**:
- Returned by: `ScoreService.getTopScores()` as a `List<ScoreRecord>`
- Displayed in: `MazeGameGUI.showScoreboard()`
- Created from: Database query results in `ScoreServiceImpl`

---

### 12. **RoundResult.java**
**Purpose**: Stores the outcome of a completed round.

#### Fields:
- `DifficultyLevel difficulty`: Which difficulty level this round was
- `long elapsedMillis`: How long the round took (milliseconds)
- `boolean completed`: Whether the round was successfully completed (both players reached goal)

#### Methods:

##### Constructor
```java
public RoundResult(DifficultyLevel difficulty, long elapsedMillis, boolean completed)
```

##### Getters/Setters
```java
public DifficultyLevel getDifficulty()
public void setDifficulty(DifficultyLevel difficulty)
public long getElapsedMillis()
public void setElapsedMillis(long elapsedMillis)
public boolean isCompleted()
public void setCompleted(boolean completed)
```

**Connections**:
- Created by: `RoundManager.getRoundResult()`
- Stored in: `GameController.roundResults` list
- Used in: `GameController.calculateTotalTime()` and `GameController.onRoundComplete()`

---

## Tile System

The tile system uses **inheritance** to model different maze elements.

### 13. **Tile.java** (Base Class)
**Purpose**: Abstract representation of a single cell in the maze. Subclasses define specific behaviors.

#### Fields:
- `Position position`: Where this tile is located
- `TileType type`: What kind of tile this is

#### Methods:

##### Constructor
```java
public Tile(Position position, TileType type)
```

##### Getters/Setters
```java
public Position getPosition()
public void setPosition(Position position)
public TileType getType()
public void setType(TileType type)
```

##### `canPlayerPass(PlayerId playerId)` - **POLYMORPHIC BEHAVIOR**
```java
public boolean canPlayerPass(PlayerId playerId)
```
- **Purpose**: Determines if a specific player can walk on this tile
- **Parameters**: `playerId` - Which player is attempting to pass
- **Returns**: `true` if the player can pass, `false` otherwise
- **Implementation**: Uses switch statement based on `type`:
  - `FLOOR`: Both players can pass (returns `true`)
  - `GOAL`: Both players can pass (returns `true`)
  - `REGULAR_WALL`: No one can pass (returns `false`)
  - `BLUE_WALL`: Only PLAYER_1 can pass
  - `RED_WALL`: Only PLAYER_2 can pass
  
- **Design Note**: Subclasses override this method to provide specific behavior

**Connections**:
- Base class for: `Floor`, `RegularWall`, `BlueWall`, `RedWall`
- Stored in: `Map<Position, Tile>` in `MazeModel`
- Used by: `MazeModel.isWalkable()` to validate moves

---

### 14. **Floor.java**
**Purpose**: Represents a walkable tile that both players can move through.

#### Inheritance:
```java
public class Floor extends Tile
```

#### Constructor:
```java
public Floor(Position position)
```
- Calls `super(position, TileType.FLOOR)`

#### Overridden Method:
```java
@Override
public boolean canPlayerPass(PlayerId playerId)
```
- Always returns `true` (both players can walk on floor)

**Rendered as**: White square in `MazePanel`

---

### 15. **RegularWall.java**
**Purpose**: Impassable wall that blocks all players.

#### Inheritance:
```java
public class RegularWall extends Tile
```

#### Constructor:
```java
public RegularWall(Position position)
```
- Calls `super(position, TileType.REGULAR_WALL)`

#### Overridden Method:
```java
@Override
public boolean canPlayerPass(PlayerId playerId)
```
- Always returns `false` (no one can pass)

**Rendered as**: Gray square in `MazePanel`

---

### 16. **BlueWall.java**
**Purpose**: Special wall that only Player 1 (Blue player) can pass through.

#### Inheritance:
```java
public class BlueWall extends Tile
```

#### Constructor:
```java
public BlueWall(Position position)
```
- Calls `super(position, TileType.BLUE_WALL)`

#### Overridden Method:
```java
@Override
public boolean canPlayerPass(PlayerId playerId)
```
- Returns `true` only if `playerId == PlayerId.PLAYER_1`
- Player 2 sees this as a wall, Player 1 can walk through it

**Rendered as**: Blue square in `MazePanel`

**Game Design Purpose**: Creates strategic cooperation - Player 1 can access areas Player 2 cannot

---

### 17. **RedWall.java**
**Purpose**: Special wall that only Player 2 (Red player) can pass through.

#### Inheritance:
```java
public class RedWall extends Tile
```

#### Constructor:
```java
public RedWall(Position position)
```
- Calls `super(position, TileType.RED_WALL)`

#### Overridden Method:
```java
@Override
public boolean canPlayerPass(PlayerId playerId)
```
- Returns `true` only if `playerId == PlayerId.PLAYER_2`
- Player 1 sees this as a wall, Player 2 can walk through it

**Rendered as**: Dark red square in `MazePanel`

**Game Design Purpose**: Complementary to BlueWall - creates areas only Player 2 can reach

---

## Game Logic

### 18. **GameTimer.java**
**Purpose**: Simple timer utility to track elapsed time during a round.

#### Fields:
- `long startTime`: Timestamp when timer started (milliseconds since epoch)
- `boolean isRunning`: Whether the timer is currently active

#### Methods:

##### Constructor
```java
public GameTimer()
```
- Initializes with `startTime = 0` and `isRunning = false`

##### Getters/Setters
```java
public long getStartTime()
public void setStartTime(long startTime)
public boolean isRunning()
public void setRunning(boolean running)
```

##### `start()` - **Begin Timing**
```java
public void start()
```
- Sets `startTime` to current system time: `System.currentTimeMillis()`
- Sets `isRunning` to `true`
- **Called by**: `RoundManager.startRound()`

##### `stop()` - **End Timing**
```java
public void stop()
```
- Sets `isRunning` to `false`
- Does NOT reset `startTime` (so elapsed time can still be calculated)
- **Called by**: `RoundManager.stopRound()`

##### `getElapsedTime()` - **Calculate Duration**
```java
public long getElapsedTime()
```
- **Returns**: Milliseconds since timer started
- **Formula**: `System.currentTimeMillis() - startTime`
- **Used by**: 
  - `MazeGameGUI.updateGameState()` to display countdown
  - `RoundResult` to record round duration

**Connections**:
- Owned by: `RoundManager`
- Updated by: `RoundManager.startRound()` and `stopRound()`
- Read by: `GameController` and `MazeGameGUI` for time display

---

### 19. **MazeModel.java**
**Purpose**: Represents the maze structure and generates random mazes using Depth-First Search algorithm.

#### Fields:
- `Map<Position, Tile> tiles`: The 2D maze grid stored as a HashMap (Position → Tile)
- `DifficultyLevel difficulty`: Current difficulty (affects size)
- `int width`: Maze width (columns)
- `int height`: Maze height (rows)
- `Position player1Start`: Where Player 1 spawns
- `Position player2Start`: Where Player 2 spawns
- `Position goalPosition`: The win condition location

#### Methods:

##### Constructor
```java
public MazeModel()
```
- Initializes empty maze
- Default to `EASY` difficulty (10x10)

##### Getters/Setters
```java
public Map<Position, Tile> getTiles()
public void setTiles(Map<Position, Tile> tiles)
public DifficultyLevel getDifficulty()
public int getWidth()
public int getHeight()
public Position getPlayer1Start()
public Position getPlayer2Start()
public Position getGoalPosition()
```

##### `setDifficulty(DifficultyLevel difficulty)` - **Size Configuration**
```java
public void setDifficulty(DifficultyLevel difficulty)
```
- Sets the difficulty and adjusts maze dimensions:
  - **EASY**: 10x10
  - **MEDIUM**: 15x15
  - **HARD**: 25x25
- **Called before**: `generateMaze()` to create appropriate sized maze

##### `getTile(int x, int y)` - **Tile Lookup**
```java
public Tile getTile(int x, int y)
```
- **Returns**: The tile at coordinates (x, y), or `null` if out of bounds
- **Uses**: HashMap lookup with `new Position(x, y)` as key
- **Called by**: `isWalkable()`, `MazePanel.paintComponent()`, `generateMaze()`

##### `isWalkable(PlayerId playerId, int x, int y)` - **Movement Validation**
```java
public boolean isWalkable(PlayerId playerId, int x, int y)
```
- **Purpose**: Checks if a player can move to a specific position
- **Returns**: `true` if the player can walk there, `false` otherwise
- **Logic**:
  1. Get the tile at (x, y)
  2. If tile is null (out of bounds), return `false`
  3. Delegate to `tile.canPlayerPass(playerId)`
- **Called by**: `RoundManager.handleMove()` before actually moving a player

##### `generateMaze()` - **MAZE GENERATION ALGORITHM** ⭐
```java
public void generateMaze()
```
This is the most complex method in the class. It generates a solvable maze using **Depth-First Search (DFS)** with backtracking.

**Step 1: Initialize all tiles as walls**
```java
for (int y = 0; y < height; y++) {
    for (int x = 0; x < width; x++) {
        tiles.put(new Position(x, y), new RegularWall(new Position(x, y)));
    }
}
```
Starts with a solid grid of walls.

**Step 2: Carve paths using DFS**
```java
Stack<Position> stack = new Stack<>();
Position start = new Position(1, 1);
tiles.put(start, new Floor(start));
stack.push(start);
```
- Begins at (1, 1) - the top-left interior position
- Uses a stack to track the path

**Step 3: DFS Loop**
```java
while (!stack.isEmpty()) {
    Position current = stack.peek();
    List<Integer> directions = new ArrayList<>();
    
    // Check all 4 directions (2 cells away)
    for (int i = 0; i < 4; i++) {
        int nx = current.getX() + dx[i];
        int ny = current.getY() + dy[i];
        
        // If neighbor is valid and still a wall
        if (nx > 0 && nx < width - 1 && ny > 0 && ny < height - 1) {
            Tile tile = getTile(nx, ny);
            if (tile instanceof RegularWall) {
                directions.add(i);
            }
        }
    }
```
- Looks 2 cells away in each direction (to leave walls between paths)
- Finds unvisited neighbors (still walls)

**Step 4: Carve or Backtrack**
```java
if (!directions.isEmpty()) {
    // Choose random direction
    int dir = directions.get(random.nextInt(directions.size()));
    int nx = current.getX() + dx[dir];
    int ny = current.getY() + dy[dir];
    int mx = current.getX() + dx[dir] / 2;
    int my = current.getY() + dy[dir] / 2;
    
    // Carve path: convert wall between current and neighbor to floor
    tiles.put(new Position(mx, my), new Floor(new Position(mx, my)));
    tiles.put(new Position(nx, ny), new Floor(new Position(nx, ny)));
    stack.push(new Position(nx, ny));
} else {
    stack.pop();  // Backtrack
}
```
- If valid neighbors exist: carve a path and continue
- If no neighbors: backtrack to previous position

**Step 5: Set Start and Goal Positions**
```java
player1Start = new Position(1, 1);
player2Start = new Position(1, 1);  // Both start at same location
goalPosition = new Position(width - 2, height - 2);  // Bottom-right corner
```

**Step 6: Ensure Goal is Accessible** (CRITICAL FIX)
```java
tiles.put(goalPosition, new Floor(goalPosition));

// Clear 3x3 area around goal
for (int dgy = -1; dgy <= 1; dgy++) {
    for (int dgx = -1; dgx <= 1; dgx++) {
        int gx = goalPosition.getX() + dgx;
        int gy = goalPosition.getY() + dgy;
        if (gx > 0 && gx < width - 1 && gy > 0 && gy < height - 1) {
            Position pos = new Position(gx, gy);
            Tile tile = getTile(gx, gy);
            if (tile instanceof RegularWall) {
                tiles.put(pos, new Floor(pos));
            }
        }
    }
}
```
- Guarantees both players can reach the goal
- Clears surrounding area to prevent goal being walled off

**Step 7: Add Colored Walls**
```java
int coloredWallCount = (int) (width * height * 0.08);  // 8% of maze
while (coloredWallCount > 0 && attempts < maxAttempts) {
    int x = random.nextInt(width - 2) + 1;
    int y = random.nextInt(height - 2) + 1;
    Position pos = new Position(x, y);
    Tile tile = getTile(x, y);
    
    int distToStart = Math.abs(x - player1Start.getX()) + Math.abs(y - player1Start.getY());
    int distToGoal = Math.abs(x - goalPosition.getX()) + Math.abs(y - goalPosition.getY());
    
    // Only place on regular walls, away from start and goal
    if (tile instanceof RegularWall && distToStart > 3 && distToGoal > 3) {
        if (random.nextBoolean()) {
            tiles.put(pos, new BlueWall(pos));
        } else {
            tiles.put(pos, new RedWall(pos));
        }
        coloredWallCount--;
    }
}
```
- Randomly replaces ~8% of regular walls with colored walls
- Keeps them away from start and goal (distance > 3)
- 50/50 chance of blue or red

**Step 8: Mark Goal Tile**
```java
Tile goalTile = new Tile(goalPosition, TileType.GOAL);
tiles.put(goalPosition, goalTile);
```
- Creates special GOAL tile (renders as green with yellow star)

**Connections**:
- Owned by: `RoundManager`
- Generated by: `RoundManager.startRound()`
- Queried by: `RoundManager.handleMove()`, `MazePanel`

---

### 20. **RoundManager.java**
**Purpose**: Manages a single round of gameplay - the maze, players, timer, and win conditions.

#### Fields:
- `MazeModel maze`: The current maze
- `Map<PlayerId, PlayerState> players`: Current state of both players
- `GameTimer timer`: Round timer
- `int currentRound`: Round number (increments each round)

#### Methods:

##### Constructor
```java
public RoundManager()
```
- Creates empty maze, timer, and player map
- Initializes `currentRound` to 1

##### Getters/Setters
```java
public MazeModel getMaze()
public void setMaze(MazeModel maze)
public Map<PlayerId, PlayerState> getPlayers()
public void setPlayers(Map<PlayerId, PlayerState> players)
public GameTimer getTimer()
public int getCurrentRound()
```

##### `startRound(DifficultyLevel difficulty)` - **Round Initialization** ⭐
```java
public void startRound(DifficultyLevel difficulty)
```
**Purpose**: Sets up a new round with the specified difficulty.

**Steps**:
1. **Set difficulty and generate maze**:
```java
maze.setDifficulty(difficulty);
maze.generateMaze();
```

2. **Reset players**:
```java
players.clear();
```

3. **Initialize Player 1**:
```java
Player player1 = new Player(PlayerId.PLAYER_1, WallColor.BLUE);
PlayerState ps1 = new PlayerState(player1, 
    new Position(maze.getPlayer1Start().getX(), maze.getPlayer1Start().getY()));
players.put(PlayerId.PLAYER_1, ps1);
```
- Creates new Player instance
- Creates PlayerState at starting position
- Adds to players map

4. **Initialize Player 2**:
```java
Player player2 = new Player(PlayerId.PLAYER_2, WallColor.RED);
PlayerState ps2 = new PlayerState(player2, 
    new Position(maze.getPlayer2Start().getX(), maze.getPlayer2Start().getY()));
players.put(PlayerId.PLAYER_2, ps2);
```
- Same process as Player 1

5. **Start timer and increment round**:
```java
timer.start();
currentRound++;
```

**Called by**: 
- `GameController.startGame()` (first round)
- `GameController.startNextRound()` (subsequent rounds)

##### `handleMove(PlayerId playerId, Position newPos)` - **Movement Processing** ⭐
```java
public boolean handleMove(PlayerId playerId, Position newPos)
```
**Purpose**: Attempts to move a player to a new position.

**Parameters**:
- `playerId`: Which player is moving
- `newPos`: The target position

**Returns**: `true` if move succeeded, `false` if blocked

**Logic**:
1. **Get player state**:
```java
PlayerState playerState = players.get(playerId);
if (playerState == null) {
    return false;
}
```

2. **Validate move**:
```java
if (maze.isWalkable(playerId, newPos.getX(), newPos.getY())) {
    // Move is valid
}
```
- Checks if the tile allows this player to pass

3. **Update position**:
```java
playerState.setPosition(newPos);
```

4. **Check for goal**:
```java
Position goal = maze.getGoalPosition();
if (newPos.getX() == goal.getX() && newPos.getY() == goal.getY()) {
    playerState.setReachedGoal(true);
}
```
- If player reached goal, mark their state

**Called by**: `GameController.onPlayerInput()` after calculating new position

##### `isRoundComplete()` - **Win Condition Check**
```java
public boolean isRoundComplete()
```
**Purpose**: Checks if both players have reached the goal.

**Logic**:
```java
for (PlayerState playerState : players.values()) {
    if (!playerState.hasReachedGoal()) {
        return false;  // At least one player hasn't reached goal
    }
}
return players.size() > 0;  // All players reached goal (and there are players)
```

**Called by**: 
- `GameController.onPlayerInput()` after each move
- `MazeGameGUI.updateGameState()` to check win condition

##### `stopRound()` - **End Round**
```java
public void stopRound()
```
- Simply calls `timer.stop()`
- Called when round ends (win or timeout)

##### `getRoundResult()` - **Result Creation**
```java
public RoundResult getRoundResult()
```
**Returns**: New `RoundResult` with:
- Current difficulty
- Elapsed time from timer
- Whether round was completed

**Connections**:
- Owned by: `GameController`
- Manages: `MazeModel`, `GameTimer`, player states
- Called by: `GameController` for all round operations

---

### 21. **GameController.java**
**Purpose**: Main controller for entire game flow. Coordinates rounds, score tracking, and state transitions.

#### Fields:
- `RoundManager roundManager`: Manages current round
- `ScoreService scoreService`: Database operations
- `Team team`: Current team playing
- `GameState gameState`: Current application state
- `DifficultyLevel currentDifficulty`: Current difficulty level
- `List<RoundResult> roundResults`: History of completed rounds

#### Methods:

##### Constructor
```java
public GameController()
```
- Creates `RoundManager` and `ScoreServiceImpl`
- Sets initial state to `MENU`
- Initializes difficulty to `EASY`
- Creates empty `roundResults` list

##### Getters/Setters
```java
public RoundManager getRoundManager()
public void setRoundManager(RoundManager roundManager)
public ScoreService getScoreService()
public void setScoreService(ScoreService scoreService)
public Team getTeam()
public void setTeam(Team team)
public GameState getGameState()
public void setGameState(GameState gameState)
public DifficultyLevel getCurrentDifficulty()
```

##### `startGame(String teamName)` - **Game Initialization** ⭐
```java
public void startGame(String teamName)
```
**Purpose**: Begins a new game session.

**Steps**:
1. **Create team**:
```java
this.team = new Team(teamName);
```

2. **Reset difficulty and results**:
```java
this.currentDifficulty = DifficultyLevel.EASY;
this.roundResults.clear();
```

3. **Change state and start first round**:
```java
gameState = GameState.PLAYING;
roundManager.startRound(currentDifficulty);
```

**Called by**: `MazeGameGUI.startGame()` after team name validation

##### `stopGame()` - **Game Termination**
```java
public void stopGame()
```
- Stops current round
- Returns to MENU state
- **Called by**: User canceling out of game

##### `onPlayerInput(PlayerId playerId, Direction direction)` - **Input Handler** ⭐
```java
public void onPlayerInput(PlayerId playerId, Direction direction)
```
**Purpose**: Processes player movement input.

**Parameters**:
- `playerId`: Which player pressed a key
- `direction`: Which direction they want to move

**Logic**:
1. **Validate game state**:
```java
if (gameState != GameState.PLAYING) {
    return;  // Ignore input if not playing
}
```

2. **Get player state**:
```java
PlayerState playerState = roundManager.getPlayers().get(playerId);
if (playerState == null) {
    return;
}
```

3. **Calculate new position**:
```java
Position currentPos = playerState.getPosition();
Position newPos = calculateNewPosition(currentPos, direction);
```
- Calls `calculateNewPosition()` helper method

4. **Attempt move**:
```java
boolean moved = roundManager.handleMove(playerId, newPos);
```

5. **Check for round completion**:
```java
if (roundManager.isRoundComplete()) {
    onRoundComplete();
}
```

**Called by**: `MazeGameGUI.handleKeyPress()` when WASD or arrow keys pressed

##### `calculateNewPosition(Position current, Direction direction)` - **Position Math**
```java
private Position calculateNewPosition(Position current, Direction direction)
```
**Purpose**: Computes new coordinates based on direction.

**Logic**:
```java
int x = current.getX();
int y = current.getY();

switch (direction) {
    case UP:
        y--;  // Move up (decrease Y)
        break;
    case DOWN:
        y++;  // Move down (increase Y)
        break;
    case LEFT:
        x--;  // Move left (decrease X)
        break;
    case RIGHT:
        x++;  // Move right (increase X)
        break;
}

return new Position(x, y);
```

**Note**: Y increases downward (standard screen coordinates)

##### `onRoundComplete()` - **Round End Handler** ⭐
```java
private void onRoundComplete()
```
**Purpose**: Called when both players reach the goal.

**Steps**:
1. **Stop round and get result**:
```java
roundManager.stopRound();
RoundResult result = roundManager.getRoundResult();
roundResults.add(result);
```

2. **Update team total time**:
```java
if (team != null) {
    team.addRoundTime(result.getElapsedMillis());
}
```

3. **Check if game is complete**:
```java
if (currentDifficulty == DifficultyLevel.HARD) {
    gameState = GameState.GAME_OVER;  // All 3 rounds done!
} else {
    gameState = GameState.ROUND_COMPLETE;  // More rounds to go
}
```

##### `onTimerExpired()` - **Timeout Handler**
```java
public void onTimerExpired()
```
- Called when time runs out
- Stops round and sets state to `GAME_OVER`
- **Called by**: `MazeGameGUI.updateGameState()` when countdown reaches 0

##### `advanceDifficulty()` - **Difficulty Progression**
```java
private void advanceDifficulty()
```
**Purpose**: Moves to next difficulty level.

**Logic**:
```java
switch (currentDifficulty) {
    case EASY:
        currentDifficulty = DifficultyLevel.MEDIUM;
        break;
    case MEDIUM:
        currentDifficulty = DifficultyLevel.HARD;
        break;
    case HARD:
        // Already at max
        break;
}
```

##### `startNextRound()` - **Round Transition**
```java
public void startNextRound()
```
**Purpose**: Begins the next round after player confirms.

**Logic**:
```java
if (gameState == GameState.ROUND_COMPLETE) {
    advanceDifficulty();  // Move to harder difficulty
    gameState = GameState.PLAYING;
    roundManager.startRound(currentDifficulty);
}
```

**Called by**: `MazeGameGUI.showRoundComplete()` when player clicks "OK"

##### `getMaxTimeForDifficulty()` - **Time Limit**
```java
public long getMaxTimeForDifficulty()
```
- Returns: `60000` (60 seconds = 1 minute per round)
- Currently same for all difficulties
- Could be modified to give less time for harder difficulties

##### `saveTeamScore(String teamName)` - **Score Persistence**
```java
public void saveTeamScore(String teamName)
```
**Purpose**: Saves the team's total time to database.

```java
if (team != null) {
    scoreService.saveScore(teamName, team.getTotalTime());
}
```

**Called by**: `MazeGameGUI.showYouWin()` after all rounds completed

##### `getTopScores()` - **Leaderboard Retrieval**
```java
public List<ScoreRecord> getTopScores()
```
- Returns top 10 scores from database via `ScoreService`
- **Called by**: `MazeGameGUI.showScoreboard()`

##### `calculateTotalTime()` - **Time Calculation**
```java
public long calculateTotalTime()
```
**Purpose**: Sums up time from all completed rounds.

```java
long total = 0;
for (RoundResult result : roundResults) {
    total += result.getElapsedMillis();
}
return total;
```

**Connections**:
- Used by: `MazeGameGUI` as main controller
- Manages: `RoundManager`, `Team`, `ScoreService`
- Central coordination point for entire game

---

## Database Layer

### 22. **JDBCConnection.java** (Interface)
**Purpose**: Defines contract for database operations using JDBC.

#### Methods:

##### `startConnection()`
```java
void startConnection()
```
- Opens connection to database
- Must be called before queries/updates

##### `endConnection()`
```java
void endConnection()
```
- Closes database connection
- Should be called after operations complete

##### `executeUpdate(String sql, Object[] params)`
```java
void executeUpdate(String sql, Object[] params)
```
- **Purpose**: Executes INSERT, UPDATE, DELETE statements
- **Parameters**:
  - `sql`: SQL statement with `?` placeholders
  - `params`: Array of values to substitute for placeholders

##### `executeQuery(String sql, Object[] params)`
```java
ResultSet executeQuery(String sql, Object[] params)
```
- **Purpose**: Executes SELECT statements
- **Returns**: `ResultSet` containing query results

**Design Pattern**: This is an example of the **Repository Pattern** / **DAO (Data Access Object) Pattern**. It abstracts database details from business logic.

---

### 23. **JDBCConnectionImpl.java**
**Purpose**: MySQL implementation of `JDBCConnection` interface.

#### Fields:
- `Connection conn`: The active JDBC connection
- `String url`: Database URL (e.g., `"jdbc:mysql://localhost:3306/mazegame_db"`)
- `String username`: Database username (default: `"root"`)
- `String password`: Database password (default: `"mysql123"`)

#### Methods:

##### Constructors
```java
public JDBCConnectionImpl()
```
- Uses default connection parameters
- **Default URL**: `jdbc:mysql://localhost:3306/mazegame_db`
- **Default User**: `root`
- **Default Password**: `mysql123`

```java
public JDBCConnectionImpl(String url, String username, String password)
```
- Allows custom connection parameters
- Useful for testing or different database configurations

##### Getters/Setters
```java
public Connection getConn()
public void setConn(Connection conn)
```

##### `startConnection()` - **Connection Establishment** ⭐
```java
@Override
public void startConnection()
```
**Purpose**: Connects to MySQL database.

**Steps**:
1. **Load JDBC driver**:
```java
Class.forName("com.mysql.cj.jdbc.Driver");
```
- Loads MySQL Connector/J driver
- Required before JDBC 4.0 (modern versions auto-load)

2. **Establish connection**:
```java
conn = DriverManager.getConnection(url, username, password);
System.out.println("Database connection established successfully");
```

3. **Error handling**:
```java
catch (ClassNotFoundException e) {
    System.err.println("MySQL JDBC Driver not found");
    e.printStackTrace();
}
catch (SQLException e) {
    System.err.println("Failed to establish database connection");
    e.printStackTrace();
}
```

##### `endConnection()` - **Connection Cleanup**
```java
@Override
public void endConnection()
```
**Purpose**: Safely closes database connection.

```java
if (conn != null && !conn.isClosed()) {
    conn.close();
    System.out.println("Database connection closed");
}
```

**Important**: Always called in `finally` blocks or after operations complete to prevent connection leaks.

##### `executeUpdate(String sql, Object[] params)` - **Data Modification** ⭐
```java
@Override
public void executeUpdate(String sql, Object[] params)
```
**Purpose**: Executes INSERT, UPDATE, or DELETE statements.

**Example Usage**:
```java
String sql = "INSERT INTO SCOREBOARD (team_name, completion_time) VALUES (?, ?)";
Object[] params = {"TeamA", 45000};
executeUpdate(sql, params);
```

**Logic**:
1. **Create PreparedStatement**:
```java
PreparedStatement stmt = conn.prepareStatement(sql);
```
- `PreparedStatement` prevents SQL injection

2. **Set parameters**:
```java
if (params != null) {
    for (int i = 0; i < params.length; i++) {
        stmt.setObject(i + 1, params[i]);  // JDBC uses 1-based indexing
    }
}
```

3. **Execute and cleanup**:
```java
stmt.executeUpdate();
stmt.close();
```

4. **Error handling**:
```java
catch (SQLException e) {
    System.err.println("Error executing update: " + sql);
    e.printStackTrace();
}
```

##### `executeQuery(String sql, Object[] params)` - **Data Retrieval** ⭐
```java
@Override
public ResultSet executeQuery(String sql, Object[] params)
```
**Purpose**: Executes SELECT statements and returns results.

**Example Usage**:
```java
String sql = "SELECT team_name, completion_time FROM SCOREBOARD WHERE team_name = ?";
Object[] params = {"TeamA"};
ResultSet rs = executeQuery(sql, params);
```

**Logic**:
1. **Create PreparedStatement**:
```java
PreparedStatement stmt = conn.prepareStatement(sql);
```

2. **Set parameters** (same as `executeUpdate`)

3. **Execute query and return results**:
```java
return stmt.executeQuery();
```

**Important Note**: 
- The returned `ResultSet` must be closed by the caller
- `PreparedStatement` is NOT closed here (unlike `executeUpdate`)
- This is intentional - closing the statement would close the ResultSet

**Connections**:
- Implemented by: This class
- Used by: `ScoreServiceImpl`
- Connects to: MySQL database via JDBC

---

### 24. **ScoreService.java** (Interface)
**Purpose**: Defines contract for score-related business operations.

#### Methods:

##### `saveScore(String teamName, long timeMillis)`
```java
void saveScore(String teamName, long timeMillis)
```
- Saves a team's completion time to database

##### `getTopScores()`
```java
List<ScoreRecord> getTopScores()
```
- Retrieves top 10 fastest completion times

##### `teamNameExists(String teamName)`
```java
boolean teamNameExists(String teamName)
```
- Checks if a team name is already used
- Enforces unique team names

**Design Pattern**: **Service Layer Pattern** - separates business logic from data access

---

### 25. **ScoreServiceImpl.java**
**Purpose**: Implementation of `ScoreService` using JDBC.

#### Fields:
- `JDBCConnection dbConnection`: Database connection handler

#### Methods:

##### Constructors
```java
public ScoreServiceImpl()
```
- Creates default `JDBCConnectionImpl`

```java
public ScoreServiceImpl(JDBCConnection connection)
```
- Accepts custom connection (useful for testing with mock connections)

##### Getters/Setters
```java
public JDBCConnection getDbConnection()
public void setDbConnection(JDBCConnection dbConnection)
```

##### `saveScore(String teamName, long timeMillis)` - **Score Persistence** ⭐
```java
@Override
public void saveScore(String teamName, long timeMillis)
```
**Purpose**: Inserts a new score record into the database.

**Logic**:
1. **Open connection**:
```java
dbConnection.startConnection();
```

2. **Prepare SQL**:
```java
String sql = "INSERT INTO SCOREBOARD (team_name, completion_time) VALUES (?, ?)";
Object[] params = {teamName, timeMillis};
```
- Table: `SCOREBOARD`
- Columns: `team_name` (VARCHAR), `completion_time` (BIGINT)

3. **Execute**:
```java
dbConnection.executeUpdate(sql, params);
```

4. **Close connection**:
```java
dbConnection.endConnection();
```

5. **Error handling**:
```java
catch (Exception e) {
    System.err.println("Error saving score: " + e.getMessage());
    e.printStackTrace();
}
```

**Called by**: `GameController.saveTeamScore()` after winning the game

##### `teamNameExists(String teamName)` - **Duplicate Check** ⭐
```java
@Override
public boolean teamNameExists(String teamName)
```
**Purpose**: Checks if a team name is already in the database.

**Logic**:
1. **Open connection and query**:
```java
dbConnection.startConnection();
String sql = "SELECT COUNT(*) as count FROM SCOREBOARD WHERE team_name = ?";
Object[] params = {teamName};
ResultSet rs = dbConnection.executeQuery(sql, params);
```

2. **Read result**:
```java
boolean exists = false;
if (rs != null && rs.next()) {
    int count = rs.getInt("count");
    exists = (count > 0);
}
```
- If count > 0, team name exists

3. **Cleanup**:
```java
if (rs != null) {
    rs.close();
}
dbConnection.endConnection();
```

4. **Return result**:
```java
return exists;
```

**Called by**: `MazeGameGUI.startGame()` to validate team name uniqueness

##### `getTopScores()` - **Leaderboard Retrieval** ⭐
```java
@Override
public List<ScoreRecord> getTopScores()
```
**Purpose**: Fetches the 10 fastest completion times.

**Logic**:
1. **Initialize result list**:
```java
List<ScoreRecord> scores = new ArrayList<>();
```

2. **Open connection and query**:
```java
dbConnection.startConnection();
String sql = "SELECT team_name, completion_time FROM SCOREBOARD ORDER BY completion_time ASC LIMIT 10";
ResultSet rs = dbConnection.executeQuery(sql, null);
```
- Orders by `completion_time` ascending (fastest first)
- Limits to top 10

3. **Process results**:
```java
while (rs != null && rs.next()) {
    String teamName = rs.getString("team_name");
    long completionTime = rs.getLong("completion_time");
    scores.add(new ScoreRecord(teamName, completionTime));
}
```
- Converts each row to a `ScoreRecord` object

4. **Cleanup**:
```java
if (rs != null) {
    rs.close();
}
dbConnection.endConnection();
```

5. **Return list**:
```java
return scores;
```
- Returns empty list if error occurs

**Called by**: `GameController.getTopScores()` for scoreboard display

**Database Schema**:
```sql
CREATE TABLE SCOREBOARD (
    id INT AUTO_INCREMENT PRIMARY KEY,
    team_name VARCHAR(20) NOT NULL UNIQUE,
    completion_time BIGINT NOT NULL
);
```

**Connections**:
- Interface: `ScoreService`
- Uses: `JDBCConnection`
- Used by: `GameController`

---

## User Interface

### 26. **MazeGameGUI.java**
**Purpose**: Main Swing GUI window. Manages all screens and user input.

#### Fields:
- `GameController gameController`: Main game controller
- `MazePanel mazePanel`: Custom panel for rendering maze
- `JLabel timerLabel`: Displays countdown timer
- `JLabel roundLabel`: Shows current round/difficulty
- `Timer gameTimer`: Swing timer for UI updates (100ms interval)

#### Methods:

##### Constructor
```java
public MazeGameGUI()
```
**Purpose**: Initializes the game window.

**Steps**:
1. **Create controller**:
```java
gameController = new GameController();
```

2. **Configure window**:
```java
setTitle("Maze Game");
setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
setSize(800, 600);
setLocationRelativeTo(null);  // Center on screen
```

3. **Show main menu**:
```java
showMainMenu();
```

4. **Make visible**:
```java
setVisible(true);
```

##### `showMainMenu()` - **Main Menu Screen** ⭐
```java
private void showMainMenu()
```
**Purpose**: Displays the main menu with title and buttons.

**UI Components**:
1. **Clear and setup**:
```java
getContentPane().removeAll();
setLayout(new BorderLayout());
```

2. **Create menu panel**:
```java
JPanel menuPanel = new JPanel();
menuPanel.setLayout(new BoxLayout(menuPanel, BoxLayout.Y_AXIS));
menuPanel.setBackground(Color.DARK_GRAY);
```

3. **Title label**:
```java
JLabel titleLabel = new JLabel("MAZE GAME");
titleLabel.setFont(new Font("Arial", Font.BOLD, 48));
titleLabel.setForeground(Color.WHITE);
titleLabel.setAlignmentX(Component.CENTER_ALIGNMENT);
```

4. **Start button**:
```java
JButton startButton = new JButton("Start Game");
startButton.setAlignmentX(Component.CENTER_ALIGNMENT);
startButton.setFont(new Font("Arial", Font.PLAIN, 24));
startButton.addActionListener(e -> startGame());
```
- Calls `startGame()` when clicked

5. **Scoreboard button**:
```java
JButton scoreboardButton = new JButton("Scoreboard");
scoreboardButton.addActionListener(e -> showScoreboard());
```

6. **Exit button**:
```java
JButton exitButton = new JButton("Exit");
exitButton.addActionListener(e -> System.exit(0));
```

7. **Layout with spacing**:
```java
menuPanel.add(Box.createVerticalGlue());
menuPanel.add(titleLabel);
menuPanel.add(Box.createRigidArea(new Dimension(0, 50)));
menuPanel.add(startButton);
menuPanel.add(Box.createRigidArea(new Dimension(0, 20)));
menuPanel.add(scoreboardButton);
menuPanel.add(Box.createRigidArea(new Dimension(0, 20)));
menuPanel.add(exitButton);
menuPanel.add(Box.createVerticalGlue());
```

##### `startGame()` - **Game Start Handler** ⭐
```java
private void startGame()
```
**Purpose**: Validates team name and begins new game.

**Steps**:
1. **Prompt for team name**:
```java
String teamName = JOptionPane.showInputDialog(this, "Enter team name (max 20 characters):");
```

2. **Validate not empty**:
```java
if (teamName == null || teamName.trim().isEmpty()) {
    return;
}
```

3. **Validate length**:
```java
if (teamName.length() > 20) {
    JOptionPane.showMessageDialog(this, "Team name must be 20 characters or less!");
    return;
}
```

4. **Check for duplicates**:
```java
try {
    boolean exists = gameController.getScoreService().teamNameExists(teamName);
    if (exists) {
        JOptionPane.showMessageDialog(
            this,
            "Team name '" + teamName + "' already exists!\nPlease choose a different name.",
            "Duplicate Team Name",
            JOptionPane.WARNING_MESSAGE
        );
        return;
    }
}
```

5. **Start game**:
```java
gameController.startGame(teamName);
showGameScreen();
```

##### `showGameScreen()` - **Gameplay Screen** ⭐
```java
private void showGameScreen()
```
**Purpose**: Displays the maze, players, and game info.

**UI Components**:
1. **Clear and setup**:
```java
getContentPane().removeAll();
setLayout(new BorderLayout());
```

2. **Info panel (top)**:
```java
JPanel infoPanel = new JPanel(new FlowLayout());
infoPanel.setBackground(Color.LIGHT_GRAY);

roundLabel = new JLabel("Round: " + getRoundName());
roundLabel.setFont(new Font("Arial", Font.BOLD, 18));

timerLabel = new JLabel("Time: 0s");
timerLabel.setFont(new Font("Arial", Font.BOLD, 18));

JLabel instructionsLabel = new JLabel("  |  P1: WASD (Blue)  |  P2: Arrows (Red)");
instructionsLabel.setFont(new Font("Arial", Font.PLAIN, 14));

infoPanel.add(roundLabel);
infoPanel.add(Box.createRigidArea(new Dimension(20, 0)));
infoPanel.add(timerLabel);
infoPanel.add(instructionsLabel);

add(infoPanel, BorderLayout.NORTH);
```

3. **Maze panel (center)**:
```java
mazePanel = new MazePanel(gameController);
add(mazePanel, BorderLayout.CENTER);
```

4. **Key listener with debouncing**:
```java
final boolean[] keyPressed = new boolean[1];
addKeyListener(new KeyAdapter() {
    @Override
    public void keyPressed(KeyEvent e) {
        if (!keyPressed[0]) {  // Prevent multiple triggers
            keyPressed[0] = true;
            handleKeyPress(e);
        }
    }
    
    @Override
    public void keyReleased(KeyEvent e) {
        keyPressed[0] = false;  // Reset flag
    }
});
```
- Prevents multiple moves from holding key down

5. **Focus setup**:
```java
setFocusable(true);
requestFocusInWindow();
```

6. **Start UI update timer**:
```java
gameTimer = new Timer(100, e -> updateGameState());
gameTimer.start();
```
- Updates every 100ms (10 times per second)

##### `handleKeyPress(KeyEvent e)` - **Keyboard Input** ⭐
```java
private void handleKeyPress(KeyEvent e)
```
**Purpose**: Converts keyboard input to game actions.

**Key Mappings**:
```java
switch (e.getKeyCode()) {
    // Player 1 (WASD)
    case KeyEvent.VK_W:
        direction = Direction.UP;
        playerId = PlayerId.PLAYER_1;
        break;
    case KeyEvent.VK_S:
        direction = Direction.DOWN;
        playerId = PlayerId.PLAYER_1;
        break;
    case KeyEvent.VK_A:
        direction = Direction.LEFT;
        playerId = PlayerId.PLAYER_1;
        break;
    case KeyEvent.VK_D:
        direction = Direction.RIGHT;
        playerId = PlayerId.PLAYER_1;
        break;
    
    // Player 2 (Arrow Keys)
    case KeyEvent.VK_UP:
        direction = Direction.UP;
        playerId = PlayerId.PLAYER_2;
        break;
    case KeyEvent.VK_DOWN:
        direction = Direction.DOWN;
        playerId = PlayerId.PLAYER_2;
        break;
    case KeyEvent.VK_LEFT:
        direction = Direction.LEFT;
        playerId = PlayerId.PLAYER_2;
        break;
    case KeyEvent.VK_RIGHT:
        direction = Direction.RIGHT;
        playerId = PlayerId.PLAYER_2;
        break;
}
```

**Action**:
```java
if (direction != null && playerId != null) {
    gameController.onPlayerInput(playerId, direction);
    mazePanel.repaint();  // Update display
}
```

##### `updateGameState()` - **Game Loop** ⭐
```java
private void updateGameState()
```
**Purpose**: Called every 100ms to update timer and check game state.

**Steps**:
1. **Update timer display**:
```java
long elapsed = gameController.getRoundManager().getTimer().getElapsedTime() / 1000;
long timeLimit = gameController.getMaxTimeForDifficulty() / 1000;
long remaining = timeLimit - elapsed;

if (remaining < 0) remaining = 0;

timerLabel.setText("Time: " + remaining + "s");
```
- Shows countdown timer

2. **Check for timeout**:
```java
if (remaining <= 0 && !gameController.getRoundManager().isRoundComplete()) {
    gameTimer.stop();
    gameController.onTimerExpired();
}
```

3. **Check game state**:
```java
GameState state = gameController.getGameState();

if (state == GameState.ROUND_COMPLETE) {
    gameTimer.stop();
    showRoundComplete();
} else if (state == GameState.GAME_OVER) {
    gameTimer.stop();
    
    if (gameController.getRoundManager().isRoundComplete() && 
        gameController.getCurrentDifficulty() == DifficultyLevel.HARD) {
        showYouWin();
    } else {
        showYouLose();
    }
}
```

##### `showRoundComplete()` - **Round Transition Dialog**
```java
private void showRoundComplete()
```
**Purpose**: Shows dialog when round is completed successfully.

```java
String completedRoundName = getRoundName();

int result = JOptionPane.showConfirmDialog(
    this,
    "Round " + completedRoundName + " Complete!\nReady for next round?",
    "Round Complete",
    JOptionPane.OK_CANCEL_OPTION
);

if (result == JOptionPane.OK_OPTION) {
    gameController.startNextRound();
    roundLabel.setText("Round: " + getRoundName());
    mazePanel.repaint();
    gameTimer.start();
} else {
    showMainMenu();
}
```

##### `showYouWin()` - **Victory Screen**
```java
private void showYouWin()
```
**Purpose**: Displays win message, saves score, shows scoreboard.

```java
long totalTime = gameController.calculateTotalTime() / 1000;

JOptionPane.showMessageDialog(
    this,
    "Congratulations! You Win!\nTotal Time: " + totalTime + " seconds",
    "Victory!",
    JOptionPane.INFORMATION_MESSAGE
);

// Save score
if (gameController.getTeam() != null) {
    gameController.saveTeamScore(gameController.getTeam().getTeamName());
}

// Show scoreboard BEFORE returning to main menu
showScoreboard();
```

##### `showYouLose()` - **Game Over Screen**
```java
private void showYouLose()
```
**Purpose**: Displays loss message when time runs out.

```java
JOptionPane.showMessageDialog(
    this,
    "Time's up! You Lose!",
    "Game Over",
    JOptionPane.ERROR_MESSAGE
);

showMainMenu();
```

##### `showScoreboard()` - **Leaderboard Screen** ⭐
```java
private void showScoreboard()
```
**Purpose**: Displays top 10 scores.

**UI Components**:
1. **Setup**:
```java
getContentPane().removeAll();
setLayout(new BorderLayout());

JPanel scorePanel = new JPanel();
scorePanel.setLayout(new BoxLayout(scorePanel, BoxLayout.Y_AXIS));
scorePanel.setBackground(Color.DARK_GRAY);
```

2. **Title**:
```java
JLabel titleLabel = new JLabel("SCOREBOARD");
titleLabel.setFont(new Font("Arial", Font.BOLD, 36));
titleLabel.setForeground(Color.WHITE);
titleLabel.setAlignmentX(Component.CENTER_ALIGNMENT);
```

3. **Retrieve and display scores**:
```java
List<ScoreRecord> scores = gameController.getTopScores();

for (int i = 0; i < scores.size(); i++) {
    ScoreRecord record = scores.get(i);
    String text = String.format("%d. %s - %d seconds", 
        i + 1, record.getTeamName(), record.getTotalTime() / 1000);
    
    JLabel scoreLabel = new JLabel(text);
    scoreLabel.setFont(new Font("Arial", Font.PLAIN, 20));
    scoreLabel.setForeground(Color.WHITE);
    scoreLabel.setAlignmentX(Component.CENTER_ALIGNMENT);
    
    scorePanel.add(scoreLabel);
    scorePanel.add(Box.createRigidArea(new Dimension(0, 10)));
}
```

4. **Back button**:
```java
JButton backButton = new JButton("Back to Menu");
backButton.setAlignmentX(Component.CENTER_ALIGNMENT);
backButton.setFont(new Font("Arial", Font.PLAIN, 20));
backButton.addActionListener(e -> showMainMenu());
```

##### `getRoundName()` - **Difficulty String**
```java
private String getRoundName()
```
**Purpose**: Converts difficulty enum to display string.

```java
DifficultyLevel diff = gameController.getCurrentDifficulty();
switch (diff) {
    case EASY: return "Easy";
    case MEDIUM: return "Medium";
    case HARD: return "Hard";
    default: return "Unknown";
}
```

##### `main(String[] args)` - **Application Entry Point**
```java
public static void main(String[] args)
```
**Purpose**: Starts the application on the Swing Event Dispatch Thread.

```java
SwingUtilities.invokeLater(() -> new MazeGameGUI());
```
- `SwingUtilities.invokeLater()` ensures thread safety for Swing components

**Connections**:
- Uses: `GameController`, `MazePanel`
- Main entry point of application

---

### 27. **MazePanel.java**
**Purpose**: Custom JPanel that renders the maze and players.

#### Fields:
- `GameController gameController`: Reference to controller (to access maze and player data)
- `static final int CELL_SIZE = 20`: Size of each tile in pixels

#### Methods:

##### Constructor
```java
public MazePanel(GameController gameController)
```
- Sets background to black
- Stores controller reference

##### `paintComponent(Graphics g)` - **Rendering Method** ⭐
```java
@Override
protected void paintComponent(Graphics g)
```
**Purpose**: Renders the entire maze and players.

**Steps**:
1. **Call super (clear background)**:
```java
super.paintComponent(g);
```

2. **Get maze and players**:
```java
MazeModel maze = gameController.getRoundManager().getMaze();
Map<PlayerId, PlayerState> players = gameController.getRoundManager().getPlayers();

if (maze == null) {
    return;
}
```

3. **Calculate centering offset**:
```java
int width = maze.getWidth();
int height = maze.getHeight();

int offsetX = (getWidth() - width * CELL_SIZE) / 2;
int offsetY = (getHeight() - height * CELL_SIZE) / 2;
```
- Centers maze on panel

4. **Draw all tiles**:
```java
for (int y = 0; y < height; y++) {
    for (int x = 0; x < width; x++) {
        Tile tile = maze.getTile(x, y);
        if (tile != null) {
            drawTile(g, tile, offsetX + x * CELL_SIZE, offsetY + y * CELL_SIZE);
        }
    }
}
```

5. **Draw players**:
```java
if (players != null) {
    PlayerState p1 = players.get(PlayerId.PLAYER_1);
    if (p1 != null) {
        drawPlayer(g, p1, offsetX, offsetY, Color.CYAN);
    }
    
    PlayerState p2 = players.get(PlayerId.PLAYER_2);
    if (p2 != null) {
        drawPlayer(g, p2, offsetX, offsetY, Color.RED);
    }
}
```

##### `drawTile(Graphics g, Tile tile, int x, int y)` - **Tile Rendering**
```java
private void drawTile(Graphics g, Tile tile, int x, int y)
```
**Purpose**: Renders a single tile with appropriate color.

**Color Mapping**:
```java
TileType type = tile.getType();

switch (type) {
    case FLOOR:
        g.setColor(Color.WHITE);
        g.fillRect(x, y, CELL_SIZE, CELL_SIZE);
        break;
    
    case REGULAR_WALL:
        g.setColor(Color.GRAY);
        g.fillRect(x, y, CELL_SIZE, CELL_SIZE);
        break;
    
    case BLUE_WALL:
        g.setColor(Color.BLUE);
        g.fillRect(x, y, CELL_SIZE, CELL_SIZE);
        break;
    
    case RED_WALL:
        g.setColor(new Color(200, 0, 0));  // Dark red
        g.fillRect(x, y, CELL_SIZE, CELL_SIZE);
        break;
    
    case GOAL:
        g.setColor(Color.GREEN);
        g.fillRect(x, y, CELL_SIZE, CELL_SIZE);
        // Draw star pattern
        g.setColor(Color.YELLOW);
        int cx = x + CELL_SIZE / 2;
        int cy = y + CELL_SIZE / 2;
        g.drawLine(cx - 5, cy, cx + 5, cy);  // Horizontal line
        g.drawLine(cx, cy - 5, cx, cy + 5);  // Vertical line
        break;
}

// Draw grid border
g.setColor(Color.BLACK);
g.drawRect(x, y, CELL_SIZE, CELL_SIZE);
```

##### `drawPlayer(Graphics g, PlayerState playerState, int offsetX, int offsetY, Color color)` - **Player Rendering**
```java
private void drawPlayer(Graphics g, PlayerState playerState, int offsetX, int offsetY, Color color)
```
**Purpose**: Renders a player as a colored circle.

**Steps**:
1. **Calculate screen position**:
```java
Position pos = playerState.getPosition();
int x = offsetX + pos.getX() * CELL_SIZE;
int y = offsetY + pos.getY() * CELL_SIZE;
```

2. **Draw colored circle**:
```java
g.setColor(color);
g.fillOval(x + 3, y + 3, CELL_SIZE - 6, CELL_SIZE - 6);
```
- 3-pixel padding on each side

3. **Draw outline**:
```java
g.setColor(Color.BLACK);
g.drawOval(x + 3, y + 3, CELL_SIZE - 6, CELL_SIZE - 6);
```

4. **Draw checkmark if reached goal**:
```java
if (playerState.hasReachedGoal()) {
    g.setColor(Color.GREEN);
    g.setFont(new Font("Arial", Font.BOLD, 12));
    g.drawString("✓", x + 5, y + 15);
}
```

**Connections**:
- Created by: `MazeGameGUI.showGameScreen()`
- Reads data from: `GameController.getRoundManager()`

---

## Class Relationships

### Dependency Diagram

```
MazeGameGUI (main entry)
    ├── GameController
    │   ├── RoundManager
    │   │   ├── MazeModel
    │   │   │   ├── Tile (Map<Position, Tile>)
    │   │   │   │   ├── Floor
    │   │   │   │   ├── RegularWall
    │   │   │   │   ├── BlueWall
    │   │   │   │   └── RedWall
    │   │   │   ├── Position (key in map)
    │   │   │   └── DifficultyLevel
    │   │   ├── PlayerState (Map<PlayerId, PlayerState>)
    │   │   │   ├── Player
    │   │   │   │   ├── PlayerId
    │   │   │   │   └── WallColor
    │   │   │   └── Position
    │   │   └── GameTimer
    │   ├── ScoreService (interface)
    │   │   └── ScoreServiceImpl
    │   │       ├── JDBCConnection (interface)
    │   │       │   └── JDBCConnectionImpl
    │   │       └── ScoreRecord
    │   ├── Team
    │   │   └── Player
    │   ├── GameState
    │   ├── DifficultyLevel
    │   └── RoundResult
    └── MazePanel
```

### Design Patterns Used

1. **Model-View-Controller (MVC)**
   - **Model**: `MazeModel`, `Player`, `PlayerState`, `Team`
   - **View**: `MazeGameGUI`, `MazePanel`
   - **Controller**: `GameController`, `RoundManager`

2. **Service Layer**
   - `ScoreService` interface
   - `ScoreServiceImpl` implementation

3. **Repository/DAO**
   - `JDBCConnection` interface
   - `JDBCConnectionImpl` implementation

4. **Strategy Pattern**
   - `Tile.canPlayerPass()` - behavior varies by tile type

5. **State Pattern**
   - `GameState` enum controls UI flow

6. **Singleton-like**
   - `GameController` is created once in `MazeGameGUI`

### Data Flow Examples

#### Example 1: Player Movement
1. User presses 'W' key
2. `MazeGameGUI.handleKeyPress()` detects key
3. Converts to `Direction.UP` and `PlayerId.PLAYER_1`
4. Calls `GameController.onPlayerInput(PLAYER_1, UP)`
5. Controller gets current position from `PlayerState`
6. Calls `calculateNewPosition()` to compute new coordinates
7. Calls `RoundManager.handleMove(PLAYER_1, newPosition)`
8. RoundManager calls `MazeModel.isWalkable(PLAYER_1, x, y)`
9. MazeModel gets tile and calls `tile.canPlayerPass(PLAYER_1)`
10. If valid, updates `PlayerState.position`
11. Checks if new position is goal
12. `MazePanel.repaint()` updates display

#### Example 2: Saving Score
1. Both players reach goal on HARD difficulty
2. `GameController.onRoundComplete()` sets state to GAME_OVER
3. `MazeGameGUI.updateGameState()` detects GAME_OVER
4. Calls `showYouWin()`
5. Calls `GameController.saveTeamScore(teamName)`
6. Controller calls `ScoreService.saveScore(name, time)`
7. `ScoreServiceImpl.saveScore()` opens database connection
8. Calls `JDBCConnection.executeUpdate()` with INSERT SQL
9. `JDBCConnectionImpl` creates PreparedStatement
10. Sets parameters and executes
11. Closes connection
12. GUI shows scoreboard with updated scores

---

## Summary

This Maze Game is a well-structured application that demonstrates:

- **Object-Oriented Design**: Inheritance (Tile hierarchy), Encapsulation (private fields), Polymorphism (canPlayerPass())
- **Design Patterns**: MVC, Service Layer, DAO, Strategy, State
- **Algorithm Implementation**: Depth-First Search for maze generation
- **Database Integration**: JDBC with MySQL
- **GUI Development**: Swing components, custom rendering, event handling
- **Game Logic**: Turn-based movement, collision detection, timer system, difficulty progression

The codebase is modular, with clear separation of concerns:
- **Data models** are simple POJOs
- **Game logic** is in controllers and managers
- **Database access** is abstracted behind interfaces
- **UI** is separated from business logic

This architecture makes the code maintainable, testable, and extensible.

