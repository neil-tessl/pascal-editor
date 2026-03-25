# Spatial Placement Grid

A module that provides axis-aligned bounding-box spatial indexing for 3D item placement. Items are registered by position, dimensions, and Y-axis rotation. The grid supports collision detection (checking whether a proposed placement overlaps any existing items), spatial radius queries (finding items near a point), and insert/remove/update lifecycle management.

## Capabilities

### Inserting and removing items

Register an item with a 3D position, dimensions, and rotation tuple. Removal deletes the item from all cells it occupied.

- Inserting an item increases the item count by one [@test](./tests/insert-item.test.ts)
- Removing an item that was inserted decreases the item count back to zero [@test](./tests/remove-item.test.ts)
- Removing an item that was never inserted is a no-op [@test](./tests/remove-nonexistent.test.ts)

### Collision detection via canPlace

canPlace checks whether a bounding box at a given position conflicts with any registered items, returning `{ valid, conflictIds }`.

- canPlace returns valid:true when the grid is empty [@test](./tests/can-place-empty.test.ts)
- canPlace returns valid:false and includes the conflicting item's ID when an overlap exists [@test](./tests/can-place-conflict.test.ts)
- canPlace with an ignoreIds list does not report conflicts with the ignored items [@test](./tests/can-place-ignore.test.ts)

### Radius query

queryRadius returns the IDs of all items whose registered cells fall within a given cell-radius of a point.

- An item inserted at a point is returned by a queryRadius call at that same point with radius 1 [@test](./tests/query-radius.test.ts)
- An item far outside the radius is not returned [@test](./tests/query-radius-out-of-range.test.ts)

## Implementation

[@generates](./src/spatial-grid.ts)

## API

```typescript { #api }
export { SpatialGrid } from '@pascal-app/core'
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides the `SpatialGrid` class with `insert`, `remove`, `update`, `canPlace`, `queryRadius`, and `getItemCount` methods.

[@satisfied-by](@pascal-app/core)
