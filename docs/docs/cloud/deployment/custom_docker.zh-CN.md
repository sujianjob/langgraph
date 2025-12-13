# 如何自定义 Dockerfile (How to customize Dockerfile)

用户可以在从父 LangGraph 镜像导入之后添加一组额外的行到 Dockerfile。为此，您只需通过将要运行的命令传递给 `dockerfile_lines` 键来修改您的 `langgraph.json` 文件。例如，如果我们想在图中使用 `Pillow`，您需要添加以下依赖项：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env",
    "dockerfile_lines": [
        "RUN apt-get update && apt-get install -y libjpeg-dev zlib1g-dev libpng-dev",
        "RUN pip install Pillow"
    ]
}
```

如果我们使用 `jpeg` 或 `png` 图像格式，这将安装使用 Pillow 所需的系统包。
