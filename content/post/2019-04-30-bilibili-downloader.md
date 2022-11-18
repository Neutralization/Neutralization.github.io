---
title: 哔哩哔哩下载工具 二
date: 2019-04-30 18:21:00
categories:
    - 爬虫
tags:
    - bilibili
    - pyinstaller
    - python
---

主要解决一些之前遇到的坑，填一些留下的 TODO。

<!-- more -->

## Vegas 不支持 HEVC 编码

思路很简单，对下载下来的`m4s`先读一遍编码，看是不是 HEVC 格式的，如果是就在转码的时候用`libx264`编码，不是就粗暴的 copy 视频流。

暂时不打算加上判断其他视频流格式的代码，遇到再说。

### 通过 ffprobe 判断编码

因为之前很少接触视频转码，也没什么机会用 ffmpeg。在判断 HEVC 编码这一步的时候，想着用类似`ffmpeg -args | find "stream"`的方式来取得视频编码。

可能是环境问题，没成功，而且还要另写正则匹配文本有些麻烦。查了一下改用 ffprobe。

> ffprobe.exe -v error -select_streams v:0 -show_entries stream=codec_name \
> -of default=nokey=1:noprint_wrappers=1 v.m4s >> ./t.txt

## ffmpeg 合并 flv

ffmpeg 的问题主要出现在合并 flv 上，dash 视频倒没有什么问题。

采用`-f concat -i file.list`的方式合并遇到了路径问题。因为听取了建议“下载的文件最好单独放个目录，直接下在脚本根目录就会很乱”。

所有的文件都下载在了子文件夹里，比如`./down`。在下载 flv 分段的时候，文件的目录结构就类似：

> ..
> ffmpeg.exe
> /down
> /down/file.list
> /down/part1.flv
> /down/part2.flv

ffmpeg 的命令写成：

> ffmpeg -safe 0 -hide_banner -f concat -i ./down/file.list -c copy ./down/finish.mp4

问题来了，file.list 里该怎么写，我原本以为应该是：

> file './down/part1.flv'
> file './down/part2.flv'
> file './down/part3.flv'

然而这样 ffmpeg 会报错找不到文件，根据错误信息里提示的文件路径，应该改成

> file 'part1.flv'
> file 'part2.flv'
> file 'part3.flv'

看来 file.list 里的根目录是 file.list 所在目录。

## 通过子进程加速下载

音频和视频同时下载。因为原本用 os.system 的方式会阻塞，改用 subprocess.Popen 子进程，然后在 ffmpeg 合并前阻塞两个子进程，确保下载完毕再进行合并。

单个任务的话就用上面的方式，然后开启多进程同时执行多个任务。但是这样又遇到了新的问题：

### 尝试多进程和 pyinstaller 的坑

因为最后要打包成 exe 给其他人用，毕竟没必要让每个人都装一个 python 环境来跑脚本。

在打包过程中发现，虽然 py 脚本能够正确执行，但是打包完成的 exe 反而变成了一个 fork 炸弹，会无限的产生子进程直到让系统宕机。

如是三次之后 Google 了一下，pyinstaller 和 multiprocessing 相性不合，使用 mulitprocessing.Process 和 mulitprocessing.Pool 的代码在 pyinstaller 编译后就会出现这个问题。

给出的解决方法只适用于 python2 版本，而我用的是 python3.7，摊手。

### 改用 threading 绕过问题

既然不能用多进程，那就用多线程好了。

而且实际上我也的确是先用的 threading 库，遇到了问题才想的用 multiprocessing。

原因是 threading 的场合，我编译 exe 完成，测试的时候想直接关闭窗口，发现做不到。

结论而且这不算什么问题，比起 fork 炸弹要可接受的多了，而且一个下载工具，本来就没有必要在中途退出。

最后上了线程池`from multiprocessing.dummy import Pool as ThreadPool`，幸好这样并不会和 pyinstaller 冲突。

## 放弃写配置文件

计划把 ffmpeg 和 aria2c 的配置单独存文件的来着，想想又要增加代码量，而且几乎不会去动这些配置，还是作罢了。

硬编码不也挺好么.jpg

## 完整代码

```python
#!/usr/bin/python
## -*- coding: utf-8 -*-

import arrow
import json
import os
import re
import requests
import subprocess

from lxml import etree
from multiprocessing.dummy import Pool as ThreadPool


today = arrow.utcnow().format('YYYY-MM-DD')
if os.path.exists('./{}'.format(today)):
    pass
else:
    os.mkdir('./{}'.format(today))

session = requests.session()
session.headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0',
    'Accept': 'application/json, text/plain, */*',
    'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
    'Referer': 'https://www.bilibili.com/video/av42038790?from=search&seid=4077958841119274554',
    'Origin': 'https://www.bilibili.com',
    'DNT': '1',
    'Connection': 'keep-alive',
    'Pragma': 'no-cache',
    'Cache-Control': 'no-cache',
}

with open('cookies', 'r') as f:
    cookies = f.read()
session.headers['cookie'] = cookies


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
        aids = [x['id'] for x in playinfo['dash']['audio']]
        aurl = playinfo['dash']['audio'][aids.index(max(aids))]['baseUrl']
        vids = [x['id'] for x in playinfo['dash']['video']]
        vurl = playinfo['dash']['video'][vids.index(max(vids))]['baseUrl']

        ap = subprocess.Popen(
            '''aria2c.exe -x10 -k1M --file-allocation=none --auto-file-renaming=false "{aurl}"\
            --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0"\
            --header="Accept: */*"\
            --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"\
            --header="Referer: https://www.bilibili.com/video/av{aid}"\
            --header="Origin: https://www.bilibili.com"\
            --header="DNT: 1"\
            --header="Connection: keep-alive"\
            --header="Pragma: no-cache"\
            --header="Cache-Control: no-cache"\
            --out ./{today}/{cid}_a.m4s'''.format(today=today, aurl=aurl, aid=aid, cid=cid), shell=True)
        vp = subprocess.Popen(
            '''aria2c -x10 -k1M --file-allocation=none --auto-file-renaming=false "{vurl}"\
            --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0"\
            --header="Accept: */*"\
            --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"\
            --header="Referer: https://www.bilibili.com/video/av{aid}"\
            --header="Origin: https://www.bilibili.com"\
            --header="DNT: 1"\
            --header="Connection: keep-alive"\
            --header="Pragma: no-cache"\
            --header="Cache-Control: no-cache"\
            --out ./{today}/{cid}_v.m4s'''.format(today=today, vurl=vurl, aid=aid, cid=cid), shell=True)
        ap.wait()
        vp.wait()
        vt = subprocess.Popen(
            '''ffprobe.exe -v error -select_streams v:0 -show_entries stream=codec_name\
            -of default=nokey=1:noprint_wrappers=1\
            ./{today}/{cid}_v.m4s >> ./{today}/{cid}_t.txt'''.format(today=today, cid=cid), shell=True)
        vt.wait()
        with open('./{today}/{cid}_t.txt'.format(today=today, cid=cid), 'r') as f:
            v_type = str(f.read()).replace('\n', '')
        if v_type == 'hevc':
            fp = subprocess.Popen(
                '''ffmpeg -n -hide_banner -i ./{today}/{cid}_v.m4s -i ./{today}/{cid}_a.m4s -c:v libx264 -c:a copy\
                ./{today}/av{aid}.mp4'''.format(today=today, cid=cid, aid=aid), shell=True)
        else:
            fp = subprocess.Popen(
                '''ffmpeg -n -hide_banner -i ./{today}/{cid}_v.m4s -i ./{today}/{cid}_a.m4s -c:v copy -c:a copy\
                ./{today}/av{aid}.mp4'''.format(today=today, cid=cid, aid=aid), shell=True)

        os.remove('./{today}/{cid}_t.txt'.format(today=today, cid=cid))
        fp.wait()
        os.remove('./{today}/{cid}_a.m4s'.format(today=today, cid=cid))
        os.remove('./{today}/{cid}_v.m4s'.format(today=today, cid=cid))

    elif playinfo.get('durl'):
        vlink = [x['url'] for x in playinfo['durl']]
        with open('./{today}/{aid}_merge.txt'.format(today=today, aid=aid), 'a', encoding='utf-8') as t:
            for p, link in enumerate(vlink):
                t.write(
                    """file './av{aid}_{p}.flv'\n""".format(aid=aid, p=p))

        for p, link in enumerate(vlink):
            subprocess.check_call(
                '''aria2c.exe -x10 -k1M --file-allocation=none --auto-file-renaming=false "{link}"\
                --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0"\
                --header="Accept: */*"\
                --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"\
                --header="Referer: https://www.bilibili.com/video/av{aid}"\
                --header="Origin: https://www.bilibili.com"\
                --header="DNT: 1"\
                --header="Connection: keep-alive"\
                --header="Pragma: no-cache" --header="Cache-Control: no-cache"\
                --out ./{today}/av{aid}_{p}.flv'''.format(today=today, link=link, aid=aid, p=p), shell=True)

        fp = subprocess.Popen(
            '''ffmpeg -safe 0 -hide_banner -f concat -i ./{today}/{aid}_merge.txt -c copy\
            ./{today}/av{aid}.mp4'''.format(today=today, aid=aid), shell=True)
        fp.wait()
        os.remove('./{today}/{aid}_merge.txt'.format(today=today, aid=aid))

        for p, link in enumerate(vlink):
            os.remove(
                './{today}/av{aid}_{p}.flv'.format(today=today, aid=aid, p=p))


def main():
    pool = ThreadPool(processes=5)
    with open('down.txt', 'r') as f:
        aids = re.findall(r'av(\d+)', f.read())
        for x in aids:
            pool.apply_async(videourl, (x, videocid(x)))
    pool.close()
    pool.join()


if __name__ == '__main__':
    main()

```

## TODO

- ~~部分视频的视频流是 HEVC 编码而不是 x264，Windows 读不出 MP4 缩略图，考虑要不要判断自动转码。~~
- ~~aria2 和 ffmpeg 的参数单独存文件方便修改，不然每次都要重新编译 exe。~~
- 分 P 选择和清晰度选择

## EOF
