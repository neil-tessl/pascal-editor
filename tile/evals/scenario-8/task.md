# Node Collection Manager

A module that manages named collections of scene nodes. Collections group arbitrary node IDs under a named label. The store maintains a bidirectional relationship: each Collection object lists its member node IDs, and each member node that supports collections has its collectionIds array kept in sync automatically.

## Capabilities

### Creating and deleting collections

createCollection takes a name and an optional initial array of node IDs and returns a CollectionId. deleteCollection removes the collection and strips its ID from all member nodes.

- Creating a collection with a name and two node IDs returns a CollectionId and adds the collection to the store [@test](./tests/create-collection.test.ts)
- Deleting a collection removes it from the store's collections map [@test](./tests/delete-collection.test.ts)

### Adding and removing nodes from a collection

addToCollection adds a node ID to an existing collection (idempotent). removeFromCollection removes it.

- Adding a node to a collection updates the collection's nodeIds list [@test](./tests/add-to-collection.test.ts)
- Removing a node from a collection updates the collection's nodeIds list [@test](./tests/remove-from-collection.test.ts)
- Adding the same node ID twice results in only one entry in nodeIds [@test](./tests/add-idempotent.test.ts)

### Updating collection metadata

updateCollection merges partial data (e.g., new name or color) into an existing collection without affecting nodeIds.

- Updating a collection's name changes only the name field [@test](./tests/update-collection.test.ts)

## Implementation

[@generates](./src/collection-manager.ts)

## API

```typescript { #api }
export type { Collection, CollectionId } from '@pascal-app/core'
export { generateCollectionId } from '@pascal-app/core'
// useScene actions: createCollection, deleteCollection, updateCollection, addToCollection, removeFromCollection
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `useScene` with collection actions (`createCollection`, `deleteCollection`, `updateCollection`, `addToCollection`, `removeFromCollection`), the `Collection` type, `CollectionId` type, and `generateCollectionId`.

[@satisfied-by](@pascal-app/core)
