---
layout: default
title: MacBook Pro 2017でLinuxを使う場合のSuspend/Resumeの設定について
---
# MacBook Pro 2017でLinuxを使う場合のSuspend/Resumeの設定について

## 概要（Overview）

MacBook Pro (2017, Retina, 13-inch, タッチバーなし) に Arch Linux を導入した際、サスペンド（suspend）およびレジューム（resume）に関していくつかの問題が発生した。本稿では、観測された問題点と暫定的な対処法（workaround）、および今後の課題について整理する。

## 問題点（Problems Observed）

- suspend に入ることはできるが、resume(復帰)後に画面が真っ黒になる
- resume 後に操作不能となる（画面は点灯しているが入力を受け付けない）

## 暫定対応（Workaround）

- 新しいカーネルの利用
- resume復帰速度の改善
    - (1) d3cold_allowed の一括無効
    - (2) PCIe Switch配下の応答遅れに対応
    - (3) Thunderboltを通常は無効にしておく

### 新しいカーネルの利用

現在、以下のカーネルで動作確認をしています。
なお、以前はLinux Mintを試用していましたが、公式に新しいカーネルを使うことは難しく、より新しいカーネルを利用したかったため、カーネルの更新が早い Arch Linux に移行しました。

```
$ uname -r
6.15.4-arch2-1
```

### resume速度の改善

上記のカーネルの場合下記コマンドでsuspendできますが、デフォルトの状態ではresumeが完了するまでに数分かかってしまいます。

```
systemctl suspend 
```

復帰が遅くなる原因はNVMe、Thunderbolt及びPCI デバイスが Unable to change power state でエラーになるためです。現状のLinuxではこれらのデバイスの"深いスリープ"(D3 Cold)が管理できないようです。これらのデバイスが"深いスリープ"にならないように設定をすることで、resumeに時間がかかることを回避します。

#### (1) d3cold_allowed の一括無効

下記のスクリプトで問題となるデバイスの"深いスリープ"(D3 Cold)を無効化します。
私の場合は~/git/tm-bin/tm-prepare_suspend.shというファイル名で保存しています。

```
#!/usr/bin/env bash

# /sys/devices/ 以下の全 d3cold_allowed を 0 に設定
echo "全デバイスの d3cold_allowed を無効化中..."
for f in $(find /sys/devices/ -name d3cold_allowed); do
  echo 0 | sudo tee "$f" > /dev/null
done

# 特定の Thunderbolt / NVMe デバイスにも個別で明示的に設定
echo "特定のデバイスも個別に設定中..."
for device in \
  0000:01:00.0 \
  0000:1c:00.0 \
  0000:05:00.0 \
  0000:05:01.0 \
  0000:05:02.0 \
  0000:06:00.0
do
  if [ -e /sys/bus/pci/devices/$device/d3cold_allowed ]; then
    echo 0 | sudo tee /sys/bus/pci/devices/$device/d3cold_allowed > /dev/null
  fi
done

echo "d3cold設定完了。suspendできます。"
```

このD3Coldの無効化は再起動されるとリセットされてしまい、元に戻ってしまいます。
そのためスリープが実行される前に、systemdのフックを利用して実行するようにします。

```
sudo nano /lib/systemd/system-sleep/tm-prepare-suspend
```

内容は下記となります。

```
#!/usr/bin/env bash

logfile="/tmp/suspend.log"

log() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> "$logfile"
}

case $1 in
  pre)
    log "running prepare_suspend.sh before suspend"
    /home/takachin/git/tm-bin/tm-prepare_suspend.sh >> "$logfile" 2>&1
    ;;
  post)
    log "woke up from suspend"
    ;;
esac
```

これで"深いスリープ"(D3 Cold)からの復帰の失敗が回避され、resumeが早く完了するようになります。

#### (2) PCIe Switch配下の応答遅れに対応

スリープが遅い原因を調査するとLinux未対応のApple固有のPCIe Switch系統の復帰が遅れていることがわかりました。

PCIe Switchについて、ChatGPTによると下記の図のようになります。
```
        ┌────────────┐
        │ CPU / SoC  │
        └────┬───────┘
             │ PCIe x4
     ┌───────┴──────────────┐
     │  PCIe Switch         │
     │ （例：0000:05:00.0)   │
     └──┬────────┬────┬─────┘
        │        │    │
     NVMe     Wi-Fi  Thunderbolt
  (05:01.0) (05:02.0) (05:04.0)
```
これらのPCIe Switch配下にあるNVMeやThunderboltなどがうまく復帰できず、resumeに時間がかかることがわかりました。

そこで、/etc/default/grub の GRUB_CMDLINE_LINUX_DEFAULT= を下記のように変更、起動パラメータ調整します。

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet splash nvme_core.default_ps_max_latency_us=0 nvme.noacpi=1 pci=noaer i915.enable_dc=0 i915.enable_fbc=0 i915.enable_psr=0"
```

それぞれのパラメータについて一部補足します。

- nvme_core.default_ps_max_latency_us=0 は、NVMeデバイスが省電力状態に入るのを防ぎ、復帰を高速化します。
- pci=noaer はPCIのAdvanced Error Reportingを無効にし、ログに余計なエラーを出さないようにします。

その後下記を実行し、起動パラメータに反映します。

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

これにより、一部の電力管理は無効化されるため、バッテリー持ちが若干悪くなる可能性がありますが、resumeの速度が改善されます。

#### (3) Thunderboltを通常は無効にしておく

私の利用方法では外部ディスプレイを繋がない(Thunderboltを使わない)ため、思い切ってThunderboltを無効にしてしまいます。以下の手順で無効化できます。


```
sudo nano /etc/modprobe.d/disable-thunderbolt.conf
```

disable-thunderbolt.confに下記の内容を記載します。

```
blacklist thunderbolt
```

再起動後、Thunderboltが無効になりますが、復帰は高速化します。

もしThunderboltを使いたい場合、下記のようにコマンドを実行して有効にします。

```
sudo modprobe thunderbolt
```

## 現時点でのまとめと今後の課題

本稿で示した対策により、MacBook Pro (2017) におけるサスペンド／レジュームの挙動は **実用的な速度と安定性を確保**できるようになりました。

特に以下の2点が効果的でした：

- `d3cold_allowed` の一括無効化による復帰失敗の回避
- `GRUB` 起動パラメータの適切な調整による resume の高速化

ただし、以下の点は今後も調査・検証が必要です：

- Thunderbolt 有効時の安定性と resume 成功率
- バッテリー駆動時の消費電力への影響
- 将来のカーネルアップデートに伴う挙動の変化

この手順を適用することにより、MacBook Pro (2017) でもある程度実用的なsuspend、resumeになりますが、suspend中の省電力機能を意図的に無効化しているため、macOS に比べるとsuspend中のバッテリー消費は多くなる点にご留意ください。

本ページの内容は随時更新予定です。

[← トップページに戻る](index.md)