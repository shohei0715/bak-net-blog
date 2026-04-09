---
title: "php70-php-gdを入れても設定が反映されなかった"
date: 2016-05-08 02:51:20
draft: false
slug: "php70-php-gdを入れても設定が反映されなかった"
---

WordpressではGDというライブラリを使い画像の処理を行っているそうな。
サムネイル(アイキャッチ)の縮小や画像編集で「ご利用中のホスティング環境は画像の回転機能に対応していません。」が出る場合はこれが上手く動いていないのが原因。

CentOSなのでこんな感じ

[bash]
# yum list |grep php-gd
php70-php-gd.x86_64                        

#yum install php70-php-gd
[/bash]

これで使えるようになると思ったらどうも反応しない。
phpinfoにもGDの項目がenable以前に表示もされていない。

というわけで下記を行なったところ解決。

 

[code title="/etc/php.d/20-gd.ini"]
extension=/opt/remi/php70/root/usr/lib64/php/modules/gd.so
[/code]
