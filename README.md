---
typora-copy-images-to: ../../../Desktop/IMG_0376.jpg
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

しかし試してみるとインバータの消費電力が思いのほか大きくバッテリーの消耗が激しいことが判明した。そこで、インバータの前段にRaspberry Pi Zeroで作ったOn/Offタイマーを設け、インバータごと通電を遮断するようにする。Raspberry Pi ZeroはMPPTについているUSBから給電する。Raspberry Pi Zeroへの通電は常時行われるが、晴れている昼間はソーラーパネルから充電が行われており、バッテリー切れが起きる心配はない。

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

## RTCの取り付けと設定

Raspberry Pi用のRTCモジュールはいくつかのベンダーから入手が可能である。今回はSeeedのPiRTCを使用した。

![IMG_0376](/Users/onoyoshiyuki/Desktop/IMG_0376.jpg)

クロックチップにDS1307を使っており、1日で2秒程度の誤差がでるが低価格である。Adafruitからも同じくDS1307を搭載したPiRTCがでている。どちらもHATになっており半田付けなどは不要でGPIOピンに差し込むだけで使うことができるので便利である。Raspberry Piとの通信にI2Cを使用するが、AdafruitのものはI2Cのピンが塞がってしまうのに対し、SeeedのものはI2Cのピンの外出しができるように工夫されており、そのための2pinヘッダーも添付されているので、こちらの方が良いと思う。使い方は、SeeedのWikiを参照のこと。

https://wiki.seeedstudio.com/High_Accuracy_Pi_RTC-DS3231/

注意すべき点は、ドキュメントにはLiコイン電池 CR1225を使用することとなっているが、この型式の電池は日本では手に入りにくいことである。同じ仕様のBR1225でも良いが、こちらも量販店ではあまり売られていない。CR1225は、電圧3V、直径12.5mm、厚さ2.5mmだが、厚さが2.0mmのCR1220でもソケットに入るので使うことができる。今回はPanasonic製 CR1220を使用した。

電池をソケットにセットしたら、Raspberry Pi ZeroのGPIOピンヘッダーの１番ピン側に差し込む。

![IMG_0377](/Users/onoyoshiyuki/Desktop/IMG_0377.jpg)

Wikiの記載に従って、ドライバーをインストールする。`git clone`を使うので、まずGitをインストールする。

```
pi@raspberrypi:~ $ sudo apt-get install git
```

インストールされたバージョンは、2.20.1。

```
pi@raspberrypi:~ $ git --version
git version 2.20.1
```

SeeedのGit Hubからpi-HATのリポジトリをクローニングする。

```
cd; git clone https://github.com/Seeed-Studio/pi-hats.git
```

pi-hatsの下にはRTC用だけでなくSeeedの他のHATモジュールのファイルも一緒にコピーされている。

```
pi@raspberrypi:~ $ cd pi-hats
pi@raspberrypi:~/pi-hats $ ls
ADC-HAT  CAN-HAT  GROVE-AI-HAT  LICENSE  README.md  RTC-HAT  images  tools
```

RTC _D S1307のドライバーをインストールする。

SeeedのWikiには、カレントディレクトリがpi-hatsでコマンドを実行するように書かれているが、間違い。一つ下のtoolsに変更して実行する必要がある（Git-Hubのreadmeにはtoolsで実行するように記述されている）。

```
pi@raspberrypi:~/pi-hats $ cd tools
pi@raspberrypi:~/pi-hats/tools $ sudo ./install.sh -u rtc_ds1307
Uninstall rtc_ds1307 ...
Uninstall rtc_ds3231 ...
Uninstall adc_ads1115 ...
Enable I2C interface ...
Install rtc_ds1307 ...
パッケージリストを読み込んでいます... 完了
依存関係ツリーを作成しています                
状態情報を読み取っています... 完了
以下のパッケージは「削除」されます:
  fake-hwclock
アップグレード: 0 個、新規インストール: 0 個、削除: 1 個、保留: 0 個。
この操作後に 32.8 kB のディスク容量が解放されます。
(データベースを読み込んでいます ... 現在 41077 個のファイルとディレクトリがインストールされています。)
fake-hwclock (0.11+rpt1) を削除しています ...
man-db (2.8.5-2) のトリガを処理しています ...
Synchronizing state of fake-hwclock.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable fake-hwclock
Unit /etc/systemd/system/fake-hwclock.service is masked, ignoring.
#######################################################
Reboot the system to take a effect of Install/Uninstall
#######################################################
```

このメッセージを読むと、I2Cを有効にしていること、fake-hwclockのモジュールは削除され無効化されていることがわかる。またRTCを有効にするためにはリブートが必要であることもわかる。今時点で、`timedatectl`コマンドを実行してみる。

```
pi@raspberrypi:~/pi-hats/tools $ timedatectl
               Local time: 水 2021-03-17 17:02:55 JST
           Universal time: 水 2021-03-17 08:02:55 UTC
                 RTC time: n/a
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

確かにまだこの時点では、RTCは機能していない。

次にMacのインターネット接続共有の機能を切って、スタンドアローンでRaspberry Pi Zeroを立ち上げて、`timedatectl`コマンドを実行し時刻がどのように設定されるのかを確認してみる。

```
pi@raspberrypi:~/pi-hats/tools $ timedatectl
Failed to query server: Failed to read RTC: Invalid argument
```

RTCの読み込みに失敗し時刻の情報が表示できなかった。`hwclock -r`を実行し、RTCの時刻データの読み出しを試みても同じようにエラーとなっている。これはRTCに時刻が設定されていない（おそらくNullになっている）ためタイプエラーで読み出しができないためであろう。

```
pi@raspberrypi:~/pi-hats/tools $ sudo hwclock -r
hwclock: ioctl(RTC_RD_TIME) to /dev/rtc0 to read the time failed: 無効な引数です
```

次にMacのインターネット接続共有をオンにしてRaspberry Pi Zeroを再度立ち上げて、同じことをやってみる。

```
pi@raspberrypi:~ $ timedatectl
               Local time: 水 2021-03-17 17:44:08 JST
           Universal time: 水 2021-03-17 08:44:08 UTC
                 RTC time: 水 2021-03-17 08:44:08
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
pi@raspberrypi:~ $ sudo hwclock -r
2021-03-17 17:45:10.245437+09:00
```

今度は、RTCから時刻が読み取れており、その時刻はUTCと同じになっている。

起動時にインターネットに繋がっていれば、UTC、Local timeとともにRTCもNTPサーバーと同期される仕様になっていることがわかる。

この仕様をさらに理解するため、RTCの時刻を手動で設定して挙動を見てみる。

```
pi@raspberrypi:~ $ sudo hwclock --set --date "17 March 2021 11:00"
pi@raspberrypi:~ $ timedatectl
               Local time: 水 2021-03-17 21:26:20 JST
           Universal time: 水 2021-03-17 12:26:20 UTC
                 RTC time: 水 2021-03-17 02:00:15
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no
```

`hwclock --set`コマンドでRTCを2021年3/17 11:00（JSTで20:00に設定するつもりでUTCで11:00と指定した）に設定して、`date`コマンドでシステムクロックを、`timedatectl`コマンドでクロックの設定状態を確認してみる。システムクロック（Local timeとUniversal time）は起動時にNTPサーバーと同期した時刻が維持されているが、RTCは3/17 11:00ではなく、3/17 2:00となってしまった。`hwclock --set`コマンドの引数はどうやらLocal time（JST）だと評価されて、その9時間前の2時に設定されたのだと思われる。

この状態でシャットダウンしインターネットに繋がないローカルで再起動してみる。システムクロックの設定はRTCに合わせて変わっているが、意図とは全く違う時刻設定になってしまっている。

```
pi@raspberrypi:~ $ date
2021年  3月 17日 水曜日 11:02:33 JST
pi@raspberrypi:~ $ timedatectl
               Local time: 水 2021-03-17 11:02:42 JST
           Universal time: 水 2021-03-17 02:02:42 UTC
                 RTC time: 水 2021-03-17 02:02:42
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: no
              NTP service: active
          RTC in local TZ: no
```

この後、インターネットに接続した状態で再起動してみる。

```
pi@raspberrypi:~ $ date
2021年  3月 17日 水曜日 11:15:33 JST
pi@raspberrypi:~ $ timedatectl
               Local time: 水 2021-03-17 21:41:56 JST
           Universal time: 水 2021-03-17 12:41:56 UTC
                 RTC time: 水 2021-03-17 12:41:57
                Time zone: Asia/Tokyo (JST, +0900)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
pi@raspberrypi:~ $ date
2021年  3月 17日 水曜日 21:42:12 JST
```

RTCはNTPサーバーと同期されて手動で設定した値は消えてしまう。RTCを手動で設定する場合はこのあたりの仕様に注意する必要がある。

解せないのは起動後、最初の`date`コマンドの実行ではNTPと同期が取れていない以前の時刻が表示されたことである。2回目以降はNTPと同期が取れた後の時刻になった。

以上をまとめると、挙動は以下の通りとなる。インターネットに繋がらないスタンドアローンの状態で起動するとRTCの時刻を基準にシステムクロックの時刻が設定され、インターネットに繋がる状態で起動するとNTPサーバーとRTC、システムクロックは同期される。

手動でRTCを設定し任意の時刻で運用するためには、NTPとの同期を無効にする必要がある。

Raspberry PiにRTCをつけるのは、インターネットに繋がないスタンドアローン環境下で時刻を利用したいという時である。しかしRasberry Pi Zeroをヘッドレスで構築しているときは多くのケースでインターネットに繋がれた環境である。Raspberry Piの起動時のfake-hwclockとNTPサーバーとの同期の仕組みは、スタンドアローンの状態とインターネットに接続された状態ではシステムクロックの挙動に違いを生じされる。筆者も構築時には問題なく動いていたものが、いざ屋外で運用を始めると想定外の動きになるということに悩まされた。そのため、RTCをつけた時の時刻同期の挙動については詳しく記載した。

なおSeeedのWikiには、PiRTCのドライバは、Raspbian Jessie/Stretchのみを対象とすると書かれているが、Busterでも問題なく動作することを確認した。

## Systemdを利用した定刻でのプログラム実行

RTCの設定ができたので、定刻でSSRを制御する仕組みを作る。すでに配線の章で説明した通り、GPIO26をHighにするとSSRは閉になりインバータが動き出す。GPIO 26をHigh、LowにするPythonのプログラムをsystemdで定刻で実行することでインバータを定刻で起動、停止する。

まず、RPi.GPIOライブラリをインストールする（その事前準備としてpipもインストールする）。systemdで起動するので実行ユーザーはrootになるため、sudoでインストールする必要がある（sudoなしの `pip3 install RPi.GPIO `だとModuleNotFoundError: No module named 'RPi'が発生しうまく動かない）。

```
pi@raspberrypi:~ $ sudo apt-get -y install python3-dev

pi@raspberrypi:~ $ sudo apt-get -y install python3-pip

pi@raspberrypi:~ $ sudo pip3 install RPi.GPIO
```

PythonのプログラムはGPIO26を制御するだけの単純なものでる。

```
# set_on.py
import RPi.GPIO as GPIO
GPIO.setmode(GPIO.BCM)
GPIO.setup(26,GPIO.OUT)
GPIO.output(26,1)

# set_off.py
import RPi.GPIO as GPIO
GPIO.setmode(GPIO.BCM)
GPIO.setup(26,GPIO.OUT)
GPIO.output(26,0)
GPIO.cleanup()

```

このプログラムを定刻で実行するためにsystemdの設定をする。

Systemdの設定に関しては、molchiro氏のqiitaの記事を参考にさせていただいた。

https://qiita.com/molchiro/items/ee32a11b81fa1dc2fd8d

### serviceファイルの作成

/etc/systemd/systemの下にseviseファイルを作成する。そのserviceファイルの中で前述のPythonプログラムを呼ぶシェルを記述する。

```
# /etc/systemd/system/set_on.service

[Unit]
Description=do something

[Service]
ExecStart=/usr/bin/python3 /home/pi/set_on.py
```

```
# /etc/systemd/system/set_off.service

[Unit]
Description=do something

[Service]
ExecStart=/usr/bin/python3 /home/pi/set_off.py
```

### timerファイルの作成

前述のserviceを定刻で呼び出すためのtimerファイルを/etc/systemd/systemの下に作成する。呼び出すserviceは明示的に指定しない場合は、同名のserviceが呼び出される。

```
# /etc/systemd/system/set_on.timer

[Unit]
Description=daily do something

[Timer]
OnCalendar=*-*-* 8:00

[Install]
WantedBy=timers.target
```

```
# /etc/systemd/system/set_off.timer

[Unit]
Description=daily do something

[Timer]
OnCalendar=*-*-* 9:00

[Install]
WantedBy=timers.target
```

### Timerの有効化

作成したtimerファイルをsystemdに登録して有効化する。

```
pi@raspberrypi:~ $ sudo systemctl enable set_on.timer
pi@raspberrypi:~ $ sudo systemctl enable set_off.timer
```

timerの登録がうまくできているかは、`systemctl status `コマンドで確認できる。

```
pi@raspberrypi:~ $ systemctl status set_on.timer
● set_on.timer - daily do something
   Loaded: loaded (/etc/systemd/system/set_on.timer; enabled; vendor preset: enabled)
   Active: active (waiting) since Fri 2021-03-19 01:52:49 JST; 7h ago
  Trigger: Sat 2021-03-20 08:00:00 JST; 22h left
```

timerを無効化するには、`systemctl disable`コマンドを実行する。`systemctl status `コマンドを実行すると無効になっていることが確認できる。

```
pi@raspberrypi:~ $ sudo systemctl disable set_on.timer
Removed /etc/systemd/system/timers.target.wants/set_on.timer.
pi@raspberrypi:~ $ systemctl status set_on.timer
● set_on.timer - daily do something
   Loaded: loaded (/etc/systemd/system/set_on.timer; disabled; vendor preset: enabled)
   Active: active (waiting) since Fri 2021-03-19 01:52:49 JST; 7h ago
  Trigger: Sat 2021-03-20 08:00:00 JST; 22h left

 3月 19 01:52:49 raspberrypi systemd[1]: Started daily do something.
```

ここまでで設定は全て完了となる。

rebootをするとtimerが動き出すので、設定した時刻に正しく動く（この場合、8時にインバータが起動し、9時にシャットダウンされる）ことを確認する。

#### 参考（サービス登録の失敗）

最初、RPi.GPIOのインストールをsudoで実行しなかったため、うまくサービスが動かなかった。その時のサービス登録時の挙動を参考までに、残しておく。

```
pi@raspberrypi:~ $ sudo systemctl enable set_on.service
Created symlink /etc/systemd/system/multi-user.target.wants/set_on.service → /etc/systemd/system/set_on.service.
pi@raspberrypi:~ $ systemctl status set_on.service
● set_on.service - do something
   Loaded: loaded (/etc/systemd/system/set_on.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Fri 2021-03-19 00:18:23 JST; 21min ago
 Main PID: 817 (code=exited, status=1/FAILURE)

 3月 19 00:18:23 raspberrypi systemd[1]: Started do something.
 3月 19 00:18:23 raspberrypi python3[817]: Traceback (most recent call last):
 3月 19 00:18:23 raspberrypi python3[817]:   File "/home/pi/set_on.py", line 1, in <module>
 3月 19 00:18:23 raspberrypi python3[817]:     import RPi.GPIO as GPIO
 3月 19 00:18:23 raspberrypi python3[817]: ModuleNotFoundError: No module named 'RPi'
 3月 19 00:18:23 raspberrypi systemd[1]: set_on.service: Main process exited, code=exited, status=1/FAILURE
 3月 19 00:18:23 raspberrypi systemd[1]: set_on.service: Failed with result 'exit-code'.
```

timerの登録の時には正常に実行できているように見えるが、実際には定刻になっても何も起こらない。`systemctl status `コマンドを実行しないと登録に失敗していることがわからないので注意が必要である。

# まとめ

以上、Raspberry Pi Zeroを使った定刻でインバータをOn/Offするタイマーの作製方法についての備忘録。

現在、このタイマーのおかげで菜園ロボットは問題なく稼働している。

今回は、インバータが接点入力だったのでSSRは単純なOpen /Closeとして使ったが、電源を直列に接続すればさまざまな機器への電力供給の時刻による制御が可能である。今回使用したSSRはAC100VやDC24Vも制御できるので、幅広い機器に利用できる。Raspberry Pi Zeroにはプログラムで制御できる26のGPIOピンを持っているので、SSRを増やせば最大、26台の機器の制御が可能である。ただしSSRで大容量の電流の制御を行うと、信号側の消費電流も大きくなるので、その点は注意する必要があるであろう。