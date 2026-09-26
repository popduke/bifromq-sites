---
title: Lua Processing
sidebar_position: 2
description: Filter, transform, route, and schedule Tenon records with Lua.
---

# Lua processing

A Flow uses `function main(event)` to process input or react to a timer. Each channel has its own Lua state.

| Event | Check | Available input |
| --- | --- | --- |
| Source | `event.type == "source"` | `event.timestamp` and typed fields in `event.payload`, defined by the Source plugin |
| Timer | `event.type == "timer"` | `event.timestamp`, `event.eligibleAt`, and `event.id` (`nil` for the anonymous timer); no payload |

`timestamp` records dispatch time and `eligibleAt` records the timer's scheduled deadline. Both are integer milliseconds since Pipeline startup, not Unix time. Events and nested payload values are read-only. See the [event field reference](/tenon/docs/development/reference/lua/#events-and-payloads).

## Build an output

Use the Builder for the exact Sink contract bound to the Flow. Method calls use `:`:

```lua
local builder = registry:getBuilder("org.apache.bifromq.tenon.stdout-sink@0.1.0")

function main(event)
  if event.type == "source" then
    builder:setMessage(event.payload.message)
    emit(builder:build())
  end
end
```

This example assumes a Source payload with a string field named `message`. Adapt the input field to your Source plugin's payload.

- `build()` creates an immutable snapshot. The Builder stays mutable; later changes do not alter an earlier snapshot.
- `emit(snapshot)` sends that snapshot to matching Sink instances in the Flow.
- `emit()` without arguments completes pending input without producing output. Use it when intentionally filtering a record.

See [Delivery & Recovery](/tenon/docs/development/user-guide/delivery/) for completion rules and the [Lua API](/tenon/docs/development/reference/lua/) for payload types, JSON, binary fields, and errors.

## Timers

Timers are one-shot. Each channel has an anonymous timer and can also keep independent named timers. Schedule the next timer from the current timer event:

```lua
local builder = registry:getBuilder("org.apache.bifromq.tenon.stdout-sink@0.1.0")
local count = 0
setTimeout(1000, "probe")

function main(event)
  if event.type == "timer" and event.id == "probe" then
    count = count + 1
    builder:setMessage("probe " .. count)
    emit(builder:build())
    setTimeout(1000, "probe")
  end
end
```

A timer can generate a local probe while Dummy Source stays idle, as in [First Pipeline](/tenon/docs/development/get-started/first-pipeline/).

- Calling `setTimeout(delay, id)` again replaces only that id's pending timer. Other named timers remain pending. Omitting `id` or passing `nil` selects the anonymous timer.
- Use `hasTimeout(id)` to check a timer and `clearTimeout(id)` to cancel it. Cancellation succeeds even if no timer exists.
- A timer is removed before its event runs; schedule the next one explicitly.
- Delays are nonnegative Lua integers. Invalid arguments and time calculations that overflow raise catchable Lua errors. Timer names are non-empty UTF-8 strings of at most 128 bytes. See [timer arguments and errors](/tenon/docs/development/reference/lua/#timers-and-state).
- Timers share the VM's memory allowance; there is no separate limit on their count.
- Restarts and applied script changes recreate Lua state and timers.
- Multiple channels have independent counters and timers. They do not provide a shared or durable schedule.

Due timers and readable Source input share the same dispatch order: the earlier timer deadline or first observed Source readiness runs first, with ties broken by registration or observation order. Repeated zero-delay timers cannot indefinitely block readable input. A busy channel or Sink backpressure can still delay execution.

A timer name does not give each key its own reliable completion boundary. The first successful `emit` in an event can complete all pending Source records in that VM, including records associated with other timer names. See [Delivery & Recovery](/tenon/docs/development/user-guide/delivery/).
