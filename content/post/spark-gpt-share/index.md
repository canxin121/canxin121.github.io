---
title: SparkGPT 思路分析
description: Nonebot 框架下 Ai 语言模型 最佳实现
date: 2023-09-10 00:00:00+0000
categories:
    - Nonebot
    - SparkGPT
tags:
    - Ai
    - Nonebot
license: 
hidden: false
comments: false
---
## 项目地址 -> [Spark-GPT](https://github.com/canxin121/Spark-GPT)

## Nonebot框架
由于SparkGPT由Python开发, 在Python中运行, 而Python目前最佳的bot框架为Nonebot框架,所以 目前2.0.0版本深度绑定Nonebot框架进行开发.  

预计将在下个大版本 将SparkGPT独立实现协议, 再实现不同语言框架的协议对接  
## 配置管理
### webui
实现了`Nonebot_plugin_web_config`来是实现webui管理配置信息, 借助pydantic的`BaseModel`来实现序列化反序列化并持久储存和方便的存取. 

`Nonebot_plugin_web_config`提供了一个父类给其他插件,直接继承并填写相关注释和属性即可实现持久储存和web编辑, 并且提供了从`Nonebot_plugin_web_config`实时获取配置的方法,而无需重新反序列化.  

ToDo:  
预计将会实现pydantic -> json schema的转换用于通信, 并重写前端页面  

## 数据储存

### 用户会话数据储存
1. 使用pydantic的`BaseModel`方便 序列化和反序列化
2. 使用`Nonebot_plugin_bind`的统一id作为用户标志储存, 实现跨平台跨账户的数据共享
3. 所有的会话数据以json形式储存在以用户id命名的文件中, 单用户单数据, 保证数据的隔断性
### 预设,指令,模型数据
1. 继承自`Nonebot_plugin_web_config`提供的父类, 直接实现了持久储存和webui编辑
2. 实现了一个装饰器, 方便的给每个子类生成一个从`Nonebot_plugin_web_config`获取数据的方法
 

## 消息事件处理
### 消息接受
1. 依赖`Nonebot_plugin_alconna`进行命令形式的消息事件匹配,分发给不同的函数进行处理
2. 实现`Nonebot_plugin_bind` 进行不同账户(可跨平台)信息的绑定, 将同一人的所有聊天平台的数据统一起来
3. 实现了会话的持久储存和一个从信息中获取会话依赖注入, 可以从一条信息中获取用户私有的或公有的会话, 分发给不同模型的Chatbot处理

### 回复生成
#### 模型回复
1. 实现`BaseChatBot`父类, 实现使用 异步生成器 实现 流式发送(每次发送消息的两段左右,如果平台支持编辑消息,那么直接加到原来的消息后面,否则发送新的消息) 和 一次性发送(可以自适应长度文转图转链接, 也可以强制设定使用文字或图片回复), 这里其实就是消息发送的步骤.
2. 所有的`ChatBot`子类只需添加特有的属性(用于储存会话信息)和`BaseChatBot`父类要求的属性, 以及一个异步生成器方法(逆向或使用官方api)和一个刷新会话方法, 即可实现一个新的api的接入  
3. 涉及的实现的逆向工程的链接" [Async-Bing-Client](https://github.com/canxin121/Async-Bing-Client) ", " [Async-Poe-Client](https://github.com/canxin121/Async-Poe-Client) ", " [Async-Claude-Client](https://github.com/canxin121/Async-Claude-Client) "  
#### 普通回复
##### 菜单和帮助
1. 实现了Nonebot_plugin_templates, 提供一些模板和构造方法, 直接构造出菜单的html并用htmlrender渲染截图返回结果.
2. 实现了一个Menu类来将 文本菜单 和  Nonebot_plugin_templates生成的菜单同时实现, 并且缓存图片, 减小开销, 提高效率
##### 基本查看和管理
1. chat的list在`用户会话数据储存类`的基础上加了一个方法, 并借助Nonebot_plugin_templates生成 会话的列表图片并缓存和动态更新, 减少开销, 提升效率
2. prompt和command的图片回复在`Nonebot_plugin_web_config`的子类的基础上借助Nonebot_plugin_templates实现图片列表和具体展示
3. 其他文本形式的回复直接发送即可

### 消息发送
1. 依赖`Nonebot_plugin_saa`进行跨适配器(跨聊天平台)的发送消息,可以实现图文发送
2. 依赖`Nonebot_plugin_htmlrender`进行文转图, 实质上就是使用playwright使用我的自制模板进行前端渲染并截图
3. 使用 `dpaste.org` 的 逆向api 进行文转网络剪切板链接, 方便用户拿去回答和信息
