---
title: jojo，python的能力是有极限的
date: 2019-10-18 12:18:24
categories:
    - 杂记
tags:
    - bash
    - python
---

--> jojo，python 的能力是有极限的。

--> 我从短暂的人生当中学到一件事......

--> 越是玩弄代码，就越会发现 python 的不足......

--> 除非超越 python。

你到底想说什么？

--> 吔我 bash 啦！

<!-- more -->

## 起因

原因其实很简单，工作上需要使用后台导入些数据。

吭哧吭哧把原始数据整理完，搞成 csv 了，点个“导入”，给我来了个报错：

-   单次导入只支持 1000 条

崽啊，9102 年末了，我 csv 里面近百万条数据，你跟我说一次 1000？我不下班啦？

然而优秀的底层员工是不会抱怨这些事情的，行吧，我自己切表总行了吧。

## 尝试 python 切割

先用 python 瞎糊了一个：

```python
datas = {}
for n, line in enumerate(open('test.csv', 'r', encoding='utf-8-sig')):
    if datas.get(n//1000):
        datas[n//1000] = datas[n//1000] + [line]
    else:
        datas[n//1000] = [line]

for k,v in datas.items():
    with open('test-{}.csv'.format(k), 'w', encoding='utf-8-sig') as f:
        f.write(''.join(v))
```

凑合着用了，顺便再写了个脚本把切好的文件自动导入到后台，把工作做完了。

但是回过头了，就开始嫌弃 python 不够快了。

虽然很明显原因在于我（毕竟瞎糊凑合用），想看能不能更快一点。

于是想起了 bash，想起了 split 和 xargs。

## 换成 bash

要切割文件的话 split 的确很方便:

> split test.csv -l 1000 -d -a 3 test\_\_

`-d -a 3`指定了按 3 位数递增添加后缀，这样就能产生 test**001 ~ test**999 命名的切割好的文件。

但新的问题也来了，切割完需要全部加上文件扩展名：

> mv test\__001 test_-001.csv

但手工操作量还是太大，这时候我们就需要 xargs 了：

> ls | grep test\_\_ | xargs -n1 -I file mv file file.csv

`ls | grep test__`筛选了所有缺少扩展名的文件，管道命令传给 xargs 后，`-n1`代表一次接受一个参数，`-I file`告诉 xargs 后续命令中的`file`全部用参数值替换。

比如只有一个文件的时候，`ls | grep test__`只有一个结果`test__001`，xargs 接收到参数，将`file`替换成`test__001`，实际执行内容就是：

> mv test**001 test**001.csv

将他们组合一下：

> split -l 1000 test.csv -d -a 3 test** && ls | grep test** | xargs -n1 -I file mv file file.csv

回车执行，1 秒都不需要，爽死了。

## 人类是懒惰的动物

但这样还是需要自己手打需要切割的文件名，不够爽。尝试把代码再变长一点：

> ls | grep .csv$ | sed 's/.csv//g' | xargs -n1 -I file split -l 1000 file.csv -d -a 3 file** && ls | grep ** | xargs -n1 -I file mv file file.csv

以管道命令分割解释一下：

1. `ls` 列出当前目录下所有文件和子目录
2. `grep .csv$` 筛选扩展名是 csv 的所有文件
3. `sed 's/.csv//g'` 去除扩展名，只留文件名
4. `xargs -n1 -I file split -l 1000 file.csv -d -a 3 file__` 切割所有的文件
5. `ls` 列出当前目录下所有文件和子目录
6. `grep __` 筛选文件名包含`__`的文件（上一步切割好的文件）
7. `xargs -n1 -I file mv file file.csv` 加上扩展名

爽又死了。

## EOF

尝试在 windows 上（Cmder+Github）做类似的操作，发现远没有在虚拟机（Ubuntu）里执行快，垃圾微软.jpg
