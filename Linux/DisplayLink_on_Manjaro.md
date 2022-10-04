# DisplayLink on Manjaro Linux

Code taken from https://forum.manjaro.org/t/complete-manjaro-noob-looking-for-help-with-displaylink-to-complete-switch-over/88071

```Bash
pamac build displaylink
pamac build evdi-git
systemctl enable displaylink.service
stemctl start displaylink.service
```
