# Agent Note: Ignore empty tool-call id/name on stream continuations

Status: implemented

English | [中文](2026-08-19-tool-call-empty-delta-clobber.zh.md)

## Problem

`dsh-llm-deepseek`'s SSE `translate()` applied `tool_calls[].id` and `function.name` whenever the fields were present (`!== undefined`). OpenAI-compatible gateways sometimes serialize omitted continuation fields as explicit empties (`id: ""`, `name: null`) instead of omitting them. Those values overwrote the real id/name from the first delta, so assembled tool calls ended with empty `name`/`callId`, failed tool-policy checks (`tool "" is disabled`), and left headless turns finishing cleanly with no output.

## Decision

Treat only non-empty strings as authoritative: assign `callId`/`name` when the delta carries a string of length > 0. Empty string and `null` are ignored so the first-delta identity sticks. A never-named call still falls back to `""` at close, matching the prior defensive empty-string behavior for streams that never send identity at all.

## Alternatives considered

**Fail the stream when a continuation carries an empty id/name after a non-empty one.** Rejected for this fix because gateways that emit the empty form are otherwise usable; overwriting was the defect, not the presence of the field.

**Normalize at `BlockAssembler` instead of the DeepSeek translator.** Rejected because the empty overwrite is a wire-translation bug owned by `dsh-llm-deepseek`; the assembler correctly keeps the first closed block.

## Consequences

Tool calls against gateways that emit empty continuation id/name keep their first-delta identity and succeed policy checks. Unit coverage locks the gateway shape next to the existing "never carried id/name" defensive case.
