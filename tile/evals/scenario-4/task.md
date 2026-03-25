# Wall Junction Miter Calculator

A module that computes miter intersection data for wall junctions in a 2D floorplan. When walls meet at corners (L-junctions), T-junctions, or X-junctions, the system must calculate the correct intersection points along the inner and outer edges of each wall so that renderers can produce clean mitered corners without gaps or overlaps.

## Capabilities

### Computing miter data for a set of walls

Given an array of wall nodes on a level, produce a WallMiterData object that describes all detected junctions and the per-wall intersection points at those junctions.

- Two walls meeting at a right-angle L-junction produce one junction entry in the junctions map [@test](./tests/l-junction.test.ts)
- Three walls meeting at a T-junction produce one junction entry containing three connected walls [@test](./tests/t-junction.test.ts)
- Walls that share no endpoints produce no junction entries [@test](./tests/no-junction.test.ts)

### Junction key serialization

pointToKey converts a Point2D ({x, y}) into a string key suitable for map lookups, snapping coordinates to a default tolerance.

- Two points within default tolerance serialize to the same key [@test](./tests/point-to-key-tolerance.test.ts)
- Two clearly distinct points serialize to different keys [@test](./tests/point-to-key-distinct.test.ts)

### Accessing junction data

The returned WallMiterData.junctionData is a map from junction key to per-wall left/right intersection points.

- For an L-junction, each wall has either a left or right miter point defined in junctionData [@test](./tests/junction-data-access.test.ts)

## Implementation

[@generates](./src/wall-miter-calculator.ts)

## API

```typescript { #api }
export {
  calculateLevelMiters,
  pointToKey,
} from '@pascal-app/core'
export type {
  WallMiterData,
  Point2D,
} from '@pascal-app/core'
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `calculateLevelMiters`, `pointToKey`, `WallMiterData`, and `Point2D` for computing wall junction miter geometry.

[@satisfied-by](@pascal-app/core)
