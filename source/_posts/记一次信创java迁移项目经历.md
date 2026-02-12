---
title: 记一次信创java迁移项目经历
date: 2026-02-12 16:44:43
categories:
 - Java
 - 开发
---

# 某项目的信创迁移避坑指南
这是一个和.Net后端与嵌入式系统通过HTTP协议通信的项目

仅记录代码迁移

上海信创技术栈`上海市卫生健康信息技术应用创新白皮书`
## 迁移要求
- 后端 Asp.Net Core 6.0 -> Java 17
- 接口不变，数据处理逻辑、数据接收格式、数据返回格式不变
- 后端连接数据库从MariaDB -> Dm8
<!-- more -->
## 案发描述
- 你拿到前同事的代码，要求完成上述迁移；由于是老项目翻新，__你被告知代码以外的资料已经找不到了__
- 你打开gitee仓库一看，代码是2024.08.16 09:40 提交到仓库的，如今master分支已经成为保护分支，__但你仍被告知代码不一定是最新的__
- 你隐约知道存在一个 __ToDesk节点__ 是模拟生产环境的，上面可能有额外的信息
  - 坏消息：ToDesk节点真有最新代码，是 __2024.08.16 13:45__ 修改的
  - 好消息：只是接口命名改动
- 这是一个纯被动的Http后端，不存在任何Websocket主动推送，迁移后的测试是一个问题
- 迁移完成后，你发现已有的唯一下位机是坏的；无法测试，但是项目生命周期不一定会因你而延长，迫使你主动介入嵌入式开发（略）

## 感谢来自DeepSeek的安慰
![](ds1.png)

__没办法，小公司老项目翻新，开发环境的复杂更多是反映在除了开发以外的方方面面，这也是末法之世的一个侧面__

![](ds2.png)


## 控制层解析
- `ClientController.cs`
  - `/Com/P1`的Post方法
    - 猜测：根据注释，这是客户端调用的，直接给下位机发指令
    - 正确答案：这是一个永远不会被调用的陷阱接口，不需要被迁移
- `DeviceController.cs`
  - `/Send/Get Get`
    - 正确答案：根据方法体，是测试用的 
  - `/Send/Post Post {}`
    - 正确答案：根据方法体，也是用来测试的
  - `/Send/Para?devID=123&data=456 Post {}` 
    - 注释里写道 `POST send/Para/?devID=101&data=E1A2` send居然是小写？但是接口名是大写
    - 猜测：这是一个参数上报信息的Post接口，接口到底大小写这块可以测试测出来
    - 正确答案：这是一个永远不会被调用的陷阱接口，任何猜测都是无用
  - ~~`/Send/Body Post  DataFormat`~~ -> `/api/AndroidApi/GetAntiSlipInfo Post  DataFormat`
    - 注释里写道 `URL后面为空，通过Body发送数据。Body里放JSON内容：{"DevID":"101","Data":"E1A2"}`
    - 猜测：这是一个参数上报信息的Post接口，请求体里的俩参数的命名都很明确了
    - 正确答案1：这确实是一个会被调用的接口，但它仍然是一个陷阱接口，因为最新代码里它的名称发生了变化
    - 正确答案2：这个接口看似用到过token鉴权，但实际上已经注释掉了，因此可以确认整个后端不存在任何鉴权
  - ~~`/Send/Login Post  UserLogin`~~ -> `/api/AndroidApi/Login Post  DataFormat`
    -  猜测：这是一个登录的Post接口，请求体是小驼峰的用户名密码，会返回一个由TokenHelper.cs生产的token
    - 正确答案：依然接口命名变更，依然伪装成投入使用的无效token逻辑
## 持久层简述
- `DBModel.cs` 定义了所有数据表，是大驼峰
  - 由于ToDesk节点存在MariaDB的表格，直接导出表结构，然后导入DM8即可
  - 然后使用mybatis-plus的代码生成器通过表生成代码<https://baomidou.com/guides/new-code-generator/> 
- `Helper.cs` 定义了所有上报数据的解析处理，以及一个周期为2秒的定时查询任务，使用了数据库的存取和更新
  - 将`Helper.java` 加上注解`@Component` 就可以在字段处自动注入持久层操作实例， 使用`@AutoWired`
  - 将`Helper.java` 和 `Application.java`启动类 加上注解`@EnableScheduling` 就可以在`Helper.java`的方法上注解 `@Scheduled(fixedDelay = 10000)` 表示为10秒一次的定时任务
- `TableAccess.cs` 定义了所有数据插入的sql语句，重点是增加数据以后，会对一个t_DevLatestDataInfo进行条件更新，只更新不为null的字段，数据库操作需要全部用MyBatis-Plus进行替换
## 日志记录
所有和写日志有关的方法全部删除

替换为在类定义上方使用注解`@Slf4j`

`log.info(字符串)`/`log.warn(字符串)`/`log.error(字符串)`

## 代码逻辑迁移
- `C# ObjectA.FieldB ` -> `Java Lombok支持的@Data注解 ObjectA.getFieldB()`
- `C# string ` -> `Java String`
- `C# int ` -> `Java Integer`
- `C# .ToString() ` -> `Java .toString()`
- `C# 字符串.Substring(起始索引, 长度)` -> `Java 字符串.substring(起始索引, 起始索引 + 长度)`
- `C# Convert.ToInt32(字符串, 基数)` -> `Java Integer.parseInt(字符串, 基数)`
- `C# 整型.ToString("X")` -> `Java Integer.toHexString(整型).toUpperCase()`
- `C# 字符串[i]` -> `Java 字符串.charAt(i)` java虽然是转成char，但是在有String表达式里会自动装箱成String的，无需额外转换
  
