---
layout:       post
title:        "python-telegram-bot 使用总结"
author:       "Hamza"
header-style: text
catalog:      true
tags:
    - python
    - bot
    - telegram
---

## 引言

最近使用 python 的 [telegram bot sdk](https://github.com/python-telegram-bot/python-telegram-bot) 写了几个机器人项目, 项目开发过程中遇到过一些问题, 也积累了一些经验, 在下面记录一下

## bot 的工作模式

### 非监听模式

非监听模式下, bot 和普通用户的表现比较类似, bot 可以借助 [`get_updates`](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html#telegram.Bot.get_updates) 接口来收发消息。

任何用户对 bot 发的消息, 或者bot 所在的群、频道中别的用户发送的消息, 会在 telegram 的服务器上以 [`update`](https://docs.python-telegram-bot.org/en/stable/telegram.update.html#telegram.Update) 的形式储存 24 小时, 调用上述接口可以获取服务器上储存的所有更新。

#### get_updates 接口参数简介

- **offset:** 是一个 `update_id`, 请求返回的消息的 `update_id` 均大于该值
- **limit:** 从 offset 开始, 返回的消息数上限
- **allowed_updates:** 筛选更新的类型, 包括普通文本消息更新、编辑消息的更新、频道消息更新、编辑频道消息更新……

### 监听模式

#### [长轮询模式](https://core.telegram.org/bots/api#getupdates)

即是定时轮询 `get_updates` 接口, 不停迭代更新 `update_id`, 从而达到(伪)实时获取更新的效果。

`python-telegram-bot` 模块中可以使用 [`Updater.start_polling`](https://docs.python-telegram-bot.org/en/stable/telegram.ext.updater.html?#telegram.ext.Updater.start_polling) 接口实现长轮询模式。

#### [webhook 模式](https://core.telegram.org/bots/api#getupdates)

初始化一个 webhook, 入参中传入目标 url, 这个 url 要对应一个服务端程序。当有用户给 bot 发消息(或者其他任何更新)时, telegram 会作为客户端把本次更新的数据序列化成 json 字符串, 作为 post data 去请求 url。

`python-telegram-bot` 模块中也有对 webhook 模式的支持, 即实现了上述目标 url 对应的服务端程序, 详见[这里](https://docs.python-telegram-bot.org/en/stable/telegram.ext.updater.html?#telegram.ext.Updater.start_webhook)

### Tips

#### multiple instances 的问题

在长轮询模式下, 同一个 bot_token 只能开启一个实例, 多个实例会报错: `Conflict: terminated by other getUpdates request; make sure that only one bot instance is running`, 也就意味着只能有一个服务端程序轮序获取更新。这种情况对负载均衡实现高并发很不友好, 一个可行的方案是采用 1+n 模式部署:

- 一台主节点 + n个运行相同业务逻辑程序的服务节点
- 主节点运行 bot 实例, 轮询获取更新, 把更新的数据和当前用户的数据分流到不同的服务节点
- 服务节点处理完数据后, 返回相应的 telegram message, 由主节点发送给用户

#### 两种监听模式之间的区别

telegram 服务端在 bot 和用户之间中转消息, 而大部分 bot 要主动响应用户的输入, 因此 bot 需要实时从 telegram 服务端获取更新。

webhook 模式实现了主动推送, 相对于长轮询模式不停发送请求获取更新的方式, 占用资源更少, 效率也更高。

## 框架细节

TODO