# AGENTS 说明

本仓库是一个 monorepo（单体仓库）。每个库都位于 `libs/` 下的子目录中。

当你修改任何库中的代码时，在创建拉取请求之前，请在该库的目录中运行以下命令：

- `make format` – 运行代码格式化工具
- `make lint` – 运行代码检查工具
- `make test` – 执行测试套件

要运行特定的测试文件或传递额外的 pytest 选项，你可以指定 `TEST` 变量：

```
TEST=path/to/test.py make test
```

其他 pytest 参数也可以在 `TEST` 变量中提供。

## 库

该仓库包含多个 Python 和 JavaScript/TypeScript 库。
以下是高级概述：

- **checkpoint** – LangGraph 检查点的基础接口。
- **checkpoint-postgres** – 检查点保存器的 Postgres 实现。
- **checkpoint-sqlite** – 检查点保存器的 SQLite 实现。
- **cli** – LangGraph 的官方命令行界面。
- **langgraph** – 用于构建有状态、多角色智能体的核心框架。
- **prebuilt** – 用于创建和运行智能体和工具的高级 API。
- **sdk-js** – 用于与 LangGraph REST API 交互的 JS/TS SDK。
- **sdk-py** – LangGraph Server API 的 Python SDK。

### 依赖关系图

下图列出了每个生产依赖项的下游库，如该库的 `pyproject.toml`（或 `package.json`）中所声明。

```text
checkpoint
├── checkpoint-postgres
├── checkpoint-sqlite
├── prebuilt
└── langgraph

prebuilt
└── langgraph

sdk-py
├── langgraph
└── cli

sdk-js (standalone)
```

对一个库的更改可能会影响上面显示的所有依赖项。
