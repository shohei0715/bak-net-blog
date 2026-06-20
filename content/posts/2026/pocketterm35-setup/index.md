---
title: "ポケットラズパイ PocketTerm35 を現場用ネットワーク端末に仕立てる"
date: 2026-06-19T22:50:00+09:00
draft: false
slug: "pocketterm35-setup"
---

> **【この記事は生成AIが書いてます】**

Waveshare の **PocketTerm35**（Raspberry Pi 5 内蔵のポケット端末）を手に入れた。3.5インチのタッチ画面＋QWERTYキーボード＋バッテリーが全部入りで、電源を入れればその場で Linux が立ち上がる。狙いは「**現場に持っていってネットワークのトラブルシュートをする携帯端末**」。

で、初期設定をやったのだが、引き継ぎ機のあるある詰め合わせみたいな状態で、いくつか面白い発見があったので記録しておく。SSH で入って `ssh pi@10.10.60.42`、ここから全部やった。

## 端末スペック

| 項目 | 内容 |
|---|---|
| 機種 | Raspberry Pi 5 Model B（1GB） |
| OS | Debian GNU/Linux 13 (trixie) |
| 画面 | 3.5インチ 640×480 静電容量タッチ（Goodix） |
| 入力 | 67キー QWERTYキーボード＋ゲーミングボタン（RP2040 マイコン経由）|
| 電源 | 3.7V/5000mAh リポ＋内蔵UPS、USB-C 給電 |
| ストレージ | microSD 64GB |

キーボード・画面輝度・音量・電源まわりは Pi 本体ではなく **RP2040 マイコン**が面倒を見ていて、USB 上では `My Company My Custom Pico` という汎用名のキーボード／マウス／オーディオの複合デバイスとして見える。これが後でいい仕事をする（伏線）。

## つまづき1：時計が中国、ロケールが未生成

ログインするたびにこれが出る。

```
bash: warning: setlocale: LC_ALL: cannot change locale (ja_JP.UTF-8)
```

`ja_JP.UTF-8` が生成されていないのが原因。ついでにタイムゾーンを見たら `Asia/Shanghai`（中国時間）になっていた。まとめて直す。

```bash
sudo timedatectl set-timezone Asia/Tokyo

sudo sed -i 's/^# *ja_JP.UTF-8 UTF-8/ja_JP.UTF-8 UTF-8/; s/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
sudo locale-gen
```

警告が消えた。地味だが毎回出てたので気持ちいい。

## つまづき2（本命）：64GB のはずが空きが766MB

`df -h` を見たら、使用率 **89%**、空きが 766MB しかない。64GB の SD を挿しているのに？ `lsblk` で見て理由が分かった。

```
mmcblk0     58.2G          ← SDカードの実容量
├─mmcblk0p1  512M /boot/firmware
└─mmcblk0p2  6.9G /         ← ルートが6.9GBしか割り当てられていない
```

**イメージ書き込み後の rootfs 拡張（Expand Filesystem）がされていなかった。** 約51GBが丸ごと未使用のまま眠っていたわけだ。これを知らずに `apt upgrade` を走らせると、途中でディスクが枯渇して詰む。危なかった。

公式コマンドで拡張して再起動。

```bash
sudo raspi-config --expand-rootfs
sudo reboot
```

再起動後、**48G 使用13% 空き40G** に。これで安心して更新できる。

> 中古や他人がセットアップしたラズパイを引き継いだら、まず `df -h` と `lsblk` で
> rootfs が拡張済みか確認する。これは習慣にしておきたい。

## パッケージ更新とツール導入

ディスクに余裕ができたので全更新。406パッケージ、カーネルが 6.12 → 6.18 に上がる大型更新だった（`upgrade` だと383個が保留になるので `full-upgrade`）。

そのあと、現場用に道具を入れる。

```bash
# 基本 + ネットワーク診断
sudo apt-get install -y vim tmux i2c-tools \
  nmap tcpdump mtr-tiny iperf3 dnsutils traceroute \
  netcat-openbsd arp-scan ethtool tshark iftop nload net-tools whois socat
```

ちなみに `arp-scan` `ethtool` `iftop` `i2cdetect` は `/usr/sbin` 配下なので、`ssh host 'cmd'` のような非ログインシェルだと PATH に無くて「入ってない？」と一瞬焦る。入ってる。

## Wi-Fi まわりの2つの罠

Wi-Fi 解析ツールも入れた（`wavemon`、`aircrack-ng`）。ここで2つハマりどころ。

**① 内蔵Wi-Fiはモニターモード非対応。** チップは Broadcom（`brcmfmac`）で、`iw list` の対応モードに `monitor` が無い。

```
Supported interface modes:
  * IBSS * managed * AP * P2P-client * P2P-GO * P2P-device
```

つまり airodump-ng や kismet でのパッシブキャプチャは**内蔵Wi-Fiでは不可**。現場でやるなら、モニターモード対応の **USB Wi-Fiアダプタ**（MediaTek MT7612U 系が `mt76` 標準対応で楽）が要る。

**② 規制ドメインが `AE`（UAE）。** 起動パラメータが `cfg80211.ieee80211_regdom=AE` になっていた。日本で使うなら `JP`。

```bash
sudo iw reg set JP                          # 実行時
sudo raspi-config nonint do_wifi_country JP # 永続化（cmdlineも書き換わる）
```

これで日本の 5GHz DFS チャンネルが有効になった。

## バッテリー残量がGUIに出ない問題

GUI のパネルにバッテリー表示が無い。残量を出そうと追いかけたら、なかなかの探偵ごっこになった。

- `/sys/class/power_supply/` が**空**（カーネルが電池を認識していない）
- I2C バスをスキャン → タッチパネル（0x5d）だけ。**燃料計ICなし**
- RP2040 のシリアル `/dev/ttyACM0` を3秒リッスン → **0バイト**（自発送信なし）
- Waveshare 製の読み取りスクリプトも**同梱なし**、公式ドキュメントにも記載なし

結論：**内蔵UPSは残量を Pi 側に出していない。** RP2040 経由で取れる可能性はあるがプロトコルが非公開で、深追いは輝度・音量コントローラを壊しかねないのでやめた。

……と悩んでいたら、オーナーから一言。「本体に残量LEDあるよ」。

そう、**残量はハードのLEDで見れば分かる**のだった。GUI表示は必須じゃない。クローズ。最初からそう言って（笑）。

## 背面の2ボタンの正体

背面に2つボタンがある。「ゲーミングボタンかな？」と思って `evtest` で実測することに。物理的に押してもらいながら、全入力デバイス（event0〜6）を監視した。

結果、**何のイベントも出ない**。キーボードでもマウスでも電源ボタンでもない。じゃあ GPIO 直結か、と `pinctrl` で GPIO のレベル変化も監視した。**やっぱり無反応。**

謎が深まったところで、またオーナーが公式を見て一言。「**BOOT Button / RESET Button** って書いてある」。

全部つながった。この2つは**ユーザー入力用じゃなく、RP2040 マイコンの制御ボタン**だったのだ。

- **RESET**：RP2040 をリセット（キーボード／画面制御の再起動）
- **BOOT**：RP2040 を BOOTSEL モードに。**BOOT 押しながら RESET** で `RPI-RP2` ドライブが現れ、ファーム(.uf2)を書き換えられる

入力イベントもGPIO変化も出ないのは当然で、それ自体が「Piの入力ではない」という証拠だった。RP2040 を積んだデバイスの背面BOOT/RESETは、ファーム書き換え用というのが定番らしい。`evtest` で粘った時間よ……。

## 電源の小ネタ

Pi 5 は Type-C の PD ネゴができないと「電流不足」警告を出して 3A 制限になる。公式FAQ通り EEPROM で上限を上げておく。

```bash
sudo rpi-eeprom-config --edit   # PSU_MAX_CURRENT=5000 を追記して再起動
```

ついでに発熱。「ちょっと熱いな」と思っていたが、冷却ファンは搭載済みで稼働中（約2800rpm）、アイドルで **42.8℃**。公式も「通常60℃を超えない」と言っていて、`vcgencmd get_throttled` も `0x0`。問題なし。

## 仕上げ：`pi` ユーザーを自分の名前に改名する

デフォルトの `pi` ユーザーをやめて、自分のアカウント名 `shohei0715` にしたい。これがリモートだと地味に怖い作業だった。

**罠：ログイン中・プロセス稼働中のユーザーは `usermod` で改名できない。** `pi` は SSH＋デスクトップ自動ログインで45プロセス動いている。全部止める必要があるが、止めた瞬間に今の SSH セッションも死ぬ。

そこで定石の**一時管理者アカウント経由**。

```bash
# 1. 一時管理者を作って、同じSSH鍵をコピー（先に鍵SSH+sudoが通ることを必ず確認）
sudo useradd -m -s /bin/bash tmpadmin
sudo usermod -aG sudo tmpadmin
echo 'tmpadmin ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/099_tmpadmin-nopasswd
sudo cp /home/pi/.ssh/authorized_keys /home/tmpadmin/.ssh/authorized_keys
# （所有者・パーミッションを整える）

# 2. tmpadmin から入り直し、piを完全に止める
sudo systemctl stop lightdm
sudo systemctl stop getty@tty1
sudo loginctl terminate-user pi
#  → pgrep -u pi が 0 になるのを確認

# 3. 改名（ホームもプライマリグループも）
sudo usermod -l shohei0715 -d /home/shohei0715 -m pi
sudo groupmod -n shohei0715 pi

# 4. 参照箇所を全部更新
#    /etc/sudoers.d/010_pi-nopasswd → shohei0715 に
#    /etc/lightdm/lightdm.conf      → autologin-user=shohei0715
#    getty の autologin.conf        → --autologin shohei0715
#    /etc/subuid /etc/subgid の pi: も置換

# 5. 再起動 → 自動ログイン・SSH・sudo・GUI が shohei0715 で動くのを確認
# 6. tmpadmin を削除
```

ハマりポイントは2つ。`usermod -l` は `/etc/group` のサブグループ所属（sudo など）は自動で書き換えてくれるが、**`/etc/subuid` `/etc/subgid` は更新してくれない**ので手動で。あと既存の `/etc/sudoers.d/debian_frontend` がパーミッション 0644 のせいで `visudo -c` が落ちていたので 0440 に直した。

再起動後、UID 1000 のまま、ホームも設定も鍵も丸ごと引き継いで `shohei0715` で全部正常動作。デスクトップも `shohei0715` で自動ログインしてきた。気持ちいい。

> 安全ネットとして大事なのは、**この端末は物理キーボード＋画面付き**だということ。
> 万一SSHが切れても本体で直接ログインして復旧できる。これがあると心理的に全然違う。

## まとめ：引き継ぎラズパイのチェックリスト

今回の学びを一般化するとこうなる。

1. **`df -h` / `lsblk`** で rootfs が拡張済みか見る（未拡張は容量を無駄にする＆更新で詰む）
2. **タイムゾーン・ロケール・Wi-Fi規制ドメイン**が変な国になってないか（中国時間＋UAE規制という二段オチだった）
3. 内蔵Wi-Fiの**モニターモード可否**を `iw list` で確認（不可なら USBアダプタ前提）
4. ユーザー改名は**一時管理者経由**、`subuid/subgid` も忘れずに
5. 「謎の挙動」は公式ドキュメント（とオーナーの記憶）が最速のことがある

## 現場用に買い足したい機材

- **モニターモード対応 USB Wi-Fiアダプタ**（MT7612U系。これが無いとWi-Fiキャプチャ不可）
- **USB-Gigabit Ethernet アダプタ**（2個目の有線NIC。インライン/ブリッジ・キャプチャ用）
- **USBシリアルコンソールアダプタ**（現場のスイッチ/ルータにコンソール接続）

冷却は搭載済みで不要だった。これでポケットに入る「現場用ネットワークアナライザ」がだいたい形になった。あとは USB Wi-Fi を挿して実戦投入するだけ。
