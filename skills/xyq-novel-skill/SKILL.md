---
name: xyq-novel-skill
description: 使用 pippit-cli 的 novel 场景能力提交和查询漫剧创作任务。覆盖漫剧生成、续写、改写、剧情扩展、人物设定、分集草稿、世界观设定等创作场景。当用户要求创作漫剧、写漫剧剧本、续写故事、修改剧情、补充角色设定、查询漫剧任务进展，或提到 pippit-cli novel / 小云雀 novel 时触发。
user-invocable: true
metadata:
  {
    "openclaw":
      {
        "emoji": "📖",
        "requires":
          {
            "bins": ["pippit-cli"]
          }
      }
  }
---

# 小云雀漫剧创作

通过 `pippit-cli novel` 命令提交漫剧创作任务、上传参考文件，并查询任务进展。

漫剧场景面向剧情、人物、分集与画面化叙事创作，用户的原始需求通过 `--message` 发送给后端 Agent。后端 Agent 负责理解任务、编排流程和生成内容；用户侧 Agent 只负责提交任务、查询进展和展示结果。

## 功能

1. **提交漫剧 Run 任务** - 创建新会话或向已有会话发送漫剧创作需求。
2. **查询会话进展** - 根据 `thread_id`、`run_id`、`after_seq` 拉取漫剧任务消息列表。
3. **上传文件** - 上传漫剧相关参考文件，得到文件 ID，供后续任务引用。

## 前置要求

需要已安装 `pippit-cli`：

```bash
npx @pippit-dev/cli@latest install
```

可选：`PIPPIT_CLI_BASE_URL`、`XYQ_OPENAPI_BASE` 或 `XYQ_BASE_URL`，默认 `https://xyq.jianying.com`。

## 使用方法

### 1. 提交漫剧任务

```bash
# 创建新会话并提交漫剧创作需求
pippit-cli novel +submit-run --message "创作一个赛博朋克漫剧开头"

# 向已有会话追加新的漫剧需求
pippit-cli novel +submit-run --message "继续写下一集，重点描写主角的逃亡" --thread-id THREAD_ID

# 携带已上传文件 ID 提交任务
pippit-cli novel +submit-run --message "参考这个大纲写第一集" --asset-ids ASSET_ID
```

### 2. 查询漫剧任务进展

```bash
# 查询会话消息列表
pippit-cli novel +get-thread --thread-id THREAD_ID --run-id RUN_ID --after-seq 0
```

> `thread_id` 和 `run_id` 由 `+submit-run` 返回。`after-seq` 用于增量拉取消息，首次查询可使用 `0`。

### 3. 上传文件

当用户提供漫剧大纲、人物设定、世界观设定、已有分集或剧本等本地文件路径时，可先上传文件。

```bash
pippit-cli novel +upload-file --path /path/to/outline.md
```

## 典型工作流

### 场景 1：用户要求生成漫剧内容

```
1. pippit-cli novel +submit-run --message "用户的原始漫剧需求"
   → 拿到 thread_id、run_id 和 web_thread_link
2. 立即将 web_thread_link 展示给用户
3. 使用 pippit-cli novel +get-thread --thread-id THREAD_ID --run-id RUN_ID --after-seq SEQUENCE 查询进展
4. 检查 messages：
   - 如果任务仍在进行中：展示过程消息，继续查询
   - 如果后端 Agent 提出问题：展示问题，等待用户回复
   - 如果已返回漫剧内容或结果：展示给用户
5. 如用户继续追加需求，使用同一 thread_id 再次 submit-run
```

### 场景 2：用户提供参考文件要求创作

```
1. pippit-cli novel +upload-file --path /path/to/file
   → 拿到 file_id
2. pippit-cli novel +submit-run --message "用户的原始漫剧需求" --asset-ids file_id
   → 拿到 thread_id、run_id 和 web_thread_link
3. 后续同场景 1 的查询流程
```

### 场景 3：在已有漫剧会话中续写或修改

```
1. pippit-cli novel +submit-run --message "用户的新需求" --thread-id THREAD_ID
   → 拿到新的 run_id 和 web_thread_link
2. pippit-cli novel +get-thread --thread-id THREAD_ID --run-id RUN_ID --after-seq SEQUENCE
   → 查询该次任务进展
```

## 轮询策略

- **间隔**：每 10 秒查询一次。
- **增量拉取**：首次使用 `--after-seq 0`，后续根据已读消息进度调整 `after-seq`。
- **用户确认**：如果消息中出现需要用户确认、补充设定或回答问题的内容，先展示给用户，等待用户回复。
- **超时**：如果长时间无结果，告知用户任务仍在生成中，可稍后通过 `web_thread_link` 查看。
- **错误处理**：单次查询失败可重试；连续失败时停止轮询并向用户说明错误。

## 输出格式

**+submit-run** 返回：

```json
{
  "thread_id": "thread_...",
  "run_id": "run_...",
  "web_thread_link": "https://xyq.jianying.com/..."
}
```

**+get-thread** 返回：

```json
{
  "messages": [
    {
      "id": "message_...",
      "role": "assistant",
      "content": [
        {
          "type": "text",
          "data": {}
        }
      ]
    }
  ]
}
```

**+upload-file** 返回：

```json
{
  "scene": "novel",
  "file_id": "file_...",
  "status": "uploaded",
  "uploaded_at": "2026-05-19T00:00:00Z",
  "request": {
    "path": "/path/to/file",
    "file_name": "file.md"
  }
}
```

## 向用户展示内容

- 任务提交后：立即展示 `web_thread_link`。
- 任务进行中：展示后端 Agent 返回的过程消息。
- 需要用户补充信息时：原样展示后端 Agent 的问题，等待用户回复。
- 任务完成后：展示漫剧内容、分集草稿、设定说明或其他结果信息。

## 核心原则：用户侧不做创作，只做传话

你（用户侧 Agent）的职责是传递用户需求和展示后端结果，不是替后端 Agent 创作漫剧。

你要做的只有三件事：

1. **上传**：如果用户给了本地参考文件，先调用 `+upload-file`。
2. **提交任务**：把用户原始漫剧需求和文件 ID 通过 `+submit-run` 发给后端。
3. **传话**：根据 `+get-thread` 返回的消息展示进展、问题和结果。

**不要做的事：**

- 不要替用户扩写、润色、翻译 prompt。
- 不要自行编排剧情、人物关系、世界观或分集大纲后再提交。
- 不要把用户的一个需求拆成多次 `+submit-run`，除非用户明确要求分多次处理。
- 不要将自己编写的漫剧内容混入后端返回结果。

后端 Agent 会负责理解漫剧任务、组织创作流程和生成内容。用户侧 Agent 越俎代庖会降低结果一致性。

## 注意事项

- `--message` 是用户的原始漫剧需求，不能为空。
- 查询进展时优先使用 `+submit-run` 返回的 `thread_id` 和 `run_id`。
- `--after-seq` 用于增量拉取消息，首次查询可设置为 `0`。
- `+upload-file` 当前用于漫剧场景文件上传链路，上传后将返回可传给 `+submit-run` 的文件 ID。
