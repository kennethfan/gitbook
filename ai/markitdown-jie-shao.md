---
description: 微软开源 MarkItDown：一键将文档变Markdown，AI训练神器来了！
---

# MarkItDown介绍

## 前言

> 还在为PDF、Word、PPT转Markdown发愁？微软官方工具免费开源，轻松提取文档文字、表格、图片甚至OCR！

最近AI圈又有一个“官方级”工具悄悄火了——微软开源了 **MarkItDown**。别看名字有点呆萌，它可是能把PDF、Word、PPT、Excel、图片（OCR）甚至HTML统统转换成Markdown格式的神器。

为什么Markdown这么重要？因为现在的AI大模型几乎都“吃”Markdown格式的数据——结构清晰、纯文本、易处理。但日常文档五花八门，转换一直是痛点。

现在，微软亲自下场解决了。本文带你3分钟看懂 MarkItDown 能做什么、怎么用。

***

### 一、MarkItDown 是什么？

**MarkItDown** 是微软开源的 Python 库，核心功能就一句话：**将各种办公文档转换成 Markdown 格式**。

它就像一台“文档粉碎机+重组机”——把 PDF 里的文字、Word 里的表格、PPT 里的幻灯片、Excel 里的数据，甚至图片里的文字（OCR）全部提取出来，整理成干干净净的 Markdown。

GitHub 地址：[microsoft/markitdown](https://github.com/microsoft/markitdown)（上线不到一周已获 2.5w+ star）

![项目首页截图](https://opengraph.githubassets.com/1/microsoft/markitdown)

***

### 二、为什么你需要它？

如果你是下面任意一种人，MarkItDown 会让你爱不释手：

* **AI 工程师/数据科学家**：需要把各种格式的文档喂给 LLM（大语言模型）
* **知识库搭建者**：想把企业文件库一键转成纯文本索引
* **普通办公族**：偶尔需要把 PDF 或 PPT 里的内容提取出来编辑
* **RAG 应用开发者**：需要高质量的结构化文档输入

过去我们可能要用 Pandoc、Adobe Acrobat、各种在线转换工具，要么收费、要么格式乱、要么不支持批量。MarkItDown 完全免费，而且由微软官方维护，质量和更新有保障。

***

### 三、支持哪些格式？（几乎全了）

| 文件类型       | 扩展名                   | 支持情况              |
| ---------- | --------------------- | ----------------- |
| PDF        | .pdf                  | ✅ 文字+表格           |
| Word       | .docx                 | ✅ 完整保留            |
| PowerPoint | .pptx                 | ✅ 每页输出            |
| Excel      | .xlsx                 | ✅ 表格转 Markdown 表格 |
| 图片         | .jpg/.png/.gif/.bmp 等 | ✅ OCR 提取文字        |
| HTML       | .html                 | ✅ 转纯文本            |
| CSV        | .csv                  | ✅ 转表格             |
| JSON       | .json                 | ✅ 格式化输出           |
| XML        | .xml                  | ✅ 文本提取            |
| 文本文件       | .txt                  | ✅ 直接读取            |
| ZIP        | .zip                  | ✅ 解压后依次处理         |
| EPUB 电子书   | .epub                 | ✅ 提取正文            |

> 表格是根据 README 整理的最新列表

***

### 四、安装与使用（只要两行代码）

#### 1. 安装

```bash
pip install markitdown
```

> 如果需要 OCR 图片功能，建议加装依赖：
>
> ```bash
> pip install markitdown[ocr]
> ```

#### 2. 命令行使用

```bash
markitdown 简历.pdf > 简历.md
markitdown 年终汇报.pptx > 汇报.md
markitdown 复杂表格.xlsx --output 表格.md
```

#### 3. 在 Python 中使用

```python
from markitdown import MarkItDown

md = MarkItDown()

# 转换 PDF
result = md.convert("document.pdf")
print(result.text_content)

# 批量转换
import glob
for file in glob.glob("docs/*.pdf"):
    result = md.convert(file)
    with open(f"{file}.md", "w") as f:
        f.write(result.text_content)
```

就这么简单。PDF 里的表格会变成 Markdown 表格，PPT 里的每一页会被 `---` 分隔，图片里的文字会被 OCR 识别出来。

***

### 五、与 Pandoc 对比：谁更胜一筹？

说到文档转换，很多人会想到 **Pandoc**——那个被称为“文档转换瑞士军刀”的老牌工具。那么 MarkItDown 和 Pandoc 有什么区别？该选哪个？

#### 对比一览表

| 对比维度               | MarkItDown        | Pandoc                |
| ------------------ | ----------------- | --------------------- |
| **开发者**            | 微软                | John MacFarlane（开源社区） |
| **定位**             | 专注 → Markdown     | 通用文档转换枢纽              |
| **支持格式数量**         | \~15 种输入          | **50+ 种输入/输出**        |
| **OCR 图片识别**       | ✅ 原生支持            | ❌ 不支持                 |
| **Excel 转表格**      | ✅ 自动转 Markdown 表格 | ⚠️ 需借助 CSV 中间格式       |
| **PPT 转 Markdown** | ✅ 每页保留为独立段落       | ⚠️ 功能有限               |
| **AI 友好度**         | ⭐⭐⭐⭐⭐ 专为 LLM 优化   | ⭐⭐⭐ 通用设计              |
| **Python API**     | ✅ 原生 Python 库     | ⚠️ 需调用子进程             |
| **双向转换**           | ❌ 仅输入→MD          | ✅ 任意格式互转              |
| **学习曲线**           | 极低（两行代码）          | 中等（需了解参数）             |

#### 核心差异解读

**1. 定位不同**

* **Pandoc**：想做“文档转换界的万能翻译”，支持 50+ 格式互转（Word ↔ PDF ↔ HTML ↔ EPUB ↔ LaTeX...），适合复杂出版流程。
* **MarkItDown**：只做一件事——**任何文档 → Markdown**，把它做到极致，特别适配 AI 数据预处理。

**2. 办公文档支持**

* Pandoc 对 Word/PPT/Excel 的支持依赖外部工具（如 LibreOffice），转换质量不稳定。
* MarkItDown 直接调用微软 Office 底层 API（Windows）或 Office 365 REST API，格式保真度更高。

**3. OCR 能力**

* Pandoc 完全无法处理扫描版 PDF 或图片中的文字。
* MarkItDown 集成 Tesseract OCR，能“读懂”图片里的字，这对处理老旧文档非常有用。

**4. 使用场景建议**

* **选 MarkItDown**：你主要需要把办公文档（Word/PPT/Excel/PDF/图片）转成 Markdown，用于 RAG、模型训练、知识库构建。
* **选 Pandoc**：你需要在多种格式之间来回转换（比如 MD → EPUB → PDF），或者写学术论文需要 LaTeX 与 Word 互转。

> 💡 **一句话总结**：MarkItDown 是“AI 时代的文档转换器”，Pandoc 是“文档转换界的万能工具箱”。两者可以共存，各取所长。

***

### 六、背后的原理（非技术可跳过）

MarkItDown 并不是自己解析所有格式，而是“借力打力”：

* **PDF**：调用 `pdfminer.six` 提取文本和布局
* **Word/PPT/Excel**：使用 `office365-rest-python-client` 或本地 COM 对象（Windows）读取
* **图片 OCR**：依赖 `pytesseract`（Tesseract OCR 引擎）
* **HTML**：使用 `beautifulsoup4` 清理标签

最后统一输出为标准 Markdown 格式。整个过程自动处理编码、表格对齐、分页符等问题。

***

### 七、常见问题 & 避坑指南

#### Q1：中文 PDF 会乱码吗？

如果 PDF 里的字体是嵌入且标准编码的，一般不会。但如果 PDF 是扫描件（图像型），需要开启 OCR 功能。

#### Q2：图片 OCR 需要额外安装什么？

* Windows：下载安装 Tesseract，并配置环境变量 `TESSERACT_CMD`
* macOS：`brew install tesseract`
* Linux：`sudo apt install tesseract-ocr`

#### Q3：能保持原文档的目录结构吗？

可以批量转换文件夹，它会依次输出每个文件的 Markdown，但不会自动生成目录索引。你可以写脚本合并。

#### Q4：MarkItDown 和 Pandoc 可以一起用吗？

当然可以！典型流程：MarkItDown 将复杂办公文档转成 Markdown → Pandoc 再将 Markdown 转成你需要的高质量 EPUB/PDF/LaTeX。

***

### 八、实际应用场景示例

#### 场景1：用 LLM 总结一份 100 页的年报

```python
with open("annual_report.pdf.md", "r") as f:
    content = f.read()
    
# 截取前 5000 字发给 ChatGPT/Claude
summary_prompt = f"请用中文总结以下内容的关键信息：\n{content[:5000]}"
```

#### 场景2：搭建企业知识库问答系统

将所有 Word/PDF 转成 Markdown 后，用向量数据库（如 Chroma、Weaviate）建立索引，就可以实现“问文档”功能。

#### 场景3：从图片表格中提取数据

先 OCR 成 Markdown 表格，再用 Python 解析为 DataFrame。

#### 场景4：混合工作流（Pandoc 兜底）

```bash
# MarkItDown 转复杂办公文档
markitdown 产品说明书.docx > temp.md

# Pandoc 转为精美 PDF（需安装 LaTeX）
pandoc temp.md -o 最终产品手册.pdf --pdf-engine=xelatex
```

***

### 九、总结

MarkItDown 是一款“小而美”的官方工具，它解决了 AI 时代一个非常实际的需求：**让非结构化文档变成 AI 友好的结构化文本**。

* ✅ 免费开源，微软官方出品
* ✅ 支持 10+ 常见文档格式
* ✅ 保留表格、页眉、列表等结构
* ✅ 提供 CLI 和 Python API 两种方式
* ✅ 支持 OCR 从图片中提取文字
* ✅ 与 Pandoc 互补，协同工作更强

如果你经常和文档打交道，或者正在搭建 RAG 应用，强烈建议把这个小工具收入你的工具箱。

**GitHub 地址**：https://github.com/microsoft/markitdown

**命令行一句安装**：

```bash
pip install markitdown
```
