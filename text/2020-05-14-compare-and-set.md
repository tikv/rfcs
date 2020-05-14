# Compare-And-Set

## Summary

It allows to conditionally update a key, the condition being that the key has a certain value(maybe not exist).

## Motivation

It allows to implement read-modify-write scenarios.

## Design

- We can run the Compare-And-Set operation in the Scheduler which use Latch to control write concurrency.
- If there is conflicts, return fail immediately or wait in the queue

## Alternatives

- Use Txn Api, get and set

## Unresolved questions
