# Make a HDD go to sleep

This tutorial is based on https://linuxconfig.org/change-hard-drive-s-sleep-standby-mode-timer-to-reduce-power-consumption

Install hdparm

```bash
sudo apt install hdparm
```

Disable hdparm for drive

```bash
sudo hdparm -s 0 /dev/sda
```

Enable hdparm for drive

```bash
sudo hdparm --yes-i-know-what-i-am-doing -s 1 /dev/sda
```

Set timeout for drive to go to sleep. The number argument is multiplied by 5 and that is the amount of seconds for the waiting period. Maximal value is 255.

```bash
sudo hdparm -S 180 /dev/sda
```
