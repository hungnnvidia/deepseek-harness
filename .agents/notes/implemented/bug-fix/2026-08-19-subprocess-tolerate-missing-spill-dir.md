# Agent Note: Subprocess spill failures stay tool-local

Status: implemented

English | [中文](2026-08-19-subprocess-tolerate-missing-spill-dir.zh.md)

## Problem

`OutputCollector.spillAll` opened the spill file with `openSync` with no try/catch. When the spill directory was missing (or open failed after an early Windows bash crash), the exception escaped a stream `data` callback and crashed the whole harness process. Collected stdout/stderr streams also lacked an `error` listener, so a broken pipe after early child death could similarly become an uncaught exception.

## Decision

Wrap spill open/write in try/catch: on failure, `discardSpill()` and keep only the bounded in-memory tail. Attach a no-op `error` listener on collected stdout/stderr so pipe errors do not escape.

## Alternatives considered

**Recreate the spill directory on failure.** Rejected for this fix: the collector must not create parent dirs outside the caller's chosen spill root, and a missing dir often signals a broader teardown race.

**Let the stream error abort the subprocess handle.** Rejected: exit status already reports child failure; an uncaught stream error is worse than losing spill.

## Consequences

A missing spill directory degrades to memory-tail-only collection instead of process death. Unit coverage asserts push/finalize against a non-existent spill path does not throw.
