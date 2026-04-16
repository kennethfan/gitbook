# openclaw使用opencode模型

Claude Code 开启 KYC 验证，大陆用户几乎无法使用。本文教你用 OpenCode 免费模型 + OpenClaw 搭建自己的 AI Agent，无需翻墙，完全免费！

## 🚨 为什么需要这个教程？

最近，Claude Code 开始强制要求 KYC（身份验证），这对大陆开发者来说简直是噩梦：

* ❌ 需要国外手机号验证
* ❌ 需要信用卡绑定
* ❌ 大陆 IP 直接被拒
* ❌ 即使翻墙也可能被风控

**但是！AI Agent 的时代已经到来，我们不能因为这些问题就放弃。**

OpenClaw + OpenCode 提供了一个完美的替代方案：

* ✅ **完全免费**：OpenCode 提供免费模型
* ✅ **无需 KYC**：只需邮箱注册即可
* ✅ **大陆可用**：无需翻墙，直连访问
* ✅ **功能完整**：支持文件操作、代码执行、消息发送等

本教程将手把手教你搭建这套方案，让你在 AI Agent 浪潮中不掉队！

## 📖 OpenClaw 是什么？

OpenClaw 是一款开源的 AI 智能体（AI Agent）框架，能够真正帮你”干活”：

* 访问本地文件、运行程序
* 发送邮件、管理日历
* 帮你写代码、运营公众号
* 甚至可以替你在微信上回复消息

简单来说：**你只需要下达指令，它就会自动执行，全程无需你亲自动手。**

## 🚀 第一步：安装 OpenClaw

### 1.1 设置 npm 镜像站（国内用户必做）

```bash
npm config set registry https://registry.npmmirror.com
```

### 1.2 安装 OpenClaw

```shellscript
npm i -g openclaw@2026.4.9
```

### 1.3 验证安装成功

![](<../.gitbook/assets/Unknown image>)

看到版本号输出，说明安装成功！

## 🔑 第二步：申请 OpenCode API（无需 KYC！）

### 2.1 注册账号

访问 [https://opencode.ai](https://opencode.ai/) 完成注册

💡 **重点**：只需要邮箱注册，无需手机号、无需信用卡、无需 KYC！

### 2.2 生成 API Key

注册完成后，在控制台生成 API Key：

![](<../.gitbook/assets/Unknown image (1)>)

💡 **提示**：点击旁边的复制按钮，稍后配置时会用到

## 🤖 第三步：申请飞书机器人

### 3.1 创建应用

访问 [飞书开放平台](https://open.feishu.cn/app?lang=zh-CN)，选择「创建企业自建应用」：

![](<../.gitbook/assets/Unknown image (2)>)

### 3.2 填写应用信息

![](<../.gitbook/assets/Unknown image (3)>)

### 3.3 添加机器人能力

![](<../.gitbook/assets/Unknown image (4)>)

### 3.4 配置应用权限

在「权限管理」页面，点击「添加权限」：

![](<../.gitbook/assets/Unknown image (5)>)

选择「批量导入」，粘贴以下权限配置：

{% code overflow="wrap" %}
```json
{
  "scopes": {
    "tenant": [
      "application:application:self_manage",
      "auth:user_access_token:read",
      "base:app:copy",
      "base:app:create",
      "base:app:read",
      "base:app:update",
      "base:collaborator:create",
      "base:collaborator:delete",
      "base:collaborator:read",
      "base:dashboard:copy",
      "base:dashboard:create",
      "base:dashboard:delete",
      "base:dashboard:read",
      "base:dashboard:update",
      "base:field:create",
      "base:field:delete",
      "base:field:read",
      "base:field:update",
      "base:field_group:create",
      "base:form:create",
      "base:form:delete",
      "base:form:read",
      "base:form:update",
      "base:history:read",
      "base:record:create",
      "base:record:delete",
      "base:record:read",
      "base:record:retrieve",
      "base:record:update",
      "base:role:create",
      "base:role:delete",
      "base:role:read",
      "base:role:update",
      "base:table:create",
      "base:table:delete",
      "base:table:read",
      "base:table:update",
      "base:view:read",
      "base:view:write_only",
      "base:workflow:create",
      "base:workflow:delete",
      "base:workflow:read",
      "base:workflow:update",
      "base:workflow:write",
      "base:workspace:list",
      "bitable:app",
      "bitable:app:readonly",
      "board:whiteboard:node:create",
      "board:whiteboard:node:delete",
      "board:whiteboard:node:read",
      "board:whiteboard:node:update",
      "calendar:calendar",
      "calendar:calendar.acl:create",
      "calendar:calendar.acl:delete",
      "calendar:calendar.acl:read",
      "calendar:calendar.event:create",
      "calendar:calendar.event:delete",
      "calendar:calendar.event:read",
      "calendar:calendar.event:reply",
      "calendar:calendar.event:update",
      "calendar:calendar.free_busy:read",
      "calendar:calendar:create",
      "calendar:calendar:delete",
      "calendar:calendar:read",
      "calendar:calendar:readonly",
      "calendar:calendar:subscribe",
      "calendar:calendar:update",
      "calendar:exchange.bindings:create",
      "calendar:exchange.bindings:delete",
      "calendar:exchange.bindings:read",
      "calendar:settings.caldav:create",
      "calendar:settings.workhour:read",
      "calendar:time_off:create",
      "calendar:time_off:delete",
      "calendar:timeoff",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:contact.base:readonly",
      "contact:user.base:readonly",
      "contact:user.basic_profile:readonly",
      "contact:user.id:readonly",
      "docs:doc",
      "docs:doc:readonly",
      "docs:document.comment:create",
      "docs:document.comment:delete",
      "docs:document.comment:read",
      "docs:document.comment:update",
      "docs:document.comment:write_only",
      "docs:document.content:read",
      "docs:document.media:download",
      "docs:document.media:upload",
      "docs:document.subscription",
      "docs:document.subscription:read",
      "docs:document:copy",
      "docs:document:export",
      "docs:document:import",
      "docs:event.document_deleted:read",
      "docs:event.document_edited:read",
      "docs:event.document_opened:read",
      "docs:event:subscribe",
      "docs:permission.member",
      "docs:permission.member:auth",
      "docs:permission.member:create",
      "docs:permission.member:delete",
      "docs:permission.member:readonly",
      "docs:permission.member:retrieve",
      "docs:permission.member:transfer",
      "docs:permission.member:update",
      "docs:permission.setting",
      "docs:permission.setting:read",
      "docs:permission.setting:readonly",
      "docs:permission.setting:write_only",
      "docx:document",
      "docx:document.block:convert",
      "docx:document:create",
      "docx:document:readonly",
      "docx:document:write_only",
      "drive:drive",
      "drive:drive.metadata:readonly",
      "drive:drive.search:readonly",
      "drive:drive:readonly",
      "drive:drive:version",
      "drive:drive:version:readonly",
      "drive:export:readonly",
      "drive:file",
      "drive:file.like:readonly",
      "drive:file.meta.sec_label.read_only",
      "drive:file:download",
      "drive:file:readonly",
      "drive:file:upload",
      "drive:file:view_record:readonly",
      "im:app_feed_card:write",
      "im:chat:read",
      "im:chat:readonly",
      "im:chat:update",
      "im:datasync.feed_card.time_sensitive:write",
      "im:message.group_at_msg:readonly",
      "im:message.p2p_msg:readonly",
      "im:message.pins:read",
      "im:message.pins:write_only",
      "im:message.reactions:read",
      "im:message.reactions:write_only",
      "im:message:readonly",
      "im:message:recall",
      "im:message:send_as_bot",
      "im:message:send_multi_users",
      "im:message:send_sys_msg",
      "im:message:update",
      "im:resource",
      "minutes:minutes",
      "minutes:minutes.basic:read",
      "minutes:minutes.media:export",
      "minutes:minutes.statistics:read",
      "minutes:minutes.transcript:export",
      "minutes:minutes:readonly",
      "search:docs:read",
      "sheets:spreadsheet",
      "sheets:spreadsheet.meta:read",
      "sheets:spreadsheet.meta:write_only",
      "sheets:spreadsheet:create",
      "sheets:spreadsheet:read",
      "sheets:spreadsheet:readonly",
      "sheets:spreadsheet:write_only",
      "slides:presentation:create",
      "slides:presentation:read",
      "slides:presentation:update",
      "slides:presentation:write_only",
      "space:document.event:read",
      "space:document:delete",
      "space:document:move",
      "space:document:retrieve",
      "space:document:shortcut",
      "space:folder:create",
      "task:comment:read",
      "task:comment:readonly",
      "task:comment:write",
      "task:task",
      "task:task.event_update_tenant:readonly",
      "task:task.privilege:read",
      "task:task:read",
      "task:task:readonly",
      "task:task:write",
      "task:task:writeonly",
      "task:tasklist.privilege:read",
      "task:tasklist:read",
      "task:tasklist:write",
      "wiki:member:create",
      "wiki:member:retrieve",
      "wiki:member:update",
      "wiki:node:copy",
      "wiki:node:create",
      "wiki:node:move",
      "wiki:node:read",
      "wiki:node:retrieve",
      "wiki:node:update",
      "wiki:setting:read",
      "wiki:setting:write_only",
      "wiki:space:read",
      "wiki:space:retrieve",
      "wiki:space:write_only",
      "wiki:wiki",
      "wiki:wiki:readonly"
    ],
    "user": [
      "base:app:copy",
      "base:app:create",
      "base:app:read",
      "base:app:update",
      "base:collaborator:create",
      "base:collaborator:delete",
      "base:collaborator:read",
      "base:dashboard:copy",
      "base:dashboard:create",
      "base:dashboard:delete",
      "base:dashboard:read",
      "base:dashboard:update",
      "base:field:create",
      "base:field:delete",
      "base:field:read",
      "base:field:update",
      "base:field_group:create",
      "base:form:create",
      "base:form:delete",
      "base:form:read",
      "base:form:update",
      "base:history:read",
      "base:record:create",
      "base:record:delete",
      "base:record:read",
      "base:record:retrieve",
      "base:record:update",
      "base:role:create",
      "base:role:delete",
      "base:role:read",
      "base:role:update",
      "base:table:create",
      "base:table:delete",
      "base:table:read",
      "base:table:update",
      "base:view:read",
      "base:view:write_only",
      "base:workflow:create",
      "base:workflow:delete",
      "base:workflow:read",
      "base:workflow:update",
      "base:workflow:write",
      "base:workspace:list",
      "bitable:app",
      "bitable:app:readonly",
      "board:whiteboard:node:create",
      "board:whiteboard:node:delete",
      "board:whiteboard:node:read",
      "board:whiteboard:node:update",
      "calendar:calendar",
      "calendar:calendar.acl:create",
      "calendar:calendar.acl:delete",
      "calendar:calendar.acl:read",
      "calendar:calendar.event:create",
      "calendar:calendar.event:delete",
      "calendar:calendar.event:read",
      "calendar:calendar.event:reply",
      "calendar:calendar.event:update",
      "calendar:calendar.free_busy:read",
      "calendar:calendar:create",
      "calendar:calendar:delete",
      "calendar:calendar:read",
      "calendar:calendar:readonly",
      "calendar:calendar:subscribe",
      "calendar:calendar:update",
      "calendar:exchange.bindings:create",
      "calendar:exchange.bindings:delete",
      "calendar:exchange.bindings:read",
      "calendar:settings.caldav:create",
      "calendar:settings.workhour:read",
      "calendar:time_off:create",
      "calendar:time_off:delete",
      "cardkit:card:read",
      "cardkit:card:write",
      "cardkit:template:read",
      "contact:contact.base:readonly",
      "contact:user.base:readonly",
      "contact:user.basic_profile:readonly",
      "contact:user.employee_id:readonly",
      "contact:user.id:readonly",
      "contact:user:search",
      "docs:doc",
      "docs:doc:readonly",
      "docs:document.comment:create",
      "docs:document.comment:delete",
      "docs:document.comment:read",
      "docs:document.comment:update",
      "docs:document.comment:write_only",
      "docs:document.content:read",
      "docs:document.media:download",
      "docs:document.media:upload",
      "docs:document.subscription",
      "docs:document.subscription:read",
      "docs:document:copy",
      "docs:document:export",
      "docs:document:import",
      "docs:event.document_deleted:read",
      "docs:event.document_edited:read",
      "docs:event.document_opened:read",
      "docs:event:subscribe",
      "docs:permission.member",
      "docs:permission.member:auth",
      "docs:permission.member:create",
      "docs:permission.member:delete",
      "docs:permission.member:readonly",
      "docs:permission.member:retrieve",
      "docs:permission.member:transfer",
      "docs:permission.member:update",
      "docs:permission.setting",
      "docs:permission.setting:read",
      "docs:permission.setting:readonly",
      "docs:permission.setting:write_only",
      "docx:document",
      "docx:document.block:convert",
      "docx:document:create",
      "docx:document:readonly",
      "docx:document:write_only",
      "drive:drive",
      "drive:drive.metadata:readonly",
      "drive:drive.search:readonly",
      "drive:drive:readonly",
      "drive:drive:version",
      "drive:drive:version:readonly",
      "drive:export:readonly",
      "drive:file",
      "drive:file.like:readonly",
      "drive:file.meta.sec_label.read_only",
      "drive:file:download",
      "drive:file:readonly",
      "drive:file:upload",
      "drive:file:view_record:readonly",
      "im:chat",
      "im:chat.access_event.bot_p2p_chat:read",
      "im:chat.announcement:read",
      "im:chat.announcement:write_only",
      "im:chat.chat_pins:read",
      "im:chat.chat_pins:write_only",
      "im:chat.collab_plugins:read",
      "im:chat.collab_plugins:write_only",
      "im:chat.managers:write_only",
      "im:chat.members:read",
      "im:chat.members:write_only",
      "im:chat.moderation:read",
      "im:chat.tabs:read",
      "im:chat.tabs:write_only",
      "im:chat.top_notice:write_only",
      "im:chat:create_by_user",
      "im:chat:delete",
      "im:chat:moderation:write_only",
      "im:chat:read",
      "im:chat:readonly",
      "im:chat:update",
      "im:message",
      "im:message.group_msg:get_as_user",
      "im:message.p2p_msg:get_as_user",
      "im:message.pins:read",
      "im:message.pins:write_only",
      "im:message.reactions:read",
      "im:message.reactions:write_only",
      "im:message.send_as_user",
      "im:message.urgent.status:write",
      "im:message:readonly",
      "im:message:recall",
      "im:message:update",
      "offline_access",
      "search:docs:read",
      "search:message",
      "sheets:spreadsheet",
      "sheets:spreadsheet.meta:read",
      "sheets:spreadsheet.meta:write_only",
      "sheets:spreadsheet:create",
      "sheets:spreadsheet:read",
      "sheets:spreadsheet:readonly",
      "sheets:spreadsheet:write_only",
      "slides:presentation:create",
      "slides:presentation:read",
      "slides:presentation:update",
      "slides:presentation:write_only",
      "space:document.event:read",
      "space:document:delete",
      "space:document:move",
      "space:document:retrieve",
      "space:document:shortcut",
      "space:folder:create",
      "task:comment:read",
      "task:comment:readonly",
      "task:comment:write",
      "task:task",
      "task:task:read",
      "task:task:readonly",
      "task:task:write",
      "task:task:writeonly",
      "task:tasklist:read",
      "task:tasklist:write",
      "wiki:member:create",
      "wiki:member:retrieve",
      "wiki:member:update",
      "wiki:node:copy",
      "wiki:node:create",
      "wiki:node:move",
      "wiki:node:read",
      "wiki:node:retrieve",
      "wiki:node:update",
      "wiki:setting:read",
      "wiki:setting:write_only",
      "wiki:space:read",
      "wiki:space:retrieve",
      "wiki:space:write_only",
      "wiki:wiki",
      "wiki:wiki:readonly"
    ]
  }
}
```
{% endcode %}

💡 **简化提示**：上面是精简版权限配置。如需完整权限，可在权限管理页面手动选择「全部」。

配置完成后：

![](<../.gitbook/assets/Unknown image (6)>)

### 3.5 配置事件订阅

在「事件订阅」页面：

1. **订阅方式**：选择「长连接」

![](<../.gitbook/assets/Unknown image (7)>)

1. **添加事件**：点击「添加事件」，选择以下 4 个事件：

im.chat.member.bot.added\_v1 # 机器人被添加到群聊im.message.reaction.created\_v1 # 消息表情反应创建im.message.reaction.deleted\_v1 # 消息表情反应删除im.message.receive\_v1 # 接收消息

### 3.6 创建版本并发布

点击「创建版本」：

![](<../.gitbook/assets/Unknown image (8)>)

填写版本信息并发布：

![](<../.gitbook/assets/Unknown image (9)>)

发布后，飞书会收到推送通知，点击打开应用：

![](<../.gitbook/assets/Unknown image (10)>)

⚠️ **注意**：需要在飞书客户端中打开应用

打开后，你就可以和机器人聊天了：

![](<../.gitbook/assets/Unknown image (11)>)

### 3.7 获取 App ID 和 App Secret

在「凭证与基础信息」页面，复制以下信息：

![](<../.gitbook/assets/Unknown image (12)>)

## ⚙️ 第四步：配置 OpenClaw

### 4.1 启动配置向导

{% code overflow="wrap" %}
```shellscript
openclaw onboard --mode local
```
{% endcode %}

### 4.2 按提示完成配置

配置过程中会出现一系列选择项：

**前几步选择**： - ✅ yes（同意条款） - ✅ manual（手动配置） - ✅ local（本地模式） - ⏎ 默认（直接回车）

![](<../.gitbook/assets/Unknown image (13)>)

**关键配置项**：

1. **Model/auth provider**：选择 opencode
2. **OpenCode auth method**：选择 OpenCode Zen catalog

![](<../.gitbook/assets/Unknown image (14)>)

1. **输入 OpenCode API Key**：粘贴第二步获取的密钥
2. **Default model**：选择带 free 字样的免费模型，推荐 minima-m2.5-free

![](<../.gitbook/assets/Unknown image (15)>)

1. **Channels**：选择 feishu

![](<../.gitbook/assets/Unknown image (16)>)

![](<../.gitbook/assets/Unknown image (17)>)

### 4.3 飞书授权

根据版本不同，有两种授权方式：

**方式一：扫码授权** - 使用飞书客户端扫描二维码 - 选择「已有机器人」

**方式二：手动填写** - 输入第三步获取的 App ID 和 App Secret

### 4.4 安装 Gateway

{% code overflow="wrap" %}
```shellscript
openclaw gateway install
```
{% endcode %}

### 4.5 完成验证

如果使用手动填写方式，飞书机器人会发送一条验证命令：

![](<../.gitbook/assets/Unknown image (18)>)

复制该命令到终端执行，即可完成配置。

## ✅ 完成！开始你的 AI Agent 之旅

现在你可以在飞书中给机器人发送消息，体验 AI Agent 的强大功能了！

**相比 Claude Code，这套方案的优势**：

| 特性     | Claude Code | OpenClaw + OpenCode |
| ------ | ----------- | ------------------- |
| KYC 验证 | ✅ 必须        | ❌ 不需要               |
| 大陆可用   | ❌ 困难        | ✅ 完全支持              |
| 费用     | 💰 付费       | 🆓 免费               |
| 翻墙需求   | ✅ 需要        | ❌ 不需要               |
| 功能完整性  | ⭐⭐⭐⭐⭐       | ⭐⭐⭐⭐                |

## 💡 常见问题

### Q1: 安装失败怎么办？

确保已正确设置 npm 镜像站，或尝试使用代理。

### Q2: 机器人不回复消息？

检查事件订阅是否正确配置，确保 Gateway 已启动。

### Q3: 如何切换模型？

重新运行 openclaw onboard 命令，在配置向导中选择其他模型。

### Q4: OpenCode 免费模型够用吗？

对于日常使用完全够用！如果需要更强大的能力，OpenCode 也提供付费模型。

## 📚 相关链接

* [OpenClaw 官方文档](https://github.com/openclaw)
* [OpenCode 平台](https://opencode.ai/)
* [飞书开放平台](https://open.feishu.cn/)

**别让 KYC 阻挡你的 AI Agent 之路！立即开始搭建你的专属 AI 助手吧！**

**如有问题，欢迎留言交流。**
