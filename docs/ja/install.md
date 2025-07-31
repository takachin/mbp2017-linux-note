---
layout: default
title: MacBook Pro 2017でArch Linuxのインストールの注意点について
---

# MacBook Pro 2017でArch Linuxのインストールの注意点について

インストールについては[Arch Linuxのインストール](https://wiki.archlinux.jp/index.php/%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%82%AC%E3%82%A4%E3%83%89)に沿ってインストールすれば問題ないですが、ディスプレイの行数が足りず、インストールを進めると画面外に表示内容がはみ出してしまい、インストールを進めることができないことがあります。

この場合、途中で下記のコマンドを実行します。下記は横120文字 × 縦40行に設定した場合です。

```
stty cols 120 rows 40
```

この数字を調整してからインストールを進めると画面外に表示内容をはみ出すことなくインストールを進めることができます。

---

<footer>
    Contact: <a href="https://x.com/takachin" target="_blank">@takachin on X</a>
</footer>
