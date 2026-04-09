---
title: "M5Stackはじめました"
date: 2019-11-27 02:56:28
draft: false
slug: "m5stackはじめました"
---

M5Stackをはじめたのでblogもお掃除してみた。

とりあえず最初の一歩。

Serial接続は115200bpsにしないと化ける。

Wi-Fiの接続はこう

[python]
import network
wlan = network.WLAN(network.STA_IF)
wlan.active()
wlan.connect('SSID','PASSWORD')
wlan.isconnected()
[/python]

isconnecte()でTrueが帰ってきたら接続できてるね！
