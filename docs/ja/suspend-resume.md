# Suspend/Resume Behavior on MacBook Pro 2017

## 概要（Overview）

MacBook Pro (2017, Retina, 13-inch) に Arch Linux を導入した際、サスペンド（suspend）およびレジューム（resume）に関していくつかの問題が発生した。本稿では、観測された問題点と暫定的な対処法（workaround）、および今後の課題について整理する。

## 問題点（Problems Observed）

- suspend に入ることはできるが、resume(復帰)後に画面が真っ黒になる
- resume 後に操作不能となる（画面は点灯しているが入力を受け付けない）
- resume 後にファンが高回転で回り続けることがある
- （今後追加予定）

## 暫定対応（Workaround）

- 新しいカーネルの利用
- resume復帰速度の改善
    - `mem_sleep_default=deep` をカーネルパラメータに追加
    - `i915.enable_psr=0` により画面復帰時の問題を軽減
    - `NVMe` デバイスに関して D3cold を無効化（`/etc/modprobe.d/` などで設定）
    - `thunderbolt` モジュールを一時的に blacklist（resume トラブルの切り分け目的）

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
#!/bin/bash

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
  echo "[tm] $(date '+%Y-%m-%d %H:%M:%S') $1" >> "$logfile"
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

#### (2) nvme.noacpi=1 をカーネルブートオプションに追加

#### (3) initramfs に resume フックが含まれていないか確認

#### (4) Thunderboltを通常は無効にしておく