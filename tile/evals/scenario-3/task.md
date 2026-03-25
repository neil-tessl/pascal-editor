# Wall Segment Factory

A utility module for creating and inspecting wall segments in a 2D floorplan. A wall is defined by 2D start and end coordinates in the level coordinate system. The module exposes helpers to compute a wall's plan-view footprint (the four corner points of the wall rectangle given its thickness) and retrieve its effective thickness.

## Capabilities

### Creating wall nodes with start/end coordinates

A wall node is created from 2D start and end points (each a [x, z] tuple). It can optionally carry thickness and height overrides. The node has a generated ID and default frontSide/backSide values of "unknown".

- A wall created with start [0, 0] and end [5, 0] stores those exact coordinates [@test](./tests/wall-coordinates.test.ts)
- A wall created without explicit thickness has no thickness override (uses the system default) [@test](./tests/wall-thickness-default.test.ts)
- A wall's frontSide and backSide default to "unknown" [@test](./tests/wall-side-defaults.test.ts)

### Computing wall plan footprint

Given a wall node, compute the four 2D corner points of the wall's rectangular footprint based on its start, end, and thickness.

- A horizontal wall from [0,0] to [4,0] with thickness 0.2 has footprint corners offset ±0.1 on the z-axis at each end [@test](./tests/wall-footprint.test.ts)

### Retrieving effective wall thickness

A utility function returns the effective thickness for a wall, falling back to the system default when the wall has no explicit thickness.

- A wall with no thickness set returns the default wall thickness value [@test](./tests/wall-thickness-fallback.test.ts)
- A wall with an explicit thickness returns that value [@test](./tests/wall-thickness-explicit.test.ts)

## Implementation

[@generates](./src/wall-factory.ts)

## API

```typescript { #api }
export {
  WallNode,
  DEFAULT_WALL_HEIGHT,
  DEFAULT_WALL_THICKNESS,
  getWallThickness,
  getWallPlanFootprint,
} from '@pascal-app/core'
export type { WallNode as WallNodeType } from '@pascal-app/core'
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `WallNode` schema, `DEFAULT_WALL_HEIGHT`, `DEFAULT_WALL_THICKNESS`, `getWallThickness`, and `getWallPlanFootprint`.

[@satisfied-by](@pascal-app/core)
