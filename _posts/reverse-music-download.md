title: 一个无损音乐下载软件的逆向
author: Kevin。
tags:
  - 逆向
categories: []
date: 2020-06-03 00:12:20
---

吾爱上面发现个音乐软件，可以下载好多无损歌曲，正好也想下几首歌，于是反编译过来玩玩![软件界面](/images/QQ截图20200602234658.png)

查了壳是.net写的，好办多了

![查壳](/images/QQ截图20200602234842.png)

用dnSpy反编译之后发现有些地方被加密了，试着用魔改的de4dot脱一下壳，没想到成功了

![魔改的de4dot](/images/QQ截图20200602235203.png)

再放到dnspy里面，发现原来加密的函数已经被解开

![解密之后的模块](/images/QQ截图20200602235318.png)

调试了不少时间都没有找到获取链接的点，一开始是一直在纠结wymusic函数里面的内容，一直在下断点结果没有停下来，一直以为是这里，之前开发python网易云api用的就是这个链接，所以没多想

![就...很坑](/images/QQ截图20200602235652.png)

上了wireshark没有抓到包，推测是https发送的，于是打算从下载的函数入手

从Form1里面找到了CommonHelper.GetAnyUrl，在进去之后发现了DES加密
![des函数](/images/QQ截图20200603000219.png)

再由DES函数找到了KEY和IV

![key和iv](/images/QQ截图20200603000318.png)

直接下断点，经过调试拿到了url的格式，用python写一个解密程序成功获取无损音乐的地址

![调试太好用了(笑)](/images/QQ截图20200603000525.png)

```python
def getFLAC(mid):
    import requests
    from base64 import b64encode
    from Cryptodome.Cipher import DES
    headers = {
        "User-Agent":"Mozill/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36"
    }
    data = "wy_999_" + mid + ".flac"
    key = bytes([36, 32, 93, 156, 78, 112, 218, 32])
    iv = bytes([55, 183, 236, 79, 36, 99, 167, 56])
    generator = DES.new(key, DES.MODE_CBC, iv)
    pad = 8 - len(data) % 8
    pad_str = ""
    for _ in range(pad):
        pad_str = pad_str + chr(pad)
    result = b64encode(generator.encrypt((data + pad_str).encode()))
    return requests.get("https://114.67.65.49/api/?uid=42a8d585757a79984ad9bdc6e5d62a9b&uin=&type=url&key=" + b64encode(result).decode(),headers=headers,verify=False).text

print(getFLAC("493735012"))
```
