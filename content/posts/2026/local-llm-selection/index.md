---
title: "Local LLM の選定 — RTX 3060 12GB × 2 で Qwen 3.5 35B MoE に落ち着いた話"
date: 2026-04-25T00:00:00+09:00
draft: false
slug: "local-llm-selection"
---

> **【この記事は生成AIが書いてます】**

社内で運用している OpenClaw（エージェント実行基盤）の既定モデルを何にするか、ここ数日比較していた。環境は RTX 3060 12GB × 2 の合計 **VRAM 24GB**。Ollama 側のチューニングは以下の通り。

```
OLLAMA_KV_CACHE_TYPE=q4_0
OLLAMA_FLASH_ATTENTION=1
```

結論から言うと、**`qwen3.5-35b-nothink` (Q4_K_S)** を primary に採用した。以下、そこに至る経緯。

## 計測結果

同一プロンプト「東京の名物を3つ短く」を 3 回計測した平均の体感値。

| モデル | ctx | t/s | VRAM fit | 備考 |
| --- | --: | --: | --- | --- |
| `qwen3.6:27b` (dense) | 32k | ~6 | 部分 CPU | thinking あり、warmup 3.5 分。実用外 |
| `qwen3.6:35b-a3b` (MoE, thinking) | 32k | ~15 | 部分 CPU | thinking で応答が冗長 |
| `qwen3.6-35b-a3b-nothink` | 16k | 22.2 | ✗ (CPU 4.6GB) | Q4_K_M 約 22GB、24GB VRAM にギリ入らず |
| `qwen3:30b-a3b-nothink` | 16k | **84** | ✓ | 最速。ただし世代が一つ古い (Qwen 3.0) |
| **`qwen3.5-35b-nothink` (Q4_K_S)** | **16k** | **53** | **✓** | **採用** |

## 学び

### 1. CPU オフロードが t/s をほぼ決める

`qwen3.6:35b-a3b` は Q4_K_M で約 22GB、24GB VRAM に**ギリギリ入らず**に 4.6GB が CPU 行きになる。たった 4.6GB のオフロードで t/s は 1/4 前後まで落ちる。VRAM に乗るか乗らないかが事実上のスイッチ。

### 2. ctx を削っても CPU オフロードは解消しない

最初「ctx を 32k → 8k に下げれば VRAM に収まるだろう」と思ったが、ダメだった。KV キャッシュを削っても **weights 本体のオーバーフロー**は解決しない。KV と weights は別物。この環境なら **16k ctx で十分**。

### 3. `SYSTEM /no_think` が効く

Qwen 3 系列で thinking ブロックを抑制するのは `SYSTEM /no_think` が確実。`TEMPLATE {{ .Prompt }}` 方式だと Qwen 3.6 では完全には抑制されなかった。

### 4. Q4_K_M と Q4_K_S の差は大きい

`qwen3.5-35b-nothink` を選べた決め手は **Q4_K_S** の存在。Q4_K_M より 1 段粗い量子化だが、そのおかげで 20GB 級に収まり、34.7B MoE のまま完全 VRAM fit する。「新しめ × 速い × VRAM に乗る」のスイートスポットがここにあった。

## 最終構成

- **primary**: `ollama/qwen3.5-35b-nothink:latest`
- **ctx**: 16384（元の Modelfile は 4096。速度はほぼ変わらず 4 倍に引き上げ）
- OpenClaw のモデルピッカーは整理。Qwen 系は primary 1 本のみ表示。日本語特化用の `hf.co/mmnga-o/llm-jp-4-8b-thinking-gguf` だけ別枠で残した。
- Ollama 上の他モデル（qwen3.6 系、qwen3:30b-a3b、qwen3.5:9b 系）は**削除せず保持**。必要になれば openclaw.json に追記するだけで戻せる。

## 今後試したいこと

- HuggingFace の **Q3_K_M / IQ3_M** GGUF 版 `qwen3.6:35b-a3b` なら完全 VRAM fit させて 50 t/s 超えが狙えるはず。品質劣化がどこまで許容できるか次第。
- RAM or VRAM 拡張があれば `qwen3.6:35b-a3b-nothink` の Q4_K_M を完全 VRAM に置けて 50–80 t/s は出ると思う。
- 作ったが未採用の派生モデル
  - `qwen3-30b-a3b-nothink`（速度優先、必要時に呼び出し）
  - `qwen3.6-35b-a3b-nothink`（最新版の試用枠）

## まとめ

**「VRAM にギリギリ乗らない構成は、乗る構成の 1/4 の速度になる」**。これが今回の一番の収穫。ベンチ数字を見るときは t/s の前に VRAM fit か確認する癖をつけたほうがいい。
