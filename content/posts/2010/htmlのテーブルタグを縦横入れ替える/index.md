---
title: "HTMLのテーブルタグを縦横入れ替える"
date: 2010-04-22 17:11:11
draft: false
slug: "htmlのテーブルタグを縦横入れ替える"
---

某大手ECサイトというか楽天の一括更新はCSVなので必然的にExcelでやる羽目になる。
商品情報をなぜかテーブルタグで書いてたのでそれを縦横入れ替えたいときは下記のようなスクリプトを書けば一括ですね、という話。

[vb]
Option Explicit
Sub main()
Dim SelectCell As Range
Set SelectCell = Range("g3")
SelectCell.Activate
While SelectCell.Value <> ""
If InStr(SelectCell.Value, "table") <> 0 Then
SelectCell.Value = ReplaceHTML(SelectCell.Value)
End If
Set SelectCell = SelectCell.Cells(2, 1)
SelectCell.Activate
Wend
End Sub
Function ReplaceHTML(str As String) As String
Dim pos_header As Integer
Dim pos_footer As Integer
Dim pos_tbl_header As Integer
Dim pos_tbl_footer As Integer
Dim header As String
Dim footer As String
Dim table_data As String
Dim table_body As String
Dim table_header As String
Dim table_footer As String
Dim table_body_th As String
Dim table_body_td As String
Dim temp As String
Dim title As Variant
Dim data As Variant
Dim i As Integer
'テーブルタグまでヘッダを検出する
pos_header = InStr(str, "")
footer = Right(str, Len(str) - pos_footer - 7)
table_data = Mid(str, pos_header, Len(str) - (Len(str) - pos_footer) - pos_header + 8)
'Debug.Print "header------" & header
'Debug.Print "table-------" & table_data
'Debug.Print "footer------" & footer
pos_tbl_header = InStr(table_data, "")
table_footer = Right(table_data, Len(table_data) - pos_tbl_footer)
'テーブルのTR的な部分だけを検出。
table_body = Mid(table_data, pos_tbl_header, Len(table_data) - pos_tbl_header - (Len(table_data) - pos_tbl_footer - 1))
'Debug.Print "テーブルのヘッダ部分" & table_header
'Debug.Print "テーブルのフッタ部分" & table_footer
'Debug.Print "テーブルの本体部分" & table_body
'部分を抜き出す
table_body_th = Left(table_body, InStr(table_body, "") + 4)
table_body_td = Right(table_body, Len(table_body) - Len(table_body_th))
'Debug.Print "Th部分" & table_body_th
'Debug.Print "Td部分" & table_body_td
title = Split(table_body_th, "")
data = Split(table_body_td, "")
For i = 0 To UBound(title)
title(i) = RemoveHTML(title(i))
Next i
For i = 0 To UBound(data)
data(i) = RemoveHTML(data(i))
Next i
For i = 0 To UBound(title) - 1
temp = temp & ""
temp = temp & "" & title(i) & ""
temp = temp & "" & data(i) & ""
temp = temp & ""
Next
ReplaceHTML = header & table_header & temp & "" & footer
End Function
Function RemoveHTML(strHTML) As String
'HTMLタグを削除するファンクション
Dim Flg As Boolean
Dim i As Integer
For i = 1 To Len(strHTML)
If Mid(strHTML, i, 1) = "" Then
Flg = False
Mid(strHTML, i, 1) = " "
ElseIf Flg Then
Mid(strHTML, i, 1) = " "
End If
Next
strHTML = Replace(strHTML, " ", "")
'Debug.Print strHTML
RemoveHTML = strHTML
End Function
Option Explicit
Sub main()Dim SelectCell As Range
Set SelectCell = Range("g3")SelectCell.Activate
While SelectCell.Value <> ""    If InStr(SelectCell.Value, "table") <> 0 Then        SelectCell.Value = ReplaceHTML(SelectCell.Value)    End If    Set SelectCell = SelectCell.Cells(2, 1)    SelectCell.ActivateWend
End Sub
Function ReplaceHTML(str As String) As String
Dim pos_header As IntegerDim pos_footer As IntegerDim pos_tbl_header As IntegerDim pos_tbl_footer As Integer
Dim header As StringDim footer As StringDim table_data As StringDim table_body As StringDim table_header As StringDim table_footer As StringDim table_body_th As StringDim table_body_td As StringDim temp As String
Dim title As VariantDim data As Variant
Dim i As Integer
'テーブルタグまでヘッダを検出するpos_header = InStr(str, "")footer = Right(str, Len(str) - pos_footer - 7)
table_data = Mid(str, pos_header, Len(str) - (Len(str) - pos_footer) - pos_header + 8)
'Debug.Print "header------" & header'Debug.Print "table-------" & table_data'Debug.Print "footer------" & footer
pos_tbl_header = InStr(table_data, "")table_footer = Right(table_data, Len(table_data) - pos_tbl_footer)

'テーブルのTR的な部分だけを検出。table_body = Mid(table_data, pos_tbl_header, Len(table_data) - pos_tbl_header - (Len(table_data) - pos_tbl_footer - 1))
'Debug.Print "テーブルのヘッダ部分" & table_header'Debug.Print "テーブルのフッタ部分" & table_footer'Debug.Print "テーブルの本体部分" & table_body
'部分を抜き出す
table_body_th = Left(table_body, InStr(table_body, "") + 4)table_body_td = Right(table_body, Len(table_body) - Len(table_body_th))
'Debug.Print "Th部分" & table_body_th'Debug.Print "Td部分" & table_body_td
title = Split(table_body_th, "")data = Split(table_body_td, "")
For i = 0 To UBound(title)    title(i) = RemoveHTML(title(i))Next i
For i = 0 To UBound(data)    data(i) = RemoveHTML(data(i))Next i
For i = 0 To UBound(title) - 1    temp = temp & ""    temp = temp & "" & title(i) & ""    temp = temp & "" & data(i) & ""    temp = temp & ""Next
ReplaceHTML = header & table_header & temp & "" & footer
End Function
Function RemoveHTML(strHTML) As String'HTMLタグを削除するファンクション
Dim Flg As Boolean
Dim i As Integer
For i = 1 To Len(strHTML)    If Mid(strHTML, i, 1) = "" Then        Flg = False        Mid(strHTML, i, 1) = " "    ElseIf Flg Then        Mid(strHTML, i, 1) = " "    End If Next strHTML = Replace(strHTML, " ", "")'Debug.Print strHTMLRemoveHTML = strHTMLEnd Function
[/vb]
