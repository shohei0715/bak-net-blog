---
title: "EC-CUBEにVIP会員だけ購入可能な機能を付加したい その2"
date: 2013-07-26 03:09:05
draft: false
slug: "ec-cubeにvip会員だけ購入可能な機能を付加したい-その2"
---

テンプレートに手を入れるだけでいけるんじゃね?

と勘違いしたおかげで遠回りしてしまった。

 

Smartyに渡る変数で、追加に必要なのは

限定商品フラグ

VIP会員フラグ

の 二種が必要なのでやっぱdetail.phpをいじらなければいけない。

結局テンプレートもこのフラグで分岐させな駄目ですけどね!

 

以下メモ

```
$search = 99; //商品ステータス
$key = in_array($search, $objPage->productStatus[2]);
$objPage->tpl_products_vip = true;
if ($key){
// 商品ステータスがみつかったのでVIP判定をする
$objCustomer = new SC_Customer();
//顧客ステータスから会員情報を判定して会員ならtrue
$customoer_status = $objCustomer->getValue('status');
if ($customer_status == 9){
$objPage->tpl_customer_vip = true;
}else{
//
}
```
