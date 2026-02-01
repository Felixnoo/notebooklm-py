# 让他人通过 API 调用你的 NotebookLM 笔记本教程

## 简介

本教程将详细介绍如何设置你的 NotebookLM 笔记本，使其可以被他人通过 API 调用和访问。通过这种方式，你可以与同事、朋友分享你的笔记本内容，甚至允许他们与笔记本进行交互。

## 前提条件

- 拥有 Google 账号并已使用 NotebookLM
- 安装了 Python 3.10 或更高版本
- 安装了 notebooklm-py 库（双方都需要）

## 第一部分：设置笔记本共享权限

### 步骤 1：安装必要工具

如果你还没有安装 notebooklm-py，请先安装：

```bash
pip3 install "notebooklm-py[browser]"
playwright install chromium
```

### 步骤 2：登录到你的 Google 账号

```bash
notebooklm login
```

这会打开一个浏览器窗口，让你登录到 Google 账号并授权访问 NotebookLM。

### 步骤 3：查看你的笔记本列表

```bash
notebooklm list
```

记录下你要共享的笔记本 ID。

### 步骤 4：设置共享权限

有两种方式可以设置共享权限：

#### 方式 A：设置为公开（任何人通过链接可访问）

```bash
# 设置为公开
notebooklm share public --enable --notebook <你的笔记本ID>

# 查看共享状态（获取共享链接）
notebooklm share status --notebook <你的笔记本ID>
```

#### 方式 B：添加特定用户为协作者

```bash
# 添加用户为查看者（只能查看和对话）
notebooklm share add user@example.com --permission viewer --notebook <你的笔记本ID>

# 或添加用户为编辑者（可以修改笔记本内容）
notebooklm share add user@example.com --permission editor --notebook <你的笔记本ID>
```

### 步骤 5：设置查看级别

```bash
# 设置为完整访问（可查看所有内容）
notebooklm share view-level full --notebook <你的笔记本ID>

# 或设置为仅聊天访问
notebooklm share view-level chat --notebook <你的笔记本ID>
```

## 第二部分：他人使用 API 访问你的笔记本

### 步骤 1：他人安装必要工具

他人需要安装：

```bash
pip3 install "notebooklm-py[browser]"
playwright install chromium
```

### 步骤 2：他人登录自己的 Google 账号

```bash
notebooklm login
```

### 步骤 3：他人使用你的笔记本 ID 进行操作

#### 查看笔记本信息

```bash
# 查看笔记本中的源
notebooklm list-sources --notebook <你的笔记本ID>

# 查看笔记本的共享状态
notebooklm share status --notebook <你的笔记本ID>
```

#### 与笔记本对话

```bash
# 直接指定笔记本 ID 进行提问
notebooklm ask --notebook <你的笔记本ID> "这个笔记本是关于什么的？"

# 或先设置默认笔记本，再提问
notebooklm use <你的笔记本ID>
notebooklm ask "总结一下主要内容"
```

#### 生成内容

```bash
# 生成音频播客
notebooklm generate audio --notebook <你的笔记本ID> "制作一个关于这个主题的播客"

# 生成视频概述
notebooklm generate video --notebook <你的笔记本ID> --style whiteboard

# 生成测验
notebooklm generate quiz --notebook <你的笔记本ID> --difficulty medium
```

#### 下载生成的内容

```bash
# 下载音频
notebooklm download audio ./podcast.mp3 --notebook <你的笔记本ID>

# 下载测验（Markdown 格式）
notebooklm download quiz --format markdown ./quiz.md --notebook <你的笔记本ID>
```

## 第三部分：使用 Python API 访问

### 示例 1：基本访问

```python
import asyncio
from notebooklm import NotebookLMClient

async def main():
    async with await NotebookLMClient.from_storage() as client:
        # 查看笔记本中的源
        sources = await client.sources.list("<你的笔记本ID>")
        print("源列表:")
        for source in sources:
            print(f"- {source.title} (ID: {source.id})")
        
        # 与笔记本对话
        result = await client.chat.ask("<你的笔记本ID>", "总结这个笔记本的内容")
        print("\n回答:")
        print(result.answer)

asyncio.run(main())
```

### 示例 2：生成内容

```python
import asyncio
from notebooklm import NotebookLMClient

async def main():
    async with await NotebookLMClient.from_storage() as client:
        # 生成音频播客
        status = await client.artifacts.generate_audio("<你的笔记本ID>", "制作一个有趣的播客")
        print(f"生成任务 ID: {status.task_id}")
        
        # 等待生成完成
        await client.artifacts.wait_for_completion("<你的笔记本ID>", status.task_id)
        
        # 下载生成的音频
        await client.artifacts.download_audio("<你的笔记本ID>", "./podcast.mp3")
        print("音频已下载到 ./podcast.mp3")

asyncio.run(main())
```

## 第四部分：权限说明

| 权限级别 | API 访问能力 |
|---------|------------|
| **公开链接** | 可查看、对话、生成内容 |
| **查看者** | 可查看、对话、生成内容 |
| **编辑者** | 可查看、对话、生成内容、修改笔记本（添加/删除源等） |

## 第五部分：注意事项和故障排除

### 常见问题

1. **认证失败**
   - 解决方案：重新运行 `notebooklm login`

2. **权限错误**
   - 症状：收到 "Permission denied" 或类似错误
   - 原因：公司账号可能有共享限制
   - 解决方案：尝试共享给特定用户，或使用个人账号

3. **API 调用失败**
   - 症状：收到 "API error" 或类似错误
   - 原因：可能是 API 速率限制或网络问题
   - 解决方案：等待一段时间后重试，或检查网络连接

4. **笔记本未找到**
   - 症状：收到 "Notebook not found" 错误
   - 原因：笔记本 ID 错误或权限不足
   - 解决方案：确认笔记本 ID 正确，且已正确设置共享权限

### 最佳实践

1. **共享前测试**：先与自己的另一个账号测试共享功能
2. **使用特定用户**：对于敏感内容，建议共享给特定用户而不是设置为公开
3. **定期检查权限**：定期查看谁有权访问你的笔记本
4. **使用环境变量**：在脚本中使用环境变量存储笔记本 ID，避免硬编码

## 第六部分：完整工作流程示例

### 你（笔记本所有者）的操作

```bash
# 1. 登录
notebooklm login

# 2. 查看笔记本
notebooklm list

# 3. 设置为公开
notebooklm share public --enable --notebook <笔记本ID>

# 4. 查看共享状态
notebooklm share status --notebook <笔记本ID>
```

### 他人（访问者）的操作

```bash
# 1. 登录
notebooklm login

# 2. 与笔记本对话
notebooklm ask --notebook <笔记本ID> "这个笔记本是关于什么的？"

# 3. 生成内容
notebooklm generate audio --notebook <笔记本ID> "制作播客"

# 4. 下载内容
notebooklm download audio ./podcast.mp3 --notebook <笔记本ID>
```

## 总结

通过本教程，你应该能够成功设置你的 NotebookLM 笔记本，使其可以被他人通过 API 调用和访问。这为团队协作、知识共享和内容创作提供了新的可能性。

如果你遇到任何问题，请参考故障排除部分，或查看 [notebooklm-py 文档](https://github.com/teng-lin/notebooklm-py/tree/main/docs) 获取更多帮助。