# Asset Storage & Space Detection

## Capabilities

### Asset Storage — IndexedDB File Storage

`saveAsset` and `loadAssetUrl` provide persistent storage for user-uploaded 3D model files and other assets using IndexedDB via `idb-keyval`. Assets are stored under a custom `asset://` URL scheme and cached in memory to prevent object URL leaks.

```typescript { .api }
import { saveAsset, loadAssetUrl } from '@pascal-app/core'

/**
 * Save a File to IndexedDB.
 * @returns A custom URL string: 'asset://<uuid>'
 * Store this URL in ItemNode.asset.src or ItemNode.asset.thumbnail.
 */
function saveAsset(file: File): Promise<string>

/**
 * Load a file from IndexedDB and return an object URL for use in Three.js loaders.
 * Handles multiple URL types:
 * - 'blob:...' or 'http...' URLs — returned as-is
 * - 'asset://<uuid>' — fetched from IndexedDB, object URL created and cached
 * - legacy data URLs — returned as-is
 * - empty/null — returns null
 * - missing asset — returns null (logs a warning)
 *
 * @param url - the URL stored in the asset (e.g. asset.src)
 * @returns object URL string, or null if not found
 */
function loadAssetUrl(url: string): Promise<string | null>
```

**Usage Example:**

```typescript
import { saveAsset, loadAssetUrl, ItemNode, useScene } from '@pascal-app/core'

// Upload a user-provided GLB file
async function importModel(file: File, levelId: string) {
  const assetUrl = await saveAsset(file)

  const item = ItemNode.parse({
    asset: {
      id: crypto.randomUUID(),
      category: 'furniture',
      name: file.name,
      thumbnail: '',
      src: assetUrl,          // 'asset://<uuid>'
      dimensions: [1, 1, 1],
    },
    position: [0, 0, 0],
  })

  useScene.getState().createNode(item, levelId)
}

// Load for Three.js GLTFLoader
async function loadModel(item: ItemNode) {
  const url = await loadAssetUrl(item.asset.src)
  if (!url) return
  const gltf = await loader.loadAsync(url)
  // ...
}
```

**Notes:**
- Object URLs created by `loadAssetUrl` are cached internally — calling it multiple times for the same asset returns the same URL without creating duplicate object URLs.
- The custom `asset://` prefix is required for persistence. Standard `blob:` URLs are not persistent across sessions.

---

## Space Detection

The space detection system analyzes wall topology to detect enclosed rooms/spaces on a floor level. It uses a discrete grid flood-fill algorithm to classify grid cells as interior, exterior, or wall, then assigns `frontSide`/`backSide` classification to each wall and extracts space polygons.

### Space Type

```typescript { .api }
import type { Space } from '@pascal-app/core'

interface Space {
  id: string                         // 'space-0', 'space-1', ...
  levelId: string
  polygon: Array<[number, number]>   // bounding box polygon of the space [x, z]
  wallIds: string[]                  // wall IDs bounding this space (currently empty)
  isExterior: boolean                // false for interior spaces
}
```

### detectSpacesForLevel

Direct space detection for a set of walls on a level.

```typescript { .api }
import { detectSpacesForLevel } from '@pascal-app/core'

/**
 * Detects enclosed spaces on a level using flood-fill grid analysis.
 * @param levelId - ID of the level
 * @param walls - array of WallNode instances on the level
 * @param gridResolution - grid cell size in meters (default: 0.5)
 * @returns wallUpdates and detected spaces
 */
function detectSpacesForLevel(
  levelId: string,
  walls: WallNode[],
  gridResolution?: number,
): {
  wallUpdates: Array<{
    wallId: string
    frontSide: 'interior' | 'exterior' | 'unknown'
    backSide:  'interior' | 'exterior' | 'unknown'
  }>
  spaces: Space[]
}
```

**Algorithm:**
1. Builds a discrete grid from wall positions + 2m boundary padding.
2. Marks cells occupied by walls (using wall thickness for rasterization).
3. Flood-fills from all grid edge cells to mark exterior space.
4. Remaining unmarked cells are interior spaces — each connected region becomes a `Space`.
5. Samples perpendicular to each wall midpoint to classify `frontSide`/`backSide`.

**Usage Example:**

```typescript
import { detectSpacesForLevel, useScene } from '@pascal-app/core'

const nodes = useScene.getState().nodes
const walls = Object.values(nodes)
  .filter(n => n.type === 'wall' && n.parentId === levelId)

const { wallUpdates, spaces } = detectSpacesForLevel(levelId, walls)

// Apply wall side updates
for (const update of wallUpdates) {
  useScene.getState().updateNode(update.wallId as any, {
    frontSide: update.frontSide,
    backSide: update.backSide,
  })
}

console.log('Detected spaces:', spaces)
```

### initSpaceDetectionSync

Automatic space detection that subscribes to `useScene` changes and runs detection when walls are added or removed.

```typescript { .api }
import { initSpaceDetectionSync } from '@pascal-app/core'

/**
 * Initialize automatic space detection sync.
 * Subscribe to scene changes and run space detection when wall topology changes.
 *
 * @param sceneStore - the useScene Zustand store
 * @param editorStore - an editor store with a setSpaces(spaces: Record<string, Space>) action
 * @returns unsubscribe function — call to stop syncing
 */
function initSpaceDetectionSync(
  sceneStore: typeof useScene,
  editorStore: { getState(): { setSpaces(spaces: Record<string, any>): void } },
): () => void
```

**Trigger conditions:**
- A new wall is added that touches other walls (within 0.1m threshold).
- An existing wall is deleted from a level that had multiple walls.

**Implementation notes:**
- Calls `sceneStore.temporal.getState().pause()` / `.resume()` during space detection to avoid recording detection side-effects in undo history.
- Only fires when wall topology changes — position-only updates do not re-trigger.

**Usage:**

```typescript
import { useScene, initSpaceDetectionSync } from '@pascal-app/core'
import useEditor from './stores/use-editor'  // your editor store

// Call once at app startup
const unsubscribe = initSpaceDetectionSync(useScene, useEditor)

// To stop:
// unsubscribe()
```

### wallTouchesOthers

Utility to check if a wall is topologically connected to any other walls. Used internally by `initSpaceDetectionSync` to decide whether space detection should run.

```typescript { .api }
import { wallTouchesOthers } from '@pascal-app/core'

/**
 * Returns true if any endpoint of `wall` is within 0.1m of any point
 * on any wall in `otherWalls`.
 * @param wall - the wall to test
 * @param otherWalls - other walls to test against
 */
function wallTouchesOthers(wall: WallNode, otherWalls: WallNode[]): boolean
```
