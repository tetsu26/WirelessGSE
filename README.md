# GSE Ver.2 の仕様書   
東北大学FROM THE EARTH用のGSE制御システムVer.2の仕様書です．    
質問等は tetsushi_m[at]outlook.jp まで．

## 使いたい部品  
* マイコン    MBED LPC1768  
まぁいつもの．ピンがいっぱいあるから．
* 3端子dcdc   BP5293-50   http://akizukidenshi.com/catalog/g/gM-11188/  
3端子レギュレーターは効率悪いし，村田のdcdcコンバータは半田付けが難しすぎるので．  
ざっと計算した感じ，1Aもあれば，IM920含めて余裕で足りる．
* リレー    946H-1C-5D  http://akizukidenshi.com/catalog/g/gP-07342/  
mbedのdigital out は40mAまでしか出せないので，リレーは動かせなさそうだった．  
5Vのリレーをトランジスタでスイッチングするのがいいと思った．   
* 無線モジュール    IM920  
協賛でもらったやつ．正直無線GSEには向いてない．
* その他諸々

## プログラムコード
\Sourses にあります．mbedオンラインコンパイラにコピペして使ってください．
ロケット側と点火点側でソースコードが分かれてるんで気をつけてください．
