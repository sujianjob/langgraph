# LangGraph 文档

有关为我们的文档做出贡献的更多信息，请参阅[贡献指南](../CONTRIBUTING.zh-CN.md)。

## 结构

主要文档位于 `docs/` 目录中。该目录包含主要文档的源文件以及 API 参考文档构建过程。

### 主要文档

主要文档文件位于 `docs/docs/` 中，采用 Markdown 格式编写。该网站使用 [**MkDocs**](https://www.mkdocs.org/) 和 [Material 主题](https://squidfunk.github.io/mkdocs-material/)，且包含：

- **概念 (Concepts)**：核心 LangGraph 概念和解释
- **教程 (Tutorials)**：循序渐进的学习指南
- **操作指南 (How-tos)**：针对特定用例的任务导向指南
- **示例 (Examples)**：实际应用和用例
- **Jupyter Notebooks**：自动转换为 markdown 的交互式教程

### API 参考

API 参考文档定义在 `docs/docs/reference/` 中。每个 `.md` 文件概述了构建每个页面的“模板”。参考内容使用 **mkdocstrings** 插件从代码库中的文档字符串自动生成。生成后，内容被插入到相应的 markdown 文件中，通过手动指令指定要记录的类和/或函数：

```markdown
::: langgraph.graph.state.StateGraph
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      members:
        - add_node
        - add_edge
        - add_conditional_edges
        - add_sequence
        - compile
```

## 构建过程

文档遵循以下步骤构建：

1.  **内容处理 (Content Processing):**
    - `_scripts/notebook_hooks.py` - 主要处理管道，用于：
        - 使用 `notebook_convert.py` 将 how-tos/tutorials Jupyter notebooks 转换为 markdown
        - 使用 `generate_api_reference_links.py` 为代码块添加自动 API 参考链接
        - 处理 Python/JS 版本的条件渲染
        - 处理高亮注释和自定义语法

2.  **API 参考生成 (API Reference Generation):**
    - **mkdocstrings** 插件从 Python 源代码中提取文档字符串
    - 参考页面 (`/docs/docs/*`) 中的手动 `::: module.Class` 指令指定要记录的内容
    - 在文档和 API 之间自动生成交叉引用

3.  **网站生成 (Site Generation):**
    - **MkDocs** 处理所有 markdown 文件并生成静态 HTML
    - 自定义钩子处理重定向并注入附加功能

4.  **部署 (Deployment):**
    - 网站使用 Vercel 部署
    - `make build-docs` 生成生产构建（也可用于本地测试）
    - 自动重定向处理版本之间的 URL 更改

### 本地开发

对于本地开发，请使用 Makefile 目标：

```bash
# 本地服务文档并支持热重载
make serve-docs

# 为生产测试进行清理构建
make build-docs

# 使用清理构建进行服务
make serve-clean-docs
```

`serve-docs` 命令：

- 监视源文件的更改
- 包含脏构建以加快迭代速度
- 服务于 [http://127.0.0.1:8000/langgraph/](http://127.0.0.1:8000/langgraph/)

## 标准

**文档字符串格式 (Docstring Format):**
API 参考使用带有 Markdown 标记的 **Google 风格文档字符串**。`mkdocstrings` 插件处理这些以生成文档。

**所需格式：**

```python
def example_function(param1: str, param2: int = 5) -> bool:
    """Brief description of the function.

    Longer description can go here. Use Markdown syntax for
    rich formatting like **bold** and *italic*.

    Args:
        param1: Description of the first parameter.
        param2: Description of the second parameter with default value.

    Returns:
        Description of the return value.

    Raises:
        ValueError: When param1 is empty.
        TypeError: When param2 is not an integer.

    !!! warning
        This function is experimental and may change.

    !!! version-added "Added in version 0.2.0"
    """
```

**特殊标记 (Special Markers):**

- **MkDocs 警告**: `!!! warning`, `!!! note`, `!!! version-added`
- **代码块**: 标准 markdown ``` 语法
- **交叉引用**: 通过 `generate_api_reference_links.py` 自动链接

## 执行 notebooks

如果您想自动执行所有 notebooks，以模拟“运行 notebooks” GitHub action，您可以运行：

```bash
python _scripts/prepare_notebooks_for_ci.py
./_scripts/execute_notebooks.sh
```

**注意**：如果您想在没有 `%pip install` 单元格的情况下运行 notebooks，可以运行：

```bash
python _scripts/prepare_notebooks_for_ci.py --comment-install-cells
./_scripts/execute_notebooks.sh
```

`prepare_notebooks_for_ci.py` 脚本将为 notebook 中的每个单元格添加 VCR cassette 上下文管理器，以便：

- 当 notebook 第一次运行时，带有网络请求的单元格将被记录到 VCR cassette 文件中
- 当 notebook 随后运行时，带有网络请求的单元格将从 cassettes 中重放

## 添加新 notebooks

如果您要添加带有 API 请求的 notebook，**建议**记录网络请求，以便随后可以重放它们。如果不这样做，notebook 运行器每次运行 notebook 时都会发出 API 请求，这可能既昂贵又缓慢。

要记录网络请求，请确保首先运行 `prepare_notebooks_for_ci.py` 脚本。

然后，运行

```bash
jupyter execute <path_to_notebook>
```

一旦 notebook 执行完毕，您应该会在 `cassettes` 目录中看到新记录的 VCR cassettes，并丢弃更新后的 notebook。

## 更新现有 notebooks

如果您要更新现有的 notebook，请确保删除 `cassettes` 目录中该 notebook 的任何现有 cassettes（每个 cassette 都以 notebook 名称为前缀），然后运行上面“添加新 notebooks”部分中的步骤。

要删除 notebook 的 cassettes，您可以运行：

```bash
rm cassettes/<notebook_name>*
```
