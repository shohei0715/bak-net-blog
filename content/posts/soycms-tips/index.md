---
title: "SoyCMS Tips"
date: 2014-07-25 15:54:20
draft: false
slug: "soycms-tips"
---

# TinyMCEの編集画面にCSSを適用する

ファイル
/soycms/js/editor/RichTextEditor.js

[php]
function onInitTinymceEditor(id){
        //スタイルの適用
	tinyMCE.get(id).dom.loadCSS(entry_css_path);
	tinyMCE.get(id).dom.loadCSS("/hogehoge.css"); //これ

[/php]
