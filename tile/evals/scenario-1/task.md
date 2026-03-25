# Building Scene Initializer

A module that initializes and manages the lifecycle of a 3D building scene. The scene has a predefined default hierarchy consisting of a site node at the root, containing a building node, which in turn contains at least one level node. The module must support loading the default scene, clearing it back to defaults, and completely unloading it.

## Capabilities

### Loading the default scene hierarchy

Calling the load action on an empty scene creates the default Site → Building → Level hierarchy. If the scene is already loaded (has root node IDs), calling load again should not reset the scene — it should be a no-op.

- Loading an empty scene creates exactly one site, one building, and one level node [@test](./tests/load-scene.test.ts)
- Loading an already-populated scene is a no-op (does not re-initialize) [@test](./tests/load-scene-noop.test.ts)

### Clearing the scene

Clearing the scene resets it back to the default hierarchy, discarding any previously added nodes.

- After adding extra nodes and then clearing, only the default hierarchy remains [@test](./tests/clear-scene.test.ts)

### Unloading the scene

Unloading removes all nodes, root node IDs, and ancillary data, leaving the store completely empty.

- After unloading, nodes is an empty object and rootNodeIds is an empty array [@test](./tests/unload-scene.test.ts)

### Setting the scene from external data

setScene replaces the entire scene with provided nodes and rootNodeIds, applying any necessary backward-compatibility migrations.

- setScene with a valid nodes map and rootNodeIds replaces the entire scene [@test](./tests/set-scene.test.ts)

## Implementation

[@generates](./src/scene-initializer.ts)

## API

```typescript { #api }
export type SceneLifecycle = {
  loadScene: () => void
  clearScene: () => void
  unloadScene: () => void
  setScene: (nodes: Record<string, unknown>, rootNodeIds: string[]) => void
}
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `useScene` with `loadScene`, `clearScene`, `unloadScene`, and `setScene` actions, as well as `SiteNode`, `BuildingNode`, and `LevelNode` schema constructors.

[@satisfied-by](@pascal-app/core)
