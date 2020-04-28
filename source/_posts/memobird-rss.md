---
title: RSS咕咕机
date: 2019-10-02 20:38:00
s: memobird-rss
categories:
- 爬虫
tags:
- memobird
- python
- rss
---

之前从同事那里二手买了个咕咕机，当玩具玩。享受了几天使用APP打印各种沙雕照片和表情包带来的乐趣后，它毫无意外的开始吃灰，就和桌子旁边的树莓派一样染上历史的颜色。

还是拿出来用用吧……

咕咕机APP里的一部分订阅号的确有点意思，比如订阅每天天气，早上8点自动打印一条天气内容之类的。那么拿来订阅微博更新好像也没什么不对的。

反正一天有12小时在公司，闲着也是闲着，不定时的来一段小纸条也不错。
<!-- more -->
## 开发者申请

为了做到这些，首先要去咕咕机的[开发者页面](http://open.memobird.cn)申请APPKEY。填写必要的信息然后提交申请就行了。等了一周左右申请就通过了。

拿到APPKEY之后，可以查阅官方给的文档，可用的API其实不多，更多还是要靠自己去生成内容。

[GitHub](https://github.com/Windfarer/pymobird)上也有其他开发者写的SDK，可以不用自己重复按照官方文档造轮子。

## RSS

打个比方，订阅[小狐狸](https://space.bilibili.com/332704117/dynamic)的B站动态。

熟练的打开小狐狸的空间，F12，刷新，过滤XHR，找到API接口`https://api.vc.bilibili.com/dynamic_svr/v1/dynamic_svr/space_history`。B站的动态有多种类型，需要根据类型判断爬取的内容。

`['data']['cards']['desc']['type']`的值表明了动态是何种类型，比如1是指转发，2是带图动态，4是纯文字动态，8是投稿视频推送。

在 `['data']['cards']['card']` 中是一串JSON数据，动态的文本内容根据上面不同的类型存在不同的键值里，比如`['dynamic']`、`['item']['content']`、`['item']['description']`这几个位置。

## 图片存储

只有文本内容的纸条太单调了，信息量也不够，还是需要能够同时打印图片内容。但图片打印会增加打印时间和下载时间，折衷选择只保存附带图片的第一张。

为了保持目录整洁，图片数据全部用sqlite3存储。

## 内容更新

为了避免重复打印浪费纸张，每次爬取数据时需要判断是否有新的内容。

建数据表，只存储必要的时间、日期、文字内容和图片数据，加上动态的ID作为主键来进行去重。

```sql
CREATE TABLE
IF
    NOT EXISTS 白上吹雪Official (
    day VARCHAR,
    _id VARCHAR PRIMARY KEY UNIQUE,
    msg VARCHAR,
    pic BLOB,
    title VARCHAR);
```

然后人工插入一条id为0的数据，用来记录是否获取到新内容，每次先单独查询这一行，有新数据再取出内容进行打印。

## POST请求打印

利用之前提到的SDK，将文字和图片的复合内容整合后一起POST给服务器。

### 小花样

为了区分普通的带图动态和视频投稿的推送，仿照动态的样式给视频投稿的图像加上了角标。同时因为咕咕机采用的是WiFi打印，为了减小文件体积，POST数据前对图片尺寸进行缩小。

```python
from pymobird import Content
from pymobird import Content

bird = SimplePymobird(appkey, device_id, user_id)
content = Content()
content.add_text('【{}】'.format(uname)) # 用户名
content.add_text(time.strftime('%Y-%m-%d %H:%M',time.gmtime(int(day)))) # 日期
content.add_text('https://t.bilibili.com/{}'.format(_id)) # 链接
content.add_text(msg) # 文字内容

from PIL import Image
from PIL import ImageDraw
from PIL import ImageFont
img = Image.open('peitu.jpg')
img.thumbnail((400, 400), Image.ANTIALIAS)
font = ImageFont.truetype('SourceHanSansCN-Regular.otf', 22)
draw = ImageDraw.Draw(img)
draw.rectangle(
    (268, 15, 368, 47), fill=(250, 115, 150), outline=(250, 140, 160))
draw.text((274, 14), '投稿视频', '#FFFFFF', font=font)
img.save('peitu.png', 'PNG')
content.add_image('peitu.png')
result = bird.print_multi_part_content(content)
```

## Crontab定时任务

开SSH把脚本扔进富有历史感的树莓派。

> crontab -e
> 
> \*/5 * * * * python3 bilibili_fubuki.py

## 完整代码

```python
#!/usr/bin/python
## -*- coding: utf-8 -*-

import json
import os
import re
import requests
import sqlite3
import time

## bilibili
uname = '白上吹雪Official'
uid = '332704117'

## memobird
appkey = ''
device_id = ''
user_id = ''

headers = {
    'User-Agent':
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:69.0) Gecko/20100101 Firefox/69.0',
    'Accept': 'application/json, text/plain, */*',
    'Accept-Language':
    'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
    'Origin': 'https://space.bilibili.com',
    'DNT': '1',
    'Connection': 'keep-alive',
    'Referer': 'https://space.bilibili.com/',
}

newfile = not os.path.exists('bilibili_{}.sqlite'.format(uname))
db = sqlite3.connect('bilibili_{}.sqlite'.format(uname), isolation_level=None)
cur = db.cursor()

cur.execute('''
CREATE TABLE
IF
    NOT EXISTS {} (
    day VARCHAR,
    _id VARCHAR PRIMARY KEY UNIQUE,
    msg VARCHAR,
    pic BLOB,
    title VARCHAR);'''.format(uname))

if newfile:
    cur.execute('''INSERT INTO {} VALUES ( ?, ?, ?, ?, ? );'''.format(uname),
                (None, '0', '1', None, None))


def bili_pic(link):
    file = requests.get(link, headers=headers)
    return file.content


def main():
    params = (
        ('visitor_uid', '0'),
        ('host_uid', '{}'.format(uid)),
        ('offset_dynamic_id', '0'),
    )

    response = requests.get(
        'https://api.vc.bilibili.com/dynamic_svr/v1/dynamic_svr/space_history',
        headers=headers,
        params=params)
    result = json.loads(response.content)
    if result.get('code') != 0:
        print(result.get('message'))
        return False

    for card in result['data']['cards']:
        card_time = card['desc']['timestamp']
        card_id = card['desc']['dynamic_id_str']
        card_type = card['desc']['type']
        data = json.loads(card['card'])
        if card_type == 8:
            title = data['title']
            pic = data['pic']
            text = data['dynamic']
        elif card_type == 4:
            title = None
            pic = None
            text = data['item']['content']
        elif card_type == 2:
            title = None
            if data['item'].get('pictures'):
                pic = data['item']['pictures'][0]['img_src']
            text = data['item']['description']
        elif card_type == 1:
            title = None
            pic = None
            text = data['item']['content']
        else:
            print(card_time, card_id, card_type)
            continue

        img = bili_pic(pic) if pic else None
        cur.execute(
            '''INSERT OR REPLACE INTO {} VALUES ( ?, ?, ?, ?, ? );'''.format(
                uname), (card_time, card_id, text, img, title))

    new = cur.execute(
        '''SELECT COUNT(_id) FROM {};'''.format(uname)).fetchone()
    old = cur.execute(
        '''SELECT msg FROM {} WHERE _id = 0;'''.format(uname)).fetchone()

    if int(new[0]) == int(old[0]):
        return False
    cur.execute('''UPDATE {} SET msg = ? WHERE _id = 0'''.format(uname),
                (str(new[0]), ))

    limit = int(new[0]) - int(old[0])
    if limit < 0:
        return False

    limit = limit if limit < 1 else 1

    from pymobird import Content
    from pymobird import SimplePymobird
    bird = SimplePymobird(ak=appkey, device_id=device_id, user_id=user_id)
    data = cur.execute(
        '''SELECT day, _id, msg, pic, title FROM {} ORDER BY _id DESC LIMIT {};'''
        .format(uname, limit)).fetchall()
    for dynamic in data[::-1]:
        day, _id, msg, pic, title = dynamic
        content = Content()
        content.add_text('【{}】'.format(uname))
        content.add_text(time.strftime('%Y-%m-%d %H:%M',time.gmtime(int(day))))
        content.add_text('https://t.bilibili.com/{}'.format(_id))
        content.add_text(msg)
        if pic:
            with open('{}.jpg'.format(uname), 'wb') as f:
                f.write(pic)
            img = Image.open('{}.jpg'.format(uname))
            img.thumbnail((400, 400), Image.ANTIALIAS)
            if title:
                from PIL import Image
                from PIL import ImageDraw
                from PIL import ImageFont
                content.add_text(title)
                font = ImageFont.truetype('SourceHanSansCN-Regular.otf', 22)
                draw = ImageDraw.Draw(img)
                draw.rectangle((268, 15, 368, 47),
                               fill=(250, 115, 150),
                               outline=(250, 140, 160))
                draw.text((274, 14), '投稿视频', '#FFFFFF', font=font)
            img.save('{}.png'.format(uname), 'PNG')
            content.add_image('{}.png'.format(uname))
        result = bird.print_multi_part_content(content)
        print(result)
        time.sleep(5)


if __name__ == '__main__':
    main()
```

## EOF
<center>{% asset_img "2019-10-02_20-38.gif" "gugugu" %}</center>