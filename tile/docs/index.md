# @pascal-app/core

`@pascal-app/core` is the core library for the Pascal 3D building editor. It provides the complete data layer and geometry systems for creating and managing architectural scenes: node schemas for all building primitives (walls, slabs, roofs, doors, windows, items, etc.), a Zustand-based scene store with full undo/redo history, geometry-update systems for use with React Three Fiber, a typed event bus, a spatial grid for collision/placement queries, and IndexedDB-backed asset storage.

## Package Information

- **Package Name**: `@pascal-app/core`
- **Package Type**: npm (scoped)
- **Language**: TypeScript (ESM only)
- **Installation**: `npm install @pascal-app/core`
- **Peer Dependencies**: `npm install react three @react-three/fiber @react-three/drei`

## Core Imports

```typescript
import {
  // Scene store
  useScene,
  clearSceneHistory,
  // Node schemas
  SiteNode, BuildingNode, LevelNode, WallNode, SlabNode, CeilingNode,
  RoofNode, RoofSegmentNode, ItemNode, ZoneNode, DoorNode, WindowNode,
  ScanNode, GuideNode,
  // Schema utilities
  AnyNode, generateId,
  // Systems (React components)
  WallSystem, SlabSystem, CeilingSystem, RoofSystem, ItemSystem, DoorSystem, WindowSystem,
  // Event bus
  emitter, eventSuffixes,
  // Registry
  sceneRegistry, useRegistry,
  // Spatial grid
  spatialGridManager, useSpatialQuery, initSpatialGridSync,
  // Asset storage
  saveAsset, loadAssetUrl,
  // Space detection
  detectSpacesForLevel, initSpaceDetectionSync,
} from '@pascal-app/core'
```

## Basic Usage

```typescript
import { useScene, WallNode, LevelNode, SiteNode, BuildingNode } from '@pascal-app/core'

// Initialize the default scene (Site → Building → Level hierarchy)
useScene.getState().loadScene()

// Read scene state
const { nodes, rootNodeIds } = useScene.getState()
const site = Object.values(nodes).find(n => n.type === 'site')

// Create a wall (must parse to fill in defaults)
const wall = WallNode.parse({
  start: [0, 0],
  end: [5, 0],
  height: 2.5,
  thickness: 0.2,
})
const levelId = Object.values(nodes).find(n => n.type === 'level')!.id
useScene.getState().createNode(wall, levelId)

// React subscription
function WallCount() {
  const nodes = useScene(state => state.nodes)
  const wallCount = Object.values(nodes).filter(n => n.type === 'wall').length
  return <div>Walls: {wallCount}</div>
}
```

## Architecture

- **Node Schemas** — Zod schemas for all building primitives, used for parse/validate and as TypeScript types
- **Scene Store** — Zustand flat-dictionary store (`nodes: Record<id, AnyNode>`) with `zundo` temporal undo/redo, up to 50 history entries
- **Systems** — React components using `useFrame` to process dirty nodes and update Three.js geometries each frame
- **Scene Registry** — Fast bidirectional lookup between node IDs and Three.js `Object3D` instances
- **Spatial Grid** — Per-level occupancy grids for floor, wall, and ceiling item placement validation
- **Event Bus** — `mitt`-based typed event emitter for pointer events on nodes and camera control events

## Capabilities

### Node Schemas & Scene Graph

All 14 building node types as Zod schemas with TypeScript types. The scene graph is a flat dictionary keyed by node ID, with parent-child references via `parentId` and typed `children` arrays.

```typescript { .api }
// All node schemas are Zod schemas + TypeScript types
const WallNode: z.ZodObject<...>  // parse, safeParse, extend...
const wall: WallNode = WallNode.parse({ start: [0,0], end: [5,0] })

// AnyNode is a discriminated union on 'type'
type AnyNode = SiteNode | BuildingNode | LevelNode | WallNode | ...
type AnyNodeId = AnyNode['id']
type AnyNodeType = AnyNode['type']

// ID generation
function generateId<T extends string>(prefix: T): `${T}_${string}`
```

[Node Schemas](./nodes.md)

### Scene Store (useScene)

Zustand store managing the flat node dictionary, undo/redo history, and collections. The default export is the Zustand store hook.

```typescript { .api }
// Default export — Zustand store hook
const useScene: UseBoundStore<StoreApi<SceneState>> & {
  temporal: StoreApi<TemporalState<Pick<SceneState, 'nodes' | 'rootNodeIds' | 'collections'>>>
}

interface SceneState {
  nodes: Record<AnyNodeId, AnyNode>
  rootNodeIds: AnyNodeId[]
  dirtyNodes: Set<AnyNodeId>
  collections: Record<CollectionId, Collection>

  loadScene(): void
  clearScene(): void
  unloadScene(): void
  setScene(nodes: Record<AnyNodeId, AnyNode>, rootNodeIds: AnyNodeId[]): void
  markDirty(id: AnyNodeId): void
  clearDirty(id: AnyNodeId): void
  createNode(node: AnyNode, parentId?: AnyNodeId): void
  createNodes(ops: { node: AnyNode; parentId?: AnyNodeId }[]): void
  updateNode(id: AnyNodeId, data: Partial<AnyNode>): void
  updateNodes(updates: { id: AnyNodeId; data: Partial<AnyNode> }[]): void
  deleteNode(id: AnyNodeId): void
  deleteNodes(ids: AnyNodeId[]): void
  createCollection(name: string, nodeIds?: AnyNodeId[]): CollectionId
  deleteCollection(id: CollectionId): void
  updateCollection(id: CollectionId, data: Partial<Omit<Collection, 'id'>>): void
  addToCollection(id: CollectionId, nodeId: AnyNodeId): void
  removeFromCollection(id: CollectionId, nodeId: AnyNodeId): void
}

// Undo/redo
function clearSceneHistory(): void
```

[Scene Store & Collections](./store.md)

### Event Bus

Typed `mitt`-based event emitter for all pointer events on scene nodes and camera/tool events.

```typescript { .api }
import { emitter, eventSuffixes, type EventSuffix } from '@pascal-app/core'

// Event keys: '<nodeType>:<suffix>'
// e.g. 'wall:click', 'item:move', 'grid:pointerdown'
const emitter: Emitter<EditorEvents>

const eventSuffixes: readonly ['click', 'move', 'enter', 'leave',
  'pointerdown', 'pointerup', 'context-menu', 'double-click']
type EventSuffix = (typeof eventSuffixes)[number]
```

[Events](./events.md)

### Scene Registry

Singleton for fast O(1) lookup between node IDs and Three.js `Object3D` instances.

```typescript { .api }
const sceneRegistry: {
  nodes: Map<string, THREE.Object3D>
  byType: Record<NodeType, Set<string>>
  clear(): void
}

function useRegistry(
  id: string,
  type: keyof typeof sceneRegistry.byType,
  ref: React.RefObject<THREE.Object3D>
): void
```

[Scene Registry & Spatial Grid](./registry-spatial.md)

### Spatial Grid & Placement Validation

Per-level occupancy grids for validating item placement on floors, walls, and ceilings.

```typescript { .api }
// Singleton
const spatialGridManager: SpatialGridManager

// React hook returning memoized callbacks
function useSpatialQuery(): {
  canPlaceOnFloor(levelId, position, dimensions, rotation, ignoreIds?): PlacementResult
  canPlaceOnWall(levelId, wallId, localX, localY, dimensions, attachType?, side?, ignoreIds?): PlacementResult
  canPlaceOnCeiling(ceilingId, position, dimensions, rotation, ignoreIds?): PlacementResult
}

interface PlacementResult { valid: boolean; conflictIds: string[] }

// Initialize sync (once at app start)
function initSpatialGridSync(): void
```

[Scene Registry & Spatial Grid](./registry-spatial.md)

### Geometry Systems

React components that process dirty nodes each frame and update Three.js mesh geometries.

```typescript { .api }
// All systems are React components returning null
const WallSystem: () => null
const SlabSystem: () => null
const CeilingSystem: () => null
const RoofSystem: () => null
const ItemSystem: () => null
const DoorSystem: () => null
const WindowSystem: () => null
```

[Systems](./systems.md)

### Asset Storage & Space Detection

IndexedDB asset storage for uploaded files, and algorithmic space (room) detection from wall topology.

```typescript { .api }
function saveAsset(file: File): Promise<string>        // returns 'asset://<uuid>'
function loadAssetUrl(url: string): Promise<string | null>

function detectSpacesForLevel(levelId, walls, gridResolution?): { wallUpdates, spaces }
function initSpaceDetectionSync(sceneStore, editorStore): () => void
function wallTouchesOthers(wall: WallNode, otherWalls: WallNode[]): boolean

interface Space { id: string; levelId: string; polygon: [number,number][]; wallIds: string[]; isExterior: boolean }
```

[Asset Storage & Space Detection](./asset-space.md)

### Interactive Items (useInteractive)

Zustand store for runtime control values of interactive items (lights, animated objects).

```typescript { .api }
const useInteractive: UseBoundStore<StoreApi<InteractiveStore>>

type ControlValue = boolean | number
interface ItemInteractiveState { controlValues: ControlValue[] }
```

[Scene Store & Collections](./store.md)
