---
title: "久々に"
date: 2010-04-02 19:59:16
draft: false
slug: "久々に"
---

VBA書いたので残しておくｗ
Set ～思い出すのに3分かかったｗ

> Sub main()
> Dim SelectCell As Range
> 
> Set SelectCell = Range("i2")
> SelectCell.Activate
> 
> While SelectCell.Value <> ""
> 
> SelectCell.Value = ReplaceHTML(SelectCell.Value)
> 
> Set SelectCell = SelectCell.Cells(2, 1)
> SelectCell.Activate
> 
> Wend
> 
> End Sub
> 
> Function ReplaceHTML(str As String) As String
> 
> Dim pos As Integer
> Dim str1 As String
> Dim str2 As String
> 
> '最初のTRまで検出する
> pos = InStr(str, " 
> str1 = Left(str, pos)
> str2 = Right(str, Len(str) - pos)
> 
> str1 = Replace(str1, " str2 = Replace(str2, " 
> 
> ReplaceHTML = str1 & str2
> 
> End Function
