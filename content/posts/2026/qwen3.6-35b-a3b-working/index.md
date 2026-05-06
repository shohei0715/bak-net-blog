---
title: "続・Local LLM 選定 — Qwen3.6 35B-a3b を完全 VRAM fit させた"
date: 2026-05-06T17:37:07+09:00
draft: true
slug: "qwen3.6-35b-a3b-working"
---

> **【この記事は生成AIが書いてます】**

[前回の記事]({{< ref "posts/2026/local-llm-selection" >}})では `qwen3.6:35b-a3b` が Q4_K_M で 24GB VRAM にギリギリ入らず、4.6GB が CPU 行きになって t/s が 1/4 まで落ちた。今回その続報。**完全 VRAM fit させたら速度が一気に伸びた話**。

## 何を変えたか

<!-- TODO: 採用した量子化（Q3_K_M / IQ3_M / その他）と入手元を書く -->

- 量子化:
- 入手元（HuggingFace のリポジトリ等）:
- ファイルサイズ:
- ctx:

## 計測

| モデル | 量子化 | ctx | t/s | VRAM fit | 備考 |
| --- | --- | --: | --: | --- | --- |
| `qwen3.6:35b-a3b` | Q4_K_M | 16k | 22.2 | ✗ (CPU 4.6GB) | 前回の値 |
| **`qwen3.6:35b-a3b`** | **?** | **16k** | **?** | **✓** | **今回** |
| `qwen3.5-35b-nothink` | Q4_K_S | 16k | 53 | ✓ | 前回 primary |

<!-- TODO: 実測値と nvidia-smi のスクショを差し込む -->

## 体感の変化

<!-- TODO: thinking ON/OFF、応答品質、長文タスクでの挙動など -->

## primary 切り替え

<!-- TODO: openclaw.json の primary を qwen3.6:35b-a3b に切り替えたか -->

## 学び

- **「乗らない」と「乗る」の差は構造的**。前回まとめた「VRAM fit が t/s をほぼ決める」がそのまま再現された。
- 量子化を 1 段下げる判断: 品質劣化の許容範囲を実タスクで確認する必要がある。
- <!-- TODO: 追加の気づき -->

## 次に試すこと

- <!-- TODO: -->
