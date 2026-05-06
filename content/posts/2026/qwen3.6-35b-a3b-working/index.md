---
title: "続・Local LLM 選定 — Qwen3.6 35B-A3B を ncmoe で 64 t/s 出した話"
date: 2026-05-06T17:37:07+09:00
draft: true
slug: "qwen3.6-35b-a3b-working"
---

> **【この記事は生成AIが書いてます】**

[前回の記事]({{< ref "posts/2026/local-llm-selection" >}})では `qwen3.6:35b-a3b` が Q4_K_M で 24GB VRAM にギリギリ入らず、4.6GB が CPU 行きになって t/s が 1/4 まで落ちた。前回は妥協策として `qwen3.5-35b-nothink` (Q4_K_S, 53 t/s) を primary に据えた。

今回その続報。**`qwen3.6:35b-a3b` を Q4_K_M のまま使いつつ 64 t/s 出るようになった**。Q3 にしなくて済んだ。鍵は `--n-cpu-moe`（`-ncmoe`）という llama.cpp のフラグ。

## 何を変えたか

- **量子化**: Q4_K_M（前回と同じ。落とさず）
- **入手元**: [unsloth/Qwen3.6-35B-A3B-GGUF](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF) の `Qwen3.6-35B-A3B-UD-Q4_K_M.gguf`
- **ファイルサイズ**: 22.1 GB
- **runtime**: **Ollama を捨てて llama.cpp 直起動**
- **ctx**: 128k（前回 16k だったのを大幅拡張）
- **新フラグ**: `-ncmoe 5`（MoE エキスパート 5 層を CPU に逃がす）
- **KV cache**: `q8_0`（前回 `q4_0` 強制してた、品質寄りに変更）

要するに：

1. Q3 に妥協する前に `-ncmoe` を試したら勝ってしまった
2. ついでに **Ollama の `OLLAMA_KV_CACHE_TYPE=q4_0` が実質効いてない** ことも判明した（これは別の罠）

起動コマンド：

```bash
docker run -d --name llamacpp-long --restart=unless-stopped --gpus all \
  -p 18080:8080 \
  -v ~/ggufs/Qwen3.6-35B-A3B-UD-Q4_K_M.gguf:/m.gguf:ro \
  ghcr.io/ggml-org/llama.cpp:server-cuda \
  -m /m.gguf -ngl 999 -ncmoe 5 -fa on \
  --cache-type-k q8_0 --cache-type-v q8_0 \
  -c 131072 -np 1 -fit off \
  --jinja --reasoning off \
  --alias qwen3.6-35b-a3b \
  --host 0.0.0.0 --port 8080
```

## `-ncmoe` とは

`--n-cpu-moe N`（短縮 `-ncmoe N`）は llama.cpp が持つ MoE 専用のオフロードフラグ。**最初の N 層分の MoE エキスパート重みを CPU/RAM 側に置き、それ以外（attention・router・残りのエキスパート）を GPU に置く**。

普通の `-ngl` だけだと「層」単位で粗くしか分けられないが、`-ncmoe` は「層内の MoE expert 部分だけ CPU」という細かい切り出しができる。

35B-A3B は active 3B の MoE なので、token ごとに使われる expert は層に対して 8 個中 4 個程度しか発火しない。**CPU に逃した expert がたまたま選ばれた時だけ CPU 計算が走る**仕組みで、平均的にはそこまでボトルネックにならない。

## ncmoe スイープ実測

同一プロンプト「東京の名物を3つ短く」を 3 回計測した平均。ctx は 128k 固定。

| ncmoe | VRAM (GPU0+GPU1) | t/s |
| ---: | ---: | ---: |
| 0 | OOM | - |
| **5** | **21.6 GB** (10.0 + 11.6) | **64.1** |
| 10 | 19.3 GB (7.6 + 11.7) | 56.1 |
| 15 | 16.9 GB (5.3 + 11.6) | 49.9 |
| 25 | 12.3 GB (2.5 + 9.8) | 41.0 |

**ncmoe=5 がスイートスポット**。`ncmoe=0` は OOM で起動失敗、`5` まで GPU 寄せで「ギリ収まる」最小値で最速。

ちなみに記事ネタにした他人の RTX 4070 12GB（単機）+ `-ncmoe 25` では 60 t/s 出てるらしい。同じ Q4_K_M で **24GB ある我々の方が `-ncmoe` を 5 まで詰められて 64 t/s**、というのは VRAM の差がそのまま GPU 比率に効いてる。

## CPU/RAM 側のコスト

推論中の実測：

| リソース | 値 |
| --- | --- |
| llama.cpp プロセス CPU | 約 33%（12 コア中 ~3 コア相当） |
| llama.cpp プロセス RSS | 6.5 GB |
| OS の mmap キャッシュ | 16 GB（GGUF ファイルを OS が抱える） |
| GPU0 / GPU1 utilization | 推論中はほぼ常時動いてる |

CPU 33%、RAM 6.5 GB を犠牲にして **dense 27B (18 t/s) の 3.5 倍速** で MoE 35B が動く。サーバ全体としては全然余裕。

## 計測まとめ（前回と並べる）

| モデル | 量子化 | ctx | t/s | 構成 |
| --- | --- | ---: | ---: | --- |
| `qwen3.5-35b-nothink` | Q4_K_S | 16k | 53 | 前回 primary（Ollama） |
| `qwen3.6:27b` (dense) | Q4_K_M | 262k | 18 | dense 27B 試用、長文脈用 |
| `qwen3.6:35b-a3b` | Q4_K_M (Ollama) | 16k | 22.2 | 前回の悲しい値（CPU 4.6GB オフロード） |
| **`qwen3.6:35b-a3b`** | **Q4_K_M (UD)** | **128k** | **64.1** | **今回（llama.cpp + ncmoe=5）** |

同じモデル・同じ量子化でランタイムを変えて **22 t/s → 64 t/s（3 倍弱）**。ctx も 16k → 128k に伸びてる。

## おまけ: Ollama の `OLLAMA_KV_CACHE_TYPE=q4_0` は効いてなかった

前回は Ollama 側で `OLLAMA_KV_CACHE_TYPE=q4_0` 設定して KV を q4 量子化してるつもりだった。が、別の検証で **同じ GGUF を llama.cpp 直で `--cache-type-k/v q4_0` 指定すると KV サイズが 14 GB 削減** されることが分かった（dense 27B で確認）。Ollama 側は env を読んでも内部で fp16 にフォールバックしてる気配。

つまり前回の「VRAM ギリ入らず 1/4 落ち」は **本来 q4_0 KV cache が効いていれば回避できたかも** しれない。Ollama を信じすぎず、`/api/ps` の `size_vram` を実測する癖をつけた方がいい。

詳細は別記事にする予定。

## primary 切り替え

`openclaw.json` の primary を `kk-ai2-long/qwen3.6-35b-a3b` に変更。OpenClaw / OpenCode / Open WebUI 全部このエンドポイント経由になった。

切替後の Slack OpenClaw2 の体感は明確に速い。前 primary の 53 t/s でもそんなに不満はなかったが、64 t/s + 一世代新しい model + 128k ctx になると「待ってる」感がほぼ消える。

## 学び

1. **`-ncmoe` は MoE モデルのコンシューマ運用における ace カード**。Q3 に量子化を落とす前に必ず試す。VRAM 容量に合わせて N を詰める（小さいほど GPU 寄せで速い、ただし OOM ギリ）。
2. **MoE は dense より「VRAM に何を置くか」の自由度が高い**。dense は 1 層丸ごと GPU か CPU か。MoE は層内で attention は GPU、expert は CPU、と分割できる。これが ncmoe の効く理由。
3. **Ollama の env 設定は信用しすぎない**。`/api/ps` の実測 VRAM と、同じ GGUF を llama.cpp 直で叩いた時の VRAM を比べると挙動の差が見える。
4. ハードウェアを買い替えなくても、**ランタイム選びとフラグ調整だけで 3 倍速**になる例。コンシューマ GPU 2 枚運用は工夫の余地が大きい。

## 次に試すこと

- **AutoRound INT4 + vLLM + MTP** ルート（ネタ元の記事で 139 t/s と報告されてた）。vLLM のセットアップコストが高いが、上記より +2x の余地がある。
- **Q3_K_M / IQ3_M** で完全 VRAM fit させた場合の比較（今のところ ncmoe で十分速いので緊急性は無い）。
- **長文脈フィル時の t/s 維持**確認。今回の 64 t/s は短プロンプト時の値。実際に 100k トークン詰めた状態でどこまで落ちるか、まだ計ってない。
