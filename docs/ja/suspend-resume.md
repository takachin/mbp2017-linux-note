# Suspend/Resume Behavior on MacBook Pro 2017

## 概要（Overview）

MacBook Pro (2017, Retina, 13-inch) に Arch Linux を導入した際、サスペンド（suspend）およびレジューム（resume）に関していくつかの問題が発生した。本稿では、観測された問題点と暫定的な対処法（workaround）、および今後の課題について整理する。

## 問題点（Problems Observed）

- suspend に入ることはできるが、resume（復帰）後に画面が真っ黒になる
- resume 後に操作不能となる（画面は点灯しているが入力を受け付けない）
- resume 後にファンが高回転で回り続けることがある
- （今後追加予定）

## 暫定対応（Workaround）

- `mem_sleep_default=deep` をカーネルパラメータに追加することで、suspend の安定性が向上した
- `i915.enable_psr=0` により画面復帰時の問題を軽減
- `NVMe` デバイスに関して D3cold を無効化（`/etc/modprobe.d/` などで設定）
- `thunderbolt` モジュールを一時的に blacklist（resume トラブルの切り分け目的）

(以下作成中)