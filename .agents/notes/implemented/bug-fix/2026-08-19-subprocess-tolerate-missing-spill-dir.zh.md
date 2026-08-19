# Agent Note: 子进程 spill 失败保持在工具局部

Status: implemented

[English](2026-08-19-subprocess-tolerate-missing-spill-dir.md) | 中文

## Problem

`OutputCollector.spillAll` 用 `openSync` 打开 spill 文件时没有任何 try/catch。当 spill 目录缺失（或 Windows 下 bash 启动即崩后打开失败）时，异常会从流的 `data` 回调逃逸并拖垮整个 harness 进程。被收集的 stdout/stderr 也缺少 `error` 监听，子进程早死后的断管同样可能变成未捕获异常。

## Decision

用 try/catch 包裹 spill 的打开与写入：失败时调用 `discardSpill()`，仅保留有界内存尾。为被收集的 stdout/stderr 挂上空的 `error` 监听，避免管道错误逃逸。

## Alternatives considered

**失败时重建 spill 目录。** 本次不采用：收集器不应在调用方选定的 spill 根之外创建父目录，且目录缺失往往意味着更广的拆卸竞态。

**让流错误中止子进程句柄。** 否决：退出状态已报告子进程失败；未捕获的流错误比丢失 spill 更糟。

## Consequences

缺失 spill 目录时降级为仅内存尾收集，而不再导致进程死亡。单测断言对不存在的 spill 路径 push/finalize 不抛异常。
