---
description: DeepSeek++ 完整指南：从了解到上手。
---

# DeepSeek++指南

> 还在手动复制粘贴 DeepSeek 的回答？那是你用它的方式不对。

![DeepSeek++ 项目 Logo](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/readme-header.png)

***

### 一、先看效果

想象一下这个场景——

你在 DeepSeek 网页上问了一个问题，它不仅能回答，还能**自动联网搜索最新信息**、**读取你指定的网页内容**、**调用本机 Office 工具检查和修改文档**，甚至**记住你的偏好并在下次对话中自动复用**。

整个过程不需要你复制任何东西、不需要切换页面、不需要手动执行任何工具。

这就是 **DeepSeek++** 带来的体验。

一个开源的浏览器扩展，把你的 DeepSeek 网页版升级成真正的 **AI Agent 工作台**。

***

### 二、为什么需要 DeepSeek++？

用过 DeepSeek 的朋友都知道，它的推理能力确实很强，尤其是 DeepSeek R1，在数学、编程、逻辑推理方面表现出色。

但有一个问题：**它只是一个聊天窗口**。

* 它不能联网搜索实时信息
* 它不能读取你指定的网页
* 它无法调用外部工具
* 它记不住你的偏好设置
* 它不能定时执行任务
* 它的对话无法方便地导出归档

每次你需要这些能力，就得手动复制粘贴、切换工具、重新说明上下文。效率低，体验割裂。

**DeepSeek++ 就是为了解决这些问题而生的。**

***

### 三、八大核心功能一览

#### 1️⃣ 侧边栏对话 + 右键发送

安装后，DeepSeek 页面会多出一个侧边栏。你可以在侧边栏直接发起新对话，互不干扰。在任意网页选中文本，右键一键发送到 DeepSeek 进行解释、总结或改写。

配置 API Key 后，**侧边栏对话在普通网页也能用**，不依赖 DeepSeek 登录态。

#### 2️⃣ 联网搜索 & 网页获取

需要查实时信息？DeepSeek++ 内置 `web_search` 和 `web_fetch` 工具。

![类原生工具调用效果](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/screenshot-inline-tools.svg) _工具调用结果以原生折叠区块展示，清爽不干扰_

模型在需要时会自动搜索互联网，读取指定网页内容，把结果带回对话继续生成。

搜索完成后，结果自动回传，模型继续整理最终回答——**搜索不再是中断，而是流程的一部分**。

#### 3️⃣ MCP 工具系统

这是 DeepSeek++ 最强大的能力之一。MCP（Model Context Protocol）是 Anthropic 提出的 AI 工具标准化协议。

![MCP 工具管理侧边栏](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/screenshot-sidepanel-mcp.svg) _侧边栏 MCP 页面：管理工具、授权、测试连接_

DeepSeek++ 支持接入任意 MCP 服务——远程的、本机的，都可以。

你可以在侧边栏管理 MCP 服务、查看工具状态、授权权限。工具执行结果自动回传，支持**多步连续执行**，模型像 Claude Code 一样根据结果决定下一步。

#### 4️⃣ 长期记忆系统

DeepSeek++ 会自动识别对话中的关键信息，保存为长期记忆。

![记忆系统侧边栏](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/screenshot-sidepanel-memory.png) _长期记忆管理：自动保存、智能注入、四种类型_

下次对话时，根据关键词匹配、置顶权重、访问频率自动筛选注入。

四种记忆类型：

* **用户画像**：你的偏好和背景
* **行为反馈**：交互习惯
* **话题上下文**：重要话题信息
* **参考资料**：常用文档和链接

支持侧边栏管理、按类型筛选、JSON 导入导出。

#### 5️⃣ Skill 技能系统

预置多组开箱即用的技能模板。

![技能管理侧边栏](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/screenshot-sidepanel-skill.png) _技能系统：内置 + 自定义 + GitHub 导入_

![/ 触发技能面板](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/screenshot-skill-popup.png) _在聊天框输入 / 一键唤起技能选择_

你可以在侧边栏创建自定义技能，也可以从 GitHub 仓库、目录甚至单个 SKILL.md 文件导入第三方技能。

在聊天框输入 `/` 即可弹出技能面板，选择后自动注入对应的系统提示词。

#### 6️⃣ 对话导出

DeepSeek 回复下方工具栏新增了导出按钮。支持导出 HTML、Markdown、PDF 三种格式。默认隐藏内部提示和工具调用标记，导出文件干净可读。

文件通过浏览器本地下载保存，**不上传到任何服务器**。

#### 7️⃣ 自动化任务

需要让 DeepSeek 定时处理某个任务？在侧边栏创建自动化任务，支持：

![自动化任务侧边栏](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/screenshot-sidepanel-automation.svg) _自动化任务：手动触发 / cron / RRULE 定时调度_

* **立即运行**：手动触发
* **定时调度**：cron 表达式或 RRULE
* **独立会话**：每个任务有专属会话，后续运行复用
* **状态追踪**：下次运行、上次运行、状态和错误信息

最小间隔 15 分钟。

#### 8️⃣ 悬浮宠物 🐋

DeepSeek 页面会出现一条 **"DeepSeek 小鲸鱼"**。

![DeepSeek 小鲸鱼悬浮宠物](https://raw.githubusercontent.com/zhu1090093659/deepseek-pp/main/assets/yuansheng.jpg) _可爱的 DeepSeek 小鲸鱼，陪伴你每一次思考_

它会跟随思考、输出、工具执行状态切换反馈，显示可爱的台词气泡。支持固定在左下或右下，也可以自由拖动，还能调整尺寸、透明度和动态漂浮效果。

***

### 四、安装方法

浏览器打开地址[https://chromewebstore.google.com/detail/deepseek++/kdmpkkahkhdmdhfkdihkopikgcocbpbf?hl=zh-CN\&utm\_source=ext\_sidebar](https://chromewebstore.google.com/detail/deepseek++/kdmpkkahkhdmdhfkdihkopikgcocbpbf?hl=zh-CN\&utm_source=ext_sidebar)

### 五、理解 MCP 体系

在开始动手配置之前，花两分钟理解 MCP——这将帮助你更好地使用。

**MCP（Model Context Protocol）** 是 Anthropic 提出的 AI 工具标准化协议。简单理解就是：**让 AI 能够调用外部工具的"通用语言"**。

通过 MCP，DeepSeek 可以：

* 操作本机文件
* 执行命令行工具
* 调用远程 API
* 处理 Office 文档
* 运行 Python 脚本

MCP 采用 **客户端-服务器架构**：

```
┌─────────────────────┐         MCP 协议        ┌──────────────────────┐
│  DeepSeek++ 扩展     │ ◄──────────────────────► │   MCP Server        │
│  (MCP Client)        │    JSON-RPC 2.0          │   (工具提供者)       │
└─────────────────────┘                           └──────────────────────┘
```

DeepSeek++ 作为 MCP Client，连接到各种 MCP Server。每个 Server 提供一组工具（Tools）。

在侧边栏中，**MCP 页面**是所有工具的管理中心：

* **添加服务**：连接远程或本机的 MCP 服务
* **权限管理**：按服务或单个工具控制执行权限（自动/手动）
* **健康检查**：测试连接状态
* **工具列表**：查看每个服务提供的工具
* **执行历史**：查看调用记录

> 🔒 **安全提示**：MCP 配置和密钥**保存在浏览器本地**，WebDAV 同步不会同步敏感信息。

***

### 六、配置 Shell MCP（本机命令执行）

Shell MCP 是 DeepSeek++ 最常用的本机工具通道。通过它，DeepSeek 可以执行本机命令、操作文件、调用 OfficeCLI 等。

#### Step 1：安装 Shell Native Host

Shell Native Host 是一个本机程序，作为浏览器和操作系统之间的桥梁：

```bash
npx deepseek-pp-shell-host install --browser chrome --extension-id <扩展ID>
```

> 💡 **如何获取扩展 ID？**
>
> 加载扩展后，在 `chrome://extensions/` 页面找到 DeepSeek++，开启"开发者模式"，就能看到一串 ID（如 `abcdefghijklmnopqrstuvwxyz123456`）。在侧边栏的 MCP 页面也会自动填入当前扩展 ID。

**各浏览器命令：**

| 浏览器     | 命令                                                                         |
| ------- | -------------------------------------------------------------------------- |
| Chrome  | `npx deepseek-pp-shell-host install --browser chrome --extension-id <ID>`  |
| Edge    | `npx deepseek-pp-shell-host install --browser edge --extension-id <ID>`    |
| Firefox | `npx deepseek-pp-shell-host install --browser firefox --extension-id <ID>` |

#### Step 2：验证安装

安装完成后：

1. **重启浏览器**
2. 打开 DeepSeek 网页，进入侧边栏 **MCP 页面**
3. 点击 **"Shell"** 创建预设
4. 点击 **"测试连接"**，确认状态正常
5. 点击 **"刷新工具"**，查看可用工具列表

如果一切正常，你应该能看到类似 `shell_exec`、`python_exec` 等工具。

#### Step 3：运行烟测验证

项目提供了专门的烟测脚本：

```bash
npm run smoke:shell
```

#### 常见问题排查

| 问题           | 可能原因           | 解决方法                          |
| ------------ | -------------- | ----------------------------- |
| 连接失败         | 扩展 ID 不匹配      | 在 MCP 页确认扩展 ID 并重新安装          |
| 工具列表为空       | Shell Host 未授权 | 检查 Native Host 配置是否正确         |
| Windows 中文乱码 | 编码问题           | 更新到 v0.6.2+，已修复 Windows 路径和编码 |
| 命令找不到        | PATH 环境变量      | 确保命令在系统 PATH 中                |

***

### 七、配置 OfficeCLI（Office 文档处理）

OfficeCLI 是 DeepSeek++ 内置的 Office 文档工具链。安装 Shell Host 时会**自动安装命令版 OfficeCLI**。

#### 内置技能

| 技能                           | 用途             |
| ---------------------------- | -------------- |
| `/officecli`                 | 通用 Office 文档处理 |
| `/officecli-docx`            | Word 文档操作      |
| `/officecli-xlsx`            | Excel 表格操作     |
| `/officecli-pptx`            | PPT 演示文稿操作     |
| `/officecli-styles`          | PPT 样式库        |
| `/officecli-pitch-deck`      | 商业计划书模板        |
| `/officecli-academic-paper`  | 学术论文模板         |
| `/officecli-financial-model` | 财务模型模板         |
| `/officecli-dashboard`       | 数据看板模板         |
| `/officecli-morph-ppt`       | PPT 风格迁移       |

#### 使用方式

在 DeepSeek 聊天框输入 `/officecli`，选择对应的技能。技能会自动：

1. **检查本机 OfficeCLI 二进制**是否包含 `view/get/set/batch/validate` 等脚本化命令
2. 如果是，使用本机命令版（更快，不走 hosted AI 额度）
3. 如果不是，提示你切换 OfficeCLI 二进制

#### 支持的命令

```
officecli create <file>          # 创建文档
officecli get <file> <field>     # 读取文档内容
officecli set <file> <field>     # 修改文档内容
officecli view <file>            # 预览文档
officecli batch <config>         # 批量处理
officecli validate <file>        # 验证文档
```

***

### 八、配置远程 MCP 服务

#### 添加远程 MCP 服务

1. 在侧边栏进入 **MCP 页面**
2. 点击 **"添加服务"**
3. 输入服务名称和 URL（MCP 服务端地址）
4. 如有需要，配置认证信息
5. 点击 **"测试连接"** 验证
6. 连接成功后，工具列表会自动加载

#### 服务管理

添加完成后，你可以：

* **切换执行模式**：默认自动执行，可改为手动确认
* **查看工具列表**：每个服务提供的工具一览
* **测试连接**：随时检查服务状态
* **刷新工具**：服务更新后重新加载工具列表
* **查看调用历史**：追踪每次工具调用

#### 安全建议

* MCP 配置和密钥**仅保存在浏览器本地**
* 为敏感服务设置手动执行模式
* 定期检查和清理不再使用的 MCP 服务

***

### 九、Agent 式持续执行

配置好 MCP 后，最值得体验的就是 DeepSeek++ 的 **Agent 式持续执行**——像 Claude Code 一样，模型可以根据工具结果继续决定下一步。

#### 工作原理

```
用户提问
    ↓
DeepSeek 生成回复 → 需要工具？ → 调用 MCP 工具
    ↓                          ↓
返回最终结果      工具结果回传 → DeepSeek 继续生成
                                        ↓
                              还需要工具？ → 循环
                                        ↓
                              任务完成，输出结果
```

#### 关键特性

1. **分步续跑**：MCP 工具结果会回传到同一会话继续生成，直到任务完成
2. **节奏控制**：连续请求之间自动留出间隔，减少长任务被平台校验打断
3. **Step 折叠区**：按 Step 展示执行过程，已完成步骤自动折叠
4. **刷新恢复**：页面刷新后仍能恢复最近执行过程和状态
5. **手动停止**：长任务可随时停止后续续跑

#### 实际场景举例

**场景：让 DeepSeek 分析一个 Excel 表格并生成报告**

```
用户：帮我分析本季度的销售数据，生成一份PPT报告

Step 1: DeepSeek 通过 Shell MCP 调用 officecli view data.xlsx
Step 2: 读取到数据后，分析销售趋势
Step 3: 调用 officecli create report.pptx 生成PPT
Step 4: 继续调用 officecli set 添加标题和图表
Step 5: 完成任务，输出分析摘要
```

整个过程自动完成，不需要你手动操作任何一步。

***

### 十、侧边栏对话与右键场景（v0.6.4+）

#### 启用侧边栏对话

1. 进入 **设置页**
2. 启用 **"侧边栏对话"**
3. 侧边栏会新增「对话」页

侧边栏对话支持：

* 独立新会话，不影响当前页面对话
* 流式展示回复
* 配置 **官方 API Key** 后，可在普通网页使用（不依赖 DeepSeek 登录态）

#### 配置右键场景

在设置页配置常用场景模板：

1. 添加场景名称和 Prompt 模板
2. 在任意网页选中文本 → 右键 → 发送到 DeepSeek
3. 也可以选择套用场景模板

适合快速解释、总结、改写页面内容。

#### Python 解释器（v0.6.4+）

Shell MCP 新增 `python_exec` 能力，侧边栏工具页提供清晰的启用、权限和状态管理。感谢社区贡献者 @IjalG 贡献此功能。

如果你是数据分析师或 Python 开发者，这个功能让你的 DeepSeek 可以直接运行 Python 代码并返回结果。

***

### 十一、完整配置检查清单

| 步骤 | 配置项                  | 验证方式                                     |
| -- | -------------------- | ---------------------------------------- |
| ✅  | 安装 DeepSeek++ 扩展     | 侧边栏正常显示                                  |
| ✅  | 安装 Shell Native Host | `npx deepseek-pp-shell-host install ...` |
| ✅  | 创建 Shell MCP 预设      | 测试连接成功                                   |
| ✅  | 刷新 MCP 工具列表          | 看到 `shell_exec` 等工具                      |
| ✅  | 运行烟测                 | `npm run smoke:shell` 通过                 |
| ✅  | 可选：配置 API Key        | 侧边栏对话在普通网页可用                             |
| ✅  | 可选：配置远程 MCP 服务       | 测试连接成功                                   |
| ✅  | 可选：启用 Python 解释器     | 工具页显示 `python_exec`                      |

***

### 十二、谁应该使用？

* **开发者**：需要 DeepSeek 配合工具完成编程任务，MCP + Agent 式续跑是生产力利器
* **研究者**：需要联网搜索和网页获取来辅助深度研究
* **内容创作者**：需要长期记忆来保持风格一致性，需要对话导出归档
* **办公人士**：通过 OfficeCLI 技能让 DeepSeek 处理 Word、Excel、PPT 文档
* **自动化爱好者**：想让 AI 定时执行固定任务

***
