

# ---

**📘 DocuFix: AI 兼容性审计与增强套件 (v0.1.0)**

Project Code: docufix-core  
定位: The "Lighthouse" for AI Documentation.  
核心价值: 让旧文档在不迁移的情况下，只需一条命令，就能被 Cursor/Copilot/Claude 完美理解并操作。

## ---

**1\. 系统架构设计 (Architecture)**

我们采用 **"Pipeline" (管道)** 架构。数据流向明确，易于扩展。

* **Input Layer (输入层):** 支持 URL (在线文档) 或 Local Path (本地 Markdown/HTML 文件)。  
* **Parsing Layer (解析层):** 针对 Docusaurus, Sphinx, MkDocs 等主流框架的特定解析器（去除 Header/Footer 噪音）。  
* **Audit Engine (审计引擎 \- 核心壁垒):** 模拟 RAG 检索过程，计算“幻觉风险分”。  
* **Retrofit Layer (改造层):** 生成 llms.txt (被动索引) 和 mcp.json (主动 Agent 协议)。  
* **Presentation Layer (表现层):** CLI 终端报告 \+ CI/CD 阻断机制。

## ---

**2\. 核心模块详细设计 (Detailed Modules)**

### **2.1 目录结构 (Project Structure)**

Bash

docufix/  
├── src/  
│   ├── audit/           \# 审计核心算法  
│   │   ├── metrics.py   \# 评分维度 (Token密度, 代码完整性等)  
│   │   └── scorer.py    \# 总分计算器  
│   ├── parser/          \# 解析不同类型的文档  
│   │   ├── html\_parser.py  
│   │   └── markdown\_parser.py  
│   ├── generator/       \# 生成文件  
│   │   ├── llms\_txt.py  
│   │   └── mcp\_server.py \# \[差异化\] MCP 协议生成器  
│   ├── cli.py           \# Typer 入口  
│   └── main.py  
├── tests/  
├── pyproject.toml  
└── README.md

### **2.2 核心算法：审计引擎 (src/audit/scorer.py)**

这是你的**商业机密**。普通的工具只提取文本，你的工具会“模拟 AI 的大脑”。

**评分维度 (The DocuFix Score 0-100):**

1. **Chunk Health (切片健康度) \- 40分**  
   * *原理:* RAG 通常按 512 或 1024 Token 切分。  
   * *扣分项:* 如果 500 个 Token 内没有出现任何 Markdown 标题 (\#, \#\#)，扣 10 分。这意味着 AI 读到这里会丢失上下文（不知道这段话属于哪个章节）。  
2. **Code Context (代码上下文) \- 30分**  
   * *原理:* 开发者最看重代码。  
   * *扣分项:* 代码块 (\<pre\>) 超过 10 行但没有注释？扣分。代码块里缺少 import 语句（导致 AI 无法推断库名）？严重扣分。  
3. **Link Rot (链接腐烂) \- 20分**  
   * *原理:* AI 顺着链接爬取，遇到 404 会中断推理。  
   * *扣分项:* 每一个死链扣 2 分。  
4. **Metadata (元数据) \- 10分**  
   * *扣分项:* 缺少 description 或 keywords meta 标签。

### **2.3 差异化功能：MCP Server 生成器 (src/generator/mcp\_server.py)**

这是你打击竞争对手的武器。我们不仅生成静态文本，还生成 **Model Context Protocol (MCP)** 配置，让 Claude Desktop 可以直接挂载这个文档。

* **输出文件:** docufix-mcp.json  
* **内容逻辑:** 自动将文档的 OpenAPI (Swagger) 定义转换为 MCP Tools 格式。

## ---

**3\. 关键代码实施规范 (Implementation Specs)**

### **3.1 审计算法伪代码 (Audit Logic)**

Python

\# src/audit/metrics.py  
import tiktoken  
from bs4 import BeautifulSoup

class ContextAuditor:  
    def \_\_init\_\_(self):  
        self.encoder \= tiktoken.get\_encoding("cl100k\_base")  
        self.chunk\_size \= 512

    def audit\_chunking\_risk(self, text: str) \-\> list\[dict\]:  
        """  
        检测是否有长文本缺乏结构化标题，导致 RAG 丢失上下文。  
        """  
        tokens \= self.encoder.encode(text)  
        chunks \= \[tokens\[i:i+self.chunk\_size\] for i in range(0, len(tokens), self.chunk\_size)\]  
          
        risks \= \[\]  
        for idx, chunk in enumerate(chunks):  
            chunk\_text \= self.encoder.decode(chunk)  
            \# 如果这一块里没有标题 (markdown header)，视为高风险  
            if "\# " not in chunk\_text and "\#\# " not in chunk\_text:  
                risks.append({  
                    "chunk\_id": idx,  
                    "risk\_level": "HIGH",  
                    "reason": "Context Drift: No headers found in 512 token window."  
                })  
        return risks

    def audit\_code\_blocks(self, html\_content: str) \-\> int:  
        """  
        检测代码块是否包含必要的 import 语句  
        """  
        soup \= BeautifulSoup(html\_content, 'html.parser')  
        score \= 100  
        for code in soup.find\_all('code'):  
            text \= code.get\_text()  
            if len(text.split('\\n')) \> 5: \# 这是一个长代码块  
                \# 简单启发式：Python 代码应该有 import/from  
                if "import " not in text and "from " not in text:  
                    score \-= 5  
        return max(0, score)

### **3.2 CLI 交互设计 (Typer \+ Rich)**

用户体验必须像 npm audit 一样丝滑。

Python

\# src/cli.py  
import typer  
from rich.console import Console  
from rich.table import Table

app \= typer.Typer()  
console \= Console()

@app.command()  
def scan(url: str):  
    """  
    运行 AI 兼容性审计  
    """  
    console.print(f"\[bold blue\]DocuFix\[/\] Scanning {url} for AI-readability...", style="blink")  
      
    \# ... 调用审计逻辑 ...  
    score \= 72  \# 假设得分  
      
    \# 打印漂亮的报告表  
    table \= Table(title="Audit Report")  
    table.add\_column("Metric", style="cyan")  
    table.add\_column("Status", style="magenta")  
    table.add\_column("Score Impact", style="red")  
      
    table.add\_row("Chunking Structure", "⚠️ Poor Headers", "-15")  
    table.add\_row("Code Snippets", "✅ Healthy", "0")  
      
    console.print(table)  
    console.print(f"\\n\[bold yellow\]Final GEO Score: {score}/100\[/\]")  
      
    if score \< 80:  
        console.print("\[red\]Critical Issues Found\! AI models will hallucinate on this doc.\[/red\]")  
        console.print("Run \[bold green\]docufix fix\[/\] to generate patches.")

## ---

**4\. 开发冲刺计划 (Development Sprint Plan)**

利用 Antigravity 的 Agent 能力，我们把开发压缩到 7 天。

### **Day 1: 骨架与爬虫 (Infrastructure)**

* **任务:** 搭建项目，实现 URL \-\> Clean Text 的转换。  
* **难点:** 处理 Docusaurus 等单页应用 (SPA) 的动态加载。  
* **Antigravity Prompt:**"Create a robust scraper using playwright-python. It needs to visit a URL, wait for network idle, and extract the content within the \<article\> or \<main\> tags. Strip out \<nav\> and \<footer\>. Save the clean HTML to a temp file."

### **Day 2-3: 审计大脑 (The Brain)**

* **任务:** 实现 src/audit/scorer.py。  
* **核心:** 把上面提到的 ContextAuditor 逻辑写死。  
* **测试:** 找一个很烂的文档（比如没有任何标题的一大段文字）和一个很好的文档，确保分数有显著差异。

### **Day 4: 生成器 (The Fix)**

* **任务:** 简单的 llms.txt 生成。  
* **格式:** 遵循 [llmstxt.org](https://llmstxt.org/) 标准。

### **Day 5: 杀手锏 MCP (The Differentiator)**

* **任务:** 编写一个转换器，把文档里的 API 描述（如果能抓取到）转成简单的 JSON 描述。  
* **MVP:** 哪怕只是生成一个 JSON 文件，告诉 Claude "Visit this URL to search docs"，也比普通文档强。

### **Day 6-7: 包装与发布 (Launch)**

* **任务:** 编写 README.md，录制一个 GIF（展示终端里跑分的酷炫效果）。  
* **发布:** PyPI, GitHub。

## ---

**5\. 立即执行：你的第一条指令**

**请复制下面的指令到你的 Google Antigravity (或 Cursor) 的 Composer 中，开始构建项目骨架：**

Plaintext

Act as a Senior Python Architect. I want to build a CLI tool called "DocuFix".  
Purpose: Audit documentation websites for "AI-readability" (GEO).

Please generate the project structure and the following key files:

1\. \`pyproject.toml\`: Include dependencies \`typer\`, \`rich\`, \`beautifulsoup4\`, \`playwright\`, \`tiktoken\`.  
2\. \`src/cli.py\`: A basic Typer app with a \`scan(url)\` command. Use \`rich\` to print a placeholder score table.  
3\. \`src/audit/scorer.py\`: Create a class \`DocAuditor\`. Include a method \`check\_token\_density(text)\` using \`tiktoken\` to count tokens.  
4\. \`src/parser/web.py\`: A function stub \`fetch\_clean\_content(url)\` that simulates fetching a webpage.

Follow modern Python practices (type hinting, docstrings). Do not implement the full logic yet, just the scaffold.

