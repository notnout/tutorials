# How to block Twitter or Reddit

Is some specific website, like Twitter, bringing you joy or is it bringing you the opposite? Do you have a hard time stopping? Here's a quick tutorial recommended by 9 out 10 dentists.

***First rule:** Never run random bash scripts you find online without reading the scripts yourself!*

---

## Blocking with /etc/hosts
If you understand the first rule, then read, copypaste and run following script in your terminal.

**On Linux or MacOS**
```shell
sudo bash -c "cat >> /etc/hosts << EOF

0.0.0.0 twitter.com
0.0.0.0 reddit.com
EOF
"
```
And that's it. This should apply the rule immediately.
If you want to check how does your `/etc/hosts` look like, then just run  `nano /etc/hosts`.

**On Windows**
```batch
echo 0.0.0.0 twitter.com >> %WINDIR%\System32\Drivers\Etc\Hosts
echo 0.0.0.0 reddit.com >> %WINDIR%\System32\Drivers\Etc\Hosts
```

---

## Blocking with pi-hole
Even better option is if you have [pi-hole](https://pi-hole.net/) set up on your network to proxy all your DNS requests. If you didn't know pi-hole is also available on [Umbrel](https://getumbrel.com/) or [Citadel](https://runcitadel.space/).

Simple steps:
1. Sign in to your your pi-hole admin interface, e.g. https://pi.hole/admin/index.php
2. Navigate to *Blacklist*
3. Add `twitter.com` in the Domain input box and click on *Add to Blacklist*
4. You are all set.

---

Posted as https://stacker.news/items/29648