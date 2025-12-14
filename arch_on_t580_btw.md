<img src="/images/hikari_256x256_round.jpg" alt="penguin icon image" width="100" height="100" />

### I use arch on T580, btw

|   |                   |
|---|-------------------|
|MEM|DDR4-2400 PC4-19200|
|CPU|Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz / coffee lake-s|
|GPU|Intel UHD Graphics 620 / Gen9.5|
|SSD|M.2 SSD / PCIe NVMe, PCIe 3.0 x 2, 16Gb/s|

***
### Thunderbolt firmware problem

T580を含む、この世代のThinkpadには**ファームウェアに関しての致命的な問題**が存在します。

[The "Thunderbolt Firmware Problem" Explained (reddit.com)](https://www.reddit.com/r/thinkpad/comments/1dfdp18/the_thunderbolt_firmware_problem_explained/)

> 2017年から2019年にかけて製造されたThinkPadの複数のモデルには、Thunderbolt 3コントローラーがしばらくすると自動的に停止するという厄介なバグがありました。状況は悪化し、Lenovoはこの問題を認め、ファームウェアアップデートをリリースして問題を解決しようとしました。このアップデートはほとんどのユーザーにとって有効で、何らかの理由でアップデートを実行できなかったユーザーは、デバイスを送付してマザーボードを交換することができました。しかし、これはもはや不可能なため、現状では自分で行うしかありません。

> ユニットを入手したらすぐに、 LenovoのウェブサイトでThunderbolt/NVMコントローラーのファームウェアバージョンを確認する方法を ご確認ください。中古市場に流通する前に、優秀なIT部門に管理されていたデバイスであれば、ほとんどのデバイスは既にアップデートされていますが、予防保守こそが最良のメンテナンスであるため、念のため確認することをお勧めします。古いバージョンを使用していて、まだ問題が発生していない場合は、Lenovoのウェブサイトの指示に従って、ファームウェアの書き換え方法をご確認ください。この作業を先延ばしにすると、すぐに対処しないと、後々問題が顕在化してしまう可能性があります。

現在（2025年）以降に入手する場合はほぼ間違いなく中古品でしょうから、引用文中にもあるように、以前のオーナーがアップデートを済ませていたかどうかが問題となります。わたしの個人的な印象ですが、ネット上で検索したときにこの問題についての日本語による情報がほとんど見つからないことを考えると、油断せずに必ずアップデート状況を確認しておくべきだと思います。

何やら大袈裟な感じがしますが、実際にやらなければいけないことは多くはありません。`LVFS（Linux Vendor Firmware Service）`という各社のファームウェア更新を一元的に配布するためのサービスと、それを利用するプログラムである`fwupd`との開発により、linux環境でのファームウェア更新は飛躍的に簡単になりました。

```bash
pacman -S fwupd                # additionally `pacman -S udisks2 efivar` for UEFI firmware update
fwupdmgr get-devices           # display all devices detected by fwupd
fwupdmgr refresh               # download latest metadata
fwupdmgr get-updates           # list available updates
fwupdmgr update                # install updates
```

……と本来なら以上のコマンドだけで十分で、この項目が書かれる必要性もなかったはずなのですが、実のところわたしの環境でこの方法ではアップデートが成功しませんでした。

1. `fwupdmgr get-devices`で`Thunderbolt host controller`を認識していることは確認できる。
   - 認識しない場合：[https://wiki.archlinux.org/title/Thunderbolt#Forcing_power](https://wiki.archlinux.org/title/Thunderbolt#Forcing_power)
2. `fwupdmgr get-updates`で、バグを完全に修正するバージョン20へのアップデートが表示されるべきところ
3. 現在のバージョンが14であるにも関わらずアップデートなしと出てしまう。

仕方ないので次に以下のように、LVFSサイトから必要なファイルを直にダウンロードして、手動でのインストールを試みました。しかし、これもうまく進みませんでした。`did not find magic`といったようなエラーが出ました。

```bash
curl https://r2.fwupd.org/lvfs-prod/86e4f830ccc892f1ca8963b1e8e34164e59af4ed-Lenovo-ThinkPad-T580-Thunderbolt-Firmware-N27TF18W_AssistMode.cab > foo.cab
fwupdmgr local-install foo.cab --verbose
```

fwupdか、archのカーネルか、わたしのハードウェアか、どこに問題があるのかわかりませんでしたが、redditを眺めているとlinuxmintでの成功例が数多く投稿されていたので、それに倣うことにしました（yes, I use mint, btw）。

1. isoをダウンロードしてインストールメディアを作成。
2. `fwupdmgr get-updates`でバージョン20へのアップデートありとの表示！
3. しかしいざ`fwupdmgr update`すると何故か通らない。万事休すかと思ったが……
4. `fwupdmgr install <DIVICE-ID>`で結局解決できた。
   - `DEVICE-ID`は`fwupdmgr get-devices`で確認できる`4214d8e87e1aa7de57c19e5954bbef2e462b86a4`のような文字列。
   - このコマンドで出てくるプロンプトに従って、まず`14->18`、その後`18->20`と段階的にやると、無事バージョン20まで到達することができました。

一応補足しておくと、実際にmintをハードドライブにまでインストールする必要はなく、インストールメディアから`fwupd`を使用してすべての操作を完了できます。

***
### Yellowish tint monitor

この機種を含んだいくつかのThinkpadのモニターの色味が黄色がかっている、という評判がネット上で見られます。わたしの経験から言わせてもらうと、これは本当です。特にT580以前に使用していたラップトップが青味の強いモニターだったので、最初のうちはその落差でかなり黄色く見えていました。ただし、使用しているうちに一ヶ月もしないで慣れてきて普通に見えるようになります。逆に言うと、普通もしくは青味のモニターと一緒に横に並べて使用する場合には、もしかするとずっと違和感が残り続けることになるかもしれません。単独で使用するだけならほとんど問題はないことをわたし［誰？］が保証します。

***
### GPU acceleration on chromium

Core-i5のモデルを使用していたとしても（そして試してはいませんがまず間違いなくCore-i3でも）、youtubeの1080p全画面再生程度では全然普通に余裕があります。とは言え、内蔵GPUを使って更にCPUに余裕を持たせられればそれはそれで嬉しいものです。発熱抑制にもなります。

[Hardware video acceleration - ArchWiki](https://wiki.archlinux.org/title/Hardware_video_acceleration)

> Intel graphics open-source drivers support VA-API:\
> HD Graphics series starting from Broadwell (2014) and newer (e.g. Intel Arc) are supported by intel-media-driver.

```
pacman -S intel-media-driver
echo "--enable-features=AcceleratedVideoDecodeLinuxGL" >> .config/chromium-flags.conf
```
これでchromiumが特定のファイルの動画再生にGPUを使うようになります。ただしyoutubeの動画で有効にするためにはもう一つ必要なものがあります。「特定のファイル」と書いたところがポイントで、現在youtubeがデフォルトで選択して送信してくるファイル形式はこれには当てはまりません。そのファイル形式をを常に`H.264`に指定するために[h264ify](https://chromewebstore.google.com/detail/h264ify/aleakchihdccplidncghkekgioiakgal)という拡張機能をダウンロードします。

正しく動作しているかの確認は`chrome://media-internals`や`watch -n 1 cat /sys/class/drm/card1/gt_cur_freq_mhz`などを見るか、あるいは`htop`を使ってCPUとGPUとを同時に見比べてもいいでしょう。`htop`には実はデフォルトでは隠れているGPUのモニターがあって`F2`の設定から選択して表示することができます。

***
### Android file transfer via usb-c

```
pacman -S gvfs-mtp
# lsusb to check Bus xxx Device yyy
gio mount mtp://[usb:xxx,yyy]/
# access via /run/user/$UID/gvfs/...
```

```
# libmtp package に含まれるツールのみでも可能？ 未検証
pacman -S libmtp

mtp-detect
mtp-connect
# search for the target file ID
mtp-folders
mtp-files

mtp-getfile <ファイルID> <PC上の保存ファイル名>
mtp-sendfile <PC上のファイル名> <デバイス上の保存先パス>
```

***
### Console font

黒いコンソール画面で、デフォルトのままだとフォントのサイズが小さすぎると感じるはずです。これへの対処は、たとえば`pacman -S terminus-font`として大きめのビットマップ・フォントを導入してもいいのですが、実は`/usr/share/kbd/consolefonts/`にデフォルトでインストールされているなかに、使用に耐えるサイズのフォントが（ほぼ唯一これだけ）入っています。
```
echo FONT=LatGrkCyr-12x22 >> /etc/vconsole.conf
echo KEYMAP=jp106 >> /etc/vconsole.conf
```

***
### Bluetooth 01 (unknown log message)

ディスプレイマネージャーを使用していないので、電源を入れてブート後にまずはコンソールに着地するのですが、その画面でほぼ毎回必ず以下のようなメッセージが割り込むような形で出力されるのが気になっていました。
```
Bluetooth: hci0: Reading supported features failed (-16)
```
周辺のログは以下のとおりでした。
```
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Waiting for firmware download to complete
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Firmware loaded in 1742667 usecs
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Waiting for device to boot
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Device booted in 12901 usecs
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Found Intel DDC parameters: intel/ibt-12-16.ddc
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Applying Intel DDC parameters completed
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Firmware revision 0.1 build 19 week 44 2021
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Reading supported features failed (-16)
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: Error reading debug features
Aug 30 09:56:26 laptop kernel: Bluetooth: hci0: HCI LE Coded PHY feature bit is set, but its usage is not supported.
Aug 30 09:56:26 laptop kernel: Bluetooth: MGMT ver 1.23
Aug 30 09:56:26 laptop bluetoothd[1083]: Battery Provider Manager created
```
ChatGPTさんに訊ねてみました。

> -16 は EBUSY（リソースがビジー状態）というエラーコードです。\
> これは、Bluetooth ファームウェアまたはチップがまだ初期化完了していないときに機能の読み取りを試みたことを意味します。\
> このエラーは Bluetooth の基本機能（ペアリング／ファイル送信など）には影響しないことがほとんどです。

気味が悪いですが問題はないようです。ちなみにこれはbluetooth系のパッケージを導入する以前から発生していました。それをインストールしたら直ったというようなこともありませんでした。

***
### Bluetooth 02 (file transfar)

アンドロイド端末へのbluetoothによるファイル転送を試みました。まず基本的な接続を確認するために以下のようにしました。

```bash
pacman -S bluez bluez-utils
systemctl enable --now bluetooth

bluetoothctl
> scan on
> scan off
> devices                           # list scanned and pairable devices
> pair <device_address>             # OK
> trust <device_address>            # OK
> connect <device_address>          # FAILED!
```
```
bluetoothd[7010]: [:1.50:method_call] > org.bluez.Device1.Connect [#31]
bluetoothd[7010]: src/device.c:connect_profiles() /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX (all), client :1.50
bluetoothd[7010]: profiles/audio/a2dp.c:a2dp_source_connect() path /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX
bluetoothd[7010]: src/service.c:btd_service_connect() a2dp-source profile connect failed for XX:XX:XX:XX:XX:XX: Protocol not available
bluetoothd[7010]: [:1.50:error] < org.bluez.Error.Failed [#31]
```

`pair`と`trust`まではいいのだが`connect`が失敗する、という状態になりました。再びChatGPTさんに訊ねてみたところ、結論としては、特に問題はないことがわかりました。

> `a2dp-source` = BlueZ 側がオーディオを送り出す側（スピーカーなど）として振る舞おうとしている\
> しかし Android 側はスピーカーではない（むしろオーディオソース）\
> よって「Protocol not available」＝**そのプロファイルは Android 側がサポートしていない**ため、接続が拒否されている

> `bluetoothctl` の `connect` は、利用可能な全プロファイルを片っ端から試す\
> その中に **Android 側が非対応なプロファイル（例: a2dp-source）** があると、それが失敗して全体が「connect failed」に見える\
> でも実際は「一部のプロファイルが使えなかっただけ」

この`A2DP`というのは要するに、スピーカーやヘッドフォンのような機器に高品質にオーディオデータを送信するためのプロトコルであって、それ以外の対応していない機器に対しては無効になるので`connect`が失敗した、ということでした。bluetooth通信においては、ペアリング`pair`こそが重要なステップであり、その後の`connect`は、使用するプロファイルに応じて、必要になったときに動的に処理されるものになるそうです。ファイル転送のためなら、そのための接続能力さえあれば、それで十分だということです。

```bash
pacman -S gnome-bluetooth-3.0
bluetooth-sendto --device=XX:XX:XX:XX:XX:XX some_file.png
```
このコマンドが使用するのが`OBEX（OBject EXchange）`というプロトコルで、必要なときに自動でAndroidとのセッションが張られます。これだけで簡単にファイル転送成功できました。ただし、このパッケージは`gtk4`を引き連れてきてしまうのがちょっといやらしい。TUIで使用できる`bluetuith`というのが同じ機能を提供していてシンプルでよさそうで、そのうちcoreレポジトリに入ってくれるのを期待しています。
