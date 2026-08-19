# Agent Note: Win32 文件夹选择器按完整 UTF-16LE 码元读取

Status: implemented

[English](2026-08-19-win32-utf16-path-truncation.md) | 中文

## Problem

原生 Win32 目录选择器里的 `readUtf16()` 在 UTF-16LE 码元的低字节为 `0x00` 时就停止 NUL 扫描。形如 U+XX00 的码点（尤其是 U+5F00「开」）编码为 `00 XX`，因此 `...\00-软件开发\...` 这类路径会在「开」之前被截断成 `...\00-软件`，随后按截断路径创建 workspace 时以 ENOENT 失败。

## Decision

把 UTF-16LE 的 NUL 当作两个零字节：仅当码元的两个字节都为零时才结束扫描。这与 Windows 宽字符串终止符一致，并保留包括低字节为零在内的全部 BMP 码点。

## Alternatives considered

**改用 `koffi.decode(addr, 'str16')`。** 此前已否决且仍不适用：该路径会把 out-param 当指针解引用，在真实 Windows 上会崩溃；仍须直接查看内存。

**拒绝或改写含 U+XX00 的路径。** 否决：这些字符在 Windows 路径中合法，且在 zh-CN 目录名中常见。

## Consequences

包含「开」「一」等字符的原生文件夹选择会把完整文件系统路径交给 workspace 创建。fake-koffi 绑定套件覆盖含 U+5F00 的路径。
