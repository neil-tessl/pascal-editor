# Building Scene Event Bus

A module that wires up a typed event bus for a 3D building editor. The bus supports subscribing to and emitting strongly-typed events for every node type (walls, items, levels, etc.), each with eight interaction suffixes (click, move, enter, leave, pointerdown, pointerup, context-menu, double-click). It also supports camera-control events and tool lifecycle events.

## Capabilities

### Subscribing and emitting node interaction events

Subscribe to a node event (e.g., a wall click or item move) and receive the correct typed payload when emitted.

- Subscribing to "wall:click" and emitting that event delivers the payload to the handler [@test](./tests/wall-click.test.ts)
- Subscribing to "item:move" and emitting that event delivers the payload to the handler [@test](./tests/item-move.test.ts)
- Handlers are not called for events they did not subscribe to [@test](./tests/no-cross-emit.test.ts)

### Unsubscribing from events

The emitter follows the mitt pattern: subscribe returns nothing; callers use `emitter.off` to unsubscribe with the same handler reference.

- After calling off with the registered handler, subsequent emits do not invoke it [@test](./tests/unsubscribe.test.ts)

### Using the eventSuffixes constant

eventSuffixes is an exported tuple of all supported interaction suffixes. It can be used to dynamically build event names.

- eventSuffixes contains exactly "click", "move", "enter", "leave", "pointerdown", "pointerup", "context-menu", and "double-click" [@test](./tests/event-suffixes.test.ts)

## Implementation

[@generates](./src/event-bus.ts)

## API

```typescript { #api }
export { emitter, eventSuffixes } from '@pascal-app/core'
export type {
  NodeEvent,
  WallEvent,
  ItemEvent,
  GridEvent,
  EventSuffix,
  CameraControlEvent,
} from '@pascal-app/core'
```

## Dependencies { .dependencies }

### @pascal-app/core { .dependency }

Provides `emitter` (mitt-based typed event bus) and `eventSuffixes` along with all event type exports for the building editor.

[@satisfied-by](@pascal-app/core)
