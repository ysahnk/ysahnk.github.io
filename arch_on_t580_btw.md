<img src="/images/iconhikari02.jpg" alt="penguin icon image" width="100" height="100" />

### I use arch on T580, btw
t580 DDR4-2400 PC4-19200
Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz / coffee lake-s
Intel UHD Graphics 620 / Gen9.5
SSD / SATA 6.0Gb/s, 2.5" wide, 7mm high (e.g. xxxGB SSD)
M.2 SSD / PCIe NVMe, PCIe 3.0 x 2, 16Gb/s
128GB M.2 SSD / PCIe NVMe, PCIe 3.0 x 2, 16Gb/s,
in WWAN slot as 2nd Storage, mutually exclusive with WWAN

### Thunderbolt firmware problem (important)
[The "Thunderbolt Firmware Problem" Explained (reddit.com)](https://www.reddit.com/r/thinkpad/comments/1dfdp18/the_thunderbolt_firmware_problem_explained/)

pacman -S fwupd udisk2 efivat
fwupdmgr refresh
fwupdmgr get-updates
fwupdmgr update
fwupdtool install foo.cab --verbose
fwupdmgr install DEVICE-ID	# from the lower version 14->18->20

### Yellowish tint monitor

### Bluetooth
pacman -S bluez bluez-utils
systemctl enable --now bluetooth

Error

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

-16 は EBUSY（リソースがビジー状態）というエラーコードです。
これは、Bluetooth ファームウェアまたはチップがまだ初期化完了していないときに機能の読み取りを試みたことを意味します。
このエラーは Bluetooth の基本機能（ペアリング／ファイル送信など）には影響しないことがほとんどです。

Bluetooth: hci0: Unable to create CMAC crypto context


###

bluetoothd[7010]: [:1.50:method_call] > org.bluez.Device1.Connect [#31]
bluetoothd[7010]: src/device.c:connect_profiles() /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX (all), client :1.50
bluetoothd[7010]: profiles/audio/a2dp.c:a2dp_source_connect() path /org/bluez/hci0/dev_XX_XX_XX_XX_XX_XX
bluetoothd[7010]: src/service.c:btd_service_connect() a2dp-source profile connect failed for XX:XX:XX:XX:XX:XX: Protocol not available
bluetoothd[7010]: [:1.50:error] < org.bluez.Error.Failed [#31]

この症状、非常によくあるパターンで、**特に Android デバイスと BlueZ を組み合わせたとき**に頻出します。要点をまとめつつ、なぜ `connect` に失敗しても `bluetooth-sendto` で送信できるのかを解説します。

---

## 🧠 結論から：**connect の失敗は重大ではない**

`bluetoothctl` の `connect <address>` で失敗するのは、特定のプロファイル（たとえば A2DP）への接続が失敗しているからです。
**しかし、OBEX（ファイル転送）は `connect` を実行しなくても Android 側から自動でセッションを張れるため、`bluetooth-sendto` は成功します。**

---

## 🔍 詳細なログ解析

あなたのログのポイントはこれです：

```
a2dp-source profile connect failed for 00:ED:B8:FF:0F:D0: Protocol not available
```

### ➤ これは何を意味するか？

* `a2dp-source` = BlueZ 側が **オーディオを送り出す側（スピーカーなど）** として振る舞おうとしている
* しかし Android 側は **スピーカーではない**（むしろオーディオソース）
* よって「Protocol not available」＝**そのプロファイルは Android 側がサポートしていない**ため、接続が拒否されている

### ➤ つまり：

* `bluetoothctl` の `connect` は、**利用可能な全プロファイルを片っ端から試す**
* その中に **Android 側が非対応なプロファイル（例: a2dp-source）** があると、それが失敗して全体が「connect failed」に見える
* でも実際は「一部のプロファイルが使えなかっただけ」

---

## 📤 なぜ `bluetooth-sendto` では送信できたのか？

* `bluetooth-sendto` は **OBEX (Object Push Profile)** を使用
* このプロファイルは Android 側がサポートしており、必要なときに Android 側が自動で OBEX セッションを張りに来る
* よって `bluetoothctl` の `connect` を成功させる必要がない

つまり：

> **`connect` に失敗しても、必要なプロファイル（OBEX）が相手側で有効なら、そのときだけセッションが確立されて使える**。

---

## ✅ 対応や補足

### ✔ 放置で問題ないケース

* ファイル送信だけが目的なら、`connect` にこだわらなくてOK。
* Android デバイス側から「受信」待機状態にしておけば、受信確認プロンプトも表示される。

### ✔ 気になるなら `connect` 対象のプロファイルを制限

`bluetoothctl` で `connect` が失敗して気になる場合は、BlueZ 側の設定を調整して `a2dp-source` を無効化することもできます（udev ルールや BlueZ ソースレベルで）。ただし少し面倒。

### ✔ Android 側が OBEX Push 対応か確認

一部の Android 端末（特に最近の Pixel やカスタム ROM）では、セキュリティ理由で Bluetooth ファイル受信（OBEX Push）機能が制限されている場合もあります。`bluetooth-sendto` で送れているなら、少なくとも今回はそれが有効だったということです。

---

## 🟢 まとめ

| 観点                | 内容                                                |
| ----------------- | ------------------------------------------------- |
| `connect` が失敗する理由 | BlueZ が使おうとしたプロファイル（例: a2dp-source）が Android に非対応 |
| ファイル送信ができた理由      | OBEX は Android 側が対応しており、自動セッション確立される              |
| 対処は必要か            | ファイル送信が目的なら特に必要なし                                 |

---

もし今後、たとえば Bluetooth スピーカーや HID（キーボード）などと接続する際に `connect` が失敗する場合は、また状況を教えてもらえれば診断できます。それぞれ違う要因が絡むので。

Bluetooth 通信において、「ペアリング」こそが最も重要なステップであり、connect はあくまで 使用したいプロファイルに応じて必要になったときに行われる動的な処理にすぎません。

🔑 ペアリング（pair, trust）の役割
操作	目的
pair	デバイス同士が互いに認証され、安全に通信できる状態を確立
trust	自動接続やサービスへのアクセスを許可。ユーザ操作なしに使えるようにする

つまり「この端末とは安全に通信していいよ」とお互いが確認し合うステップ。

これが成功していれば、必要なときに必要なプロファイルだけが接続されるという流れになります。

###

bluetoothctl
> scan on
> scan off
> devices						# list scanned and pairable devices
> pair <device_address>
> trust <device_address>
> connect <device_address>

###

pacman -S gnome-bluetooth-3.0
bluetooth-sendto --device=XX:XX:... file.png	# it works

### GPU acceleration
pacman -S intel-media-driver
pacman -S intel-gpu-tools	# intel_gpu_top for monitoring GPU usage
pacman -S libva-utils
vainfo: VA-API version: 1.22 (libva 2.22.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 25.2.1 ()
.config/chromium-flags.conf
--enable-features=AcceleratedVideoDecodeLinuxGL
chrome://gpu 
chrome://media-internals 
chrome extension h264ify
watch -n 1 cat /sys/class/drm/card1/gt_cur_freq_mhz

### Android file transfer
pacman -S gvfs-mtp
android file transfer
lsusb to check Bus xxx Device yyy
gio mount mtp://[usb:xxx,yyy]/
access via /run/user/$UID/gvfs/...

### Console font
echo FONT=LatGrkCyr-12x22 >> /etc/vconsole.conf
echo KEYMAP=jp106 >> /etc/vconsole.conf
