# Scene Store & Collections

`@pascal-app/core` provides two Zustand stores: `useScene` for the scene graph and `useInteractive` for runtime interactive item state.

## Capabilities

### useScene — Scene Graph Store

The primary data store. Uses a flat dictionary (`nodes: Record<AnyNodeId, AnyNode>`) with `zundo`-based undo/redo.

```typescript { .api }
import { useScene, clearSceneHistory } from '@pascal-app/core'

// useScene is a named export — SceneState type is available via ReturnType<typeof useScene.getState>
const useScene: UseBoundStore<StoreApi<SceneState>> & {
  temporal: StoreApi<TemporalState<Pick<SceneState, 'nodes' | 'rootNodeIds' | 'collections'>>>
}
```

**State:**

```typescript { .api }
interface SceneState {
  // Flat dictionary of all nodes keyed by ID
  nodes: Record<AnyNodeId, AnyNode>

  // IDs of root-level nodes (typically the SiteNode ID)
  rootNodeIds: AnyNodeId[]

  // Set of node IDs needing geometry recalculation
  dirtyNodes: Set<AnyNodeId>

  // Named collections keyed by CollectionId
  collections: Record<CollectionId, Collection>

  // --- Scene lifecycle ---
  loadScene(): void
  clearScene(): void
  unloadScene(): void
  setScene(nodes: Record<AnyNodeId, AnyNode>, rootNodeIds: AnyNodeId[]): void

  // --- Dirty tracking (used by systems) ---
  markDirty(id: AnyNodeId): void
  clearDirty(id: AnyNodeId): void

  // --- Node CRUD ---
  createNode(node: AnyNode, parentId?: AnyNodeId): void
  createNodes(ops: { node: AnyNode; parentId?: AnyNodeId }[]): void
  updateNode(id: AnyNodeId, data: Partial<AnyNode>): void
  updateNodes(updates: { id: AnyNodeId; data: Partial<AnyNode> }[]): void
  deleteNode(id: AnyNodeId): void
  deleteNodes(ids: AnyNodeId[]): void

  // --- Collection management ---
  createCollection(name: string, nodeIds?: AnyNodeId[]): CollectionId
  deleteCollection(id: CollectionId): void
  updateCollection(id: CollectionId, data: Partial<Omit<Collection, 'id'>>): void
  addToCollection(id: CollectionId, nodeId: AnyNodeId): void
  removeFromCollection(id: CollectionId, nodeId: AnyNodeId): void
}
```

### Scene Lifecycle Methods

```typescript { .api }
/**
 * Initialize default scene: Site → Building → Level (level 0).
 * Idempotent — if scene already loaded, marks all nodes dirty.
 */
loadScene(): void

/**
 * Reset to default scene (calls unloadScene then loadScene).
 */
clearScene(): void

/**
 * Clear all nodes, rootNodeIds, dirtyNodes, and collections.
 */
unloadScene(): void

/**
 * Load a serialized scene. Runs backward-compatibility migrations
 * (item scale, old roof format) and marks all nodes dirty.
 */
setScene(nodes: Record<AnyNodeId, AnyNode>, rootNodeIds: AnyNodeId[]): void
```

### Node CRUD Methods

```typescript { .api }
/**
 * Create a single node. Sets node.parentId = parentId.
 * Adds node to parent's children array (if parent has children).
 * Adds to rootNodeIds if no parentId. Marks both node and parent dirty.
 */
createNode(node: AnyNode, parentId?: AnyNodeId): void

/**
 * Batch create multiple nodes atomically.
 * ops: Array of { node, parentId? }
 */
createNodes(ops: { node: AnyNode; parentId?: AnyNodeId }[]): void

/**
 * Partial update a node. Handles reparenting if data.parentId changes:
 * removes from old parent's children, adds to new parent's children.
 * Marks node and affected parents dirty (via requestAnimationFrame).
 */
updateNode(id: AnyNodeId, data: Partial<AnyNode>): void

/**
 * Batch update multiple nodes.
 * updates: Array of { id, data }
 */
updateNodes(updates: { id: AnyNodeId; data: Partial<AnyNode> }[]): void

/**
 * Delete a node. Removes from parent's children and rootNodeIds.
 * Removes from all collections. Recursively deletes child nodes.
 * Marks parent and remaining siblings dirty.
 */
deleteNode(id: AnyNodeId): void

/**
 * Batch delete nodes.
 */
deleteNodes(ids: AnyNodeId[]): void
```

**Usage Examples:**

```typescript
import { useScene, WallNode, ItemNode } from '@pascal-app/core'

const store = useScene.getState()

// Create a wall
const wall = WallNode.parse({ start: [0, 0], end: [3, 0] })
store.createNode(wall, levelId)

// Update wall height
store.updateNode(wall.id, { height: 3.2 })

// Delete wall (and recursively all children)
store.deleteNode(wall.id)

// Batch create multiple nodes
const walls = [
  WallNode.parse({ start: [0, 0], end: [5, 0] }),
  WallNode.parse({ start: [5, 0], end: [5, 4] }),
]
store.createNodes(walls.map(n => ({ node: n, parentId: levelId })))
```

### Dirty Node Tracking

```typescript { .api }
/**
 * Mark a node as needing geometry update.
 * Systems check dirtyNodes each frame and process them.
 */
markDirty(id: AnyNodeId): void

/**
 * Clear dirty flag after geometry has been updated.
 */
clearDirty(id: AnyNodeId): void
```

### Undo/Redo (Temporal State)

The store uses `zundo` for undo/redo, tracking `nodes`, `rootNodeIds`, and `collections`. History limit: 50 entries.

```typescript { .api }
// Access temporal store
useScene.temporal  // StoreApi<TemporalState<...>>

// Read undo/redo state
const { pastStates, futureStates } = useScene.temporal.getState()

// Undo/redo
useScene.temporal.getState().undo()
useScene.temporal.getState().redo()

// Clear history
useScene.temporal.getState().clear()

// Pause/resume history recording (e.g., during batch operations)
useScene.temporal.getState().pause()
useScene.temporal.getState().resume()

/**
 * Clears temporal history and resets internal diff tracking state.
 * Call when loading a new scene from server to avoid stale undo history.
 */
function clearSceneHistory(): void
```

**Usage Example:**

```typescript
import { useScene, clearSceneHistory } from '@pascal-app/core'

// Undo last action
useScene.temporal.getState().undo()

// Redo
useScene.temporal.getState().redo()

// Batch multiple mutations as one undo step
useScene.temporal.getState().pause()
useScene.getState().createNode(wall1, levelId)
useScene.getState().createNode(wall2, levelId)
useScene.temporal.getState().resume()

// After loading scene from server
store.setScene(savedNodes, savedRootIds)
clearSceneHistory()
```

### React Subscription

```typescript
import { useScene } from '@pascal-app/core'

// Subscribe with selector (re-renders only when walls change)
function WallList() {
  const nodes = useScene(state => state.nodes)
  const walls = Object.values(nodes).filter(n => n.type === 'wall')
  return <ul>{walls.map(w => <li key={w.id}>{w.id}</li>)}</ul>
}

// Read state outside React
const { nodes, rootNodeIds } = useScene.getState()

// Subscribe outside React (Zustand v5 API — single listener, no selector overload)
const unsub = useScene.subscribe((state, prevState) => {
  if (state.nodes !== prevState.nodes) {
    console.log('nodes changed', Object.keys(state.nodes).length)
  }
})
```

### Collection Management

Collections group nodes by name for organizational purposes.

```typescript { .api }
/**
 * Create a named collection, optionally with initial nodeIds.
 * Returns the new CollectionId.
 * Denormalizes: stamps collectionId onto each node's collectionIds array.
 */
createCollection(name: string, nodeIds?: AnyNodeId[]): CollectionId

/**
 * Delete a collection. Removes collectionId from all member nodes.
 */
deleteCollection(id: CollectionId): void

/**
 * Partial update a collection (name, color, etc.).
 */
updateCollection(id: CollectionId, data: Partial<Omit<Collection, 'id'>>): void

/**
 * Add a node to a collection (idempotent).
 * Updates both the collection's nodeIds and the node's collectionIds.
 */
addToCollection(id: CollectionId, nodeId: AnyNodeId): void

/**
 * Remove a node from a collection.
 */
removeFromCollection(id: CollectionId, nodeId: AnyNodeId): void
```

---

## useInteractive — Item Interactive State

Zustand store managing runtime control values for interactive items (e.g., dimmable lights, animated doors).

```typescript { .api }
import { useInteractive, type ControlValue, type ItemInteractiveState } from '@pascal-app/core'

type ControlValue = boolean | number

interface ItemInteractiveState {
  // Indexed by control position in asset.interactive.controls[]
  controlValues: ControlValue[]
}

interface InteractiveStore {
  items: Record<AnyNodeId, ItemInteractiveState>

  /**
   * Initialize a node's interactive state from its asset.interactive definition.
   * Idempotent — safe to call multiple times, won't overwrite existing state.
   * Note: If interactive.controls is empty (length 0), this is a no-op — no state is stored.
   */
  initItem(itemId: AnyNodeId, interactive: Interactive): void

  /**
   * Set a single control value by index.
   * @param index - position in asset.interactive.controls[]
   */
  setControlValue(itemId: AnyNodeId, index: number, value: ControlValue): void

  /**
   * Remove an item's interactive state (call on item unmount).
   */
  removeItem(itemId: AnyNodeId): void
}

const useInteractive: UseBoundStore<StoreApi<InteractiveStore>>
```

**Usage Example:**

```typescript
import { useInteractive } from '@pascal-app/core'

// In a React component rendering an interactive item
function InteractiveItem({ itemNode }) {
  useEffect(() => {
    if (itemNode.asset.interactive) {
      useInteractive.getState().initItem(itemNode.id, itemNode.asset.interactive)
    }
    return () => useInteractive.getState().removeItem(itemNode.id)
  }, [itemNode.id])

  const controlValues = useInteractive(
    state => state.items[itemNode.id]?.controlValues ?? []
  )

  const handleToggle = () => {
    // Toggle the first control (index 0)
    useInteractive.getState().setControlValue(itemNode.id, 0, !controlValues[0])
  }

  return <button onClick={handleToggle}>Toggle</button>
}
```

**Default control values** (set by `initItem`):
- `ToggleControl` — `control.default ?? false`
- `SliderControl` — `control.default ?? control.min`
- `TemperatureControl` — `control.default ?? control.min`
