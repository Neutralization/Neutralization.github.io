---
title: 给 OpenWrt 加上 DDNS
date: 2023-06-06 15:27:30
categories:
    - 杂记
tags:
    - openwrt
    - hostker
---

给家里的宽带申请了公网 IP，为了方便连回家绑了自己的一个域名。虽说一般也不会关掉路由器，为防万一还是简单做个 DDNS 确保 IP 地址能更新。

<!-- more -->

因为用的是主机壳的域名解析服务，先从[后台](https://console.hostker.net/index.html#/Account)拿到 API 密钥。

然后去域名解析管理添加一条新的 A 记录，指向家里路由的公网 IP 地址。

简单[写个 bash 脚本](https://gist.github.com/Neutralization/aaf154b058450bb05894291d1c53bb32)，搞定。

```bash
domain="example.com"
header="router" # router.example.com <-> public ip address
email="*************************"
token="*************************"
trueip=$(ifconfig pppoe-wan | grep -Eo 'inet (addr:)?([0-9]*\.){3}[0-9]*' | grep -Eo '([0-9]*\.){3}[0-9]*' | grep -v '127.0.0.1')
echo "${trueip}"

if [[ "${trueip}" = $(cat /root/ip) ]]; then
    echo "pass"
else
    getRecord="{\"action\": \"DomainDNS.getRecords\", \"domain\": \"${domain}\"}"
    echo "${getRecord}"
    sign=$(echo -n "${getRecord}" | openssl sha256 -hmac "${token}" | sed 's/^.* //')
    echo "${sign}"
    curl -sL -X POST "https://api.hostker.net/v2" -H 'Content-Type: application/json' -H "Sign: ${sign}" -H "Email: ${email}" -d "${getRecord}" -o records.json

    recordip=$(jq -r '.result.records[] | select(.header=="'$header'").value' records.json)
    echo "${recordip}"
    if [[ "${trueip}" = "${recordip}" ]]; then
        echo "${recordip}" > /root/ip
    else
        old_record=$(jq -c -r ".result.records[] | select(.header==\"${header}\")" records.json)
        echo "${old_record}"
        new_record="{\"type\":\"A\",\"ttl\":600,\"header\":\"${header}\",\"value\":\"${trueip}\"}"
        echo "${new_record}"

        upRecord="{\"action\": \"DomainDNS.updateRecord\", \"domain\": \"${domain}\", \"old_record\": ${old_record}, \"new_record\": ${new_record}}"
        echo "${upRecord}"
        sign=$(echo -n "${upRecord}" | openssl sha256 -hmac "${token}" | sed 's/^.* //')
        echo "${sign}"
        curl -sL -X POST "https://api.hostker.net/v2" -H 'Content-Type: application/json' -H "Sign: ${sign}" -H "Email: ${email}" -d "${upRecord}" -o resp.json
    fi
fi
```

接着去 OpenWrt 添加一条计划任务：

> opkg update  
> opkg install openssl-util jq  
> chmod +x /root/hostker_ddns.sh  
> crontab -e  
>  
> \*/5 \* \* \* \* /root/hostker_ddns.sh
