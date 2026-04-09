---
title: "EC-CUBEにVIP会員だけ購入可能な機能を付加したい その1"
date: 2013-07-19 01:02:12
draft: false
slug: "ec-cubeにvip会員だけ購入可能な機能を付加したい-その1"
---

やったこと

- 商品ステータスを適当に設定する
- 会員状態を適当に設定する
- 商品ステータスを元に商品詳細ページでは限定商品かどうか判定する
- VIP判定はまたこんど

今日はここまで。

ちなみに/html/products/detail.php が

[php]
init();
//商品IDを元に限定フラグの確認
SC_Utils::sfPrintR( $objPage->productStatus );
$search = 99; //商品ステータス
$key = in_array($search, $objPage->productStatus[2]);
if ($key){
 // 商品ステータスがみつかったのでVIP判定をする
 $objCustomer = new SC_Customer();
 if($objCustomer->isLoginSuccess()) {
 $this->tpl_login = true;
 }
}else{
 //
}
$objPage->process();

[/php]

/data/class_extends/page_extends/products/
[php]
class LC_Page_Products_Detail_Ex extends LC_Page_Products_Detail {
// }}}
 // {{{ functions
 /** 各商品のステータス */
 var $productStatus;
 /** 商品ステータス設定 */
 var $arrSTATUS;
 /** 商品ステータス画像 */
 var $arrSTATUS_IMAGE;
/**
 * Page を初期化する.
 *
 * @return void
 */
 function init() {
 parent::init();
 $masterData = new SC_DB_MasterData_Ex();
 $this->arrSTATUS = $masterData->getMasterData("mtb_status");
 $this->arrSTATUS_IMAGE = $masterData->getMasterData("mtb_status_image");
 $this->productStatus = $this->lfGetProductStatus();
 }
/**
 * Page のプロセス.
 *
 * @return void
 */
 function process() {
 parent::process();
 }
 function lfGetProductStatus(){
 $objQuery =& SC_Query_Ex::getSingletonInstance();
 $objProduct = new SC_Product_Ex();
 // おすすめ商品取得
 $col = 'product_id';
 $table = 'dtb_best_products';
 $where = 'del_flg = 0';
 $objQuery->setOrder('rank');
 $objQuery->setLimit(RECOMMEND_NUM);
 $arrBestProducts = $objQuery->select($col, $table, $where);
 $objQuery =& SC_Query_Ex::getSingletonInstance();
 if (count($arrBestProducts) > 0) {
 // 商品一覧を取得
 // where条件生成&セット
 $arrProductId = array();
 $where = 'product_id IN (';
 foreach ($arrBestProducts as $key => $val) {
 $arrProductId[] = $val['product_id'];
 }
 // 商品ステータスを設定
 $objProduct->setProductsClassByProductIds($arrProductId);
 $productStatus = $objProduct->getProductStatus($arrProductId);
 }
 return $productStatus;
 }
/**
 * デストラクタ.
 *
 * @return void
 */
 function destroy() {
 parent::destroy();
 }
}
[/php]
