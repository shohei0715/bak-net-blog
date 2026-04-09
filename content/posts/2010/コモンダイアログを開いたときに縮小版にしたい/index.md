---
title: "コモンダイアログを開いたときに縮小版にしたい"
date: 2010-01-19 12:07:44
draft: false
slug: "コモンダイアログを開いたときに縮小版にしたい"
---

職場で事情によりネットショップ・オーナーというソフトを使うことがあるんですけど、画像選択するコモンダイアログがいつもアイコンでしか開かない。Windowsって不便です。

でも毎回縮小表示にするのが面倒なので、何とかしたいと思った。そもそもコモンダイアログだろうとSendMessageでなんとか出来るとだろうと思い検索。すると既に[公開されている方](http://blog.kumacchi.com/2007/05/post_107.html)がいた！それでソースをそのままコピーして、利用するWindows2003とネットショップ・オーナーでも動作するように変更しただけです。ソール公開しているくまっちさんに感謝。

[ここにバイナリ置いてきますね。](/images/SBAN2.zip)主に自分のため。

以下改変したソース

```
#uselib "user32.dll"
#cfunc FindWindow "FindWindowA" sptr, sptr
#cfunc FindWindowEx "FindWindowExA" int, int, sptr, sptr, 

#define WM_COMMAND 0x0111
#define THUMBNAIL_XP 0x702d //XP
#define THUMBNAIL_W2K 0x7031 //W2K

screen 0,200,100,1
title "縮小版にしちゃう"

a = sysinfo(0)
;mes a
b = instr(a,0,"NT")
c = instr(a,0,"5.0")
d = instr(a,0,"5.1")

if(b > 0){
	mes "NT系"
}else{
	mes "NT系以外"
	mes "利用できません。"
stop
}

if( c > 0){
	mes "Win 2K"
	THUMBNAIL = THUMBNAIL_W2K
}else{
if( d > 0){
	mes "Win Xp"
	THUMBNAIL = THUMBNAIL_XP
}else{
	mes "Vista?"
	//mes "利用できません。"
	//stop 別にとめない
	THUMBNAIL = THUMBNAIL_XP
	}
}

while(1)
hwnd1 = FindWindow("#32770", "イメージソースの選択")

if(hwnd1 == 0){
	hwnd1 = FindWindow("#32770", "別名で保存")
}

if(hwnd1 == 0){
	hwnd1 = FindWindow("#32770", "名前を付けて保存")
}

if(hwnd1 == 0){
	hwnd1 = FindWindow("#32770", "画像選択")	//これを追加した
}

hwnd2 = FindWindowEx(hwnd1, 0, "SHELLDLL_DefView",0 )

if(hwnd2){
	sendmsg hwnd2,WM_COMMAND,THUMBNAIL,0
	while(FindWindowEx(hwnd1, 0, "SHELLDLL_DefView",0 ))
		wait(50)
	wend
}
wait(50)
wend

```
