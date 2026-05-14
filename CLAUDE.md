# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`rubiks-ttt.html` is a single-file browser game — no build step, no dependencies to install. Open the file directly in a browser or serve it with any static file server. The only external dependency is Three.js r160, loaded from CDN.

## Running the Game

Open `rubiks-ttt.html` directly in a browser, or serve statically. The `.claude/launch.json` has a Python static server config for use with `preview_start`.

## Architecture

Everything lives in one file: inline CSS → HTML shell → `<script>` block. The script is structured as clearly-commented sections (use the `─── Section ───` headers to navigate).

### Data model

The cube is 26 cubies (3×3×3 minus center). Each cubie:
```js
{
  pos: { x, y, z },       // each ∈ {-1, 0, 1}, integer, never all-zero
  stickers: {              // null = inner face (no sticker)
    U: 'empty'|'X'|'O'|null,
    D, F, B, L, R          // same values
  },
  boxMesh: THREE.Mesh,
  stickerMeshes: { U: Mesh, ... }
}
```

Coordinate axes: Y up, X right, Z toward viewer. Face layer values: U=y+1, D=y−1, F=z+1, B=z−1, R=x+1, L=x−1.

### Scene graph

```
THREE.Scene
  └─ orbitGroup (THREE.Group)  ← user drag rotates this
       ├─ 26 black box meshes
       ├─ up to 54 sticker meshes (BoxGeometry, canvas textures)
       ├─ 6 arrow planes (one per face, FrontSide, hidden by default)
       └─ winLineMesh (TubeGeometry, added on game over)
```

**Critical**: the pivot group used during move animation must be a child of `orbitGroup` (not `scene`) so its rotation axes match the data-model coordinate system regardless of how the user has dragged.

### Move animation

`executeMoveAnimation(face, isCW)` uses a temporary pivot group:
1. Filter cubies whose `pos[def.layer] === def.value`
2. `pivot.attach(mesh)` (not `.add()`) for each affected box + sticker mesh — preserves world transform
3. Animate `pivot.rotation[axis]` with easeInOut over `ANIM_MS` (380ms)
4. On complete: `orbitGroup.attach(mesh)` back, round positions to integers, reset rotation, update `c.pos` and `c.stickers` via `applyPosCW/CCW` and `applyFaceMap`
5. Call `buildStickerMeshes()` (full teardown + rebuild — simpler than tracking individual meshes through reparenting)
6. Call `findWin(cubies)` — win is only checked after rotation, never after placement

### Game state machine

```
PLACE → (player clicks sticker) → switch currentPlayer → MOVE
MOVE  → (player/AI clicks move button) → ANIMATING
ANIMATING → (animation done, no win) → PLACE  [currentPlayer unchanged — same player now places]
ANIMATING → (animation done, win) → GAME_OVER
```

**Rule**: the player who places does NOT rotate. After X places, currentPlayer switches to O who must rotate. After O rotates, currentPlayer stays O who then places. The rotation is constrained to layers containing the marked cubie (`currentValidFaces`).

### AI difficulty dispatch

`aiDifficulty`: `'off' | 'easy' | 'medium' | 'hard'`

- **Easy**: fully random placement and rotation
- **Medium**: one-ply — scores all placement×rotation combos, assumes X picks the worst response for O
- **Hard**: two-ply rotation (simulates X's best placement+rotation after O rotates); placement explicitly gives −99999 score to any rotation that lets X win

`aiRotateTurn(validFaces)` fires after X places. `aiPlace()` fires after animation completes when `currentPlayer === 'O'`.

### Win detection

`getFaceStickersOrderedFrom(cs, face)` returns a consistent 9-element array per face using face-specific sort orders (see the switch in that function). `findWin(cs)` returns `{ winner, face, line }` and is used to draw the win line.

### Sticker textures

Pre-created via `makeTexture(mark)` and cached in `textureCache`. Arrow textures are created via `makeArrowTexture(isCW)` — transparent background canvas, yellow arc + arrowhead.

### Rendering notes

- Arrow planes: `FrontSide` only (so they're invisible through the cube), `depthWrite: false`, offset at 1.56 units (just above sticker surface ~1.51)
- Marked cubie outline: `THREE.LineSegments` added as child of `boxMesh` so it animates with the cube
- Win line: opaque `TubeGeometry`, no `renderOrder` override — depth-tested naturally
- Post-game-over rotation works because the overlay has `pointer-events: none`
