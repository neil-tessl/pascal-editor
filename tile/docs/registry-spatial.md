# Scene Registry & Spatial Grid

## Capabilities

### sceneRegistry — Three.js Object Lookup

Singleton providing O(1) bidirectional lookup between node IDs and Three.js `Object3D` instances. Used by geometry systems and any code that needs to access 3D objects from the scene graph.

```typescript { .api }
import { sceneRegistry } from '@pascal-app/core'

const sceneRegistry: {
  /** Master lookup: node ID → Three.js Object3D */
  nodes: Map<string, THREE.Object3D>

  /** Categorized lookup: node type → Set of node IDs */
  byType: {
    site:           Set<string>
    building:       Set<string>
    ceiling:        Set<string>
    level:          Set<string>
    wall:           Set<string>
    item:           Set<string>
    slab:           Set<string>
    zone:           Set<string>
    roof:           Set<string>
    'roof-segment': Set<string>
    scan:           Set<string>
    guide:          Set<string>
    window:         Set<string>
    door:           Set<string>
  }

  /**
   * Remove all entries. Call when unloading a scene to prevent stale 3D refs.
   */
  clear(): void
}
```

**Usage:**

```typescript
import { sceneRegistry } from '@pascal-app/core'
import * as THREE from 'three'

// Get a Three.js object by node ID
const mesh = sceneRegistry.nodes.get(wallId) as THREE.Mesh

// Get all wall Object3D instances
for (const id of sceneRegistry.byType.wall) {
  const obj = sceneRegistry.nodes.get(id)
}

// Clear on scene unload
sceneRegistry.clear()
```

### useRegistry — React Registration Hook

React hook that registers a Three.js `ref` in `sceneRegistry` on mount and unregisters on unmount.

```typescript { .api }
import { useRegistry } from '@pascal-app/core'

/**
 * Register a Three.js Object3D in the scene registry.
 * Call inside a React Three Fiber component.
 *
 * @param id - node ID (e.g., wall.id)
 * @param type - node type key matching sceneRegistry.byType
 * @param ref - React ref containing the Three.js Object3D
 */
function useRegistry(
  id: string,
  type: keyof typeof sceneRegistry.byType,
  ref: React.RefObject<THREE.Object3D>
): void
```

**Usage:**

```typescript
import { useRegistry } from '@pascal-app/core'
import { useRef } from 'react'
import * as THREE from 'three'

function WallMesh({ wall }: { wall: WallNode }) {
  const meshRef = useRef<THREE.Mesh>(null)
  useRegistry(wall.id, 'wall', meshRef)

  return <mesh ref={meshRef} />
}
```

---

### spatialGridManager — Placement Validation

Singleton managing per-level occupancy grids for floor items, wall items, and ceiling items. Used to validate item placement and detect collisions.

```typescript { .api }
import { spatialGridManager } from '@pascal-app/core'

// spatialGridManager is a singleton instance — the SpatialGridManager class
// is not directly exported. Use the singleton for all spatial queries.
const spatialGridManager: SpatialGridManager
```

**`SpatialGridManager` — singleton interface:**

```typescript { .api }
class SpatialGridManager {
  constructor(cellSize?: number)  // default 0.5 meters

  // --- Node registration (called by initSpatialGridSync) ---

  /**
   * Register a newly created node in the appropriate grid.
   * Handles: 'slab', 'ceiling', 'wall', 'item' (floor/wall/ceiling attachment).
   */
  handleNodeCreated(node: AnyNode, levelId: string): void

  /**
   * Update a node's placement in the grid after position/rotation change.
   */
  handleNodeUpdated(node: AnyNode, levelId: string): void

  /**
   * Remove a node from the grid.
   * @returns Array of item IDs removed when a wall is deleted.
   */
  handleNodeDeleted(nodeId: string, nodeType: string, levelId: string): string[]

  // --- Placement queries ---

  /**
   * Check if a floor item can be placed at the given position.
   * @param levelId - ID of the level
   * @param position - [x, y, z] world position
   * @param dimensions - [w, h, d] item dimensions
   * @param rotation - [x, y, z] Euler rotation
   * @param ignoreIds - item IDs to exclude from collision check
   */
  canPlaceOnFloor(
    levelId: string,
    position: [number, number, number],
    dimensions: [number, number, number],
    rotation: [number, number, number],
    ignoreIds?: string[],
  ): { valid: boolean; conflictIds: string[] }

  /**
   * Check if an item can be placed on a wall.
   * @param levelId - ID of the level
   * @param wallId - ID of the target wall
   * @param localX - X position in wall-local space (distance from wall start along the wall)
   * @param localY - Y position (height from floor)
   * @param dimensions - [w, h, d] item dimensions
   * @param attachType - 'wall' (through-wall, needs both sides) or 'wall-side' (one side only)
   * @param side - 'front' | 'back' for wall-side items
   * @param ignoreIds - item IDs to exclude from collision check
   */
  canPlaceOnWall(
    levelId: string,
    wallId: string,
    localX: number,
    localY: number,
    dimensions: [number, number, number],
    attachType?: 'wall' | 'wall-side',
    side?: 'front' | 'back',
    ignoreIds?: string[],
  ): { valid: boolean; conflictIds: string[] }

  /**
   * Check if an item can be placed on a ceiling.
   * Validates that the footprint is within the ceiling polygon (not in holes)
   * and doesn't overlap other ceiling items.
   */
  canPlaceOnCeiling(
    ceilingId: string,
    position: [number, number, number],
    dimensions: [number, number, number],
    rotation: [number, number, number],
    ignoreIds?: string[],
  ): { valid: boolean; conflictIds: string[] }

  /**
   * Get the wall ID for an item that is wall-attached.
   */
  getWallForItem(levelId: string, itemId: string): string | undefined

  /**
   * Get slab surface elevation at a world (x, z) point on a level.
   * Returns the highest slab elevation at that point, or 0 if no slab.
   */
  getSlabElevationAt(levelId: string, x: number, z: number): number

  /**
   * Get slab elevation for an item's full footprint.
   * Returns the highest overlapping slab elevation, or 0 if none.
   */
  getSlabElevationForItem(
    levelId: string,
    position: [number, number, number],
    dimensions: [number, number, number],
    rotation: [number, number, number],
  ): number

  /**
   * Get slab elevation for a wall segment.
   * Returns the highest overlapping slab elevation, or 0 if none.
   */
  getSlabElevationForWall(
    levelId: string,
    start: [number, number],
    end: [number, number],
  ): number

  /** Clear grids for a single level. */
  clearLevel(levelId: string): void

  /** Clear all grids. */
  clear(): void
}
```

### initSpatialGridSync

Initializes the `spatialGridManager` singleton to automatically sync with `useScene` store changes. Call once at application startup.

```typescript { .api }
import { initSpatialGridSync } from '@pascal-app/core'

/**
 * Syncs spatialGridManager with the useScene store.
 * - Processes all existing nodes in the current scene.
 * - Subscribes to future node additions, updates, and deletions.
 * - Automatically marks affected nodes dirty on slab changes.
 *
 * Call once during app initialization (before first render).
 */
function initSpatialGridSync(): void
```

**Usage:**

```typescript
import { initSpatialGridSync } from '@pascal-app/core'

// In your app entry point (e.g., layout.tsx or _app.tsx)
initSpatialGridSync()
```

### resolveLevelId

Walks up the parent chain from any node to find the containing level's ID.

```typescript { .api }
import { resolveLevelId } from '@pascal-app/core'

/**
 * Find the level ID for any node by walking up parentId chain.
 * @returns level ID string, or 'default' if not found.
 */
function resolveLevelId(node: AnyNode, nodes: Record<string, AnyNode>): string
```

### useSpatialQuery — React Hook

React hook that exposes memoized placement-check callbacks for use in React components.

```typescript { .api }
import { useSpatialQuery } from '@pascal-app/core'

function useSpatialQuery(): {
  canPlaceOnFloor(
    levelId: LevelNode['id'],
    position: [number, number, number],
    dimensions: [number, number, number],
    rotation: [number, number, number],
    ignoreIds?: string[],
  ): { valid: boolean; conflictIds: string[] }

  canPlaceOnWall(
    levelId: LevelNode['id'],
    wallId: WallNode['id'],
    localX: number,
    localY: number,
    dimensions: [number, number, number],
    attachType?: 'wall' | 'wall-side',
    side?: 'front' | 'back',
    ignoreIds?: string[],
  ): { valid: boolean; conflictIds: string[] }

  canPlaceOnCeiling(
    ceilingId: CeilingNode['id'],
    position: [number, number, number],
    dimensions: [number, number, number],
    rotation: [number, number, number],
    ignoreIds?: string[],
  ): { valid: boolean; conflictIds: string[] }
}
```

**Usage Example:**

```typescript
import { useSpatialQuery } from '@pascal-app/core'

function ItemPlacementTool({ levelId, wallId }) {
  const { canPlaceOnFloor, canPlaceOnWall } = useSpatialQuery()

  const tryPlaceFloor = (position, dimensions, rotation) => {
    const result = canPlaceOnFloor(levelId, position, dimensions, rotation)
    if (result.valid) {
      // Place item
    } else {
      console.log('Conflicts with:', result.conflictIds)
    }
  }

  const tryPlaceOnWall = (localX, localY, dimensions) => {
    const result = canPlaceOnWall(levelId, wallId, localX, localY, dimensions, 'wall')
    return result
  }
}
```

### pointInPolygon — Geometry Utility

```typescript { .api }
import { pointInPolygon } from '@pascal-app/core'

/**
 * Ray-casting point-in-polygon test.
 * @param px - point X coordinate
 * @param pz - point Z coordinate
 * @param polygon - array of [x, z] polygon vertices
 * @returns true if point is inside the polygon
 */
function pointInPolygon(px: number, pz: number, polygon: Array<[number, number]>): boolean
```
