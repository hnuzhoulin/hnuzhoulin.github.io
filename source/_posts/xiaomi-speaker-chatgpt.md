---
title: 小米音响接入ChatGPT
date: 2023-09-23 17:55:56
updated: 2023/09/23 23:06:41
categories: tool
tags: [xiaomi,chatgpt]
description: 偶然看到有将小米音箱的小爱同学问答接入ChatGPT，实现小米音箱的更智能的回答，来实践一把。
---


# 引言

本文核心内容不是原创，相关项目都来自github，我这里只是记录一下实践过程，也从小白的视角来记录期间碰到的问题和解答，首先感谢原作者的项目[yihong0618/xiaogpt](https://github.com/yihong0618/xiaogpt)，还有更底层的小米智能设备的库[MiService](https://github.com/Yonsm/MiService)，有了这个库，能对小米智能设备有更多认识，也能实现更精准的控制。

# 实践过程

事后来看哈，相对于直接参照项目的README，不如参照这个issue[不用 root 使用小爱同学和 ChatGPT 交互折腾记](https://github.com/yihong0618/gitblog/issues/258)更精准。

## 拉取代码安装依赖

```shell
git clone https://github.com/yihong0618/xiaogpt.git
cd xiaogpt
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple  #比较多，用国内源
```



## 获取小米音响信息

### 获取音箱具体型号

```
export MI_USER="XXX"
export MI_PASS="XXX"  #这是音箱绑定的小米账户的用户名密码
# 前面安装依赖的时候会安装miservice_fork，如果只是想拿到小米的设备，直接用MiService项目也可以
micli list
[
..... #省略其他设备
  {
    "name": "小爱音箱Play（2019款）",
    "model": "xiaomi.wifispeaker.lx05",
    "did": "AAAAA",
    "token": "XXXXXX"
  },
  {
    "name": "小米小爱音箱Play 增强版",
    "model": "xiaomi.wifispeaker.l05c",
    "did": "BBBBB",
    "token": "XXXX"
  },
  {
    "name": "小爱触屏音箱Pro 8",
    "model": "xiaomi.wifispeaker.x08a",
    "did": "CCCCCCCC",
    "token": "XXXXX"
  }
]
```

- model就是设备的品牌、类别和型号，有了这个就可以获得这个设备所有可以读取设置和可以执行的操作，类似物联网常说的物模型，具体示例可以参见下一篇博文 {% post_link control-xiaomi-speaker-cmdline 命令行控制小米音响 %} 。
- 从上面的输出可以看到我这里有三个音箱，关注model的最后一节，不同型号可以执行的功能和操作方法不一样，在原作者的介绍中有这样一句话：**前已知 LX04、X10A 和 L05B L05C 可能需要使用 `--use_command`，否则可能会出现终端能输出GPT的回复但小爱同学不回答GPT的情况**，具体后面再讲
- did：就是设备的唯一ID，所有的操作都是需要先有did，原作者文档里面第一步就是拿到这个did



### 设置设备ID全局变量

设定了这个did变量，后续的所有操作默认就是操作这个设备了

```
export MI_DID="AAAAA"  # 我这里是举例，实际要跟前面的命令输出一致，
```



### 设置ChatGPT信息

这里我们使用ChatGPT哈，项目作者描述是支持多种大模型的，详情见：[支持的 AI 类型](https://github.com/yihong0618/xiaogpt#%E6%94%AF%E6%8C%81%E7%9A%84-ai-%E7%B1%BB%E5%9E%8B)

```
export OPENAI_API_KEY=${your_api_key}
```

因为默认访问ChatGPT是有地区限定的，为了能在家里的群晖上跑，又不想给它配科学上网，在后面的命令中就用了[zhile大佬的pandora项目](https://github.com/zhile-io/pandora)的域名（担心key泄露或者认为用这个会被封的自行准备科学上网的方法就行，就无需api_base参数），额外添加的参数为：`--api_base "https://ai.fakeopen.com/v1"`，默认到这里基本的配置就OK了。



### 运行

```
# 设置使用ChatGPT，自定义ChatGPT api接口地址的base地址，使用流模式，快速停掉小爱的回答
python3.10 ./xiaogpt.py --hardware lx05 --mute_xiaoai --stream --api_base "https://ai.fakeopen.com/v1" --use_chatgpt_ap
## 输出下面的文字表示成功运行
Running xiaogpt now, 用`帮我/请回答`开头来提问
或用`开始持续对话`开始持续对话

# 命令行模拟问答
root@dsm:~# micli action '{"did":"265611160","siid":5,"aiid":5,"in":["请回答今天天气如何","false"]}'
{
  "did": "265611160",
  "miid": 0,
  "siid": 5,
  "aiid": 5,
  "code": 0,
  "exe_time": 160,
  "net_cost": 44,
  "ot_cost": 19,
  "otlocalts": 1695488652393921,
  "oa_cost": -1,
  "_oa_rpc_cost": 226,
  "withLatency": 0
}

# 命令输出，表示拦截的很好。
--------------------
问题：今天天气如何？
小爱没回
以下是 CHATGPTAPI 的回答: 今天天气晴朗，温度适宜。阳光明媚，没有云层遮挡。气温大约在20℃左右，适合户外活动。风力较轻，不会影响出行。天气晴好，空气清新，适宜散步或进行户外运动。这样的天气让人心情愉悦，带来了一天的好心情。
回答完毕
```

**注意**：

- 上面的参数**hardware**的值是最前面执行list命令时model的最后一节，
- 前面提过，这个型号比较关键，我另一个**l05c**型号的音响在执行上述命令后，终端有输出ChatGPT的回答，但是音响并不会播放出来，另一个**x08a**没试
- **mute_xiaoai**参数源文档说是快速停掉小爱的回答，测试过十来次发现偶尔可以停掉，貌似是去获取音响的状态发现是小爱自己的回答就通过停止指令停止播放，有比没有好。
- **stream**感觉确实会快一点，不至于等ChatGPT返回所有回答内容后再播放，因为它其实比较啰嗦，一般话都挺多的，时间较长，容易让人误以为失败了
- 这里提一个小需求，如果ChatGPT超时或者确实报错了，可以播报一下”失败请重试“，要不然真的不知道是报错了还是在准备一大段废话中；另外也可以设置一下对应ChatGPT账号的custom prompt，力求精简
- 为了跟标准的小爱区分，需要ChatGPT回答的提问需要以**帮我/请回答**开头。



# 小结

其实看到这里，在回过头看确实蛮简单的，只是一开始看项目说明时比较懵，尤其是最前面说获取DID，我是先调到MiService项目，参照它的项目说明去操作才大概理解了如何获取小米智能设备的DID等信息，也大概理解了跟小米智能设备的交互套路。
