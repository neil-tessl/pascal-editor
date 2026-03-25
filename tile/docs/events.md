# Event Bus

`@pascal-app/core` exports a `mitt`-based typed event emitter (`emitter`) for all pointer interactions on scene nodes, camera controls, tool actions, and thumbnail generation.

## Capabilities

### emitter — Typed Event Emitter

```typescript { .api }
import { emitter, eventSuffixes, type EventSuffix } from '@pascal-app/core'

const emitter: Emitter<EditorEvents>

const eventSuffixes: readonly [
  'click', 'move', 'enter', 'leave',
  'pointerdown', 'pointerup', 'context-menu', 'double-click'
]

type EventSuffix = 'click' | 'move' | 'enter' | 'leave' |
                   'pointerdown' | 'pointerup' | 'context-menu' | 'double-click'
```

### Event Key Pattern

Node events follow the pattern `'<nodeType>:<suffix>'`:

| Node Type | Example Event Keys |
|-----------|-------------------|
| `wall` | `'wall:click'`, `'wall:move'`, `'wall:enter'`, ... |
| `item` | `'item:click'`, `'item:pointerdown'`, ... |
| `site` | `'site:click'`, ... |
| `building` | `'building:click'`, ... |
| `level` | `'level:click'`, ... |
| `zone` | `'zone:click'`, ... |
| `slab` | `'slab:click'`, ... |
| `ceiling` | `'ceiling:click'`, ... |
| `roof` | `'roof:click'`, ... |
| `roof-segment` | `'roof-segment:click'`, ... |
| `window` | `'window:click'`, ... |
| `door` | `'door:click'`, ... |
| `grid` | `'grid:click'`, `'grid:move'`, ... |

### Node Event Types

```typescript { .api }
import type {
  NodeEvent, GridEvent,
  WallEvent, ItemEvent, SiteEvent, BuildingEvent, LevelEvent,
  ZoneEvent, SlabEvent, CeilingEvent, RoofEvent, RoofSegmentEvent,
  WindowEvent, DoorEvent,
  CameraControlEvent,
} from '@pascal-app/core'

interface GridEvent {
  position: [number, number, number]   // world position
  nativeEvent: ThreeEvent<PointerEvent>
}

interface NodeEvent<T extends AnyNode = AnyNode> {
  node: T
  position: [number, number, number]       // world position
  localPosition: [number, number, number]  // local position on the node
  normal?: [number, number, number]        // surface normal
  stopPropagation: () => void
  nativeEvent: ThreeEvent<PointerEvent>
}

// Concrete node event types
type WallEvent         = NodeEvent<WallNode>
type ItemEvent         = NodeEvent<ItemNode>
type SiteEvent         = NodeEvent<SiteNode>
type BuildingEvent     = NodeEvent<BuildingNode>
type LevelEvent        = NodeEvent<LevelNode>
type ZoneEvent         = NodeEvent<ZoneNode>
type SlabEvent         = NodeEvent<SlabNode>
type CeilingEvent      = NodeEvent<CeilingNode>
type RoofEvent         = NodeEvent<RoofNode>
type RoofSegmentEvent  = NodeEvent<RoofSegmentNode>
type WindowEvent       = NodeEvent<WindowNode>
type DoorEvent         = NodeEvent<DoorNode>
```

### Camera Control Events

```typescript { .api }
interface CameraControlEvent {
  nodeId: AnyNode['id']
}

// Camera control event keys and payloads:
// 'camera-controls:view'               → CameraControlEvent
// 'camera-controls:focus'              → CameraControlEvent
// 'camera-controls:capture'            → CameraControlEvent
// 'camera-controls:top-view'           → undefined
// 'camera-controls:orbit-cw'           → undefined
// 'camera-controls:orbit-ccw'          → undefined
// 'camera-controls:generate-thumbnail' → { projectId: string }
```

### Tool & Preset Events

```typescript { .api }
// Tool events:
// 'tool:cancel' → undefined

// Preset events:
// 'preset:generate-thumbnail' → { presetId: string; nodeId: string }
// 'preset:thumbnail-updated'  → { presetId: string; thumbnailUrl: string }
```

### Usage

```typescript
import { emitter, eventSuffixes, type WallEvent, type GridEvent } from '@pascal-app/core'

// Listen for wall clicks
emitter.on('wall:click', (event: WallEvent) => {
  console.log('Clicked wall:', event.node.id)
  console.log('World position:', event.position)
  event.stopPropagation()
})

// Listen for grid pointer move (e.g., for placement preview)
emitter.on('grid:move', (event: GridEvent) => {
  console.log('Grid position:', event.position)
})

// Emit a camera control event
emitter.emit('camera-controls:focus', { nodeId: wall.id })

// Emit tool cancel
emitter.emit('tool:cancel', undefined)

// Unsubscribe
const handler = (event: WallEvent) => { ... }
emitter.on('wall:click', handler)
emitter.off('wall:click', handler)

// Dynamically build event keys for all suffixes
for (const suffix of eventSuffixes) {
  emitter.on(`wall:${suffix}`, handler)
}
```
