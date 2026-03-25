# Scene History Manager

A module that manages undo and redo history for a 3D building scene. The scene store is backed by a temporal middleware that tracks past and future states. The module exposes undo and redo operations that traverse the history, and a function to clear all history so the current state becomes the new baseline.

## Capabilities

### Undoing scene changes

After making one or more changes to the scene (e.g., creating or deleting nodes), calling undo reverts the scene to its prior state.

- After creating a node and then undoing, the node is no longer present in the scene [@test](./tests/undo-create.test.ts)
- Undo on an empty history (no past states) does not throw and leaves the scene unchanged [@test](./tests/undo-empty-history.test.ts)

### Redoing reverted changes

After undoing a change, calling redo re-applies it.

- After creating a node, undoing, then redoing, the node is present in the scene again [@test](./tests/redo-create.test.ts)

### Clearing scene history

clearSceneHistory removes all past and future states from the temporal store so subsequent undo/redo operations have no history to traverse.

- After calling clearSceneHistory, the temporal store has zero past states and zero future states [@test](./tests/clear-history.test.ts)

## Implementation

[@generates](./src/scene-history.ts)

## API

```typescript { #api }
export { clearSceneHistory, default as useScene } from '@pascal-app/core'
// useScene.temporal.getState().undo()
// useScene.temporal.getState().redo()
// useScene.temporal.getState().pastStates
// useScene.temporal.getState().futureStates
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `useScene` (the Zustand store with zundo temporal middleware) and `clearSceneHistory` for managing scene undo/redo history.

[@satisfied-by](@pascal-app/core)
