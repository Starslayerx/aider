# Aider 的代码编辑机制详解

## 核心问题

**Q: Aider 是用工具调用（Function Calling）来修改代码的吗？**

**A: 不是！** Aider 使用的是**基于文本格式的编辑指令**，而非 LLM 的工具调用功能。

---

## 为什么不用工具调用？

### 工具调用的方式（Aider 没有采用）

```python
# 假设使用工具调用的方式（这不是 Aider 的做法）
tools = [{
    "name": "edit_file",
    "description": "Edit a file",
    "parameters": {
        "file_path": "string",
        "old_string": "string",
        "new_string": "string"
    }
}]

response = llm.call(
    messages=messages,
    tools=tools  # ❌ Aider 不这么做
)

# LLM 会返回结构化的工具调用
tool_call = response.tool_calls[0]
edit_file(
    file_path=tool_call.arguments["file_path"],
    old_string=tool_call.arguments["old_string"],
    new_string=tool_call.arguments["new_string"]
)
```

### Aider 的实际方式：文本格式编辑指令

```python
# Aider 的实际做法
system_prompt = """
使用 *SEARCH/REPLACE* 块来编辑代码：

filename.py
```python
<<<<<<< SEARCH
旧代码内容
=======
新代码内容
>>>>>>> REPLACE
```
"""

response = llm.call(messages=messages)  # ✅ 普通文本响应

# LLM 返回的是包含特定格式的文本
# Aider 解析这个文本，提取编辑指令
edits = parse_search_replace_blocks(response.content)
apply_edits(edits)
```

---

## Aider 的编辑格式

Aider 支持多种文本格式的编辑指令，每种格式对应一个 Coder 类：

### 1. SEARCH/REPLACE 格式 (EditBlockCoder)

**最常用的格式**，类似 Git 冲突标记。

#### 示例

```python
# LLM 输出的文本（不是函数调用）：

mathweb/flask/app.py
```python
<<<<<<< SEARCH
from flask import Flask
=======
import math
from flask import Flask
>>>>>>> REPLACE
```

mathweb/flask/app.py
```python
<<<<<<< SEARCH
def factorial(n):
    "compute factorial"

    if n == 0:
        return 1
    else:
        return n * factorial(n-1)

=======
>>>>>>> REPLACE
```
```

#### 解析流程

从 `editblock_coder.py` 的代码可以看到：

```python
def find_original_update_blocks(content, fence=DEFAULT_FENCE, valid_fnames=None):
    """解析 LLM 文本响应中的 SEARCH/REPLACE 块"""

    # 1. 按行分割文本
    lines = content.splitlines(keepends=True)

    # 2. 查找模式
    HEAD = r"^<{5,9} SEARCH>?\s*$"        # <<<<<<< SEARCH
    DIVIDER = r"^={5,9}\s*$"              # =======
    UPDATED = r"^>{5,9} REPLACE\s*$"      # >>>>>>> REPLACE

    # 3. 提取文件名
    filename = find_filename(lines[i-3:i], fence, valid_fnames)

    # 4. 提取 SEARCH 部分（原始代码）
    original_text = []
    while not divider_pattern.match(lines[i]):
        original_text.append(lines[i])
        i += 1

    # 5. 提取 REPLACE 部分（新代码）
    updated_text = []
    while not updated_pattern.match(lines[i]):
        updated_text.append(lines[i])
        i += 1

    # 6. 返回编辑元组
    yield (filename, "".join(original_text), "".join(updated_text))
```

#### 应用编辑

```python
def do_replace(fname, content, before_text, after_text, fence=None):
    """应用 SEARCH/REPLACE 编辑"""

    # 1. 读取文件内容
    content = io.read_text(fname)

    # 2. 查找匹配（完美匹配）
    new_content = replace_most_similar_chunk(content, before_text, after_text)

    # 3. 如果完美匹配失败，尝试容错匹配
    if not new_content:
        # - 忽略前导空格差异
        # - 处理 ... 省略号
        # - 模糊匹配（相似度 > 0.8）

    # 4. 写回文件
    if new_content:
        io.write_text(fname, new_content)
    else:
        raise ValueError("SEARCH block 无法精确匹配文件内容")
```

### 2. Unified Diff 格式 (UnifiedDiffCoder)

**标准 Git diff 格式**。

#### 示例

```python
# LLM 输出：

--- mathweb/flask/app.py
+++ mathweb/flask/app.py
@@ -1,4 +1,5 @@
+import math
 from flask import Flask

 app = Flask(__name__)
@@ -10,12 +11,7 @@
-def factorial(n):
-    "compute factorial"
-
-    if n == 0:
-        return 1
-    else:
-        return n * factorial(n-1)
-
 def get_factorial(n):
-    return str(factorial(n))
+    return str(math.factorial(n))
```

### 3. Whole File 格式 (WholeFileCoder)

**替换整个文件**，适合小文件。

#### 示例

```python
# LLM 输出：

app.py
```python
import math
from flask import Flask

app = Flask(__name__)

def get_factorial(n):
    return str(math.factorial(n))
```
```

### 4. Patch 格式 (PatchCoder)

**Git patch 格式**，支持重命名、权限等。

---

## 为什么选择文本格式？

### ✅ 优势

#### 1. **模型兼容性好**

```python
# 所有模型都能输出文本，但不是所有模型都支持工具调用
models_with_text_output = [
    "gpt-3.5-turbo",    # ✅
    "gpt-4",            # ✅
    "claude-3",         # ✅
    "deepseek-chat",    # ✅
    "gemini-pro",       # ✅
    "local-llama",      # ✅ 本地模型也行
]

# 工具调用支持有限
models_with_function_calling = [
    "gpt-4",            # ✅
    "claude-3",         # ✅
    "deepseek-chat",    # ❌ 早期版本不支持
    "gemini-pro",       # ✅ 但格式不同
    "local-llama",      # ❌ 大多数不支持
]
```

#### 2. **容错性强**

```python
# 文本格式可以模糊匹配
def replace_most_similar_chunk(whole, part, replace):
    # 1. 尝试完美匹配
    res = perfect_replace(whole_lines, part_lines, replace_lines)
    if res:
        return res

    # 2. 尝试忽略前导空格
    res = replace_part_with_missing_leading_whitespace(...)
    if res:
        return res

    # 3. 尝试处理 ... 省略号
    res = try_dotdotdots(whole, part, replace)
    if res:
        return res

    # 4. 模糊匹配（相似度阈值 0.8）
    res = replace_closest_edit_distance(...)
    return res

# 工具调用是结构化的，要么成功要么失败，没有容错空间
```

#### 3. **可读性和可调试性**

```python
# 文本格式：用户可以直接看到 LLM 的编辑意图
"""
我将修改 app.py 来添加 math 导入：

app.py
```python
<<<<<<< SEARCH
from flask import Flask
=======
import math
from flask import Flask
>>>>>>> REPLACE
```
"""

# 工具调用：用户看不到，是黑盒
{
    "tool_calls": [{
        "name": "edit_file",
        "arguments": {
            "file": "app.py",
            "old": "from flask import Flask",
            "new": "import math\nfrom flask import Flask"
        }
    }]
}
```

#### 4. **支持多种编辑风格**

不同模型擅长不同格式：
- GPT-4: 擅长 SEARCH/REPLACE
- Claude: 擅长 Unified Diff
- Gemini: 两者都可以

文本格式允许灵活切换。

#### 5. **更好的提示词控制**

```python
# 可以在系统提示词中详细说明格式
system_prompt = """
Every *SEARCH/REPLACE block* must use this format:
1. The *FULL* file path alone on a line
2. The opening fence: ```python
3. <<<<<<< SEARCH
4. A contiguous chunk of lines to search for
5. =======
6. The lines to replace
7. >>>>>>> REPLACE
8. The closing fence: ```

Rules:
- SEARCH section must EXACTLY MATCH existing code
- Include enough context for uniqueness
- Keep blocks concise
- Use 2 blocks to move code (delete + insert)
"""

# 工具调用的提示词控制有限
```

### ❌ 工具调用的劣势（对 Aider 场景）

1. **模型覆盖面小** - 很多模型不支持
2. **格式不统一** - OpenAI/Anthropic/Google 各有各的格式
3. **无法容错** - 参数错误就失败
4. **调试困难** - 结构化数据不如文本直观
5. **Token 开销** - 工具定义本身消耗 tokens
6. **复杂度高** - 需要处理工具调用的解析、验证、错误处理

---

## 实际代码流程

### 完整的编辑流程

```python
# 1. 用户输入
user_message = "Add math import to app.py"

# 2. Coder 准备上下文（base_coder.py）
context = {
    "system_prompt": EditBlockPrompts.main_system,  # 说明格式规则
    "files": ["app.py"],                            # 文件内容
    "repo_map": generate_repo_map(),                # 代码地图
    "chat_history": previous_messages               # 历史对话
}

# 3. 调用 LLM（models.py）
response = model.send_message(
    messages=[
        {"role": "system", "content": context["system_prompt"]},
        {"role": "user", "content": user_message}
    ]
)

# 4. LLM 返回文本（不是工具调用！）
response.content = """
To add the math import, we need to modify app.py:

app.py
```python
<<<<<<< SEARCH
from flask import Flask
=======
import math
from flask import Flask
>>>>>>> REPLACE
```
"""

# 5. 解析编辑指令（editblock_coder.py）
edits = find_original_update_blocks(
    content=response.content,
    fence=("```", "```"),
    valid_fnames=["app.py"]
)
# edits = [("app.py", "from flask import Flask", "import math\nfrom flask import Flask")]

# 6. 应用编辑（editblock_coder.py）
for file_path, old_str, new_str in edits:
    content = io.read_text(file_path)
    new_content = do_replace(file_path, content, old_str, new_str)
    io.write_text(file_path, new_content)

# 7. Git 提交（repo.py）
repo.commit(
    message="Add math import",
    author="User (aider)"
)
```

### 错误处理

```python
# 当 SEARCH 块无法匹配时
try:
    new_content = do_replace(file_path, content, old_str, new_str)
except ValueError as e:
    # 提示 LLM 修正
    error_message = f"""
# SEARCH/REPLACE block failed to match!

## SearchReplaceNoExactMatch: This SEARCH block failed to exactly match lines in {file_path}
<<<<<<< SEARCH
{old_str}
=======
{new_str}
>>>>>>> REPLACE

Did you mean to match some of these actual lines from {file_path}?

```
{find_similar_lines(old_str, content)}  # 模糊匹配建议
```

The SEARCH section must exactly match existing code including all whitespace.
Just reply with a fixed version of the block.
"""

    # 发送错误消息给 LLM，让它重试
    response = model.send_message(error_message)
```

---

## 与工具调用的对比

### 场景：添加一个导入语句

#### 使用工具调用（假设）

```python
# 系统提示词
system = "You can use edit_file(path, old, new) to edit files"

# LLM 响应
{
    "content": "I'll add the math import",
    "tool_calls": [{
        "id": "call_123",
        "type": "function",
        "function": {
            "name": "edit_file",
            "arguments": {
                "path": "app.py",
                "old": "from flask import Flask",
                "new": "import math\nfrom flask import Flask"
            }
        }
    }]
}

# 优点：
# ✅ 结构化，易于程序解析
# ✅ 类型安全（如果有 schema）

# 缺点：
# ❌ 需要模型支持 function calling
# ❌ 用户看不到编辑意图（黑盒）
# ❌ 如果 old 字符串不完全匹配就失败
# ❌ 无法容错（空格、换行差异）
# ❌ Token 开销（工具定义）
```

#### 使用文本格式（Aider 实际方式）

```python
# 系统提示词
system = """Use SEARCH/REPLACE blocks:

filename.py
```language
<<<<<<< SEARCH
old code
=======
new code
>>>>>>> REPLACE
```
"""

# LLM 响应
"""
I'll add the math import to app.py:

app.py
```python
<<<<<<< SEARCH
from flask import Flask
=======
import math
from flask import Flask
>>>>>>> REPLACE
```
"""

# 优点：
# ✅ 所有模型都能输出文本
# ✅ 用户可见（可读性强）
# ✅ 可以模糊匹配（容错）
# ✅ 支持多种格式（diff, whole-file, etc.）
# ✅ 易于调试

# 缺点：
# ❌ 需要解析文本（正则表达式）
# ❌ 格式错误处理更复杂
```

---

## Prompt Engineering 的威力

Aider 的成功很大程度上依赖于精心设计的提示词：

### 提示词示例（editblock_prompts.py）

```python
system_reminder = """
# *SEARCH/REPLACE block* Rules:

Every *SEARCH/REPLACE block* must use this format:
1. The *FULL* file path alone on a line, verbatim.
2. The opening fence: ```python
3. The start of search block: <<<<<<< SEARCH
4. A contiguous chunk of lines to search for in the existing source code
5. The dividing line: =======
6. The lines to replace into the source code
7. The end of the replace block: >>>>>>> REPLACE
8. The closing fence: ```

Use the *FULL* file path, as shown to you by the user.

Every *SEARCH* section must *EXACTLY MATCH* the existing file content,
character for character, including all comments, docstrings, etc.

*SEARCH/REPLACE* blocks will *only* replace the first match occurrence.
Include enough lines in each SEARCH section to uniquely match each set
of lines that need to change.

Keep *SEARCH/REPLACE* blocks concise.
Break large blocks into a series of smaller blocks.

To move code within a file, use 2 *SEARCH/REPLACE* blocks:
1 to delete it from its current location,
1 to insert it in the new location.

If you want to put code in a new file, use a *SEARCH/REPLACE block* with:
- A new file path
- An empty `SEARCH` section
- The new file's contents in the `REPLACE` section

ONLY EVER RETURN CODE IN A *SEARCH/REPLACE BLOCK*!
"""
```

### Few-Shot 示例

Aider 还提供了示例对话（few-shot learning）：

```python
example_messages = [
    {
        "role": "user",
        "content": "Change get_factorial() to use math.factorial"
    },
    {
        "role": "assistant",
        "content": """
To make this change we need to modify `mathweb/flask/app.py` to:

1. Import the math package.
2. Remove the existing factorial() function.
3. Update get_factorial() to call math.factorial instead.

Here are the *SEARCH/REPLACE* blocks:

mathweb/flask/app.py
```python
<<<<<<< SEARCH
from flask import Flask
=======
import math
from flask import Flask
>>>>>>> REPLACE
```

mathweb/flask/app.py
```python
<<<<<<< SEARCH
def factorial(n):
    "compute factorial"

    if n == 0:
        return 1
    else:
        return n * factorial(n-1)

=======
>>>>>>> REPLACE
```

mathweb/flask/app.py
```python
<<<<<<< SEARCH
    return str(factorial(n))
=======
    return str(math.factorial(n))
>>>>>>> REPLACE
```
"""
    }
]
```

这些示例教会 LLM：
1. 如何思考问题（step-by-step）
2. 如何格式化输出
3. 如何拆分大的编辑为多个小块
4. 如何删除代码（空的 REPLACE 部分）

---

## 总结

### Aider 的设计哲学

1. **通用性优先** - 支持所有能输出文本的 LLM
2. **用户可见性** - 编辑意图清晰展示
3. **容错性** - 模糊匹配处理格式差异
4. **灵活性** - 多种编辑格式适配不同场景
5. **Prompt Engineering** - 用精心设计的提示词引导 LLM

### 关键洞察

> **Aider 证明了：通过精心设计的文本格式 + 优秀的提示词工程，可以实现比工具调用更强大、更灵活的代码编辑能力。**

这也是为什么 Aider 能支持 100+ 种 LLM 模型，包括很多没有工具调用功能的模型。

---

## 延伸思考

### 什么时候应该用工具调用？

工具调用更适合：
- **简单的、结构化的操作**（如查询数据库、调用 API）
- **需要严格类型检查的场景**
- **单一模型的专用工具**

### 什么时候应该用文本格式？

文本格式更适合：
- **复杂的、创造性的任务**（如代码编辑）
- **需要支持多种模型的场景**
- **需要用户可见性和可调试性**
- **需要容错处理**

Aider 选择了后者，并证明了这是正确的选择！

---

*最后更新：2025年10月*
*基于 Aider v0.86.0 源码分析*
