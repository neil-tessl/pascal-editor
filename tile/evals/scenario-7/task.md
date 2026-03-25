# Enclosed Space Detector

A module that detects enclosed interior spaces formed by wall segments on a building level. Given a set of wall nodes defining a floorplan, the module uses a grid-based flood-fill approach to identify connected interior regions. It also classifies each wall's front and back sides as "interior", "exterior", or "unknown" based on which type of space each side faces.

## Capabilities

### Detecting interior spaces from walls

detectSpacesForLevel takes a level ID and an array of wall nodes and returns the detected spaces and wall side updates. Each Space has an id, levelId, polygon, wallIds, and isExterior flag.

- A closed rectangular arrangement of walls on a level produces at least one interior Space [@test](./tests/closed-room.test.ts)
- A set of walls that do not form a closed loop produces no interior spaces [@test](./tests/open-walls.test.ts)
- An empty walls array returns no spaces and no wall updates [@test](./tests/empty-walls.test.ts)

### Wall side classification

For each wall, the returned wallUpdates array contains frontSide and backSide values ("interior" | "exterior" | "unknown") indicating which space each side of the wall faces.

- Walls forming a closed room have at least one side classified as "interior" in wallUpdates [@test](./tests/wall-side-interior.test.ts)

### Wall connectivity check

wallTouchesOthers checks whether a wall shares an endpoint or has an endpoint within a threshold distance of another wall's segment.

- A wall with an endpoint coinciding with another wall's endpoint returns true [@test](./tests/wall-touches.test.ts)
- Two walls with no shared endpoints or segment proximity return false [@test](./tests/wall-not-touches.test.ts)

## Implementation

[@generates](./src/space-detector.ts)

## API

```typescript { #api }
export {
  detectSpacesForLevel,
  wallTouchesOthers,
} from '@pascal-app/core'
export type { Space } from '@pascal-app/core'
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `detectSpacesForLevel`, `wallTouchesOthers`, and the `Space` type for grid-based interior space detection.

[@satisfied-by](@pascal-app/core)
