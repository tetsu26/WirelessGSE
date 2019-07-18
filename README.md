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
\SourceCode にあります．mbedオンラインコンパイラにコピペして使ってください．
ロケット側と点火点側でソースコードが分かれてるんで気をつけてください．

### コード解説_点火点側
ソースコードはc++(ほぼC)で書いてあります．一応頭から説明しておきます．

```cpp
#include<mbed.h>
#include<IM920.h>
#include<cstring>
```
ライブラリのインポートです（それはそう）．

```cpp
#define IM920_TX p13
#define IM920_RX p14
#define IM920_BUSY p11
#define IM920_RESET p12
#define IM920_SERIAL_BANDRATE 19200//im920の通信レート
```
IM920関係のピン設定ですね．

```cpp
DigitalOut myled(LED1);
DigitalOut myled2(LED2);
DigitalOut myled3(LED3);
DigitalOut myled4(LED4);
DigitalOut led_fill(p16);
DigitalOut led_dump(p18);
DigitalOut led_oxy(p20);
DigitalOut led_fire(p21);
DigitalOut led_1(p24);
DigitalOut led_2(p23);
PwmOut buzzer(p26);

DigitalIn fill(p15);
DigitalIn dump(p17);
DigitalIn oxy(p19);
DigitalIn fire(p22);

IM920 im920(p13, p14, p11, p12,19200);//無線通信用のserialクラス
Serial pc(USBTX, USBRX);
```
そのたもろもろのピン設定です．基板自作したときは適宜変えてください．

```cpp
// グローバルな奴ら
char send_data[8];    //送信するデータを格納する変数
char recv_data[8];    //受信すry)
int cb_count = 0;     //割り込み回数を数えるやつ
int i = 0;            //一秒間隔で赤いLEDを点滅させるのに使う
```
グローバル関数です．

```cpp
int fire_buzzer(int i)
{
    if(i == 1)
    {
        buzzer.period(1.0/440.0);  //多分"ラ"
        buzzer.write(0.5f);
        wait(0.5f);
        buzzer.write(0.0f);
    }
    
    return 0;
}
```
これは fire をオンにしたときにブザーから音を出すための関数です．音程が気に入らない人は変えてください．

#### callback() 関数

```cpp
void callback () {  //受信時の割り込み関数（受信したときに送信するコード）
    int i;

    cb_count++; //割り込みの回数を数える変数
    
    i = im920.recv(recv_data, 8);
    recv_data[i] = 0;
    pc.printf("recv: '%s' (%d)\r\n", recv_data, i);
    
    if (recv_data[0] == '5' || recv_data[7] == '5')     //きちんとデータを受け取れたか確認
    {
        led_fill = recv_data[1] - '0';           
        led_dump = recv_data[2] - '0';
        led_oxy = recv_data[3] - '0';
        led_fire = recv_data[4] - '0';
        fire_buzzer(led_fire);
    }
    //8文字1セットで送ることにする
    send_data[0] = '5';   //データ始まりの合図
    send_data[1] = '0'+(fill.read()^ 1);      //プルアップしてるからHiとLoを反転
    send_data[2] = '0'+(dump.read()^ 1);
    send_data[3] = '0'+(oxy.read()^ 1);
    send_data[4] = '0'+(fire.read()^ 1);
    // 5~6はとりあえず未使用
    send_data[5] = '0';
    send_data[6] = '0';
    send_data[7] = '5';   //データ終わりの合図
        
    im920.sendData(send_data,8);    //8文字送りますよーってやつ
    //pc.printf("send:%s\r\n cb_count:%d\r\n",send_data,cb_count);
        
    
}
```
これはim920が受信したときに実行される予定の関数です．グローバル関数にiがあるのにローカルでもiを宣言して使うくそコードになってますね，今気づきました．ここからわかりにくいと思うので詳しく書いておきます．
```cpp
i = im920.recv(recv_data, 8);
recv_data[i] = 0;
pc.printf("recv: '%s' (%d)\r\n", recv_data, i);
```
この辺はおまじないです．書いておいてください．
```cpp
if (recv_data[0] == '5' || recv_data[7] == '5')     //きちんとデータを受け取れたか確認
    {
        led_fill = recv_data[1] - '0';           
        led_dump = recv_data[2] - '0';
        led_oxy = recv_data[3] - '0';
        led_fire = recv_data[4] - '0';
        fire_buzzer(led_fire);
    }
```
これは効果あるかは謎ですが，受信したデータの整合性を確認しているつもりです．受信データ（8桁）の最初と最後の数字は`5`なのでそれだけ確認しています．

その後の`led_fill = recv_data[1] - '0';`ですが，これは受信した8桁の2文字目がfillの状態を示しているのでそれをスイッチについたLEDに反映しています．受信したデータは`recv_data`に格納されているのですが，ASCIIコードなので`0`のASCIIコードを引くことで元の送信した数字に戻しています．ほかも同じです．
```cpp
 //8文字1セットで送ることにする
    send_data[0] = '5';   //データ始まりの合図
    send_data[1] = '0'+(fill.read()^ 1);      //プルアップしてるからHiとLoを反転
    send_data[2] = '0'+(dump.read()^ 1);
    send_data[3] = '0'+(oxy.read()^ 1);
    send_data[4] = '0'+(fire.read()^ 1);
    // 5~6はとりあえず未使用
    send_data[5] = '0';
    send_data[6] = '0';
    send_data[7] = '5';   //データ終わりの合図
```
この辺はスイッチの状態を読み取って送信するデータを作っています．`send_data`配列の５と６は将来のために未使用です．送る文字数を８文字にしたのはなんとなくきりがいいからです．
```cpp
im920.sendData(send_data,8);    //8文字送りますよーってやつ
```
そしてここで上で作った文字列を送信しています．

#### callback2() 関数
```cpp
void callback2()    //受信割り込みの回数を数える関数
{
    
    if(cb_count == 0)
    {
        pc.printf("yabame\r\n");
        NVIC_SystemReset();         //一秒間に一回も通信しなかったらリセットする
        
    }
    else
    {
        cb_count = 0;
        if(i == 0)
        {
            led_2 = 1;
            i++;
        }else{
            led_2 = 0;
            i = 0;
        }
            
        
    }
}
```
これは接続が切れたときの対処です．コードをざっくり見てもらうと気づくと思うのですが，このコードには「受信したら割り込み処理で送信する」機能しかないので，一度接続が切れる（というかデータのやりとりが途絶える）と復帰できなくなる問題があります．なので`callback2()`関数を1秒ごとに起動してもしその前1秒間に一度も通信していなかったらこのmbedをリセットして`main()`関数をもう一度頭から実行します．`NVIC_SystemReset()`ってのはmbedのリセットボタンを押した状態をコードから作れる魔法の言葉です．

#### main()関数
```cpp
int main()
{
    //各スイッチをプルアップする
    fill.mode(PullUp);
    dump.mode(PullUp);
    oxy.mode(PullUp);
    fire.mode(PullUp);          

    
    im920.init();
    //Im920受信の割り込みを設定
    im920.attach(callback);
    pc.baud(115200);
    pc.printf("Wireless GSE controll system started!!\r\n");
    
    Ticker timeout;
    timeout.attach(&callback2,1);
    
    myled = 1;  //設定完了の合図
    led_1 = 1; 
    im920.poll();
    im920.sendData(send_data,8);    
    
    while(1){   
        im920.poll();//双方向送受信に必要らしい
       // wait_ms(1);  //意味なさそうだけどあると安定する
        
    }
    
}
```
ここは特に説明することもない気がします．セットアップの時に1度データを送信します．