# Node Schemas

All node schemas are Zod schemas that double as TypeScript types. Every node extends `BaseNode`. Nodes are stored in a flat dictionary keyed by their `id`.

## Capabilities

### BaseNode & Schema Utilities

Common base schema extended by all node types.

```typescript { .api }
import { BaseNode, generateId, objectId, nodeType, Material } from '@pascal-app/core'

const BaseNode: z.ZodObject<{
  object: z.ZodDefault<z.ZodLiteral<'node'>>
  id: z.ZodString
  type: z.ZodDefault<z.ZodLiteral<'node'>>
  name: z.ZodOptional<z.ZodString>
  parentId: z.ZodDefault<z.ZodNullable<z.ZodString>>
  visible: z.ZodOptional<z.ZodDefault<z.ZodBoolean>>
  camera: z.ZodOptional<typeof CameraSchema>
  metadata: z.ZodOptional<z.ZodDefault<z.ZodJson>>
}>

type BaseNode = z.infer<typeof BaseNode>

/**
 * Generates a prefixed nanoid string: `${prefix}_${nanoid(16)}`
 * Used to create typed IDs for nodes.
 */
function generateId<T extends string>(prefix: T): `${T}_${string}`

/**
 * Creates a Zod schema for a typed ID with a generated default value.
 * Usage: objectId('wall') gives a schema for 'wall_...' strings.
 */
function objectId<T extends string>(prefix: T): z.ZodSchema

/**
 * Creates a Zod literal schema with a default equal to the type string.
 * Usage: nodeType('wall') gives z.literal('wall').default('wall')
 */
function nodeType<T extends string>(type: T): z.ZodDefault<z.ZodLiteral<T>>

/**
 * Material preset name reference — optional string for material names like 'white', 'brick'.
 */
const Material: z.ZodOptional<z.ZodString>
```

### CameraSchema

Embedded camera state on any node.

```typescript { .api }
import { CameraSchema } from '@pascal-app/core'

const CameraSchema: z.ZodObject<{
  position: z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>
  target:   z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>
  mode:  z.ZodDefault<z.ZodEnum<['perspective', 'orthographic']>>
  fov:   z.ZodOptional<z.ZodNumber>   // for perspective
  zoom:  z.ZodOptional<z.ZodNumber>   // for orthographic
}>

type Camera = z.infer<typeof CameraSchema>
```

### AnyNode / Union Types

Discriminated union of all 14 node types.

```typescript { .api }
import { AnyNode, type AnyNodeType, type AnyNodeId } from '@pascal-app/core'

const AnyNode: z.ZodDiscriminatedUnion<'type', [
  typeof SiteNode, typeof BuildingNode, typeof LevelNode, typeof WallNode,
  typeof ItemNode, typeof ZoneNode, typeof SlabNode, typeof CeilingNode,
  typeof RoofNode, typeof RoofSegmentNode, typeof ScanNode, typeof GuideNode,
  typeof WindowNode, typeof DoorNode,
]>

type AnyNode = z.infer<typeof AnyNode>
type AnyNodeType = AnyNode['type']   // 'site' | 'building' | 'level' | 'wall' | ...
type AnyNodeId   = AnyNode['id']     // 'site_...' | 'building_...' | ...
```

### SiteNode

Root container of the scene. Contains the site polygon boundary and direct children (buildings and items).

```typescript { .api }
import { SiteNode } from '@pascal-app/core'

const SiteNode: z.ZodObject<BaseNode & {
  id:       /* objectId('site') */   z.ZodDefault<z.ZodString>  // 'site_...'
  type:     z.ZodDefault<z.ZodLiteral<'site'>>
  polygon: z.ZodDefault<z.ZodObject<{
    type:   z.ZodLiteral<'polygon'>
    points: z.ZodArray<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>
  }>>
  children: z.ZodDefault<z.ZodArray<z.ZodDiscriminatedUnion<'type', [BuildingNode, ItemNode]>>>
}>

type SiteNode = z.infer<typeof SiteNode>
```

Fields:
- `id` — `site_<nanoid>` (auto-generated)
- `type` — `'site'`
- `polygon.type` — `'polygon'`
- `polygon.points` — array of `[x, z]` 2D coordinates defining the site boundary. Default: 30×30 square.
- `children` — child `BuildingNode` or `ItemNode` objects (not IDs — SiteNode contains embedded children). **Default: `[BuildingNode.parse({})]`** (one default building). Pass `children: []` explicitly to create an empty site.

### BuildingNode

Represents a building within the site. Children are level IDs.

```typescript { .api }
import { BuildingNode } from '@pascal-app/core'

const BuildingNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'building_...'
  type:     z.ZodDefault<z.ZodLiteral<'building'>>
  position: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  children: z.ZodDefault<z.ZodArray<z.ZodString>>  // LevelNode IDs
}>

type BuildingNode = z.infer<typeof BuildingNode>
```

Fields:
- `id` — `building_<nanoid>`
- `type` — `'building'`
- `position` — `[x, y, z]` in site coordinate system, default `[0, 0, 0]`
- `rotation` — `[x, y, z]` Euler angles in radians, default `[0, 0, 0]`
- `children` — array of `LevelNode` IDs

### LevelNode

Represents a floor level within a building. Children are IDs of walls, zones, slabs, ceilings, roofs, scans, and guides.

```typescript { .api }
import { LevelNode } from '@pascal-app/core'

const LevelNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'level_...'
  type:     z.ZodDefault<z.ZodLiteral<'level'>>
  level:    z.ZodDefault<z.ZodNumber>
  children: z.ZodDefault<z.ZodArray<z.ZodUnion<[...]>>>
  // children union: WallNode['id'] | ZoneNode['id'] | SlabNode['id'] |
  //                 CeilingNode['id'] | RoofNode['id'] | ScanNode['id'] | GuideNode['id']
}>

type LevelNode = z.infer<typeof LevelNode>
```

Fields:
- `id` — `level_<nanoid>`
- `type` — `'level'`
- `level` — floor level number, default `0` (ground floor; use 1, 2, etc. for upper floors, -1 for basement)
- `children` — array of node IDs (WallNode, ZoneNode, SlabNode, CeilingNode, RoofNode, ScanNode, GuideNode)

### WallNode

Represents a straight wall segment in the level coordinate system.

```typescript { .api }
import { WallNode } from '@pascal-app/core'

const WallNode: z.ZodObject<BaseNode & {
  id:        z.ZodDefault<z.ZodString>  // 'wall_...'
  type:      z.ZodDefault<z.ZodLiteral<'wall'>>
  start:     z.ZodTuple<[z.ZodNumber, z.ZodNumber]>
  end:       z.ZodTuple<[z.ZodNumber, z.ZodNumber]>
  thickness: z.ZodOptional<z.ZodNumber>
  height:    z.ZodOptional<z.ZodNumber>
  frontSide: z.ZodDefault<z.ZodEnum<['interior', 'exterior', 'unknown']>>
  backSide:  z.ZodDefault<z.ZodEnum<['interior', 'exterior', 'unknown']>>
  children:  z.ZodDefault<z.ZodArray<z.ZodString>>  // ItemNode IDs
}>

type WallNode = z.infer<typeof WallNode>
```

Fields:
- `id` — `wall_<nanoid>`
- `type` — `'wall'`
- `start` — `[x, z]` start point in level coordinate system (Y axis is up)
- `end` — `[x, z]` end point in level coordinate system
- `thickness` — wall thickness in meters (default `0.1` via `DEFAULT_WALL_THICKNESS`)
- `height` — wall height in meters (default `2.5` via `DEFAULT_WALL_HEIGHT`)
- `frontSide` / `backSide` — set by space detection system (`'interior'|'exterior'|'unknown'`)
- `children` — array of `ItemNode` IDs attached to this wall

**Usage Example:**
```typescript
import { WallNode, useScene } from '@pascal-app/core'

const wall = WallNode.parse({
  start: [0, 0],
  end: [4, 0],
  height: 2.5,
  thickness: 0.2,
})
useScene.getState().createNode(wall, levelId)
```

### SlabNode

Represents a floor slab defined by a polygon boundary.

```typescript { .api }
import { SlabNode } from '@pascal-app/core'

const SlabNode: z.ZodObject<BaseNode & {
  id:        z.ZodDefault<z.ZodString>  // 'slab_...'
  type:      z.ZodDefault<z.ZodLiteral<'slab'>>
  polygon:   z.ZodArray<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>
  holes:     z.ZodDefault<z.ZodArray<z.ZodArray<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>>>
  elevation: z.ZodDefault<z.ZodNumber>
}>

type SlabNode = z.infer<typeof SlabNode>
```

Fields:
- `id` — `slab_<nanoid>`
- `type` — `'slab'`
- `polygon` — array of `[x, z]` points defining the slab boundary (min 3 points)
- `holes` — array of polygons representing holes (e.g., for staircases), default `[]`
- `elevation` — slab surface height in meters, default `0.05`

### CeilingNode

Represents a ceiling surface.

```typescript { .api }
import { CeilingNode } from '@pascal-app/core'

const CeilingNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'ceiling_...'
  type:     z.ZodDefault<z.ZodLiteral<'ceiling'>>
  polygon:  z.ZodArray<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>
  holes:    z.ZodDefault<z.ZodArray<z.ZodArray<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>>>
  height:   z.ZodDefault<z.ZodNumber>
  children: z.ZodDefault<z.ZodArray<z.ZodString>>  // ItemNode IDs (ceiling-attached)
}>

type CeilingNode = z.infer<typeof CeilingNode>
```

Fields:
- `id` — `ceiling_<nanoid>`
- `type` — `'ceiling'`
- `polygon` — boundary points as `[x, z]` pairs
- `holes` — array of hole polygons, default `[]`
- `height` — ceiling height in meters, default `2.5`
- `children` — array of `ItemNode` IDs attached to the ceiling

### ZoneNode

Spatial zone or room polygon for annotation/analysis.

```typescript { .api }
import { ZoneNode } from '@pascal-app/core'

const ZoneNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'zone_...'
  type:     z.ZodDefault<z.ZodLiteral<'zone'>>
  name:     z.ZodString
  polygon:  z.ZodArray<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>
  color:    z.ZodDefault<z.ZodString>
  metadata: z.ZodOptional<z.ZodDefault<z.ZodJson>>
}>

type ZoneNode = z.infer<typeof ZoneNode>
```

Fields:
- `id` — `zone_<nanoid>`
- `type` — `'zone'`
- `name` — zone/room name (required)
- `polygon` — boundary as `[x, z]` pairs
- `color` — hex color string, default `'#3b82f6'`
- `metadata` — arbitrary JSON, default `{}`

### RoofNode

Container node grouping one or more `RoofSegmentNode` children into a combined roof.

```typescript { .api }
import { RoofNode } from '@pascal-app/core'

const RoofNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'roof_...'
  type:     z.ZodDefault<z.ZodLiteral<'roof'>>
  position: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation: z.ZodDefault<z.ZodNumber>
  children: z.ZodDefault<z.ZodArray<z.ZodString>>  // RoofSegmentNode IDs
}>

type RoofNode = z.infer<typeof RoofNode>
```

Fields:
- `id` — `roof_<nanoid>`
- `type` — `'roof'`
- `position` — center position `[x, y, z]`, default `[0, 0, 0]`
- `rotation` — Y-axis rotation in radians, default `0`
- `children` — array of `RoofSegmentNode` IDs

### RoofSegmentNode

An individual parametric roof module. Multiple segments can be combined.

```typescript { .api }
import { RoofSegmentNode, RoofType } from '@pascal-app/core'

const RoofSegmentNode: z.ZodObject<BaseNode & {
  id:              z.ZodDefault<z.ZodString>  // 'rseg_...'
  type:            z.ZodDefault<z.ZodLiteral<'roof-segment'>>
  position:        z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation:        z.ZodDefault<z.ZodNumber>
  roofType:        z.ZodDefault<typeof RoofType>
  width:           z.ZodDefault<z.ZodNumber>
  depth:           z.ZodDefault<z.ZodNumber>
  wallHeight:      z.ZodDefault<z.ZodNumber>
  roofHeight:      z.ZodDefault<z.ZodNumber>
  wallThickness:   z.ZodDefault<z.ZodNumber>
  deckThickness:   z.ZodDefault<z.ZodNumber>
  overhang:        z.ZodDefault<z.ZodNumber>
  shingleThickness: z.ZodDefault<z.ZodNumber>
}>

type RoofSegmentNode = z.infer<typeof RoofSegmentNode>

const RoofType: z.ZodEnum<['hip', 'gable', 'shed', 'gambrel', 'dutch', 'mansard', 'flat']>
type RoofType = 'hip' | 'gable' | 'shed' | 'gambrel' | 'dutch' | 'mansard' | 'flat'
```

Fields:
- `id` — `rseg_<nanoid>`
- `type` — `'roof-segment'`
- `position` — relative to parent `RoofNode`, default `[0, 0, 0]`
- `rotation` — Y-axis in radians, default `0`
- `roofType` — roof shape, default `'gable'`
- `width` — footprint width in meters, default `8`
- `depth` — footprint depth in meters, default `6`
- `wallHeight` — height of knee walls, default `0.5`
- `roofHeight` — height of roof peak above walls, default `2.5`
- `wallThickness` — structural wall thickness, default `0.1`
- `deckThickness` — roof deck thickness, default `0.1`
- `overhang` — eave overhang distance, default `0.3`
- `shingleThickness` — outer shingle layer, default `0.05`

### ItemNode

Furniture, fixtures, appliances and other 3D model items.

```typescript { .api }
import { ItemNode, type Asset, type AssetInput, type Interactive,
         type Control, type ToggleControl, type SliderControl, type TemperatureControl,
         type Effect, type AnimationEffect, type LightEffect,
         getScaledDimensions } from '@pascal-app/core'

const ItemNode: z.ZodObject<BaseNode & {
  id:            z.ZodDefault<z.ZodString>  // 'item_...'
  type:          z.ZodDefault<z.ZodLiteral<'item'>>
  position:      z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation:      z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  scale:         z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  side:          z.ZodOptional<z.ZodEnum<['front', 'back']>>
  children:      z.ZodDefault<z.ZodArray<z.ZodString>>
  wallId:        z.ZodOptional<z.ZodString>
  wallT:         z.ZodOptional<z.ZodNumber>
  collectionIds: z.ZodOptional<z.ZodArray<z.ZodCustom<CollectionId>>>
  asset:         z.ZodObject<AssetSchema>
}>

type ItemNode = z.infer<typeof ItemNode>
type AssetInput = z.input<typeof assetSchema>
type Asset = z.infer<typeof assetSchema>
```

**`asset` fields:**
- `id: string` — asset library ID
- `category: string`
- `name: string`
- `thumbnail: string` — URL of thumbnail image
- `src: string` — URL of 3D model (GLB)
- `dimensions: [w, h, d]` — size in meters, default `[1, 1, 1]`
- `attachTo?: 'wall' | 'wall-side' | 'ceiling'` — omit for floor items
- `tags?: string[]`
- `offset: [x, y, z]` — corrective position offset for the GLB model, default `[0, 0, 0]`
- `rotation: [x, y, z]` — corrective rotation for the GLB model, default `[0, 0, 0]`
- `scale: [x, y, z]` — corrective scale for the GLB model, default `[1, 1, 1]`
- `surface?: { height: number }` — surface height for stacking items on top; `undefined` if not stackable
- `interactive?: Interactive` — optional interactive descriptor for controls/effects

**`Interactive` type:**
```typescript { .api }
interface Interactive {
  controls: Control[]   // UI controls (toggle, slider, temperature)
  effects: Effect[]     // Driven effects (animation, light)
}

type Control = ToggleControl | SliderControl | TemperatureControl
type Effect  = AnimationEffect | LightEffect

interface ToggleControl {
  kind: 'toggle'
  label?: string
  default?: boolean
}

interface SliderControl {
  kind: 'slider'
  label: string
  min: number
  max: number
  step: number  // default 1
  unit?: string
  displayMode: 'slider' | 'stepper' | 'dial'  // default 'slider'
  default?: number
}

interface TemperatureControl {
  kind: 'temperature'
  label: string   // default 'Temperature'
  min: number     // default 16
  max: number     // default 30
  unit: 'C' | 'F'  // default 'C'
  default?: number
}

interface AnimationEffect {
  kind: 'animation'
  clips: { on?: string; off?: string; loop?: string }
}

interface LightEffect {
  kind: 'light'
  color: string              // hex, default '#ffffff'
  intensityRange: [number, number]
  distance?: number
  offset: [number, number, number]  // default [0,0,0]
}
```

**`getScaledDimensions(item: ItemNode): [number, number, number]`**

Returns effective world-space dimensions after applying `item.scale` to `item.asset.dimensions`. Use this instead of `item.asset.dimensions` for spatial calculations.

### DoorNode

Parametric door placed on a wall with configurable frame, segments, swing, and hardware.

```typescript { .api }
import { DoorNode, DoorSegment } from '@pascal-app/core'

const DoorNode: z.ZodObject<BaseNode & {
  id:             z.ZodDefault<z.ZodString>  // 'door_...'
  type:           z.ZodDefault<z.ZodLiteral<'door'>>
  position:       z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation:       z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  side:           z.ZodOptional<z.ZodEnum<['front', 'back']>>
  wallId:         z.ZodOptional<z.ZodString>
  width:          z.ZodDefault<z.ZodNumber>   // default 0.9
  height:         z.ZodDefault<z.ZodNumber>   // default 2.1
  frameThickness: z.ZodDefault<z.ZodNumber>   // default 0.05
  frameDepth:     z.ZodDefault<z.ZodNumber>   // default 0.07
  threshold:      z.ZodDefault<z.ZodBoolean>  // default true
  thresholdHeight: z.ZodDefault<z.ZodNumber>  // default 0.02
  hingesSide:     z.ZodDefault<z.ZodEnum<['left', 'right']>>   // default 'left'
  swingDirection: z.ZodDefault<z.ZodEnum<['inward', 'outward']>>  // default 'inward'
  segments:       z.ZodDefault<z.ZodArray<typeof DoorSegment>>
  handle:         z.ZodDefault<z.ZodBoolean>  // default true
  handleHeight:   z.ZodDefault<z.ZodNumber>   // default 1.05
  handleSide:     z.ZodDefault<z.ZodEnum<['left', 'right']>>   // default 'right'
  contentPadding: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber]>>  // default [0.04, 0.04]
  doorCloser:     z.ZodDefault<z.ZodBoolean>  // default false
  panicBar:       z.ZodDefault<z.ZodBoolean>  // default false
  panicBarHeight: z.ZodDefault<z.ZodNumber>   // default 1.0
}>

type DoorNode = z.infer<typeof DoorNode>
```

**`DoorSegment` (leaf row, stacked top to bottom):**
```typescript { .api }
const DoorSegment: z.ZodObject<{
  type:             z.ZodEnum<['panel', 'glass', 'empty']>
  heightRatio:      z.ZodNumber
  columnRatios:     z.ZodDefault<z.ZodArray<z.ZodNumber>>  // default [1]
  dividerThickness: z.ZodDefault<z.ZodNumber>  // default 0.03
  panelDepth:       z.ZodDefault<z.ZodNumber>  // default 0.01 (+raised, -recessed)
  panelInset:       z.ZodDefault<z.ZodNumber>  // default 0.04
}>

type DoorSegment = z.infer<typeof DoorSegment>
```

Notes:
- `position` is the center of the door in wall-local coordinate system (Y = height/2 sets door at floor)
- `type: 'empty'` = flush fill, `'panel'` = raised/recessed panel, `'glass'` = glazed
- `doorCloser` and `panicBar` — commercial/emergency hardware

### WindowNode

Parametric window with configurable frame, pane divisions, and sill.

```typescript { .api }
import { WindowNode } from '@pascal-app/core'

const WindowNode: z.ZodObject<BaseNode & {
  id:                      z.ZodDefault<z.ZodString>  // 'window_...'
  type:                    z.ZodDefault<z.ZodLiteral<'window'>>
  position:                z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation:                z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  side:                    z.ZodOptional<z.ZodEnum<['front', 'back']>>
  wallId:                  z.ZodOptional<z.ZodString>
  width:                   z.ZodDefault<z.ZodNumber>   // default 1.5
  height:                  z.ZodDefault<z.ZodNumber>   // default 1.5
  frameThickness:          z.ZodDefault<z.ZodNumber>   // default 0.05
  frameDepth:              z.ZodDefault<z.ZodNumber>   // default 0.07
  columnRatios:            z.ZodDefault<z.ZodArray<z.ZodNumber>>  // default [1]
  rowRatios:               z.ZodDefault<z.ZodArray<z.ZodNumber>>  // default [1]
  columnDividerThickness:  z.ZodDefault<z.ZodNumber>   // default 0.03
  rowDividerThickness:     z.ZodDefault<z.ZodNumber>   // default 0.03
  sill:                    z.ZodDefault<z.ZodBoolean>  // default true
  sillDepth:               z.ZodDefault<z.ZodNumber>   // default 0.08
  sillThickness:           z.ZodDefault<z.ZodNumber>   // default 0.03
}>

type WindowNode = z.infer<typeof WindowNode>
```

Notes:
- `position` is the center in wall-local coordinate system
- `columnRatios` / `rowRatios` control pane divisions: `[1]` = single pane, `[0.5, 0.5]` = two equal panes

### ScanNode

3D scan reference image placed in the scene.

```typescript { .api }
import { ScanNode } from '@pascal-app/core'

const ScanNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'scan_...'
  type:     z.ZodDefault<z.ZodLiteral<'scan'>>
  url:      z.ZodString
  position: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  scale:    z.ZodDefault<z.ZodNumber>
  opacity:  z.ZodDefault<z.ZodNumber>  // 0–100, default 100
}>

type ScanNode = z.infer<typeof ScanNode>
```

### GuideNode

2D guide image (floor plan overlay) placed in the scene.

```typescript { .api }
import { GuideNode } from '@pascal-app/core'

const GuideNode: z.ZodObject<BaseNode & {
  id:       z.ZodDefault<z.ZodString>  // 'guide_...'
  type:     z.ZodDefault<z.ZodLiteral<'guide'>>
  url:      z.ZodString
  position: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  rotation: z.ZodDefault<z.ZodTuple<[z.ZodNumber, z.ZodNumber, z.ZodNumber]>>
  scale:    z.ZodDefault<z.ZodNumber>
  opacity:  z.ZodDefault<z.ZodNumber>  // 0–100, default 50
}>

type GuideNode = z.infer<typeof GuideNode>
```

### Collections

Named groups of nodes (for organizing scene items into logical sets).

```typescript { .api }
import { type Collection, type CollectionId, generateCollectionId } from '@pascal-app/core'

type CollectionId = `collection_${string}`

interface Collection {
  id: CollectionId
  name: string
  color?: string
  nodeIds: AnyNodeId[]
  controlNodeId?: AnyNodeId
}

function generateCollectionId(): CollectionId
```

## Scene Hierarchy

The Pascal scene graph is a flat dictionary. The default hierarchy is:

```
SiteNode (root)
└── BuildingNode
    └── LevelNode (level: 0)
        ├── WallNode
        │   └── ItemNode (wall-attached)
        ├── SlabNode
        ├── CeilingNode
        │   └── ItemNode (ceiling-attached)
        ├── ZoneNode
        ├── RoofNode
        │   └── RoofSegmentNode
        ├── ScanNode
        └── GuideNode

SiteNode.children contains BuildingNode objects (embedded, not IDs)
BuildingNode.children contains LevelNode IDs
LevelNode.children contains WallNode, SlabNode, CeilingNode, ZoneNode, RoofNode, ScanNode, GuideNode IDs
WallNode.children contains ItemNode IDs (wall-attached items)
CeilingNode.children contains ItemNode IDs (ceiling-attached items)
```

Note: `SiteNode.children` is an array of embedded `BuildingNode` objects (not IDs), while all other `children` arrays store ID strings.
