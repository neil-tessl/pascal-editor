# Geometry Systems

Geometry systems are React components that use `useFrame` from `@react-three/fiber` to process dirty nodes each frame and update the corresponding Three.js mesh geometries. They must be rendered inside a React Three Fiber `<Canvas>` component.

All systems:
- Subscribe to `useScene` to get `dirtyNodes` and `clearDirty`
- Look up Three.js objects from `sceneRegistry.nodes`
- Generate new geometry for dirty nodes and update the mesh
- Return `null` (no visible output)

## Capabilities

### WallSystem

Processes dirty wall nodes. Generates wall geometry with correct mitering at junctions and CSG boolean cutouts for windows and doors.

```typescript { .api }
import { WallSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'wall' nodes each frame.
 * - Calculates miter junctions for connected walls.
 * - Generates wall mesh geometry.
 * - Performs CSG boolean subtraction for windows and doors.
 */
const WallSystem: () => null
```

### SlabSystem

Processes dirty slab nodes. Generates floor polygon geometry with holes.

```typescript { .api }
import { SlabSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'slab' nodes each frame.
 * - Generates floor polygon geometry from slab.polygon and slab.holes.
 * - Applies SLAB_OUTSET (0.05m) to extend slab geometry under wall thickness.
 */
const SlabSystem: () => null
```

### CeilingSystem

Processes dirty ceiling nodes. Generates ceiling polygon geometry with holes.

```typescript { .api }
import { CeilingSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'ceiling' nodes each frame.
 * - Generates ceiling geometry from ceiling.polygon and ceiling.holes.
 * - Also updates a child 'ceiling-grid' mesh with the same geometry.
 */
const CeilingSystem: () => null
```

### RoofSystem

Processes dirty roof-segment nodes. Generates roof geometry using CSG operations. Throttled to avoid frame drops on complex roofs.

```typescript { .api }
import { RoofSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'roof-segment' nodes (throttled: max 3 segments/frame).
 * Processes pending merged roof updates (throttled: max 1 roof/frame).
 * - Generates individual segment geometry per roof type.
 * - Merges all segments into a combined solid for the parent RoofNode.
 * - Uses CSG (three-bvh-csg) and BVH acceleration (three-mesh-bvh).
 * - Clears pending updates when scene is unloaded (rootNodeIds = []).
 */
const RoofSystem: () => null
```

### ItemSystem

Processes dirty item nodes. Updates item position, including slab elevation compensation.

```typescript { .api }
import { ItemSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'item' nodes each frame.
 * - Updates item mesh position and rotation.
 * - Queries spatialGridManager.getSlabElevationForItem() to lift floor items
 *   above slab surfaces.
 */
const ItemSystem: () => null
```

### DoorSystem

Processes dirty door nodes. Generates parametric door geometry.

```typescript { .api }
import { DoorSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'door' nodes each frame.
 * - Generates door frame, leaf segments (panel/glass/empty), handle,
 *   threshold, door closer, and panic bar geometry based on DoorNode properties.
 */
const DoorSystem: () => null
```

### WindowSystem

Processes dirty window nodes. Generates parametric window geometry.

```typescript { .api }
import { WindowSystem } from '@pascal-app/core'

/**
 * React component. Render once inside a <Canvas>.
 * Processes dirty 'window' nodes each frame.
 * - Generates window frame, pane divisions (columnRatios/rowRatios),
 *   and sill geometry based on WindowNode properties.
 */
const WindowSystem: () => null
```

## Usage

Mount all required systems once inside your React Three Fiber Canvas. The systems are stateless and can be mounted in any order.

```typescript
import {
  WallSystem, SlabSystem, CeilingSystem, RoofSystem,
  ItemSystem, DoorSystem, WindowSystem,
} from '@pascal-app/core'
import { Canvas } from '@react-three/fiber'

function Scene() {
  return (
    <Canvas>
      {/* Geometry systems — process dirty nodes each frame */}
      <WallSystem />
      <SlabSystem />
      <CeilingSystem />
      <RoofSystem />
      <ItemSystem />
      <DoorSystem />
      <WindowSystem />

      {/* Your scene graph renderer */}
      <SceneGraph />
    </Canvas>
  )
}
```

## Wall Geometry Utilities

Helper functions used by WallSystem, also useful for custom rendering logic.

```typescript { .api }
import {
  DEFAULT_WALL_HEIGHT,
  DEFAULT_WALL_THICKNESS,
  getWallThickness,
  getWallPlanFootprint,
  calculateLevelMiters,
  pointToKey,
  type Point2D,
  type WallMiterData,
} from '@pascal-app/core'

/** Default wall height: 2.5 meters */
const DEFAULT_WALL_HEIGHT: number  // 2.5

/** Default wall thickness: 0.1 meters */
const DEFAULT_WALL_THICKNESS: number  // 0.1

/**
 * Returns wallNode.thickness ?? DEFAULT_WALL_THICKNESS.
 */
function getWallThickness(wallNode: WallNode): number

/**
 * Returns the 2D plan footprint polygon of a wall accounting for miter offsets at junctions.
 * Points are in level coordinate system.
 * @param wallNode - the wall node
 * @param miterData - computed from calculateLevelMiters
 * @returns array of Point2D vertices, or [] for zero-length walls
 */
function getWallPlanFootprint(wallNode: WallNode, miterData: WallMiterData): Point2D[]

/**
 * Calculate miter junction data for all walls on a level.
 * Returns junction intersection points that define how wall corners meet.
 * @param walls - all WallNode instances on the level
 */
function calculateLevelMiters(walls: WallNode[]): WallMiterData

/**
 * Convert a Point2D to a deterministic string key for hash maps.
 * Snaps to 0.001m tolerance.
 */
function pointToKey(p: Point2D): string

interface Point2D {
  x: number
  y: number
}

interface WallMiterData {
  /** Junction data keyed by snapped junction position string */
  junctionData: Map<string, Map<string, { left?: Point2D; right?: Point2D }>>
  /** All detected junctions */
  junctions: Map<string, {
    meetingPoint: Point2D
    connectedWalls: Array<{ wall: WallNode; endType: 'start' | 'end' | 'passthrough' }>
  }>
}
```

**Usage Example:**

```typescript
import { calculateLevelMiters, getWallPlanFootprint, useScene } from '@pascal-app/core'

// Get all walls on a level
const nodes = useScene.getState().nodes
const walls = Object.values(nodes).filter(n => n.type === 'wall' && n.parentId === levelId)

// Compute mitering
const miterData = calculateLevelMiters(walls)

// Get the footprint polygon for one wall
const footprint = getWallPlanFootprint(walls[0], miterData)
// footprint: Point2D[] — the 2D plan polygon (XZ plane)
```

## isObject Utility

```typescript { .api }
import { isObject } from '@pascal-app/core'

/**
 * Type guard: returns true if value is a non-null, non-array plain object.
 * Useful for narrowing Zod's generic JSON types.
 */
function isObject(val: unknown): val is Record<string, any>
```
