# Aider 学习文档

这个目录包含了帮助你学习和理解 Aider 项目的文档。

## 📚 文档列表

### 1. [INTRODUCE.md](./INTRODUCE.md) - 项目学习指南

**适合人群**：初级 Python 程序员，想要学习 Aider 并参与贡献

**内容包括**：
- 项目简介和技术栈
- 为什么学习 Aider
- 快速开始指南
- **7个可视化架构图** (Mermaid)
  - 整体架构图
  - 数据流程图
  - 核心类关系图
  - 编辑流程详细图
  - Token 管理策略图
  - 文件处理流程图
  - RepoMap 生成流程图
- 项目结构详解
- 核心概念解析
- 代码阅读指南（推荐顺序）
- 贡献指南（从易到难）
- 4周学习路径
- 常见问题解答

**适合场景**：
- 刚开始学习 Aider
- 想要了解整体架构
- 准备第一次贡献

---

### 2. [EDITING_MECHANISM.md](./EDITING_MECHANISM.md) - 代码编辑机制详解

**适合人群**：想深入理解 Aider 核心工作原理的开发者

**内容包括**：
- **核心问题**：Aider 是否使用工具调用（Function Calling）？
- Aider 的实际机制：文本格式编辑指令
- 4种编辑格式详解：
  - SEARCH/REPLACE 格式 (EditBlockCoder)
  - Unified Diff 格式 (UnifiedDiffCoder)
  - Whole File 格式 (WholeFileCoder)
  - Patch 格式 (PatchCoder)
- 为什么选择文本格式而非工具调用
- 实际代码流程和错误处理
- Prompt Engineering 的威力
- 设计哲学和延伸思考

**适合场景**：
- 好奇 Aider 如何让 LLM 编辑代码
- 想理解为什么不用工具调用
- 对 Prompt Engineering 感兴趣
- 计划贡献 Coder 相关代码

---

### 3. [CONTEXT_ENGINEERING.md](./CONTEXT_ENGINEERING.md) - 上下文工程详解

**适合人群**：想理解 Aider 如何高效管理 LLM 上下文的开发者

**内容包括**：
- **核心挑战**：上下文窗口限制、相关性问题、成本控制
- Aider 的分层上下文策略
- **RepoMap 深度剖析**：
  - Tree-sitter 代码解析
  - PageRank 相关性排序
  - 二分搜索 token 优化
  - 树形视图生成
- Context 的 8 个组成部分详解
- Token 预算动态管理
- 缓存策略（Anthropic Prompt Caching）
- 完整代码流程演示
- 最佳实践和进阶技巧

**适合场景**：
- 好奇 Aider 如何在大代码库中工作
- 想理解 RepoMap 的工作原理
- 对 LLM 上下文优化感兴趣
- 需要优化自己的 AI 应用的上下文管理
- 计划贡献 RepoMap 相关代码

---

## 🎯 推荐阅读顺序

### 初学者路线

1. **第一天**：阅读 `INTRODUCE.md` 的前 3 章
   - 了解项目是什么
   - 为什么值得学习
   - 搭建开发环境

2. **第二天**：研究架构可视化部分
   - 看懂 7 个 Mermaid 图表
   - 建立全局认知

3. **第三天**：阅读项目结构和核心概念
   - 知道各个文件的作用
   - 理解 Coder、RepoMap、GitRepo 等概念

4. **第一周内**：跟随学习路径，开始实践
   - 运行测试
   - 阅读简单的源码文件

5. **需要时**：阅读 `EDITING_MECHANISM.md`
   - 当你想深入理解编辑流程时
   - 当你准备修改 Coder 相关代码时

### 进阶路线

如果你已经熟悉项目基础：

1. **深入架构**：`INTRODUCE.md` → 架构可视化 + 核心概念
2. **理解编辑机制**：`EDITING_MECHANISM.md` → 完整阅读
3. **理解上下文管理**：`CONTEXT_ENGINEERING.md` → 深入 RepoMap
4. **阅读源码**：按照 `INTRODUCE.md` 的推荐顺序阅读代码
5. **开始贡献**：找 `good first issue`，提交第一个 PR

### 主题学习路线

**路线 1：理解 Aider 如何工作**
- `INTRODUCE.md` (架构概览)
- `EDITING_MECHANISM.md` (如何编辑代码)
- `CONTEXT_ENGINEERING.md` (如何管理上下文)

**路线 2：深入 LLM 应用开发**
- `EDITING_MECHANISM.md` (Prompt Engineering)
- `CONTEXT_ENGINEERING.md` (Context Engineering)
- 源码：`aider/coders/base_coder.py`, `aider/repomap.py`

**路线 3：贡献代码**
- `INTRODUCE.md` (快速开始 + 贡献指南)
- 源码：相关模块
- `CLAUDE.md` (开发规范)

---

## 💡 使用建议

### 结合实践学习

```bash
# 1. 边读文档，边运行代码
cd /path/to/aider
source ../aider_venv/bin/activate

# 2. 运行测试理解功能
pytest tests/basic/test_commands.py::test_cmd_add -v -s

# 3. 用 Aider 自己学习 Aider
aider --read-only aider/main.py
# 然后问: "解释 main() 函数的流程"

# 4. 添加调试输出
# 在代码中加 print() 语句，运行看效果
```

### 善用图表

- 在 GitHub 上查看（自动渲染 Mermaid）
- 用支持 Mermaid 的编辑器（VS Code、Typora）
- 使用 [Mermaid Live Editor](https://mermaid.live/) 编辑

### 做笔记

在文档末尾的"你的学习计划"部分填写：
- 每周目标
- 完成情况
- 遇到的问题
- 学到的知识点

---

## 🤝 贡献这些文档

这些文档也欢迎改进！如果你发现：

- 错别字或不清楚的描述
- 需要补充的内容
- 更好的示例或解释
- 新的学习路径建议

请提交 PR 改进这些文档！

---

## 📖 其他资源

### 官方文档
- [Aider 官网](https://aider.chat/)
- [完整文档](https://aider.chat/docs/)
- [GitHub 仓库](https://github.com/Aider-AI/aider)

### 项目根目录的文档
- [CLAUDE.md](../CLAUDE.md) - Claude Code 工作指南（针对 AI 助手）
- [README.md](../README.md) - 项目主页
- [CONTRIBUTING.md](../CONTRIBUTING.md) - 贡献指南

### 社区
- [Discord 社区](https://discord.gg/Y7X7bhMQFV)
- [GitHub Issues](https://github.com/Aider-AI/aider/issues)

---

## 📝 文档历史

- **2025-10-15**：创建学习文档目录
  - 添加 `INTRODUCE.md` - 完整的学习指南，包含 7 个架构图
  - 添加 `EDITING_MECHANISM.md` - 深入讲解编辑机制
  - 添加 `README.md` - 文档导航

---

**祝你学习愉快！** 🚀

如果有任何问题，欢迎在 [GitHub Issues](https://github.com/Aider-AI/aider/issues) 或 [Discord](https://discord.gg/Y7X7bhMQFV) 提问。
