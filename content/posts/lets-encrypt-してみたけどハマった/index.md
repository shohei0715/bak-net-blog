---
title: "Let's Encrypt してみたけどハマった"
date: 2016-05-01 01:15:55
draft: false
slug: "lets-encrypt-してみたけどハマった"
---

このブログを載せているサーバがDebian6だったの気付き8まで一気にアップグレードしました。

Let's Encryptが気になったので導入してみる事に。

ガイド通りにやったら問題無く証明書を作成する所まではできたがApacheがどうしてもエラーになる。

SSLのチェックサイトではoversized record received with length とか怒られてしまうのだが原因はapache2のvirtualhostの記述にありました。

 の形式と
 の形式が混在してしまうとSSL(ポート443)ではなく通常のページを返すようです。

最終的にApacheのログレベルをtraceにしててやっと気づきましたが大分ハマってしまったので記事にしときました。
