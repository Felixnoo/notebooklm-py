# NotebookLM-Py 使用教程

## 本地电脑使用教程

### 准备工作

#### 步骤 1：检查 Python 版本
在 Mac 终端中运行：
```bash
python3 --version
```
确保输出显示 Python 3.10 或更高版本。

#### 步骤 2：安装 notebooklm-py 库
在终端中运行：
```bash
pip3 install "notebooklm-py[browser]"
```

#### 步骤 3：安装浏览器支持
在终端中运行：
```bash
playwright install chromium
```

#### 步骤 4：登录到 NotebookLM
在终端中运行：
```bash
notebooklm login
```
这会打开一个浏览器窗口，让你登录到 Google 账号并授权访问 NotebookLM。

### 基本使用（命令行方式）

#### 步骤 5：创建笔记本
```bash
notebooklm create "我的研究"
```

#### 步骤 6：添加源（例如 URL）
```bash
notebooklm use 笔记本ID  # 替换为上一步创建的笔记本ID
notebooklm source add "https://example.com"
```

#### 步骤 7：与源对话
```bash
notebooklm ask "总结这个内容"
```

#### 步骤 8：生成内容（例如音频播客）
```bash
notebooklm generate audio "制作一个有趣的播客" --wait
```

#### 步骤 9：下载生成的内容
```bash
notebooklm download audio ./播客.mp3
```

### 简单 Python 脚本

创建一个名为 `notebooklm_example.py` 的文件，内容如下：

```python
import asyncio
from notebooklm import NotebookLMClient

async def main():
    async with await NotebookLMClient.from_storage() as client:
        # 创建笔记本
        nb = await client.notebooks.create("我的项目")
        print(f"创建了笔记本: {nb.title} (ID: {nb.id})")
        
        # 添加一个 URL 源
        print("正在添加源...")
        await client.sources.add_url(nb.id, "https://en.wikipedia.org/wiki/Artificial_intelligence", wait=True)
        
        # 与源对话
        print("正在询问问题...")
        result = await client.chat.ask(nb.id, "什么是人工智能？简单解释一下")
        print("\n回答:")
        print(result.answer)

asyncio.run(main())
```

然后在终端中运行：
```bash
python3 notebooklm_example.py
```

## 在 AI Agent 中使用教程

### 准备工作

#### 步骤 1：安装 notebooklm-py 库
在终端中运行：
```bash
pip3 install "notebooklm-py[browser]"
```

#### 步骤 2：安装浏览器支持
在终端中运行：
```bash
playwright install chromium
```

#### 步骤 3：登录到 NotebookLM
在终端中运行：
```bash
notebooklm login
```
完成 Google 账号登录和授权。

#### 步骤 4：安装 Claude Code 技能
在终端中运行：
```bash
notebooklm skill install
```

### 在 Claude Code 中使用

#### 步骤 5：打开 Claude Code
访问 Claude Code 网站并登录。

#### 步骤 6：使用自然语言命令
在 Claude Code 中，你可以使用自然语言来控制 NotebookLM，例如：

**示例 1：创建笔记本并添加源**
```
/notebooklm create "研究项目"
/notebooklm use 研究项目
/notebooklm source add "https://example.com"
```

**示例 2：生成内容**
```
/notebooklm generate audio "制作一个关于人工智能的播客"
/notebooklm download audio ./ai_podcast.mp3
```

**示例 3：与源对话**
```
/notebooklm ask "总结这个网站的主要内容"
```

### 常见问题解决

1. **登录失败**：如果登录失败，尝试重新运行 `notebooklm login`。

2. **命令未找到**：如果出现 `command not found` 错误，可能是 Python 包路径问题。尝试使用 `python3 -m notebooklm` 代替 `notebooklm`。

3. **源添加失败**：确保 URL 可访问，且不是需要登录才能查看的内容。

4. **生成内容超时**：大型内容生成可能需要更长时间，使用 `--wait` 参数等待完成。

### 提示

- 对于复杂任务，可以将多个命令组合成一个脚本。
- 查看生成的内容时，直接在 Finder 中找到下载的文件并打开。
- 如果遇到问题，可以查看项目的 [故障排除文档](https://github.com/teng-lin/notebooklm-py/blob/main/docs/troubleshooting.md)。