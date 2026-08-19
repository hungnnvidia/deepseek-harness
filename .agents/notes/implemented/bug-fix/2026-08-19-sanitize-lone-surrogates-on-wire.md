# Agent Note: Sanitize lone UTF-16 surrogates on the DeepSeek wire

Status: implemented

English | [中文](2026-08-19-sanitize-lone-surrogates-on-wire.zh.md)

## Problem

Tool results (and other message strings) can contain unpaired UTF-16 surrogates when binary-derived text is decoded as UTF-8/UTF-16. `JSON.stringify` turns those into `\uD800`–`\uDFFF` escapes, which DeepSeek's chat-completions JSON parser rejects with HTTP 400. Because the poisoned message is durable in the session log, every later turn of that session fails the same way.

## Decision

`serialize.ts` replaces unpaired surrogates with U+FFFD on every string placed on the wire (user/system/tool content, reasoning, tool-call id/name/arguments, tool descriptions). Valid surrogate pairs are preserved.

## Alternatives considered

**Surface a clearer provider error and leave the body unchanged.** Rejected as the sole fix: the session remains bricked until the log is edited by hand; sanitizing at serialize recovers the session.

**Sanitize only tool-result content.** Rejected because the same escape can appear in user text, reasoning, or tool-call arguments.

## Consequences

Previously-failing replay requests serialize to valid JSON the API accepts. Unit coverage asserts stringify no longer emits lone-surrogate escapes while keeping a valid emoji pair intact.
