---
layout: default
title: Points to note when installing Arch Linux on a 2017 MacBook Pro
---

# Points to note when installing Arch Linux on a 2017 MacBook Pro

You can generally follow the [Arch Linux Installation guide](https://wiki.archlinux.org/title/Installation_guide) without issues, but there may be cases where the number of lines on the display is insufficient, causing the display content to overflow outside the screen and preventing the installation from proceeding.

If this happens during the installation process, run the following command. The following example sets the display to 120 characters horizontally × 40 lines vertically.

```
stty cols 120 rows 40
```

After adjusting these numbers, you can proceed with the installation without the display content overflowing the screen.

[Return to Top Page](index.md)

---

<footer>
    Contact: <a href="https://x.com/takachin" target="_blank">@takachin on X</a>
</footer>
