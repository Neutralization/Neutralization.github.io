---
title: 哔哩哔哩下载工具 三
date: 2019-11-16 23:48:20
s: bilibili-downloader-3
categories:
- 爬虫
tags:
- bilibili
- Shell
- awk
- sed
- curl
---

有生之年我最后居然是用Shell来填了分P的TODO……

## cURL - 命令行下的文件传输工具

cURL 这个工具太过强大，我也不知道该怎么去解释它比较好。几乎所有的网络访问，上传下载，都能够使用cURL来完成。

也难怪诸如Chrome和Firefox等浏览器，会支持复制网络访问为cURL。而其实我也一直依靠这个功能，通过[https://curl.trillworks.com/](https://curl.trillworks.com/)来快速的写我的python爬虫。

大部分的工作都可以直接靠右键复制为cURL来完成，所以只需要简单的查看下cURL的说明，便于进行一些小的修改就行。

然后开始对照之前写的python脚本的逻辑，一步步转成Shell脚本。
<!-- more -->
## jq - 命令行下的JSON处理工具

B站API接口的各种返回数据以JSON为主，直接靠awk/sed手撸解析是一件很痛苦的事。所以我们需要用上jq这个工具减轻负担。

jq解析JSON数据非常的便捷，比如获取分P数据时：

```json
{
  "code": 0,
  "message": "0",
  "ttl": 1,
  "data": [
    {
      "cid": 73802454,
      "page": 1,
      "from": "vupload",
      "part": "3",
      "duration": 66,
      "vid": "",
      "weblink": "",
      "dimension": {
        "width": 1920,
        "height": 1080,
        "rotate": 0
      }
    }
  ]
}
```
↑ jq的另一个作用就是格式化json数据，比如上面的代码就是通过`echo result.json | jq .`自动格式化出来的。

要取得分P的cid，我们可以：
`curl -sL 'https://api.bilibili.com/x/player/pagelist?aid=42038790&jsonp=jsonp' | jq -r '.data[0].cid'`

如果有多个分P，我们可以这样取得所有的cid：
`curl -sL 'https://api.bilibili.com/x/player/pagelist?aid=42038790&jsonp=jsonp' | jq -r '.data[].cid'`

真的是非常方便。

## sed - 支持正则表达式的流编辑器

考虑到windows上的兼容性，如果用视频标题做文件名，势必要排除`/?!.*|:`等特殊字符。这时我们就可以用sed来完成：
`curl -sL -H https://api.bilibili.com/x/web-interface/view?aid=42038790 | jq -r '.data.title' | sed 's/[/?!.*|:]//g'`

再后面我们会更多的用到sed。`(Flag)`

## Shell脚本的编写

`$1` 代表了命令行传给脚本的第一个参数，使用场景类似 `bash download.sh av42038790`
通过sed，我们可以兼容 `bash download.sh https://www.bilibili.com/video/av42038790?from=search&seid=3037567923943401758` 的链接模式。
```bash
aid=`echo $1 | sed -e 's/.*av//g' -e 's/[a-zA-Z?/].*//g'`
```
Shell脚本中声明变量时等号前后都不能有空格，使用变量时在变量前加上`$`
```bash
pagelist='https://api.bilibili.com/x/player/pagelist?aid='$aid'&jsonp=jsonp'
cids=`curl -sL $pagelist | jq -r '.data[].cid'`
```

这里的cids只是一个包含空格分隔的字符串`123 234 345`，为了让Shell正确计数，把它转为数组
```bash
cids_arr=($cids)
```

这样一来通过`${ #cids_arr[@] }`就能够正确得出cids中的元素个数

接下来通过for循环遍历cids中的元素，为了能同时计算元素的个数，需要另行计数：
```bash
part=0
for cid in $cids
do
    episode=$(( ++part ))
    #some code
done
```
Shell中的if else判断以if开始，fi结束：
```bash
    if [ "${#cids_arr[@]}" == "1" ]
    then
        filename=av$aid.$title.mp4 # 视频只有1P时，不标注分P
    else
        filename=av$aid.$title【P$episode】.mp4 # 多P视频标注分P
    fi
```
判断视频是DASH还是FLV：
```bash
json_url='https://api.bilibili.com/x/player/playurl?avid='$aid'&cid='$cid'&qn=116&fnver=0&fnval=16&otype=json&type='
json=`curl -sL $json_url`
dash=`echo $json | jq '.data|has("dash")'`
durl=`echo $json | jq '.data|has("durl")'`
```
处理DASH视频：
```bash
if [ "$dash" == "true" ]
then
    vp=`echo $json | jq -r '.data.dash.video[0].baseUrl'` # 视频
    ap=`echo $json | jq -r '.data.dash.audio[0].baseUrl'` # 音频

aria2c --args $vp --out ./v_$cid.m4s
aria2c --args $vp --out ./a_$cid.m4s
ffmpeg -i ./v_$cid.m4s -i ./a_$cid.m4s -c:v copy -c:a copy ./$filename
rm *.m4s
```
处理FLV视频：
```bash
elif [ "$durl" == "true" ]
then
    flvs=`echo $json | jq -r '.data.durl[].url'`
    for flv in $flvs
    do
        echo "file './"$flvname"'" >./merge_$cid.txt
        flvname=`echo $flv | sed -e 's/\?.*//g' -e 's/.*\///g'`
        aria2c --args $flv --out ./$flvname
    ffmpeg -safe 0 -f concat -i ./merge_$cid.txt -c copy ./$filename
    rm *.flv
    rm ./merge_$cid.txt
```
## shift - 支持传递多个参数

shift命令的作用是左移参数，举例来说，如果我们传给脚本3个参数a,b,c，脚本接收到的参数为：
```bash
echo $1 $2 $3
a  b  c
```
当我们执行shift后
```bash
shift
echo $1 $2 $3
b c
```
这样就可以在循环中一直只处理$1，直到所有参数左移完毕。

先判断没有参数时退出脚本：
```bash
if [ $# -eq 0 ]
then
    echo -e "->Need AVID to download!\n"
    exit 1
fi
```
加上shift和循环：
```bash
until [ $# -eq 0 ]
do
    aid=`echo $1 | sed -e 's/.*av//g' -e 's/[a-zA-Z?/].*//g'`
    #some code
    shift
done
```
## Cookies - 记得加上饼干

要获得高清源必须在访问API时加上Cookies，cURL要做到这点很简单。

测试得知判断的关键是Cookies的如下键值：
`DedeUserID=; DedeUserID__ckMd5=; SESSDATA=; bili_jct=`

F12打开浏览器，找到对应的键值，保存成Cookies的文本文件，然后修改代码：
```bash
cookies=`cat ./cookies`
curl -sL -H "Cookie: "$cookies
aria2c --header="Cookie: "$cookies
```
## 完整代码

```bash
#!/bin/bash
IFS=$'\n'
if [ $# -eq 0 ]
then
    echo -e "->Need AVID to download!\n"
    exit 1
fi
until [ $# -eq 0 ]
do
    aid=`echo $1 | sed -e 's/.*av//g' -e 's/[a-zA-Z?/].*//g'`
    cookies=`cat ./cookies`

    pagelist='https://api.bilibili.com/x/player/pagelist?aid='$aid'&jsonp=jsonp'
    echo -e "->Getting video list: \n"$pagelist
    cids=`curl -sL -H "Cookie: "$cookies $pagelist | jq -r '.data[].cid'`
    title=`curl -sL -H "Cookie: "$cookies 'https://api.bilibili.com/x/web-interface/view?aid='$aid | jq -r '.data.title' | sed 's/[/?!.*|:]//g'`
    cids_arr=($cids)
    echo -e "->Found video pages: "${#cids_arr[@]}

    part=0
    for cid in $cids
    do
        episode=$(( ++part ))
        echo -e "->Download video page: "$episode
        if [ "${#cids_arr[@]}" == "1" ]
        then
            filename=av$aid.$title.mp4
        else
            filename=av$aid.$title【P$episode】.mp4
        fi
        json_url='https://api.bilibili.com/x/player/playurl?avid='$aid'&cid='$cid'&qn=116&fnver=0&fnval=16&otype=json&type='
        echo -e "->Getting video source: \n"$json_url    
        json=`curl -sL -H "Cookie: "$cookies $json_url`
        dash=`echo $json | jq '.data|has("dash")'`
        durl=`echo $json | jq '.data|has("durl")'`
        
        if [ "$dash" == "true" ]
        then
            vp=`echo $json | jq -r '.data.dash.video[0].baseUrl'`
            ap=`echo $json | jq -r '.data.dash.audio[0].baseUrl'`

            echo -e "->Downloading Video Dash"
            aria2c -x10 -k1M --file-allocation=none --auto-file-renaming=false --allow-overwrite=true $vp\
                --show-console-readout false --quiet \
                --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0"\
                --header="Accept: */*"\
                --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"\
                --header="Referer: https://www.bilibili.com/video/av"$aid\
                --header="Origin: https://www.bilibili.com"\
                --header="DNT: 1"\
                --header="Connection: keep-alive"\
                --header="Pragma: no-cache"\
                --header="Cache-Control: no-cache"\
                --header="Cookie: "$cookies \
                --out ./v_$cid.m4s

            echo -e "->Downloading Audio Dash"
            aria2c -x10 -k1M --file-allocation=none --auto-file-renaming=false --allow-overwrite=true $ap\
                --show-console-readout false --quiet \
                --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0"\
                --header="Accept: */*"\
                --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"\
                --header="Referer: https://www.bilibili.com/video/av"$aid\
                --header="Origin: https://www.bilibili.com"\
                --header="DNT: 1"\
                --header="Connection: keep-alive"\
                --header="Pragma: no-cache"\
                --header="Cache-Control: no-cache"\
                --header="Cookie: "$cookies \
                --out ./a_$cid.m4s

            echo -e "->Merge into file: "$filename
            ffmpeg -i ./v_$cid.m4s -i ./a_$cid.m4s -c:v copy -c:a copy\
                -y -hide_banner -loglevel panic \
                ./$filename
            echo -e "->Removing temp files\n"
            rm *.m4s
        elif [ "$durl" == "true" ]
        then
            flvs=`echo $json | jq -r '.data.durl[].url'`
            for flv in $flvs
            do
                flvname=`echo $flv | sed -e 's/\?.*//g' -e 's/.*\///g'`
                echo "file './"$flvname"'" >./merge_$cid.txt
                echo "->Downloading Video Part: "$flvname
                aria2c -x10 -k1M --file-allocation=none --auto-file-renaming=false --allow-overwrite=true $flv\
                    --show-console-readout false --quiet \
                    --header="User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:65.0) Gecko/20100101 Firefox/65.0"\
                    --header="Accept: */*"\
                    --header="Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2"\
                    --header="Referer: https://www.bilibili.com/video/av"$aid\
                    --header="Origin: https://www.bilibili.com"\
                    --header="DNT: 1"\
                    --header="Connection: keep-alive"\
                    --header="Pragma: no-cache" --header="Cache-Control: no-cache"\
                    --header="Cookie: "$cookies \
                    --out ./$flvname
            done
            echo "->Merge into file: "$filename
            ffmpeg -safe 0 -f concat -i ./merge_$cid.txt -c copy\
                -y -hide_banner -loglevel panic \
                ./$filename
            echo -e "->Removing temp files\n"
            rm *.flv
            rm ./merge_$cid.txt
        else
            echo -e "->Error: Video not found!\n"
        fi
    done
    shift
done

```

## EOF
清晰度选择我觉得就真的没必要了吧.jpg

但是jq毕竟是个第三方工具，难道真的不能用awk/sed来解析json吗？

可以，但没必要。

但今天就尝试一次，用awk和sed组合，替换掉jq的工作：

```bash
# 获取cids
cids=`curl -sL -H "Cookie: "$cookies $pagelist | sed 's/[{,}]/\n/g' | awk -F: '/"cid"/ {print $2}'`

# 获取视频标题
title=`curl -sL -H "Cookie: "$cookies 'https://api.bilibili.com/x/web-interface/view?aid='$aid | sed -e 's/[{}]//g' -e 's/,"/\n"/g' | awk -F: '/"title"/ {print $2}' | sed -e 's/"//g' -e 's/[/?!.*|:]/-/g'`

# 是否是DASH
dash=`echo $json | sed -e 's/[{}]//g' -e 's/,"/\n"/g' | awk '/"dash"/'`

# 是否是FLV
durl=`echo $json | sed -e 's/[{}]//g' -e 's/,"/\n"/g' | awk '/"durl"/'`

if [ "$dash" != "" ]
then
    # 视频 DASH
    vp=`echo $json | sed -e 's/[{}]//g' -e 's/,"/\n"/g' -e 's/\]//g' | awk '/"video"/,/"audio"/' | awk '/"baseUrl"/' | sed -e 's/"baseUrl"://g' -e 's/"//g' -e 's/\u0026/\&/g' -e 's/\\\\//g' | awk 'NR==1'`

    # 音频 DASH
    ap=`echo $json | sed -e 's/[{}]//g' -e 's/,"/\n"/g' -e 's/\]//g' | awk '/"audio"/,/""/' | awk '/"baseUrl"/' | sed -e 's/"baseUrl"://g' -e 's/"//g' -e 's/\u0026/\&/g' -e 's/\\\\//g' | awk 'NR==1'`

elif [ "$durl" != "" ]
then
    # FLV 视频
    flvs=`echo $json | sed -e 's/[{}]//g' -e 's/,"/\n"/g' -e 's/\]//g' | awk '/"url"/' | sed -e 's/"url"://g' -e 's/"//g' -e 's/\u0026/\&/g' -e 's/\\\\//g'`
else
    echo -e "->Error: Video not found!\n"
fi
```