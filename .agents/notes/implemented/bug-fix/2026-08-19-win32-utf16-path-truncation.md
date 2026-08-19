# Agent Note: Win32 folder picker reads full UTF-16LE code units

Status: implemented

English | [中文](2026-08-19-win32-utf16-path-truncation.zh.md)

## Problem

`readUtf16()` in the native Win32 directory picker stopped the NUL scan when the low byte of a UTF-16LE code unit was `0x00`. Code points of the form U+XX00 (notably U+5F00 开) encode as `00 XX`, so paths such as `...\00-软件开发\...` truncated before 开 (`...\00-软件`). Workspace creation then failed with ENOENT against the truncated path.

## Decision

Treat a UTF-16LE NUL as two zero bytes: advance while either byte of the code unit is non-zero. That matches the Windows wide-string terminator and keeps every BMP code point, including those whose low byte is zero.

## Alternatives considered

**Decode via `koffi.decode(addr, 'str16')`.** Rejected earlier and still: that path dereferences the out-param as a pointer and crashes on real Windows; the direct memory view remains required.

**Reject or rewrite paths containing U+XX00.** Rejected because those characters are valid in Windows paths and common in zh-CN directory names.

## Consequences

Native folder picks that include characters like 开 / 一 keep the full filesystem path into workspace creation. The fake-koffi bindings suite covers a path containing U+5F00.
