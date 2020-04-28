---
title: 哔哩哔哩下载工具
s: bilibili-downloader
date: 2019-04-07 01:36:19
categories:
- 爬虫
tags:
- bilibili
---

给哔哩哔哩周刊组写的下载工具。每次都要下载十几个视频，一个个手动来实在是太麻烦了。

看了下，发现下载最高清一路的视频源还是需要cookie认证，写不来登录，只能用dirty的方式读cookie文件了。
<!-- more -->
## 分析

获取视频需要知道视频的av号和视频id，api接口里分别是`avid`和`cid`，有了两者之后就可以通过api获得视频不同清晰度的url地址。

B站现在新视频全部使用了dash，但是老视频没有dash，还是单个flv，两者在api上返回有区别，前者是`dash`字段，后者是`durl`字段，需要判断是哪一种视频流。

对于dash，音频和视频流分割两路，都是m4s格式，下载后需要合并。而flv根据视频时长会有多个分段文件，同样也需要合并。

## 代码

```python
#!/usr/bin/python
## -*- coding: utf-8 -*-

import json
import os
import re
import requests

from lxml import etree

session = requests.session()
session.headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0',
    'Accept': 'application/json, text/plain, */*',
    'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
    # 'Referer': 'https://www.bilibili.com/video/av42038790',
    'Origin': 'https://www.bilibili.com',
    'DNT': '1',
    'Connection': 'keep-alive',
    'Pragma': 'no-cache',
    'Cache-Control': 'no-cache',
}

with open('cookies', 'r') as f:
    cookies = f.read()
session.headers["cookie"] = cookies


def videocid(aid=42038790, part=1):
    params = (
        ('aid', aid),
        ('jsonp', 'jsonp'),
    )

    response = session.get(
        'https://api.bilibili.com/x/player/pagelist', params=params, )  # cookies=cookies)
    result = json.loads(response.content)
    cid = result['data'][part - 1]['cid']
    return cid


def videourl(aid=42038790, cid=73802454):
    params = (
        ('avid', aid),
        ('cid', cid),
        ('qn', '112'),
        ('fnver', '0'),
        ('fnval', '16'),
        ('type', ''),
        ('otype', 'json'),
    )

    response = session.get(
        'https://api.bilibili.com/x/player/playurl', params=params,)  # cookies=cookies)
    result = json.loads(response.content)
    if result.get('data') is None:
        response = session.get(
            'https://api.bilibili.com/pgc/player/web/playurl', params=params,)  # cookies=cookies)
        result = json.loads(response.content)
        playinfo = result.get('result')
    else:
        playinfo = result.get('data')
    if playinfo is None:
        print(aid, 'Not Found\n')
        return False
    if playinfo.get('dash'):
        vids = [x['id'] for x in playinfo['dash']['video']]

        vurl = playinfo['dash']['video'][vids.index(
            max(vids))]['baseUrl']

        aids = [x['id'] for x in playinfo['dash']['audio']]

        aurl = playinfo['dash']['audio'][aids.index(
            max(aids))]['baseUrl']

        os.system('''aria2c -x10 -k1M --file-allocation=none "{}" --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0" --header="Accept: */*" --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2" --header="Referer: https://www.bilibili.com/video/av{}" --header="Origin: https://www.bilibili.com" --header="DNT: 1" --header="Connection: keep-alive" --header="Pragma: no-cache" --header="Cache-Control: no-cache" --out {}_v.m4s'''.format(vurl, aid, cid))

        os.system('''aria2c.exe -x10 -k1M --file-allocation=none "{}" --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0" --header="Accept: */*" --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2" --header="Referer: https://www.bilibili.com/video/av{}" --header="Origin: https://www.bilibili.com" --header="DNT: 1" --header="Connection: keep-alive" --header="Pragma: no-cache" --header="Cache-Control: no-cache" --out {}_a.m4s'''.format(aurl, aid, cid))

        os.system(
            '''ffmpeg -i {}_v.m4s -i {}_a.m4s -c:v copy -c:a copy -strict experimental av{}.mp4'''.format(cid, cid, aid))
        os.remove('{}_a.m4s'.format(cid))
        os.remove('{}_v.m4s'.format(cid))

    elif playinfo.get('durl'):
        vlink = [x['url'] for x in playinfo['durl']]
        print(vlink)
        for p, link in enumerate(vlink):
            os.system('''aria2c.exe -x10 -k1M --file-allocation=none "{}" --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0" --header="Accept: */*" --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2" --header="Referer: https://www.bilibili.com/video/av{}" --header="Origin: https://www.bilibili.com" --header="DNT: 1" --header="Connection: keep-alive" --header="Pragma: no-cache" --header="Cache-Control: no-cache" --out av{}_{}.flv'''.format(link, aid, aid, p))
        with open('merge.txt', 'a', encoding='utf-8') as t:
            for x in os.listdir('.'):
                if str(aid) in x:
                    t.write("file '{}'\n".format(x))
        os.system('ffmpeg -f concat -i merge.txt -c copy av{}.mp4'.format(aid))
        os.remove('merge.txt')

        for x in os.listdir('.'):
            if x.endswith('.flv'):
                pass
                os.remove(x)


def main():
    with open('down.txt', 'r') as f:
        aids = re.findall(r'av(\d+)', f.read())
        for x in aids:
            videourl(x, videocid(x))


if __name__ == '__main__':
    main()

```

## TODO
* 部分视频的视频流是HEVC编码而不是x264，Windows读不出MP4缩略图，考虑要不要判断自动转码。
* aria2和ffmpeg的参数单独存文件方便修改，不然每次都要重新编译exe。
* 分P选择和清晰度选择

## EOF
