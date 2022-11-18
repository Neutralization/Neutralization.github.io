---
title: 来一场圆周率的飞花令
date: 2019-11-26 16:21:04
categories:
    - 杂记
tags:
    - poetry
    - python
    - π
---

逛微博看到了一些有意思的东西：山东卫视《国学小名士》第三季的节目，进行了一场 π(圆周率) 的飞花令。

节目里选手们接到了小数点后 204 位，那，能不能更长一点？

<!-- more -->

## 数据储备

首先需要有足够多的古诗词储备，不然程序生成的时候耗光弹药就显得非常尴尬了。

非常感谢[chinese-poetry](https://github.com/chinese-poetry/chinese-poetry)这样的开源项目，积累了大量的已经结构化的古诗词数据。

这里只用宋词部分的数据。

## 算法逻辑

逻辑很简单粗暴：找出所有包含中文数字的词句进行汇总，然后按小数点后位数迭代 π 值随机取对应的诗词就行。

整理诗词数据：

```python
import json
import re
import os

filelist = [
    file for file in os.listdir('./json') if re.search(r'poet.song', file)
]

files = [
    json.load(open('./json/' + file, 'r', encoding='utf-8-sig'))
    for file in filelist
]

result = {
    '一': [],
    '二': [],
    '三': [],
    '四': [],
    '五': [],
    '六': [],
    '七': [],
    '八': [],
    '九': [],
    '零': []
}

for file in files:
    for x in file:
        for y in x['paragraphs']:
            for z in ['一', '二', '三', '四', '五', '六', '七', '八', '九', '零']:
            　   if re.search(z, y):
                     result[z] = result.get(z) + [
                         '{}\n\t\t\t\t{}/{}'.format(y, x['author'], x['title'])
                    ]

with open('step.json', 'w', encoding='utf-8-sig') as f:
    f.write(json.dumps(result, ensure_ascii=False))
```

## π 值计算

无脑套用了一下别人的代码：<http://z-rui.github.io/post/2015/06/pi-digits/>

```python
def pidigits():
    q, r, t, u, i = 1, 180, 60, 168, 2
    while True:
        y = r // t
        yield y
        q, r, t, u, i = 10 * q * i * (2 * i - 1), 10 * u * (
            q * (5 * i - 2) + r - y * t), t * u, u + 54 * (i + 1), i + 1
```

不停迭代 pidigits() 就能得到后一位小数数值。

## 开始飞花令

测试一下迭代圆周率 1000 位

```python
import json
import sys
from random import randint

poets = json.load(open('step.json', 'r', encoding='utf-8-sig'))

number = {
    1: '一',
    2: '二',
    3: '三',
    4: '四',
    5: '五',
    6: '六',
    7: '七',
    8: '八',
    9: '九',
    0: '零'
}

n = 0


def pidigits():
    q, r, t, u, i = 1, 180, 60, 168, 2
    while True:
        y = r // t
        yield y
        q, r, t, u, i = 10 * q * i * (2 * i - 1), 10 * u * (
            q * (5 * i - 2) + r - y * t), t * u, u + 54 * (i + 1), i + 1


while True:
    f = open('result.md', 'w', encoding='utf-8-sig')
    for num in pidigits():
        used = []
        new = poets[number[num]].pop(randint(0, len(poets[number[num]])))
        while new in used:
            new = poets[number[num]].pop(randint(0, len(poets[number[num]])))
        print(num, number[num], new)
        f.write('{}\t{}\t{}\n'.format(
            num, number[num],
            new.replace(number[num], '`{}`'.format(number[num]))))
        used = used.append(new)
        n += 1
        if n > 1000:
            f.close()
            sys.exit()
```

## EOF
