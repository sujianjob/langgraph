# LangGraph CLI

LangGraph 命令行界面包含在 [Docker](https://www.docker.com/) 中本地构建和运行 LangGraph Platform API 服务器的命令。对于开发和测试，您可以使用 CLI 部署本地 API 服务器。

## 安装 (Installation)

1.  确保已安装 Docker (例如 `docker --version`)。
2.  安装 CLI 包：

    === "Python"
        ```bash
        pip install langgraph-cli
        ```

    === "JS"
        ```bash
        npx @langchain/langgraph-cli

        # Install globally, will be available as `langgraphjs`
        npm install -g @langchain/langgraph-cli
        ```

3.  运行命令 `langgraph --help` 或 `npx @langchain/langgraph-cli --help` 以确认 CLI 正常工作。

[](){#langgraph.json}

## 配置文件 (Configuration File) {#configuration-file}

LangGraph CLI 需要一个遵循此 [架构](https://raw.githubusercontent.com/langchain-ai/langgraph/refs/heads/main/libs/cli/schemas/schema.json) 的 JSON 配置文件。它包含以下属性：

<div class="admonition tip">
    <p class="admonition-title">Note</p>
    <p>
        LangGraph CLI 默认使用当前目录中的配置文件 <strong>langgraph.json</strong>。
    </p>
</div>

=== "Python"

    | 键 (Key)                                                     | 描述 (Description)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
    | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | <span style="white-space: nowrap;">`dependencies`</span>     | **必需**. LangGraph Platform API 服务器的依赖项数组。依赖项可以是以下之一：<ul><li>单个句点 (`"."`)，这将查找本地 Python 包。</li><li>`pyproject.toml`、`setup.py` 或 `requirements.txt` 所在的目录路径。</br></br>例如，如果 `requirements.txt` 位于项目目录的根目录中，请指定 `"./"`。如果它位于名为 `local_package` 的子目录中，请指定 `"./local_package"`。不要指定字符串 `"requirements.txt"` 本身。</li><li>Python 包名称。</li></ul> |
    | <span style="white-space: nowrap;">`graphs`</span>           | **必需**. 从图 ID 到定义编译图或生成图的函数的路径的映射。示例：<ul><li>`./your_package/your_file.py:variable`，其中 `variable` 是 `langgraph.graph.state.CompiledStateGraph` 的实例</li><li>`./your_package/your_file.py:make_graph`，其中 `make_graph` 是一个函数，该函数接受配置字典 (`langchain_core.runnables.RunnableConfig`) 并返回 `langgraph.graph.state.StateGraph` 或 `langgraph.graph.state.CompiledStateGraph` 的实例。有关详细信息，请参阅 [如何在运行时重建图](../../cloud/deployment/graph_rebuild.md)。</li></ul>                                    |
    | <span style="white-space: nowrap;">`auth`</span>             | _(在 v0.0.11 中添加)_ 包含身份验证处理程序路径的身份验证配置。示例：`./your_package/auth.py:auth`，其中 `auth` 是 `langgraph_sdk.Auth` 的实例。有关详细信息，请参阅 [身份验证指南](../../concepts/auth.md)。                                                                                                                                                                                                                                                                                                                        |
    | <span style="white-space: nowrap;">`base_image`</span>       | 可选. 用于 LangGraph API 服务器的基本映像。默认为 `langchain/langgraph-api` 或 `langchain/langgraphjs-api`。使用它将构建固定到 langgraph API 的特定版本，例如 `"langchain/langgraph-server:0.2"`。有关更多详细信息，请参阅 https://hub.docker.com/r/langchain/langgraph-server/tags。(在 `langgraph-cli==0.2.8` 中添加) |
    | <span style="white-space: nowrap;">`image_distro`</span>     | 可选. 基本映像的 Linux 发行版。必须是 `"debian"` 或 `"wolfi"`。如果省略，则默认为 `"debian"`。在 `langgraph-cli>=0.2.11` 中可用。|
    | <span style="white-space: nowrap;">`env`</span>              | `.env` 文件的路径或从环境变量到其值的映射。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
    | <span style="white-space: nowrap;">`store`</span>            | 为 BaseStore 添加语义搜索和/或生存时间 (TTL) 的配置。包含以下字段：<ul><li>`index` (可选): 具有字段 `embed`、`dims` 和可选 `fields` 的语义搜索索引配置。</li><li>`ttl` (可选): 项目过期的配置。具有可选字段的对象：`refresh_on_read` (布尔值，默认为 `true`)、`default_ttl` (浮点数，以 **分钟** 为单位的寿命，默认为不过期) 和 `sweep_interval_minutes` (整数，检查过期项目的频率，默认为不清理)。</li></ul> |
    | <span style="white-space: nowrap;">`ui`</span>               | 可选. 代理发出的 UI 组件的命名定义，每个均指向 JS/TS 文件。(在 `langgraph-cli==0.1.84` 中添加)                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
    | <span style="white-space: nowrap;">`python_version`</span>   | `3.11`, `3.12`, 或 `3.13`. 默认为 `3.11`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
    | <span style="white-space: nowrap;">`node_version`</span>     | 指定 `node_version: 20` 以使用 LangGraph.js。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
    | <span style="white-space: nowrap;">`pip_config_file`</span>  | `pip` 配置文件路径。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
    | <span style="white-space: nowrap;">`pip_installer`</span> | _(在 v0.3 中添加)_ 可选. Python 包安装程序选择器。可以设置为 `"auto"`, `"pip"`, 或 `"uv"`. 从版本 0.3 开始，默认策略是运行 `uv pip`，这通常提供更快的构建速度，同时保持直接替换。在极少数情况下，如果 `uv` 无法处理您的依赖关系图或 `pyproject.toml` 的结构，请在此处指定 `"pip"` 以恢复到以前的行为。 |
    | <span style="white-space: nowrap;">`keep_pkg_tools`</span> | _(在 v0.3.4 中添加)_ 可选. 控制是否在最终映像中保留 Python 打包工具 (`pip`, `setuptools`, `wheel`)。接受的值：<ul><li><code>true</code> : 保留所有三个工具 (跳过卸载)。</li><li><code>false</code> / 省略 : 卸载所有三个工具 (默认行为)。</li><li><code>list[str]</code> : **要保留** 的工具名称。每个值必须是 "pip", "setuptools", "wheel" 之一。</li></ul>. 默认情况下，将卸载所有三个工具。 |
    | <span style="white-space: nowrap;">`dockerfile_lines`</span> | 在从父映像导入之后添加到 Dockerfile 的附加行数组。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
    | <span style="white-space: nowrap;">`checkpointer`</span>   | 检查点的配置。包含一个 `ttl` 字段，该字段是一个具有以下键的对象：<ul><li>`strategy`: 如何处理过期的检查点 (例如 `"delete"`)。</li><li>`sweep_interval_minutes`: 检查过期检查点的频率 (整数)。</li><li>`default_ttl`: 检查点的默认生存时间，以 **分钟** 为单位 (整数)。定义在应用指定策略之前保留检查点的时间。</li></ul> |
    | <span style="white-space: nowrap;">`http`</span>            | 具有以下字段的 HTTP 服务器配置：<ul><li>`app`: 自定义 Starlette/FastAPI 应用程序的路径 (例如 `"./src/agent/webapp.py:app"`)。请参阅 [自定义路由指南](../../how-tos/http/custom_routes.md)。</li><li>`cors`: 具有 `allow_origins`、`allow_methods`、`allow_headers` 等字段的 CORS 配置。</li><li>`configurable_headers`: 定义要在运行的可配置值中排除或包含哪些请求标头。</li><li>`disable_assistants`: 禁用 `/assistants` 路由</li><li>`disable_mcp`: 禁用 `/mcp` 路由</li><li>`disable_meta`: 禁用 `/ok`、`/info`、`/metrics` 和 `/docs` 路由</li><li>`disable_runs`: 禁用 `/runs` 路由</li><li>`disable_store`: 禁用 `/store` 路由</li><li>`disable_threads`: 禁用 `/threads` 路由</li><li>`disable_ui`: 禁用 `/ui` 路由</li><li>`disable_webhooks`: 禁用所有路由中运行完成时的 webhook 调用</li><li>`mount_prefix`: 挂载路由的前缀 (例如 "/my-deployment/api")</li></ul> |

=== "JS"

    | 键 (Key)                                                     | 描述 (Description)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
    | ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
    | <span style="white-space: nowrap;">`graphs`</span>           | **必需**. 从图 ID 到定义编译图或生成图的函数的路径的映射。示例：<ul><li>`./src/graph.ts:variable`，其中 `variable` 是 `CompiledStateGraph` 的实例</li><li>`./src/graph.ts:makeGraph`，其中 `makeGraph` 是一个函数，该函数接受配置字典 (`LangGraphRunnableConfig`) 并返回 `StateGraph` 或 `CompiledStateGraph` 的实例。有关详细信息，请参阅 [如何在运行时重建图](../../cloud/deployment/graph_rebuild.md)。</li></ul>                                    |
    | <span style="white-space: nowrap;">`env`</span>              | `.env` 文件的路径或从环境变量到其值的映射。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
    | <span style="white-space: nowrap;">`store`</span>            | 为 BaseStore 添加语义搜索和/或生存时间 (TTL) 的配置。包含以下字段：<ul><li>`index` (可选): 具有字段 `embed`、`dims` 和可选 `fields` 的语义搜索索引配置。</li><li>`ttl` (可选): 项目过期的配置。具有可选字段的对象：`refresh_on_read` (布尔值，默认为 `true`)、`default_ttl` (浮点数，以 **分钟** 为单位的寿命，默认为不过期) 和 `sweep_interval_minutes` (整数，检查过期项目的频率，默认为不清理)。</li></ul> |
    | <span style="white-space: nowrap;">`node_version`</span>     | 指定 `node_version: 20` 以使用 LangGraph.js。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
    | <span style="white-space: nowrap;">`dockerfile_lines`</span> | 在从父映像导入之后添加到 Dockerfile 的附加行数组。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
    | <span style="white-space: nowrap;">`checkpointer`</span>   | 检查点的配置。包含一个 `ttl` 字段，该字段是一个具有以下键的对象：<ul><li>`strategy`: 如何处理过期的检查点 (例如 `"delete"`)。</li><li>`sweep_interval_minutes`: 检查过期检查点的频率 (整数)。</li><li>`default_ttl`: 检查点的默认生存时间，以 **分钟** 为单位 (整数)。定义在应用指定策略之前保留检查点的时间。</li></ul> |

### 示例 (Examples)

=== "Python"
    
    #### 基本配置 (Basic Configuration)

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "chat": "./chat/graph.py:graph"
      }
    }
    ```

    #### 使用 Wolfi 基本映像 (Using Wolfi Base Images)

    您可以使用 `image_distro` 字段指定基本映像的 Linux 发行版。有效选项为 `debian` 或 `wolfi`。Wolfi 是推荐选项，因为它提供更小、更安全的映像。这在 `langgraph-cli>=0.2.11` 中可用。

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "chat": "./chat/graph.py:graph"
      },
      "image_distro": "wolfi"
    }
    ```

    #### 向存储添加语义搜索 (Adding semantic search to the store)

    所有部署都附带一个基于数据库的 BaseStore。向您的 `langgraph.json` 添加 "index" 配置将在部署的 BaseStore 中启用 [语义搜索](../deployment/semantic_search.md)。

    `index.fields` 配置确定嵌入文档的哪些部分：

    - 如果省略或设置为 `["$"]`，则将嵌入整个文档
    - 要嵌入特定字段，请使用 JSON 路径表示法：`["metadata.title", "content.text"]`
    - 缺少指定字段的文档仍将被存储，但这些字段没有嵌入
    - 您仍然可以使用 `index` 参数在 `put` 时覆盖要在特定项目上嵌入的字段

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "memory_agent": "./agent/graph.py:graph"
      },
      "store": {
        "index": {
          "embed": "openai:text-embedding-3-small",
          "dims": 1536,
          "fields": ["$"]
        }
      }
    }
    ```

    !!! note "常见模型维度 (Common model dimensions)" 
        - `openai:text-embedding-3-large`: 3072 
        - `openai:text-embedding-3-small`: 1536 
        - `openai:text-embedding-ada-002`: 1536 
        - `cohere:embed-english-v3.0`: 1024 
        - `cohere:embed-english-light-v3.0`: 384 
        - `cohere:embed-multilingual-v3.0`: 1024 
        - `cohere:embed-multilingual-light-v3.0`: 384 

    #### 使用自定义嵌入函数进行语义搜索 (Semantic search with a custom embedding function)

    如果要将语义搜索与自定义嵌入函数一起使用，则可以传递自定义嵌入函数的路径：

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "memory_agent": "./agent/graph.py:graph"
      },
      "store": {
        "index": {
          "embed": "./embeddings.py:embed_texts",
          "dims": 768,
          "fields": ["text", "summary"]
        }
      }
    }
    ```

    存储配置中的 `embed` 字段可以引用一个自定义函数，该函数接受字符串列表并返回嵌入列表。实现示例：

    ```python
    # embeddings.py
    def embed_texts(texts: list[str]) -> list[list[float]]:
        """Custom embedding function for semantic search."""
        # Implementation using your preferred embedding model
        return [[0.1, 0.2, ...] for _ in texts]  # dims-dimensional vectors
    ```

    #### 添加自定义身份验证 (Adding custom authentication)

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "chat": "./chat/graph.py:graph"
      },
      "auth": {
        "path": "./auth.py:auth",
        "openapi": {
          "securitySchemes": {
            "apiKeyAuth": {
              "type": "apiKey",
              "in": "header",
              "name": "X-API-Key"
            }
          },
          "security": [{ "apiKeyAuth": [] }]
        },
        "disable_studio_auth": false
      }
    }
    ```

    有关详细信息，请参阅 [身份验证概念指南](../../concepts/auth.md)，有关该过程的实际演练，请参阅 [设置自定义身份验证](../../tutorials/auth/getting_started.md) 指南。

    #### 配置存储项目生存时间 (Configuring Store Item Time-to-Live (TTL))

    您可以使用 `store.ttl` 键为 BaseStore 中的项目/记忆配置默认数据过期时间。这确定了项目在上次访问后保留的时间（读取可能会根据 `refresh_on_read` 刷新计时器）。请注意，可以通过修改 `get`、`search` 等中的相应参数，在每次调用的基础上覆盖这些默认值。
    
    `ttl` 配置是一个包含可选字段的对象：

    - `refresh_on_read`: 如果为 `true`（默认值），则通过 `get` 或 `search` 访问项目会重置其过期计时器。设置为 `false` 仅在写入（`put`）时刷新 TTL。
    - `default_ttl`: 项目的默认生存时间（以 **分钟** 为单位）。如果未设置，则默认情况下项目不会过期。
    - `sweep_interval_minutes`: 系统运行后台进程以删除过期项目的频率（以分钟为单位）。如果未设置，则不会自动进行清理。

    这是一个启用 7 天 TTL（10080 分钟），在读取时刷新并每小时清理一次的示例：

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "memory_agent": "./agent/graph.py:graph"
      },
      "store": {
        "ttl": {
          "refresh_on_read": true,
          "sweep_interval_minutes": 60,
          "default_ttl": 10080 
        }
      }
    }
    ```

    #### 配置检查点生存时间 (Configuring Checkpoint Time-to-Live (TTL))

    您可以使用 `checkpointer` 键为检查点配置生存时间 (TTL)。这决定了检查点数据在根据指定策略（例如删除）自动处理之前保留多长时间。`ttl` 配置是一个包含以下内容的对象：

    - `strategy`: 对过期检查点采取的操作（当前 `"delete"` 是唯一接受的选项）。
    - `sweep_interval_minutes`: 系统检查过期检查点的频率（以分钟为单位）。
    - `default_ttl`: 检查点的默认生存时间（以 **分钟** 为单位）。

    这是一个设置默认 TTL 为 30 天（43200 分钟）的示例：

    ```json
    {
      "dependencies": ["."],
      "graphs": {
        "chat": "./chat/graph.py:graph"
      },
      "checkpointer": {
        "ttl": {
          "strategy": "delete",
          "sweep_interval_minutes": 10,
          "default_ttl": 43200
        }
      }
    }
    ```

    在此示例中，早于 30 天的检查点将被删除，并且检查每 10 分钟运行一次。


=== "JS"
    
    #### 基本配置 (Basic Configuration)

    ```json
    {
      "graphs": {
        "chat": "./src/graph.ts:graph"
      }
    }
    ```


## 命令 (Commands)

**用法 (Usage)**

=== "Python"

    LangGraph CLI 的基本命令是 `langgraph`。

    ```
    langgraph [OPTIONS] COMMAND [ARGS]
    ```
=== "JS"

    LangGraph.js CLI 的基本命令是 `langgraphjs`。

    ```
    npx @langchain/langgraph-cli [OPTIONS] COMMAND [ARGS]
    ```

    我们建议使用 `npx` 以始终使用最新版本的 CLI。

### `dev`

=== "Python"

    在开发模式下运行 LangGraph API 服务器，具有热重载和调试功能。此轻量级服务器不需要安装 Docker，适用于开发和测试。状态持久保存到本地目录。

    !!! note

        目前，CLI 仅支持 Python >= 3.11。

    **安装 (Installation)**

    此命令需要安装 "inmem" 额外组件：

    ```bash
    pip install -U "langgraph-cli[inmem]"
    ```

    **用法 (Usage)**

    ```
    langgraph dev [OPTIONS]
    ```

    **选项 (Options)**

    | 选项 (Option)                 | 默认值 (Default) | 描述 (Description)                                                                  |
    | ----------------------------- | ---------------- | ----------------------------------------------------------------------------------- |
    | `-c, --config FILE`           | `langgraph.json` | 声明依赖项、图和环境变量的配置文件的路径 |
    | `--host TEXT`                 | `127.0.0.1`      | 服务器绑定的主机                                                          |
    | `--port INTEGER`              | `2024`           | 服务器绑定的端口                                                          |
    | `--no-reload`                 |                  | 禁用自动重新加载                                                                 |
    | `--n-jobs-per-worker INTEGER` |                  | 每个工人的作业数。默认为 10                                            |
    | `--debug-port INTEGER`        |                  | 调试器监听的端口                                                      |
    | `--wait-for-client`           | `False`          | 在启动服务器之前等待调试器客户端连接到调试端口   |
    | `--no-browser`                |                  | 跳过在服务器启动时自动打开浏览器                       |
    | `--studio-url TEXT`           |                  | 要连接的 LangGraph Studio 实例的 URL。默认为 https://smith.langchain.com |
    | `--allow-blocking`            | `False`          | 在代码中不为同步 I/O 阻塞操作引发错误 (在 `0.2.6` 添加)|
    | `--tunnel`                    | `False`          | 通过公共隧道 (Cloudflare) 公开本地服务器以进行远程前端访问。这避免了浏览器（如 Safari）或阻止本地主机连接的网络的问题        |
    | `--help`                      |                  | 显示命令文档                                                       |


=== "JS"

    在开发模式下运行 LangGraph API 服务器，具有热重载功能。此轻量级服务器不需要安装 Docker，适用于开发和测试。状态持久保存到本地目录。

    **用法 (Usage)**

    ```
    npx @langchain/langgraph-cli dev [OPTIONS]
    ```

    **选项 (Options)**

    | 选项 (Option)                 | 默认值 (Default) | 描述 (Description)                                                                  |
    | ----------------------------- | ---------------- | ----------------------------------------------------------------------------------- |
    | `-c, --config FILE`           | `langgraph.json` | 声明依赖项、图和环境变量的配置文件的路径 |
    | `--host TEXT`                 | `127.0.0.1`      | 服务器绑定的主机                                                          |
    | `--port INTEGER`              | `2024`           | 服务器绑定的端口                                                          |
    | `--no-reload`                 |                  | 禁用自动重新加载                                                                 |
    | `--n-jobs-per-worker INTEGER` |                  | 每个工人的作业数。默认为 10                                            |
    | `--debug-port INTEGER`        |                  | 调试器监听的端口                                                      |
    | `--wait-for-client`           | `False`          | 在启动服务器之前等待调试器客户端连接到调试端口   |
    | `--no-browser`                |                  | 跳过在服务器启动时自动打开浏览器                       |
    | `--studio-url TEXT`           |                  | 要连接的 LangGraph Studio 实例的 URL。默认为 https://smith.langchain.com |
    | `--allow-blocking`            | `False`          | 在代码中不为同步 I/O 阻塞操作引发错误            |
    | `--tunnel`                    | `False`          | 通过公共隧道 (Cloudflare) 公开本地服务器以进行远程前端访问。这避免了浏览器或阻止本地主机连接的网络的问题        |
    | `--help`                      |                  | 显示命令文档                                                       |

### `build`

=== "Python"

    构建 LangGraph Platform API 服务器 Docker 映像。

    **用法 (Usage)**

    ```
    langgraph build [OPTIONS]
    ```

    **选项 (Options)**

    | 选项 (Option)        | 默认值 (Default) | 描述 (Description)                                                                                              |
    | -------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------- |
    | `--platform TEXT`    |                  | 为其构建 Docker 映像的目标平台。示例：`langgraph build --platform linux/amd64,linux/arm64`              |
    | `-t, --tag TEXT`     |                  | **必需**. Docker 映像的标签。示例：`langgraph build -t my-image`                                               |
    | `--pull / --no-pull` | `--pull`         | 使用最新的远程 Docker 映像构建。使用 `--no-pull` 运行带有本地构建映像的 LangGraph Platform API 服务器。 |
    | `-c, --config FILE`  | `langgraph.json` | 声明依赖项、图和环境变量的配置文件的路径。                                         |
    | `--help`             |                  | 显示命令文档。                                                                                               |

=== "JS"

    构建 LangGraph Platform API 服务器 Docker 映像。

    **用法 (Usage)**

    ```
    npx @langchain/langgraph-cli build [OPTIONS]
    ```

    **选项 (Options)**

    | 选项 (Option)        | 默认值 (Default) | 描述 (Description)                                                                                              |
    | -------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------- |
    | `--platform TEXT`    |                  | 为其构建 Docker 映像的目标平台。示例：`langgraph build --platform linux/amd64,linux/arm64`              |
    | `-t, --tag TEXT`     |                  | **必需**. Docker 映像的标签。示例：`langgraph build -t my-image`                                               |
    | `--no-pull`          |                  | 使用本地构建的映像。默认为 `false` 以使用最新的远程 Docker 映像构建。                                      |
    | `-c, --config FILE`  | `langgraph.json` | 声明依赖项、图和环境变量的配置文件的路径。                                         |
    | `--help`             |                  | 显示命令文档。                                                                                               |


### `up`

=== "Python"

    启动 LangGraph API 服务器。对于本地测试，需要可以访问 LangGraph Platform 的 LangSmith API 密钥。生产使用需要许可证密钥。

    **用法 (Usage)**

    ```
    langgraph up [OPTIONS]
    ```

    **选项 (Options)**

    | 选项 (Option)                | 默认值 (Default)          | 描述 (Description)                                                                                                      |
    | ---------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
    | `--wait`                     |                           | 在返回之前等待服务启动。隐含 --detach                                                           |
    | `--base-image TEXT`          | `langchain/langgraph-api`  | 用于 LangGraph API 服务器的基本映像。使用版本标签固定到特定版本。                            |
    | `--image TEXT`               |                           | 用于 langgraph-api 服务的 Docker 映像。如果指定，则跳过构建并直接使用此映像。           |
    | `--postgres-uri TEXT`        | Local database            | 用于数据库的 Postgres URI。                                                                                   |
    | `--watch`                    |                           | 在文件更改时重新启动                                                                                                 |
    | `--debugger-base-url TEXT`   | `http://127.0.0.1:[PORT]` | 调试器用于访问 LangGraph API 的 URL。                                                                       |
    | `--debugger-port INTEGER`    |                           | 在本地拉取调试器映像并在指定端口上提供 UI                                                      |
    | `--verbose`                  |                           | 显示来自服务器日志的更多输出。                                                                                  |
    | `-c, --config FILE`          | `langgraph.json`          | 声明依赖项、图和环境变量的配置文件的路径。                                    |
    | `-d, --docker-compose FILE`  |                           | 包含要启动的其他服务的 docker-compose.yml 文件的路径。                                                     |
    | `-p, --port INTEGER`         | `8123`                    | 要公开的端口。示例：`langgraph up --port 8000`                                                                     |
    | `--pull / --no-pull`         | `pull`                    | 拉取最新映像。使用 `--no-pull` 运行带有本地构建映像的服务器。示例：`langgraph up --no-pull` |
    | `--recreate / --no-recreate` | `no-recreate`             | 即使容器的配置和映像未更改，也重新创建容器                                               |
    | `--help`                     |                           | 显示命令文档。                                                                                          |

=== "JS"

    启动 LangGraph API 服务器。对于本地测试，需要可以访问 LangGraph Platform 的 LangSmith API 密钥。生产使用需要许可证密钥。

    **用法 (Usage)**

    ```
    npx @langchain/langgraph-cli up [OPTIONS]
    ```

    **选项 (Options)**

    | 选项 (Option)                                                          | 默认值 (Default)          | 描述 (Description)                                                                                                      |
    | ---------------------------------------------------------------------- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
    | <span style="white-space: nowrap;">`--wait`</span>                     |                           | 在返回之前等待服务启动。隐含 --detach                                                           |
    | <span style="white-space: nowrap;">`--base-image TEXT`</span>          | <span style="white-space: nowrap;">`langchain/langgraph-api`</span> | 用于 LangGraph API 服务器的基本映像。使用版本标签固定到特定版本。 |
    | <span style="white-space: nowrap;">`--image TEXT`</span>               |                           | 用于 langgraph-api 服务的 Docker 映像。如果指定，则跳过构建并直接使用此映像。 |
    | <span style="white-space: nowrap;">`--postgres-uri TEXT`</span>        | Local database            | 用于数据库的 Postgres URI。                                                                                   |
    | <span style="white-space: nowrap;">`--watch`</span>                    |                           | 在文件更改时重新启动                                                                                                 |
    | <span style="white-space: nowrap;">`-c, --config FILE`</span>          | `langgraph.json`          | 声明依赖项、图和环境变量的配置文件的路径。                                    |
    | <span style="white-space: nowrap;">`-d, --docker-compose FILE`</span>  |                           | 包含要启动的其他服务的 docker-compose.yml 文件的路径。                                                     |
    | <span style="white-space: nowrap;">`-p, --port INTEGER`</span>         | `8123`                    | 要公开的端口。示例：`langgraph up --port 8000`                                                                     |
    | <span style="white-space: nowrap;">`--no-pull`</span>                  |                           | 使用本地构建的映像。默认为 `false` 以使用最新的远程 Docker 映像构建。                                 |
    | <span style="white-space: nowrap;">`--recreate`</span>                 |                           | 即使容器的配置和映像未更改，也重新创建容器                                               |
    | <span style="white-space: nowrap;">`--help`</span>                     |                           | 显示命令文档。                                                                                          |

### `dockerfile`

=== "Python"

    生成用于构建 LangGraph Platform API 服务器 Docker 映像的 Dockerfile。

    **用法 (Usage)**

    ```
    langgraph dockerfile [OPTIONS] SAVE_PATH
    ```

    **选项 (Options)**

    | 选项 (Option)       | 默认值 (Default) | 描述 (Description)                                                                                              |
    | ------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------- |
    | `-c, --config FILE` | `langgraph.json` | 声明依赖项、图和环境变量的 [配置文件](#configuration-file) 的路径。 |
    | `--help`            |                  | 显示此消息并退出。                                                                                     |

    示例：

    ```bash
    langgraph dockerfile -c langgraph.json Dockerfile
    ```

    这将生成一个类似于以下的 Dockerfile：

    ```dockerfile
    FROM langchain/langgraph-api:3.11

    ADD ./pipconf.txt /pipconfig.txt

    RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir -c /api/constraints.txt langchain_community langchain_anthropic langchain_openai wikipedia scikit-learn

    ADD ./graphs /deps/outer-graphs/src
    RUN set -ex && \
        for line in '[project]' \
                    'name = "graphs"' \
                    'version = "0.1"' \
                    '[tool.setuptools.package-data]' \
                    '"*" = ["**/*"]'; do \
            echo "$line" >> /deps/outer-graphs/pyproject.toml; \
        done

    RUN PIP_CONFIG_FILE=/pipconfig.txt PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir -c /api/constraints.txt -e /deps/*

    ENV LANGSERVE_GRAPHS='{"agent": "/deps/outer-graphs/src/agent.py:graph", "storm": "/deps/outer-graphs/src/storm.py:graph"}'
    ```

    ???+ note "更新您的 langgraph.json 文件"
         `langgraph dockerfile` 命令将 `langgraph.json` 文件中的所有配置转换为 Dockerfile 命令。使用此命令时，每当更新 `langgraph.json` 文件时，都必须重新运行它。否则，构建或运行 dockerfile 时，您的更改将不会反映出来。

=== "JS"

    生成用于构建 LangGraph Platform API 服务器 Docker 映像的 Dockerfile。

    **用法 (Usage)**

    ```
    npx @langchain/langgraph-cli dockerfile [OPTIONS] SAVE_PATH
    ```

    **选项 (Options)**

    | 选项 (Option)       | 默认值 (Default) | 描述 (Description)                                                                                              |
    | ------------------- | ---------------- | --------------------------------------------------------------------------------------------------------------- |
    | `-c, --config FILE` | `langgraph.json` | 声明依赖项、图和环境变量的 [配置文件](#configuration-file) 的路径。 |
    | `--help`            |                  | 显示此消息并退出。                                                                                     |

    示例：

    ```bash
    npx @langchain/langgraph-cli dockerfile -c langgraph.json Dockerfile
    ```

    这将生成一个类似于以下的 Dockerfile：

    ```dockerfile
    FROM langchain/langgraphjs-api:20
    
    ADD . /deps/agent
    
    RUN cd /deps/agent && yarn install
    
    ENV LANGSERVE_GRAPHS='{"agent":"./src/react_agent/graph.ts:graph"}'
    
    WORKDIR /deps/agent
    
    RUN (test ! -f /api/langgraph_api/js/build.mts && echo "Prebuild script not found, skipping") || tsx /api/langgraph_api/js/build.mts
    ```

    ???+ note "更新您的 langgraph.json 文件"
         `npx @langchain/langgraph-cli dockerfile` 命令将 `langgraph.json` 文件中的所有配置转换为 Dockerfile 命令。使用此命令时，每当更新 `langgraph.json` 文件时，都必须重新运行它。否则，构建或运行 dockerfile 时，您的更改将不会反映出来。
