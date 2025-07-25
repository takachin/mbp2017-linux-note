
# Suspend/Resume Behavior on MacBook Pro 2017

## Overview.

When installing Arch Linux on a MacBook Pro (2017, Retina, 13-inch), I encountered some issues with suspend and resume behavior. This paper summarizes the observed problems, workarounds, and future issues.

## Problems Observed

- The screen goes black after "resume" (recovery), although it is possible to enter "suspend".
- After resume, operation becomes inoperable (screen lights up, but does not accept input).

## Temporary fix (Workaround)

- Use new kernel
- Improvement of resume speed
  - (1) Disable d3cold_allowed at once
  - (2) Address response delay under PCIe Switch
  - (3) Disable Thunderbolt normally.

### Use of new kernel

I'm currently testing the following kernels.
Note that I used to try Linux Mint, but it was difficult to officially use a newer kernel, and I wanted to use a newer kernel, so I moved to Arch Linux, which has a faster kernel update.

```
$ uname -r
6.15.4-arch2-1
```

### Improving resume speed

With the above kernel, you can suspend with the following command, but in the default state, it takes several minutes for resume to complete.

```
systemctl suspend 
```

The reason for the slow resume is that NVMe, Thunderbolt, and PCI devices will fail in the Unable to change power state. Currently Linux does not seem to be able to manage "deep sleep" (D3 Cold) for these devices. You can avoid the long time to resume by configuring these devices not to go into "deep sleep".

#### (1) Bulk disabling of d3cold_allowed

The following script disables "deep sleep" (D3 Cold) for the device in question.
In my case, I have saved it under the file name ~/git/tm-bin/tm-prepare_suspend.sh.

```
#!/usr/bin/env bash

# Set all d3cold_allowed under /sys/devices/ to 0
echo "Disabling d3cold_allowed for all devices..."
 for f in $(find /sys/devices/ -name d3cold_allowed); do
  echo 0 | sudo tee "$f" > /dev/null
done

# Explicitly set specific Thunderbolt / NVMe devices individually as well
echo "Specific device also being configured individually..."
for device in \
  0000:01:00.0 \
  0000:1c:00.0 \
  0000:05:00.0 \
  0000:05:01.0 \ \ 0000:05:01.0 \ 0000:05:02.0 \ 0000:05:02.0
  0000:05:02.0 \ \ 0000:05:02.0 \ 0000:06:00.0 \ 0000:06:00.0
  \ 0000:06:00.0
do
  if [ -e /sys/bus/pci/devices/$device/d3cold_allowed ]; then
    echo 0 | sudo tee /sys/bus/pci/devices/$device/d3cold_allowed > /dev/null
  fi /sysbus/pci/devices
done

echo "d3cold configuration complete, can suspend."
```

This disabling of D3Cold will be reset and reverted when the system is rebooted.
Therefore, before sleep is executed, use the systemd hook to execute it.

```
sudo nano /lib/systemd/system-sleep/tm-prepare-suspend
```

The contents will be as follows.

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
   ;;;
post)
    log "woke up from suspend"
   ;;;
esac
```

This avoids failure to wake up from "deep sleep" (D3 Cold) and allows resume to complete faster.

#### (2) Address response delay under PCIe Switch

I investigated the cause of the slow sleep and found that the recovery of the Apple-specific PCIe Switch system, which is not yet supported by Linux, was delayed.

 According to ChatGPT, the PCIe Switch is shown in the figure below.
```
        ┌────────────┐
        │ CPU / SoC │
        └────┬───────┘
             PCIe x4
     ┌───────┴──────────────┐
     │ PCIe Switch │
     │ (e.g. 0000:05:00.0) │
     └──┬────────┬────┬─────┘
        │        │    │
     NVMe Wi-Fi Thunderbolt
  (05:01.0) (05:02.0) (05:04.0)
```
I found that NVMe, Thunderbolt, etc. under these PCIe Switches were not restoring properly and taking a long time to resume.

Therefore, I changed GRUB_CMDLINE_LINUX_DEFAULT= in /etc/default/grub as follows and adjusted the boot parameters.

```
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet splash nvme_core.default_ps_max_latency_us=0 nvme.noacpi=1 pci=noaer i915.enable_dc=0 i915. enable_fbc=0 i915.enable_psr=0"
```

Some additional information about each parameter.

- nvme_core.default_ps_max_latency_us=0 prevents NVMe devices from entering a power-saving state and speeds up their recovery.
- pci=noaer disables Advanced Error Reporting for PCI and prevents extra errors in the log.

Then run the following to reflect it in the boot parameters

```
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

This will disable some power management, which may result in slightly worse battery life, but will improve resume speed.

#### (3) Leave Thunderbolt normally disabled

Since I do not connect an external display (i.e., do not use Thunderbolt) in my usage, I will disable Thunderbolt to the best of my ability. You can disable it by the following procedure.


```
sudo nano /etc/modprobe.d/disable-thunderbolt.conf
```

Put the following information in disable-thunderbolt.conf.

```
blacklist thunderbolt
```

After reboot, Thunderbolt will be disabled, but restoration will be faster.

If you want to use Thunderbolt, run the following command to enable it

```
sudo modprobe thunderbolt
```

## Summary at this time and future issues

With the measures presented in this paper, the suspend/resume behavior on the MacBook Pro (2017) is now **practically fast and stable**.
 
The following two points were particularly effective:

- Avoiding resumption failures by bulk disabling of `d3cold_allowed`.
- Speeding up resume by properly adjusting `GRUB` startup parameters.

However, the following points require further investigation and verification:

- Stability and resume success rate when Thunderbolt is enabled
- Impact on power consumption when running on battery
- Changes in behavior with future kernel updates

However, please note that the battery consumption during suspend is higher than that of macOS because the power saving function during suspend is intentionally disabled. Please note that the battery consumption during suspend is higher than that of macOS.

The contents of this page will be updated as needed.

[Return to Top Page](index.md)
