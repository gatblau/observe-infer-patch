# NATS subject diff classification

Subjects are the contract. Payload formats are the sub-contract. Both need diffing.

## Subject pattern

A subject pattern like `tenant.*.site.*.appliance.*.command.accepted` has:
- Token positions (7 tokens here).
- Wildcard positions (`*` at 1, 3, 5).
- Literal terminals (`command.accepted`).

## Additive

- New subject pattern (new publisher, new subscriber on a new pattern).
- New optional field in a payload schema.
- New token position appended to a subject pattern where no existing subscriber uses a trailing `>`.
- Broadened subscription (`tenant.*.site.*` → `tenant.>`) in a subscriber — absorbs more messages but does not affect publishers.

## Deprecation

- Subject marked deprecated in documentation or via a `deprecated.` prefix convention (project-specific).
- Publisher continues to emit to both old and new subjects for a transition period.

## Breaking

### Subject
- Subject removed.
- Subject renamed (token changed).
- Token position changed (meaning of position 3 used to be site, now appliance).
- Wildcard position changed (specificity moved).
- Queue group changed on subscribers (affects load distribution — technically breaking for ops).

### Payload
- Field removed from payload.
- Field renamed.
- Field type narrowed or changed.
- Payload format changed (JSON → protobuf, or vice versa).
- Required field added to payload.

### Semantics
- Message frequency changed in a way that floods subscribers.
- Retention policy changed on a JetStream stream (work-queue → limits) — breaking for consumers that relied on at-most-once or max-msgs.

## Ambiguous — flag

- New subject added that overlaps with an existing wildcard subscription. Additive for new consumers but existing subscribers now receive unexpected payloads.
- Publisher stops emitting to a subject without announcing — subscribers silently get zero messages. Classify as breaking; consumers relying on the stream will notice only by absence.

## Evidence format

```
- Subject: tenant.*.site.*.appliance.*.camera.*.status.changed
- Role in base: publisher in lib/nats.go:120
- Role in head: removed
- Classification: breaking — consumers lose the stream
```

## Payload identification

When the subject's payload type is a Go struct or proto message, diff the struct/proto using the corresponding surface's rules (`go-public-rules.md` for structs, `grpc-rules.md` for protos). Report a NATS finding as "subject X: payload struct Y changed — see Go public findings".
