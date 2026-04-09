---
title: "OpenOffice Base BasicでSQLクエリを送るときのコツ"
date: 2012-02-15 17:46:22
draft: false
slug: "openoffice-base-basicでsqlクエリを送るときのコツ"
---

・タイムスタンプをSQLに埋め込みたい時は変換する

Dim NowText as string

NowText = now()
NowText = Replace(NowText,"/","-")

・接続はこんな感じ

Dim oDoc
Dim Connection

' フォームオブジェクト取得
oDoc = ThisComponent

' 接続を取得
oForm = oDoc.getDrawPage().getForms().getByName("MainForm")
Connection = oForm.ActiveConnection

'SQLを発行する
Statement = Connection.createStatement()
ResultSet = Statement.executeQuery("SELECT ""HOGE"" FROM tbl_FugaFuga")

' 接続を閉じる
ResultSet.Close()
Statement.Close()
