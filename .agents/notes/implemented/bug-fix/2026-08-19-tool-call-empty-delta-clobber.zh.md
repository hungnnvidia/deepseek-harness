# Agent Note: 忽略流式续片中空的 tool-call id/name

Status: implemented

[English](2026-08-19-tool-call-empty-delta-clobber.md) | 中文

## Problem

`dsh-llm-deepseek` 的 SSE `translate()` 在 `tool_calls[].id` 与 `function.name` 只要“有字段”（`!== undefined`）就会写入。部分 OpenAI 兼容网关会把续片里本应省略的字段序列化成显式空值（`id: ""`、`name: null`），从而覆盖首片里的真实 id/name。组装后的 tool call 变成空 `name`/`callId`，工具策略校验失败（`tool "" is disabled`），headless 回合却仍以干净的 `finish=stop` 结束、无输出。

## Decision

只把非空字符串当作权威值：当续片携带长度 > 0 的字符串时才赋值 `callId`/`name`。空字符串与 `null` 被忽略，首片身份得以保留。若整段流从未给出身份，关闭时仍回退到 `""`，与原有防御行为一致。

## Alternatives considered

**在续片出现空 id/name 且此前已有非空值时直接让流失败。** 本次修复不采用：这类网关在其他方面可用，缺陷是覆盖写入，不是字段出现本身。

**改在 `BlockAssembler` 规范化，而不是 DeepSeek 翻译层。** 不采用：空值覆盖属于 `dsh-llm-deepseek` 的线协议翻译问题；组装器正确保留先关闭的块。

## Consequences

对续片发送空 id/name 的网关，tool call 保留首片身份并通过策略校验。单测在既有“从未携带 id/name”防御用例旁锁定该网关形态。
