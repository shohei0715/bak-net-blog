---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
draft: true
slug: "{{ .File.ContentBaseName }}"
---

ここに本文を書く。

## 見出し

テキストテキスト。

**太字** や *斜体* や `コード` が使えます。

### 画像を入れる場合

記事フォルダに画像を置いて参照：

![画像の説明](images/photo.jpg)

### リンク

[リンクテキスト](https://example.com)

### コードブロック

```python
print("hello")
```

### リスト

- 項目1
- 項目2
- 項目3
