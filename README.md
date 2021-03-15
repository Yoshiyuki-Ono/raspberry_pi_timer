---
typora-copy-images-to: ./インターネット共有設定.png
---

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

##　Raspberry Pi Zeroのセットアップ

### Raspberry Pi OSの書き込み

以下の操作はMacで行っている。

使い古しのSDカード（IOデータ製 16GB）を使用したので、SD Card Formatterを使用してフォーマットを行う。

Raspberry Pi財団の公式サイトからOSのイメージファイルをダウンロードする。

https://www.raspberrypi.org/software/operating-systems/#raspberry-pi-os-32-bit

ダウンロードしたのは、Raspberry Pi OS Lite。2021年3月13日時点でのKernelバージョンは5.4。

ダウンロードされたファイルは2021-01-11-raspios-buster-armhf-lite.zip。

このファイルをBalenaEtcherを使ってSDカードに書き込む。BalenaEtcherであれば解凍せずにZipのままで書き込みが可能。

セットアップはUSB OTG (On-The-Go) EthernetでホストPCと接続して行う。

（Qiitaの記事、https://qiita.com/Liesegang/items/dcdc669f80d1bf721c21を参考にした）

SSHを有効にするため、SSDのルートディレクトリにSSHという名前の空のファイルを作製する。

touchコマンドでSSHを作成する。

```
$ touch /Volumes/boot/ssh
```

次に、`config.txt`の一番最後に下記の行を追加する。

```
dtoverlay=dwc2
```

さらに、`cmdline.txt`の`rootwait`の後に`modules-load=dwc2,g_ether`を追記する。

```
console=serial0,115200 console=tty1 root=PARTUUID=e8af6eb2-02 rootfstype=ext4 elevator=deadline fsck.repair=yes rootwait modules-load=dwc2,g_ether quiet init=/usr/lib/raspi-config/init_resize.sh
```

## Raspberry Pi Zeroの起動と設定変更

Raspberry Pi ZeroをUSB（2つある左側、基盤上にUSBと印刷してある方）でMacと接続する。Bootが終わるのを待って（初回は少し時間がかかる。LEDが点灯状態になるまで待つ）、ターミナルからSSHでログインすると、Raspberry Py Zeroのコンソールに入ることができる。

まず、`timedatectl`コマンドで、クロックの設定状態を調べる。

```
pi@raspberrypi:~ $ timedatectl
               Local time: Mon 2021-01-11 13:30:17 GMT
           Universal time: Mon 2021-01-11 13:30:17 UTC
                 RTC time: n/a
                Time zone: Europe/London (GMT, +0000)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no
```

RTCが有効になっていないこと、Time zoneがGMTになっていること、時刻が2021年1月11日 XX ;XX：XXに設定されていることがわかる。この日付はこのバージョン（buster）のディスクイメージファイルが作成された時のものである。

またこの状態はRaspberry Pi Zeroがインターネットに接続されていないスタンドアローンの状態であることも重要なポイントになる。NPT serverとの時刻同期はできていない。

Raspberry Pi ZeroをMacを介してインターネットに接続するためには、Mac側の共有設定を変更する必要がある。

![](/Users/onoyoshiyuki/Documents/GitHub/raspberry_pi_timer/インターネット共有設定.png)

システム環境設定＞＞共有を開き、インターネット共有でRNDIS/Ethernet Gadgetのチェックボックスにチェックを入れる。

一旦、ターミナルを閉じて、SSHでRaspberry Pi Zeroのターミナルに再びログインして、`timedatectl`コマンドを実行する。

```
pi@raspberrypi:~ $ timedatectl
               Local time: Mon 2021-03-15 12:40:52 GMT
           Universal time: Mon 2021-03-15 12:40:52 UTC
                 RTC time: n/a
                Time zone: Europe/London (GMT, +0000)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

Local time、Universal time どちらも現在時刻（GMTで）になっていることがわかる。NTPサーバーとの通信が可能になり、時刻同期が行われたことが確認できる。

RTCを使う上では、これらの挙動に注意すべきである。

一つは、デフォルトの状態で、インターネットに繋がれば（いずれかの）NTPサーバーに接続して時刻同期をするということである。

もう一つは、NTPサーバーとの通信ができない時には、直近のログイン状態時のタイムスタンプを起点に現在時刻を設定するということである。これはRTCを持たないRaspberry Pi固有の仕様であり、fake-hwclockという機能である。

インターネットに接続できたところで、`update`と`upgrade`コマンドを実施する。

`raspi-config`を使って、ロケールとタイムゾーンを変更する。

```
pi@raspberrypi:~ $ sudo raspi-config nonint do_change_locale ja_JP.UTF-8
Generating locales (this might take a while)...
  ja_JP.UTF-8... done
Generation complete.
pi@raspberrypi:~ $ sudo raspi-config nonint do_change_timezone Asia/Tokyo
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
	LANGUAGE = (unset),
	LC_ALL = (unset),
	LANG = "en_GB.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_MESSAGES to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory

Current default time zone: 'Asia/Tokyo'
Local time is now:      Mon Mar 15 22:38:18 JST 2021.
Universal Time is now:  Mon Mar 15 13:38:18 UTC 2021.
```

これでRTCを使っていない普通のRaspberry Pi Zeroのセットアップが完了した。念の為、`timedatectl`コマンドで時刻の設定がどうなったかを確認する。

```
pi@raspberrypi:~ $ timedatectl
               Local time: Mon 2021-03-15 22:38:53 JST
           Universal time: Mon 2021-03-15 13:38:53 UTC
                 RTC time: n/a
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

