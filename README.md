# unifi-uxg-dnsmasq
Script to bring static dns to unifi uxg. 

Basically, you could just add your static entries to `/run/dnsmasq.conf.d/host.dns`, but that is not persistant to we automatically create a softlink with a bit of bash scriptery. Nothing too complicated so dont worry.

## 1. Scripts

The following script adds a softlink from `/run/dnsmasq.conf.d/host.dns` (which is not reboot persistent) to `/data/custom/host.dns`.

### /data/custom/dnsinject.sh
```sh
#!/bin/bash


## Original credit for idea: https://ubiquiti-networks-forum.de/board/thread/8876-dns-alias-f%C3%BCr-uxg-lite/


DNSMASQ_CONF="/run/dnsmasq.conf.d/shared.conf"
DNSMASQ_HOSTFILE="/run/dnsmasq.conf.d/host.dns"
DNSMASQ_SOURCE="/data/custom/host.dns"


DATE=$(date +%F_%H:%M:%S)


while [ ! -e "${DNSMASQ_CONF}" ]
  do
    echo "$DATE -> DNSMASQ -> No Config, waiting 10s..." >> /data/custom/dnslog.txt
    sleep 10
  done


if [ -f $DNSMASQ_HOSTFILE ]; then
    #File Exist, do nothing
    exit 0
  else
    #File NOT exist, so symlink, restart
    ln -s $DNSMASQ_SOURCE $DNSMASQ_HOSTFILE
    echo "$DATE -> DNSMASQ -> Symlink to DNS Host file created!" >> /data/custom/dnslog.txt
    sleep 5

    ## Kill current dnsmasq, ubios-udapi-server automatically restarts it
    kill $(ps -ef | awk '/dnsmas[q]/ {print $2}')
    echo "$DATE -> DNSMASQ -> Service restarted!" >> /data/custom/dnslog.txt
fi

```


### /data/custom/restartdns.sh

This can be called by hand to reload dnsmasq if you edited the host.dns file. The above script will not restart the service if the symlink has already been created. While this script is not strictly necessary, it is very convinient.

```sh
#!/bin/bash

## Kill current dnsmasq, ubios-udapi-server automatically restarts it
kill $(ps -ef | awk '/dnsmas[q]/ {print $2}')
```

### /data/custom/host.dns

This is where you put your actual static entries. 
```cfg
host-record=example.url.net,10.10.1.2
host-record=example2.blablub.foobar,10.10.2.21
```


### Make them executable

```
~# chmod 777 /data/custom/dnsinject.sh
~# chmod 777 /data/custom/restartdns.sh
```

## 2. Cronjob to run the above script

This cronjob runs the script after a reboot and once every 5 minutes (just to be safe).

### /etc/cron.d/custom_dnsinject
```sh
## run the dnsmasq_fix script
## link to /etc/cron.d

MAILTO=""
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin


@reboot root /data/custom/dnsinject.sh
*/5 * * * * root /data/custom/dnsinject.sh
```

### Reload cron 

Don't forget to reload cron so the cronjob actually runs ^^

```~# service cron reload```

## 3. Done, enjoy :)

# License 
This is free and unencumbered software released into the public domain.

Anyone is free to copy, modify, publish, use, compile, sell, or distribute this software, either in source code form or as a compiled binary, for any purpose, commercial or non-commercial, and by any means.

In jurisdictions that recognize copyright laws, the author or authors of this software dedicate any and all copyright interest in the software to the public domain. We make this dedication for the benefit of the public at large and to the detriment of our heirs and successors. We intend this dedication to be an overt act of relinquishment in perpetuity of all present and future rights to this software under copyright law.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

For more information, please refer to <http://unlicense.org/>
