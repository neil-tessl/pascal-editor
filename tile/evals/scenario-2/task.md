# Building Node Schema Parser

A module that parses and validates building scene node data using typed schemas. Each node type has a schema that auto-generates its ID, applies default values, and validates the shape of the input. The module should demonstrate creating various node types from partial input and working with the discriminated union type that covers all node types.

## Capabilities

### Parsing individual node types

Each node schema (Wall, Level, Site, Building, Item, etc.) accepts partial input, fills in defaults, and returns a fully typed node. The ID field is auto-generated when not provided.

- Parsing a wall node with only start/end coordinates produces a valid wall with a generated ID starting with "wall_" [@test](./tests/parse-wall.test.ts)
- Parsing a level node with only a level number produces a valid level node with a generated ID starting with "level_" [@test](./tests/parse-level.test.ts)
- Parsing a node without an explicit parentId defaults parentId to null [@test](./tests/parse-defaults.test.ts)

### Discriminated union node type

AnyNode is a discriminated union on the "type" field covering all supported node types. Parsing raw JSON through AnyNode correctly identifies and validates the node type.

- Parsing an object with type "wall" via AnyNode results in a WallNode [@test](./tests/any-node-wall.test.ts)
- Parsing an object with an unknown type via AnyNode throws a validation error [@test](./tests/any-node-invalid.test.ts)

### BaseNode defaults

All nodes extend BaseNode which provides common fields: object, id, type, parentId, visible, metadata.

- A parsed node has object set to "node" and visible defaulting to true [@test](./tests/base-node-defaults.test.ts)

## Implementation

[@generates](./src/node-schema-parser.ts)

## API

```typescript { #api }
// Re-exports from schema
export { WallNode, LevelNode, SiteNode, BuildingNode, ItemNode, AnyNode, BaseNode, generateId } from '@pascal-app/core'
export type { AnyNodeId, AnyNodeType } from '@pascal-app/core'
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides Zod-based node schemas: `WallNode`, `LevelNode`, `SiteNode`, `BuildingNode`, `ItemNode`, `AnyNode`, `BaseNode`, `generateId`, and the `objectId` / `nodeType` helpers.

[@satisfied-by](@pascal-app/core)
