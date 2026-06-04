# Matrix thread replies and owner commands

## Summary

Two Matrix regressions were reproduced and fixed against the `v2026.6.1`
source tree:

1. Owner-only Matrix text commands such as `/status` and `/model` could be
   rejected before reaching the core command dispatcher.
2. Replies to messages in Matrix threads carried the thread root but lost the
   current-event reply fallback, so clients rendered bot responses as normal
   replies instead of in the active thread.

The affected runtime configuration included Matrix thread replies enabled and
an explicit Matrix owner entry for command authorization:

```json
{
  "channels": {
    "matrix": {
      "threadReplies": "always",
      "dm": {
        "threadReplies": "always"
      },
      "replyToMode": "all"
    }
  },
  "commands": {
    "ownerAllowFrom": ["<explicit matrix owner id>"]
  }
}
```

Wildcard channel allow lists were not relied on as owner authorization.

## Message path

Inbound Matrix messages enter through `createMatrixRoomMessageHandler`, where
the event is normalized, access is checked, mentions are stripped, and thread
metadata is resolved. Thread routing uses `resolveMatrixThreadRootId` and
`resolveMatrixThreadRouting`, then the handler builds the core inbound channel
context via `buildChannelInboundEventContext`.

Normal agent replies flow from the core channel reply plumbing into
`deliverMatrixReplies`, which calls `sendMessageMatrix`. For Matrix threads,
the outbound payload must include both:

```json
{
  "m.relates_to": {
    "rel_type": "m.thread",
    "event_id": "<thread root event>",
    "is_falling_back": true,
    "m.in_reply_to": {
      "event_id": "<current triggering event>"
    }
  }
}
```

The `event_id` anchors the thread, while `m.in_reply_to.event_id` preserves the
reply fallback to the current thread message.

Owner command messages follow a separate pre-dispatch check:

```text
Matrix event
  -> createMatrixRoomMessageHandler
  -> hasControlCommand
  -> resolveMatrixMonitorCommandAccess
  -> core command dispatcher
```

The Matrix pre-dispatch gate must include explicit `commands.ownerAllowFrom`
entries, otherwise valid owner commands can fail closed before the core command
authorization logic sees them.

## Root causes

`deliverMatrixReplies` dropped `replyToId` whenever `threadId` was present:

```ts
const replyToId = params.threadId || params.replyToMode === "off" ? undefined : replyToIdRaw;
```

That prevented `sendMessageMatrix` from emitting the `m.in_reply_to` relation
for threaded responses. The handler also set `reply.replyToId` to `undefined`
for threaded inbound messages, so the current triggering event was not carried
through the reply context.

For owner commands, `resolveMatrixMonitorCommandAccess` only considered the
channel/room allow lists during the Matrix-specific command pre-gate. Explicit
Matrix owners configured in `commands.ownerAllowFrom` were not merged into that
pre-gate, so `/status` and `/model` could be filtered out before core command
handling.

## Fix

Thread replies now preserve `replyToId` unless `replyToMode` is `off`. The
handler sets `reply.replyToId` to the current Matrix event ID when the inbound
message belongs to a thread, and draft streaming uses the same current event ID
for threaded replies.

Matrix command access state now carries normalized `commands.ownerAllowFrom`
entries. `resolveMatrixMonitorCommandAccess` merges those explicit owner
entries into the direct-message and room command allow lists used by the
Matrix-specific pre-dispatch gate.

## Reproduction tests

The tests were written first to reproduce the observed failures:

- `extensions/matrix/src/matrix/send.test.ts` verifies threaded sends include
  `m.in_reply_to` pointing at the current thread message while `event_id`
  remains the thread root.
- `extensions/matrix/src/matrix/monitor/replies.test.ts` verifies
  `deliverMatrixReplies` preserves `replyToId` for threaded replies.
- `extensions/matrix/src/matrix/monitor/access-state.test.ts` verifies
  explicit Matrix owner entries authorize Matrix owner commands at the
  pre-dispatch gate.
- `extensions/matrix/src/matrix/monitor/handler.test.ts` verifies `/status`
  and `/model` Matrix owner text commands dispatch, and threaded inbound
  messages carry the current event as `ReplyToId`.
- `extensions/matrix/src/matrix/monitor/handler.body-for-agent.test.ts`
  verifies agent context for threaded Matrix messages includes the expected
  reply/thread identifiers.

## Verification

Focused Matrix tests:

```sh
node scripts/run-vitest.mjs run \
  --config test/vitest/vitest.extension-matrix.config.ts \
  extensions/matrix/src/matrix/send.test.ts \
  extensions/matrix/src/matrix/monitor/replies.test.ts \
  extensions/matrix/src/matrix/monitor/access-state.test.ts \
  extensions/matrix/src/matrix/monitor/handler.test.ts \
  extensions/matrix/src/matrix/monitor/handler.body-for-agent.test.ts
```

Result: 5 test files passed, 180 tests passed.

Full Matrix extension suite:

```sh
node scripts/run-vitest.mjs run \
  --config test/vitest/vitest.extension-matrix.config.ts \
  extensions/matrix/src
```

Result: 117 test files passed, 1327 tests passed.

Production source build:

```sh
pnpm build
```

Result: completed successfully.

## Runtime notes

The live gateway was migrated from the packaged CLI to a source-built
`v2026.6.1` checkout using the Volta-managed Node runtime. A stale profile-local
`@openclaw/matrix` npm install initially shadowed the bundled source extension;
it was removed after backing up the profile npm metadata. Plugin inspection then
resolved Matrix to the source checkout at version `2026.6.1`.

Rollback snapshots were kept for the systemd user service, profile npm metadata,
and plugin registry state so the runtime can be returned to the previously
installed package if needed.
