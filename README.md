<div align="center">

# 🎮 BLOCKFALL

### Tetris-style game built in C++ with custom data structures

<img src="docs/demo/gameplay.gif" alt="Blockfall gameplay demo — movement, rotation, hard drop" width="700">

<p><sub>Real gameplay capture: movement, rotation, hard drop, next-piece preview, and the ghost-piece landing guide.</sub></p>

<img src="https://img.shields.io/badge/C%2B%2B-17-00599C?style=for-the-badge&logo=cplusplus&logoColor=white" alt="C++17">
<img src="https://img.shields.io/badge/Raylib-5.5-000000?style=for-the-badge&logo=raylib&logoColor=white" alt="Raylib 5.5">
<img src="https://img.shields.io/badge/Platform-Windows-0078D6?style=for-the-badge&logo=windows&logoColor=white" alt="Windows">

<p>
  <a href="#️-gameplay">Gameplay</a> ·
  <a href="#-data-structures">Data Structures</a> ·
  <a href="#️-architecture">Architecture</a> ·
  <a href="#-gameplay-flow">Flow</a> ·
  <a href="#-project-at-a-glance">At a Glance</a> ·
  <a href="#️-build--run">Build & Run</a>
</p>

</div>

---

**Blockfall** is a Tetris-style desktop game built in **C++17** with **Raylib**, developed as a semester project for **Data Structures and Algorithms**. Instead of treating data structures as isolated exercises, each one has a real job inside the game — a queue feeds upcoming pieces, a stack powers undo, an AVL tree keeps the leaderboard sorted, and linked lists drive both the piece bag and the board itself.

---

## 🕹️ Gameplay

<table>
<tr>
<td align="center" width="50%">
<img src="docs/screenshots/welcome.jpg" alt="Welcome screen" width="100%">
<br><sub><b>Welcome screen</b> — falling-letter title animation</sub>
</td>
<td align="center" width="50%">
<img src="docs/screenshots/gameplay.png" alt="Active gameplay" width="100%">
<br><sub><b>Active gameplay</b> — board, HUD, next-piece preview</sub>
</td>
</tr>
<tr>
<td align="center" width="50%">
<img src="docs/screenshots/pause.png" alt="Pause screen" width="100%">
<br><sub><b>Paused</b> — board dims, pause icon flips to play</sub>
</td>
<td align="center" width="50%">
<img src="docs/screenshots/gameover.png" alt="Game over and leaderboard screen" width="100%">
<br><sub><b>Game over</b> — final score + AVL-backed Top 5 leaderboard</sub>
</td>
</tr>
</table>

### Controls

```text
              ↑
           Rotate

   ←        ↓        →
  Move   Soft Drop   Move

        SPACE = Hard Drop
          H = Hold
       CTRL+Z = Undo
```

| Key | Action |
|---|---|
| `←` / `→` | Move piece left / right |
| `↓` | Soft drop |
| `↑` | Rotate |
| `Space` | Hard drop |
| `H` | Hold piece |
| `Ctrl + Z` | Undo |

---

## ✨ Features

* 🎮 Tetris-style gameplay on a **15 × 20** board, 7 standard piece types
* 🔄 Movement, rotation, and collision detection
* 🧱 Line clearing with on-screen **SINGLE! / DOUBLE!! / TRIPLE!!! / TETRIS!!!!** messages
* 🏅 Classic scoring — 100 / 200 / 500 / 800 points for 1–4 lines, +100 per extra line
* 👀 Next-piece preview, 3 pieces deep
* 👻 Toggleable ghost-piece landing guide
* ↩️ Undo via saved game-state snapshots
* 🏆 AVL-tree-backed score leaderboard (Top 5)
* 🔊 Background music, rotate/clear sound effects, in-game mute toggle
* ⏸️ Pause/resume

---

## 🧩 Data Structures

This is a DSA project first — every structure below is doing real work, not sitting there as an exercise.

```text
                          BLOCKFALL
                              │
          ┌──────────┬────────┼────────┬──────────┐
          ▼          ▼        ▼        ▼          ▼
       Queue      UndoStack  ScoreAVL  LinkedList  LinkedList
          │          │          │          │          │
     Upcoming      Undo      Leaderboard  Piece      Board
      Pieces      States       Scores      Bag        Rows
          │
          ▼
      PieceQueue
   (combines Queue +
    LinkedList bag to
    generate pieces)
```

| Structure | Used for | Why it fits |
|---|---|---|
| **Linked List** | Piece bag | Dynamic sequence pieces are drawn from |
| **Linked List** | Board rows | Each row is a node holding 15 cells |
| **Queue** | Upcoming pieces | FIFO — pieces enter play in the order they're drawn |
| **Stack** | Undo | Most recent state must come back first (LIFO) |
| **AVL Tree** | Leaderboard | Keeps scores ordered and balanced on every insert |

### Code callouts

**Queue — feeding the next-piece preview:**
```cpp
void Queue::enqueue(Piece piece) {
    if (isFull()) return;
    QueueNode* newNode = new QueueNode(piece);
    if (isEmpty()) { front = rear = newNode; }
    else { rear->next = newNode; rear = newNode; }
    size++;
}
```
Standard linked-list FIFO — pieces are added at the rear and drawn from the front, so the "Next Pieces" panel always shows them in arrival order.

**Stack — undo:**
```cpp
void UndoStack::Push(const Piece& snapshot) {
    if (IsFull()) stack.erase(stack.begin());  // drop oldest when full
    stack.push_back(snapshot);
}
Piece UndoStack::Pop() {
    if (IsEmpty()) return Piece();
    Piece top = stack.back();
    stack.pop_back();
    return top;
}
```
`Ctrl+Z` pops the most recent snapshot — board, score, and piece queue all roll back together, not just the last move.

**AVL Tree — leaderboard insert:**
```cpp
AVLScoreNode* ScoreAVL::InsertNode(AVLScoreNode* node, int val) {
    if (!node) return new AVLScoreNode(val);
    if (val < node->score) node->left = InsertNode(node->left, val);
    else if (val > node->score) node->right = InsertNode(node->right, val);
    else return node;

    UpdateHeight(node);
    int balance = GetBalance(node);
    // 4 standard rotation cases keep the tree balanced here
    ...
    return node;
}
```
Called once per game, right when `GameOver` is set — the final score is inserted and the tree rebalances itself via rotations, keeping `GetTopScores()` cheap to query.

---

## 🏗️ Architecture

```text
                       main.cpp
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
        WelcomeScreen    Game         Manager
                          │           (UI/HUD)
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
          Board        PieceQueue    UndoStack
     (LinkedList of      │
         rows)    ┌──────┴──────┐
                   ▼             ▼
                 Queue       LinkedList
               (upcoming)    (piece bag)

                     Game
                      │
                      ▼
                  Leaderboard
                      │
                      ▼
                  ScoreAVL
```

| Directory | Responsibility |
|---|---|
| `game/` | Core Tetris gameplay and piece logic |
| `data_structures/` | Custom DSA implementations |
| `leaderboard/` | Score and leaderboard management |
| `ui/` | Screens, HUD, colours, presentation |
| `assets/` | Images, audio, fonts |

---

## 🔄 Gameplay Flow

```text
         Start Game
              │
              ▼
       Generate Piece ◄─────────────┐
              │                     │
              ▼                     │
         PieceQueue                 │
         (bag + queue)              │
              │                     │
              ▼                     │
       Player Movement              │
      (move / rotate / drop)        │
              │                     │
              ▼                     │
       Collision Check              │
              │                     │
              ▼                     │
         Piece Locks                │
              │                     │
              ▼                     │
      Row(s) complete? ─── No ──────┘
              │
             Yes
              │
              ▼
     Clear row(s) + Score
              │
              ▼
     Can next piece spawn? ── Yes ──► (loop back to Generate Piece)
              │
              No
              │
              ▼
          GAME OVER
              │
              ▼
    AVL Insert → Leaderboard
```

---

## 📊 Project at a Glance

| Area | Implementation |
|---|---|
| Language | C++17 |
| Graphics / audio / input | Raylib 5.5 |
| Board | 15 × 20, linked-list rows |
| Piece management | Queue + linked-list piece bag (`PieceQueue`) |
| Undo system | Stack (snapshot-based) |
| Leaderboard | AVL Tree, Top 5 |
| Build system | Makefile (MinGW-w64 / GCC) |

---

## 👥 Team & Project Context

**Blockfall** originated as a semester group project for Data Structures and Algorithms. The original project was developed collaboratively in **Aiman-Misbah/DSA-Project** before being reorganised and continued under this repository.

| Team Member | Main Contributions |
|---|---|
| **Aiman** | AVL tree and leaderboard implementation, early game/application integration |
| **Maryam** | Queue, PieceQueue, UndoStack, early game-controller/game integration |
| **Umaima** | Board, LinkedList, pieces, positions, UI components, game integration, later restructuring and build/documentation work |

---

## ⚙️ Build & Run

### Requirements
* Windows
* C++17-compatible compiler (MinGW-w64 / GCC)
* GNU Make
* Raylib 5.5

### Build
```powershell
mingw32-make
```

### Run
```powershell
.\main.exe
```

### Clean
```powershell
mingw32-make clean
```

> **Note:** the Makefile targets Windows (MinGW + vcpkg), but the source itself is portable standard C++ — it compiles and runs cleanly on Linux too against a native Raylib 5.5 build, with no Windows-specific code paths. Every screenshot and the GIF above were captured from a Linux build of this exact source, driven with simulated input — not mockups.

<details>
<summary><b>How the gameplay GIF/screenshots were actually captured</b> (worth reading if you're curious)</summary>

<br>

raylib 5.5 was built from source and this repo's `src/*.cpp` compiled against it natively on Linux — zero code changes needed. The binary was then run under a virtual display (`Xvfb`) with software OpenGL rendering, driven with real synthetic mouse/keyboard events (`xdotool`) to click through the menu and actually play, while `ImageMagick` captured frames. The welcome screen, HUD, pause screen, and game-over/leaderboard screen are all genuine output from your own compiled code.

One honest limitation: getting a real **line clear** on camera would need the capture script to read the board state back and plan placements against it — it was only sending blind keystrokes with no feedback loop, so it could stack pieces close (a full row got left with just 1–2 open cells in testing) but couldn't reliably finish one. The hard-drop/rotation GIF above and the game-over screenshot are both genuine play; a completed line just didn't land in the samples taken. Worth a real recording of your own if you want that exact moment on screen.

</details>

---

<div align="center">

### 🎮 Built as a practical application of Data Structures & Algorithms

**C++17 • Raylib • Custom Data Structures**

</div>
