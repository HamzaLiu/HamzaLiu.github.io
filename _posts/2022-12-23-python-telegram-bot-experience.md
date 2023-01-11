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

> 最近使用 python 的 [telegram bot sdk](https://github.com/python-telegram-bot/python-telegram-bot) 写了几个机器人项目，项目开发过程中遇到过一些问题，也积累了一些经验，记录一下。

## 概念介绍

### chat

telegram 中的实体，包括：supergroup（群组）、channel（频道）、机器人对话窗口（是的，对于 bot 而言这也是一种特殊的 chat）

### [update](https://docs.python-telegram-bot.org/en/stable/telegram.update.html#telegram.Update)

用户在使用 telegram 中产生的事件的抽象，是开发 bot 程序的核心，包括以下几类：

- **CALLBACK_QUERY：** 点击普通按钮
- **CHANNEL_POST：** 频道推送的消息
- **CHAT_JOIN_REQUEST：** 请求加入群组
- **CHAT_MEMBER：** 群组或频道中用户的权限发生变化
- **CHOSEN_INLINE_RESULT：** 选择机器人的内联命令返回的结果
- **EDITED_CHANNEL_POST：** 编辑频道中的消息
- **EDITED_MESSAGE：** 编辑群组和机器人窗口中的消息
- **INLINE_QUERY：** 编辑框中输入机器人的内联命令
- **MESSAGE：** 群组和机器人窗口中的普通消息，包括各种常见类型的消息，详见[这里](https://docs.python-telegram-bot.org/en/stable/telegram.message.html)
- **MY_CHAT_MEMBER：** 用户启用或禁用机器人
- **POLL：** 发起投票
- **POLL_ANSWER：** 参与投票
- **PRE_CHECKOUT_QUERY：** 点击支付按钮
- **SHIPPING_QUERY：** 订单请求

update 会在 telegram 的服务器储存 24 小时，在此期间可以通过 bot api 获取。

## bot 的工作模式

### 非监听模式

非监听模式下，bot 和普通用户的表现比较类似，不会一直监听 [chat](#chat) 中的消息。初始化[bot对象](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html)后，bot 可以使用 [get_updates](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html#telegram.Bot.get_updates) 接口来接收 [update](#update)，使用 [send_message](https://docs.python-telegram-bot.org/en/stable/telegram.bot.html?#telegram.Bot.send_message) 接口及相关的一些接口发送不同类型的消息。

机器人调用完 `get_updates` 接口后直接退出，退出后自然无法响应 `update`，因此非监听模式不适合用于开发和用户交互的机器人，适合用于开发类似**定时推送消息**功能的机器人。

#### get_updates 接口参数简介

- **offset：** 是一个 `update_id`，请求返回的消息的 `update_id` 均大于该值
- **limit：** 从 `offset` 开始，返回的消息数上限
- **allowed_updates：** 筛选更新的类型

### 监听模式

#### [长轮询模式](https://core.telegram.org/bots/api#getupdates)

即是定时轮询 `get_updates` 接口，不停迭代更新 `update_id`，从而达到(伪)实时获取更新的效果。python-telegram-bot 模块中可以使用 [start_polling](https://docs.python-telegram-bot.org/en/stable/telegram.ext.updater.html?#telegram.ext.Updater.start_polling) 接口实现长轮询模式。

#### [webhook 模式](https://core.telegram.org/bots/api#getupdates)

初始化一个 webhook，入参中传入目标 url，这个 url 要对应一个服务端程序。当有用户给 bot 发消息（或者其他任何更新）时，telegram 会作为客户端把本次更新的数据序列化成 json 字符串，作为 post data 去请求 url。

python-telegram-bot 模块中也有对 webhook 模式的支持，即实现了上述目标 url 对应的服务端程序，详见[这里](https://docs.python-telegram-bot.org/en/stable/telegram.ext.updater.html?#telegram.ext.Updater.start_webhook)。

### Tips

#### multiple instances 的问题

在长轮询模式下，同一个 bot_token 只能开启一个实例，多个实例会报错：`Conflict: terminated by other getUpdates request; make sure that only one bot instance is running`，也就意味着只能有一个服务端程序轮序获取更新。这种情况对负载均衡实现高并发很不友好，一个可行的方案是采用 1+n 模式部署:

- 一台主节点 + n个运行相同业务逻辑程序的服务节点
- 主节点运行 bot 实例，轮询获取更新，把更新的数据 push 到公共队列中
- 每一个服务节点以[非监听模式](#非监听模式)运行 bot，用该 bot 对象和公共队列初始化 [dispatcher](https://docs.python-telegram-bot.org/en/stable/telegram.ext.dispatcher.html)，给 dispatcher 绑定用于处理更新的 [handler](https://docs.python-telegram-bot.org/en/stable/telegram.ext.handler.html)，对用户的操作予以响应

#### 两种监听模式之间的区别

1. telegram 服务端在 bot 和用户之间中转消息，而大部分 bot 要主动响应用户的输入，因此 bot 需要实时从 telegram 服务端获取更新。webhook 模式实现了主动推送，相对于长轮询模式不停发送请求获取更新的方式，占用资源更少，效率也更高。

2. webhook 模式中，开启网络服务的主机需要作为服务端监听 telegram 服务器发送的 update 数据，因此主机需要有公网 ip，内网的主机要做内网穿透。长轮询模式中，主机充当的是客户端，因此无需有公网 ip。

## Handler 详解

对于通用的 bot 而言，工作流程总结为：处理 update，做出响应。python-telegram-bot 提供了 handler 的方式处理不同类型的更新。到目前（v13.15）为止，模块支持如下handler：

- **CallbackQueryHandler：** 对应 `CALLBACK_QUERY` update，可基于正则表达式捕获其中的 callback_data（创建按钮时指定的字符串），从而给不同的按钮绑定不同的处理函数
- **ChatJoinRequestHandler：** supergroup 开启入群审核的情况下，捕获 `CHAT_JOIN_REQUEST` update
- **ChatMemberHandler：** 捕获 [chat](#chat) 中的成员权限变化，对应 `MY_CHAT_MEMBER` 和 `CHAT_MEMBER` 两种 update
- **ChosenInlineResultHandler：** 对应 `CHOSEN_INLINE_RESULT` update，没用过
- **CommandHandler：** 捕获含有命令的消息，即是捕获 [entities](https://docs.python-telegram-bot.org/en/v20.0b0/telegram.message.html#telegram.Message.entities) 中包含 [bot command](https://docs.python-telegram-bot.org/en/v20.0b0/telegram.messageentity.html#telegram.MessageEntity.BOT_COMMAND) 的 message，对应 `Message` update
- **ConversationHandler：** 模块提供的一种特殊 handler，用于实现会话，[下文](#conversionhandler)会详细描述
- **InlineQueryHandler：** 对应 `InlineQuery` update，没用过
- **MessageHandler：** 对应 `Message` update，通过指定参数中的 filter 实现捕获不同类型的消息。由于 filter 的种类很多，因此该 handler 十分强大，[下文](#messagehandler)会详细描述
- **PollAnswerHandler：** 对应 `PollAnswer` update，没用过
- **PollHandler：** 对应 `Poll` update，没用过
- **PreCheckoutQueryHandler：** 对应 `PreCheckoutQuery` update，一般会在该 handler 中判断点击支付按钮的来源，防止错误处理乱七八糟的支付请求，结合附带 [Filters.successful_payment](https://docs.python-telegram-bot.org/en/stable/telegram.ext.filters.html?#telegram.ext.filters.Filters.successful_payment) 的 `MessageHandler` 可以实现支付功能
- **PrefixHandler：** 同时处理一批类似的命令，加强版 `CommandHandler` 吗？没用过，也许在某些场景下很省力气
- **ShippingQueryHandler：** 对应 `ShippingQuery` update，没用过
- **StringCommandHandler：** 和 `CommandHandler` 功能一致，只不过是处理手动加入 update queue 的 update（神奇的 handler，这次翻文档才发现）
- **StringRegexHandler：** 基于正则表达式处理 `Message` update，同样是处理手动加入 update queue 的 update
- **TypeHandler：** 给特定的 update 类型绑定处理函数，通过参数 `type` 传入 update 类型

### ChatMemberHandler

#### 权限的种类

成员在 chat 中的权限有以下几种：

> 对于 `MY_CHAT_MEMBER` 类型的 update，只可能出现 `MEMBER` 和 `KICKED` 两种权限，前者是启用机器人，后者是禁用机器人。而对于 `CHAT_MEMBER` 类型的 update，拥有全部的权限类型。

- ADMINISTRATOR：管理员，可以有零个或多个
- CREATOR：创建者，有且只能有一个
- KICKED：被踢出，即被 ban
- LEFT：用户主动退出
- MEMBER：普通成员，和其他权限不兼容，管理员和创建者不是 `MEMBER`
- RESTRICTED：权限没有全部开放。举例来说，群组的全局权限有三个，包括发消息、发送媒体文件、发起投票，某个群员的权限只有发消息和发送媒体文件，不能发起投票，那么这个群员的状态就是 `RESTRICTED`。**值得注意的是**，如果管理员的权限也不全的话，那么该管理员的权限是 `RESTRICTED` 而不是 `ADMINISTRATOR`

#### 什么时候权限会发生变化

权限之间的切换用状态图来表示可能比较直观，由于权限的变化符合人的普遍理解以及画图很麻烦，这里仅列出权限变化的时机，如下：

- 用户加入群组：LEFT -> MEMBER
- 用户主动离开群组：MEMBER/ADMINISTRATOR -> LEFT
- 设置管理员：MEMBER/RESTRICTED -> ADMINISTRATOR
- 取消管理员：ADMINISTRATOR -> MEMBER
- 管理员封禁群员：MEMBER/ADMINISTRATOR -> KICKED
- 在群管理窗口中解除封禁：KICKED -> LEFT
- 限制群员权限：MEMBER/ADMINISTRATOR -> RESTRICTED
- 解除对群员（在群里）的权限限制：RESTRICTED -> MEMBER
- 解除对群员（不在群里）的权限限制：RESTRICTED -> LEFT

#### update 格式

前文提到，`ChatMemberHandler` 用于捕获用户权限发生改变的事件，因此在更新中包含两类信息，一个是 `old_chat_member`，里面有该用户老的权限，另一个是 `new_chat_member`，里面有该用户新获得的权限。两者的类型都是 [ChatMember](https://docs.python-telegram-bot.org/en/stable/telegram.chatmember.html?#telegram.ChatMember)。

此外，上述两类数据在 update 的 chat_member（CHAT_MEMBER 类型）或 my_chat_member（MY_CHAT_MEMBER 类型）的成员变量中，这两个成员变量都是 [ChatMemberUpdated](https://docs.python-telegram-bot.org/en/stable/telegram.chatmemberupdated.html) 类型。

#### 和 Filters.new_chat_members, Filters.left_chat_member 的区别

除了 `ChatMemberHandler` 外，`MessageHandler` 搭配 `Filters.new_chat_members` 和 `Filters.left_chat_member` 也能监测到用户加群或者退群的事件。假如有一个需求，要求在用户进群时发送祝福语，在退群时发送致谢辞，使用 `MessageHandler` 不用识别复杂且交织在一起的权限变化，是不是更优的选择呢？**是也不全是**。

为什么这样回答？究其原因，主要是 `MessageHandler` 监听的是 telegram 的消息，`Filters.new_chat_members` 对应的是当新用户加群或被邀请进群时，群组里显示的`xxx加入群组` 的提示消息。同样的，`Filters.left_chat_member` 对应的是当用户主动退群或者被踢出群组时，显示的 `xxx离开群组` 的提示消息。这就带来几个问题：

- channel 是没有这种提示消息的，所以机器人无法捕获用户关注频道的提示
- 如果群组里正好有另一个删除进群退群提示消息的机器人，那么我们的机器人的监测就完全失效了。在进群验证的情境下，这种失效是致命的（用户不经验证直接入群）
- telegram 发疯，用户进群退群没有提示消息（我就遇到过很多次，debug 自己的代码很难找到原因，令人崩溃）

总结下来，使用 `ChatMemberHandler` 比使用 `MessageHandler` 更稳定，同时也更繁琐。

### ConversionHandler

TODO

### MessageHandler

TODO
