# raspberry_pi_timer
## 概要・目的
Raspberry Pi Zeroを使ったOn／Offタイマーの作製についてまとめたメモ。

CNC菜園ロボット（FarmBot)を太陽電池で駆動する電源システムを定刻でOn/Offするのが目的。

ロボットシステム全体の構成は下記の通り。

![](/Users/onoyoshiyuki/Documents/GitHub/raspberry_pi_timer/全体システム構成.png)

上記の構成図のOn/OffタイマーをRaspberry Pi Zeroを使って作製する。

菜園ロボットが稼働するのは昼間のみなので、夜間は電源供給を止めるためのもの。

当初はインバータ（DENRYO製 SK200）のAC100V出力のコンセントに市販のダイヤル式タイマーを取り付けて行う予定だった（下図参照）。

![IMG_0357](/Users/onoyoshiyuki/Documents/GitHub/raspberry_pi_timer/IMG_0357.png)

しかし試してみるとインバータの消費電力が思いのほか大きくバッテリーの消耗が激しいことが判明した。そこで、インバータの前段にOn/Offタイマーを設け、インバータごと通電を遮断するようにする。

インバータにはリモートコントロールのための接点入力端子があるので、それを利用して制御する。

![IMG_0358](/Users/onoyoshiyuki/Documents/GitHub/raspberry_pi_timer/IMG_0358.png)

##  使用するパーツ

- Raspberry Pi Zero W

- SSRモジュール（KKHMF 1チャンネル 分離駆動リレー制御モジュール）

  駆動電圧：DC　3V/3.3V

  制御可：10A 250VAC、10A 30VDC

- RTCモジュール（seeed PiRTC DS1307）

　　クロックチップ：DS1307
　　電源：3V CR1255リチウム電池

![IMG_0353](/Users/onoyoshiyuki/Documents/GitHub/raspberry_pi_timer/IMG_0353.png)

### 配線

- RTC

インターフェースはI2C。HATになっているので1番ピン側に合わせて差し込むだけである。

- RasPi Zero === SSRモジュール

RasPi Zero 17番ピン（3.3V)　====　SSR VCC

RasPi Zero 39番ピン（GND)　====　SSR GND

RasPi Zero 37番ピン（GPIO26)　====　SSR IN

（今回は信号出力にGPIO26を使用した）

- SSRモジュール === インバータ

SSR COM　===　インバータ GND

SSR NO　===　インバータ ~~ENB~~(ENBに上線がある端子)

（NO : Nomaly Open  常時開で動作時に閉）

インバータをリモートモードで運転する場合、GNDと~~ENB~~が短絡すると運転開始となり、オープンだと運転停止になる。GPIO 26がHighになるとSSRが閉になり運転開始になる。GPIO26をLoにすると運転停止になる。RasPi Zeroの電源が落ちている時、SSRへの電源が供給されていない時もNOなのでインバータは運転停止になる。

インバータをリモート運転させるときはインバータの裏面パネルのPOWERスイッチをREMO.側にセットする。

![20210311_0001](/Users/onoyoshiyuki/Documents/GitHub/raspberry_pi_timer/20210311_0001.png)

