title: '[MISC] 俄罗斯套娃'
author: Kevin。
tags:
  - misc
categories:
  - ctf
date: 2020-05-14 23:45:00
img: /images/article-banner/elstw.jpg
---
给学弟们瞎出的一道misc题，为的就是~~为难~~鼓励他们

附上下载链接：[http://file.dsb.ink/俄罗斯套娃.zip](http://file.dsb.ink/俄罗斯套娃.zip)

## 包含的内容

对各类文件头的熟悉程度

使用编程辅助解题

修改png文件的宽高

## 1.解压缩包

先解几个发现有文件名有规律的往上加，可以写一个脚本先将套娃zip全部解压出来，由于最后一个压缩包解出来会报错，所以程序会自动停止运行

把```俄罗斯套娃.zip```重命名为```flag0.zip```

运行一下shell脚本

```bash
for i in `seq 0 999`
do
	unzip flag${i}.zip
	mv zip/* .
	rm flag${i}.zip
done;
```

## 2.查看二进制

最后解出来的flag1000.zip无法解压，所以肯定有错误，拖到winhex中查看，发现有类似png尾的符号，可以确定是一张倒过来的png

![png尾部](/images/20200515000347.png)

写一个python程序把图片二进制倒过来

```python
f = open('flag1000.zip','rb')
g = open('flag.png','wb')
g.write(f.read()[::-1])
g.close()
f.close()
```


## 3.修改png

MISC中常规套路，直接修改二进制，修改png头第一个字符和png的高得到flag

![未修改高的png](/images/20200515000958.png)

![修改高度](/images/20200515001143.png)
