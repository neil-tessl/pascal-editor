# Scene Node Manager

A utility module that manages nodes in a 3D building scene using a flat-dictionary store. The store holds all nodes keyed by their IDs, tracks which nodes are at the root level, and provides actions for creating, updating, and deleting nodes with proper parent–child relationship management.

## Capabilities

### Creating single and multiple nodes

Create nodes in the scene and register them in the flat node dictionary. When a parent ID is provided, the new node should appear as a child of that parent node.

- Creating a node with no parent adds it to rootNodeIds and to the nodes dictionary [@test](./tests/create-node.test.ts)
- Creating a node with a parentId adds it to the nodes dictionary but not to rootNodeIds [@test](./tests/create-node-with-parent.test.ts)
- Creating multiple nodes in a single operation (batch) persists all of them atomically [@test](./tests/create-nodes-batch.test.ts)

### Updating nodes

Update one or more nodes by partial data merge. The update should preserve all existing fields not mentioned in the update payload.

- Updating a single node with partial data preserves unchanged fields [@test](./tests/update-node.test.ts)
- Updating multiple nodes in one operation updates each independently [@test](./tests/update-nodes-batch.test.ts)

### Deleting nodes

Remove nodes from the scene. Deleting a node should remove it from the flat dictionary and from any parent's children list.

- Deleting a node removes it from the nodes dictionary [@test](./tests/delete-node.test.ts)
- Deleting multiple nodes at once removes all of them [@test](./tests/delete-nodes-batch.test.ts)

## Implementation

[@generates](./src/scene-manager.ts)

## API

```typescript { #api }
export type AnyNodeId = string

export type AnyNode = {
  id: AnyNodeId
  type: string
  parentId: AnyNodeId | null
  [key: string]: unknown
}

export type SceneState = {
  nodes: Record<AnyNodeId, AnyNode>
  rootNodeIds: AnyNodeId[]
  createNode: (node: AnyNode, parentId?: AnyNodeId) => void
  createNodes: (ops: { node: AnyNode; parentId?: AnyNodeId }[]) => void
  updateNode: (id: AnyNodeId, data: Partial<AnyNode>) => void
  updateNodes: (updates: { id: AnyNodeId; data: Partial<AnyNode> }[]) => void
  deleteNode: (id: AnyNodeId) => void
  deleteNodes: (ids: AnyNodeId[]) => void
}
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides the flat-dictionary Zustand scene store with node CRUD actions, root node tracking, and parent–child relationship management via `useScene`.

[@satisfied-by](@pascal-app/core)
