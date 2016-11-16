title: about linux music

date: 2015-12-06 00:39:57

tags:
    - music
-----------

linux下音乐文件处理
-------------------

在[verycd](https://www.verycd.com)上下载的音乐很多都是采用音轨抓取的，linux下没有一个像样的音乐播放软件，比如[foobar2000]，不好较好的切割音乐，所以这里需要命令行来根据cue来进行文件的切割。

这里采用切割工具是shntool系列的工具，其中需要mac来对ape格式的音乐进行支持。

```
shntool split -t "%n.%p-%t" -f [*.cue] -o flac -i ape [*.ape]
```

-	-t 表示的是切割后的文件的名字定义
-	-f 指定切割的文件
-	-o 输出的文件格式
-	-i 输入的文件格式
-	最后指定音轨文件

详细内容请看shntool的帮助文档。
