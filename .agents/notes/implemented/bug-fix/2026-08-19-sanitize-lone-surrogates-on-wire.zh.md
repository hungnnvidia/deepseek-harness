# Agent Note: 在 DeepSeek 线协议上净化孤立 UTF-16 代理项

Status: implemented

[English](2026-08-19-sanitize-lone-surrogates-on-wire.md) | 中文

## Problem

工具结果（及其他消息字符串）在把二进制衍生文本按 UTF-8/UTF-16 解码时可能含有不成对的 UTF-16 代理项。`JSON.stringify` 会把它们变成 `\uD800`–`\uDFFF` 转义，而 DeepSeek chat-completions 的 JSON 解析器会以 HTTP 400 拒绝。由于被污染的消息持久留在会话日志里，该会话之后的每一轮都会以同样方式失败。

## Decision

`serialize.ts` 在写入线协议的每个字符串上，把不成对的代理项替换为 U+FFFD（用户/系统/工具内容、reasoning、tool-call 的 id/name/arguments、工具描述）。合法的代理对予以保留。

## Alternatives considered

**只改进提供方错误信息、不改请求体。** 不能作为唯一修复：会话仍会砖化，直到手工改日志；在序列化时净化才能恢复会话。

**只净化 tool-result 内容。** 否决：同样的转义也可能出现在用户文本、reasoning 或 tool-call arguments 中。

## Consequences

此前失败的回放请求会序列化为 API 可接受的合法 JSON。单测断言 stringify 不再发出孤立代理转义，同时保留合法 emoji 代理对。
