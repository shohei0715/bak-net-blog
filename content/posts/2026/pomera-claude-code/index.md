---
title: "Pomera を Claude Code 端末にしたら優勝した"
date: 2026-05-27T11:44:54+09:00
draft: false
slug: "pomera-claude-code"
---

> **【この記事は生成AIが書いてます】**

ポメラ DM250 に Debian 12 を入れて Claude Code が動く開発端末に仕立てた話。バッテリ駆動・キーボード一体型・LTEなしの隙間にWi-Fi接続でモバイル Claude Code 端末が完成して、これは控えめに言って優勝だった。

## なぜポメラなのか

- キーボードが本物（折りたためる打鍵感）
- バッテリーが20時間級
- 電源即起動・即執筆
- 余計な通知が来ない
- そして **Linux が動く**（@ichinomoto 氏の DM250 向け Debian イメージ）

iPad mini や軽量ノートでは「気が散る装置」になってしまうのに対し、ポメラは「物理的にテキストしか打てない」のが偉い。ここで Claude Code が動けば、Mac を開かなくても外出先で思考→指示→生成のループが回せる。

## 環境のベース

@ichinomoto 氏配布の DM250 向け Debian イメージ（2022年8月版）を SD に書いて起動。出発点は Debian 11（bullseye）。カーネルは **Linux 3.10.0+**（Pomera 専用、変更不可）。

これが地味にあとあと効いてくる伏線。

## 1. Debian 11 → 12 アップグレード

`sources.list` を bookworm に書き換えて `apt full-upgrade`、これ自体は教科書通り。

ところが完了直後に **SSH が一切通らなくなる**。原因を追うと sshd の seccomp 内で glibc の arc4random が `getrandom()` を呼ぼうとして死んでいた。

```
Fatal glibc error: cannot get entropy for arc4random
```

`getrandom()` システムコールは **Linux 3.17 で追加**。ポメラのカーネルは 3.10 なので、Debian 12 の glibc が前提とする syscall が無い。sshd の seccomp サンドボックスが `/dev/urandom` フォールバックも塞ぐので詰む。

**解決策**: OpenSSH をやめて **dropbear** に切替。

```bash
sudo apt install dropbear
sudo systemctl stop ssh
sudo systemctl start dropbear
```

dropbear は seccomp を使わない軽量実装なので、3.10 でも問題なく動く。カーネルを上げられない環境のリアルな知恵。

## 2. Claude Code を載せる

`@anthropic-ai/claude-code` を `npm -g` で入れて API キーをセット。動いた瞬間に「ポメラの中で Claude Code が応答してる」という事実だけでもう優勝。

> ※ 当時は npm 経由インストールだったが、現在は **tmux + Claude Code** の構成で常用中。

GitHub CLI（`gh`）と git も入れて、PR 作成までポメラ側でできるようにした。

## 3. ターミナルジプシーの旅

ここからが長かった。ポメラはウィンドウマネージャを動かしたくない（メモリとバッテリの都合）ので、`startx` で **ターミナル1本だけを全画面表示**したい。"WMなし・1024x600・絵文字対応・日本語入力可" を満たすターミナルを延々試した。

### fbterm

そもそも X すら起動しない最小構成。コンソールで日本語が出るが、UTF-8 ブロック文字（`█▀▄` 系）が **豆腐で表示できない**。フォントを Unifont/DejaVu に切り替えても改善せず。fbterm 自体が一部 Unicode を描画できない設計上の制限らしい。
→ 却下

### mlterm

xft エンジンだと **BMP 外（U+10000 以降）の絵文字描画が壊れる**。cairo エンジンに切り替えれば BMP は綺麗に描画されるが、🎉🚀 などは1文字目しか描画されないという妙な挙動。
→ 却下

### xterm

絵文字未対応（白黒グリフのみ）。代わりに全機能が動く堅実派。
→ カラー絵文字諦めるならアリ

### xfce4-terminal

VTE ベース、カラー絵文字が綺麗に出る。ただし WM なし環境で `--fullscreen` が完全には効かず、右に5文字分の余白が出る。`--zoom=103` で微調整しようとしたら **クラッシュした**。
→ カラー絵文字の表示は最初に成功した記念碑

### lxterminal（採用）

VTE ベースで軽い。`fontname=Ricty Diminished 12` を設定ファイルで指定。最終的にこれに落ち着いた。

## 4. WM なしで画面いっぱい問題

`--fullscreen` で起動しても、フォントサイズの都合で「文字グリッドが 1024x600 ピクセルにピッタリ収まらない」のは物理的にどうしようもない。14 だと収まらず、12 だと余る。VTE は **エスケープシーケンス `printf '\e[8;rows;cols t'` を意図的に無視する**（セキュリティ理由）ので、ターミナル内からのリサイズも効かない。

最終解は **xdotool で起動後にウィンドウサイズを強制**：

```bash
#!/bin/sh
sleep 1
WID=$(xdotool search --class lxterminal | tail -1)
xdotool windowmove $WID 0 0
xdotool windowsize --sync $WID 1024 600
```

これを `start-uim-screen` の冒頭に仕込んで、起動と同時に 1024x600 にスナップ。WM がなくても完全フルスクリーン化できた。

副産物として `~/bin/termsize` という汎用リサイザもできた：

```bash
termsize 1024 600
```

## 5. 細かい仕上げ

- **絵文字フォント**: `fonts-symbola` + `fontconfig` のフォールバックチェーンで補完
- **罫線崩れ**: `screen` に `cjkwidth off`、ロケールを `ja_JP.UTF-8` に
- **配色**: lxterminal のデフォルト VGA パレットは青が暗すぎて読めないので **Tango** へ変更
- **DNS**: `/etc/resolv.conf` がシンボリックリンクで壊れていたので直接固定
- **ネットワーク起動待ち**: WiFi デフォルトオフ前提なので、ネット系サービスは待たせない設定
- **GitHub キー**: ポメラ専用の ed25519 を生成して登録

## 完成形

- ポメラ電源ON → 約30秒で X 起動 → lxterminal フルスクリーン → tmux + Claude Code
- バッテリ20時間級
- カラー絵文字も罫線も日本語も綺麗に出る
- Tailscale 経由で他端末（Mac, Raspberry Pi）にも SSH 可
- 1kg なくて鞄に放り込める

これで **電車の中でも公園のベンチでも Claude Code でコード書ける**。ポメラ起動の物理的儀式（カチッと展開）が「集中モードに入るスイッチ」として予想以上に効く。

優勝とはこのことかと。

## 教訓メモ

- カーネル更新不可の環境では **dropbear** を覚えておく
- WM なしで絵文字も出したいなら **lxterminal**（VTE 系）
- VTE のリサイズは **xdotool** で強制
- 古いカーネルの罠（`getrandom` など）は OS バージョンを上げて初めて顕在化する
