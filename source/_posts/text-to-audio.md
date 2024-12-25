---
categories:
- - AI工具
date: 2024-06-04 13:53:55
description: 本文介绍文本转语音的两种方法，分别是在线工具类和代码调用方法。
id: '286'
tags:
- 文本转语音
title: 文本转语音不完全指南
---


本文介绍文本转语音的两种方法，分别是在线工具类和代码调用方法。

## 1.在线工具

这种方式使用最为简单便捷，但存在网站关闭等不稳定情况，下面推荐几个网站。

第一个网站：[https://www.text-to-speech.online/](https://www.text-to-speech.online/)，主界面如下： ![](https://www.bplan.top/2024-06-01.webp)   第二个网站：[http://www.ttsonline.cn/](http://www.ttsonline.cn/)，主界面如下： ![](https://www.bplan.top/2024-06-02.webp)   第三个网站：[https://cloudtts.com/u/index.html](https://cloudtts.com/u/index.html)，主界面如下： ![](https://www.bplan.top/2024-06-03.webp)

## 2.代码调用

这种方式需要一定的基础编程知识，但代码胜在比较灵活，可以和其他任务编排使用。这里我们使用开源库edge-tts来省去编写原始API调用的过程，开源库使用的是Microsoft Edge的在线文本转语音服务，其地址为：[https://github.com/rany2/edge-tts](https://github.com/rany2/edge-tts)。

使用前确保已经安装Python程序，然后使用pip安装库即可： `pip install edge-tts`

通过命令行简单使用： `edge-tts --text "Hello, world!" --write-media hello.mp3 --write-subtitles hello.vtt` 其中--text指定输入的文本，--write-media指定输出的音频文件。

可以查看支持的声音列表： `edge-tts --list-voices`

然后指定声音： `edge-tts --voice ar-EG-SalmaNeural --text "مرحبا كيف حالك؟" --write-media hello_in_arabic.mp3 --write-subtitles hello_in_arabic.vtt`

说明文档中还提供了更改速率、音量和音调的选项，可以按需使用。

我们还可以通过代码方式调用，代码样例在[https://github.com/rany2/edge-tts/tree/master/examples](https://github.com/rany2/edge-tts/tree/master/examples)，其中同步调用的方式如下：

```python
import edge_tts

TEXT = "Hello World!"         # 指定文本
VOICE = "en-GB-SoniaNeural"   # 指定声音
OUTPUT_FILE = "test.mp3"      # 指定输出文件

def main() -> None:
    """Main function"""
    communicate = edge_tts.Communicate(TEXT, VOICE)
    communicate.save_sync(OUTPUT_FILE)

if __name__ == "__main__":
    main()
```

将文件保存命名为main.py，在命令行运行`python main.py`即可生成音频文件。