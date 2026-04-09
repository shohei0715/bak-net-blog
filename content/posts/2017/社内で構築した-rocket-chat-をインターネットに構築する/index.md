---
title: "社内で構築した Rocket.Chat をインターネットから繋がるようにする"
date: 2017-02-25 17:56:56
draft: false
slug: "社内で構築した-rocket-chat-をインターネットに構築する"
---

社内環境で既に構築してしまった Rocket.Chat を外部から評価したかったのでやり方を模索。

前提としては社内からインターネット上の踏み台サーバーへSSH接続可能な事。

 

Reverse ssh tunnel を安定運用する
[http://qiita.com/syoyo/items/d31e9db6851dfee3ef82]

ここを参考に3000番ポートをポートフォワードで踏み台から転送できるようにする。

これだけで踏み台から(もしくは踏み台にSSHしてポートフォワード掛ければ) Rocket.Chat へ接続できるようになる。

しかしこれだと一々接続しなければ行けないのでさらにこの踏み台サーバーのポートを直接叩けば Rocket.Chatが使えるようになるには下記の設定を入れる。

まずは、踏み台側sshdの設定変更、これで外部から受け付けられる

[code]
nano /etc/ssh/sshd_config
GatewayPorts yesを追加
sudo /etc/init.d/sshd restart
[/code]

次にIPのFORWARD設定

[code]
sudo nano /etc/default/ufw
#DEFAULT_FORWARD_POLICY="DROP"
DEFAULT_FORWARD_POLICY="ACCEPT"

sudo nano /etc/ufw/sysctl.conf
net/ipv4/ip_forward=1
[/code]

最後にiptables(ufw)の変更、

[code]
nano /etc/ufw/before.rules
*nat
:PREROUTING ACCEPT [0:0]
-A PREROUTING -m tcp -p tcp --dst  --dport  -j DNAT --to-destination 127.0.0.1:
COMMIT
ufw reload
[/code]

これで繋がるようになるはず。
