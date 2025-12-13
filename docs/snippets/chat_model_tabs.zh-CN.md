=== "OpenAI"

    ```shell
    pip install -U langchain-openai
    ```
    ```python
    import getpass
    import os

    def _set_env(var: str):
        if not os.environ.get(var):
            os.environ[var] = getpass.getpass(f"{var}: ")

    _set_env("OPENAI_API_KEY")
    from langchain_openai import ChatOpenAI

    model = ChatOpenAI(model="gpt-4o")
    ```

=== "Anthropic"

    ```shell
    pip install -U langchain-anthropic
    ```
    ```python
    import getpass
    import os

    def _set_env(var: str):
        if not os.environ.get(var):
            os.environ[var] = getpass.getpass(f"{var}: ")

    _set_env("ANTHROPIC_API_KEY")
    from langchain_anthropic import ChatAnthropic

    model = ChatAnthropic(model="claude-3-5-sonnet-20240620")
    ```

=== "Google VertexCI"

    ```shell
    pip install -U langchain-google-vertexai
    ```
    ```python
    import getpass
    import os

    def _set_env(var: str):
        if not os.environ.get(var):
            os.environ[var] = getpass.getpass(f"{var}: ")

    _set_env("GOOGLE_API_KEY")
    from langchain_google_vertexai import ChatVertexAI

    model = ChatVertexAI(model="gemini-1.5-flash")
    ```

=== "TogetherAI"

    ```shell
    pip install -U langchain-together
    ```
    ```python
    import getpass
    import os

    def _set_env(var: str):
        if not os.environ.get(var):
            os.environ[var] = getpass.getpass(f"{var}: ")

    _set_env("TOGETHER_API_KEY")
    from langchain_together import ChatTogether

    model = ChatTogether(model="meta-llama/Llama-3-70b-chat-hf")
    ```

=== "MistralAI"

    ```shell
    pip install -U langchain-mistralai
    ```
    ```python
    import getpass
    import os

    def _set_env(var: str):
        if not os.environ.get(var):
            os.environ[var] = getpass.getpass(f"{var}: ")

    _set_env("MISTRAL_API_KEY")
    from langchain_mistralai import ChatMistralAI

    model = ChatMistralAI(model="mistral-large-latest")
    ```

=== "Groq"

    ```shell
    pip install -U langchain-groq
    ```
    ```python
    import getpass
    import os

    def _set_env(var: str):
        if not os.environ.get(var):
            os.environ[var] = getpass.getpass(f"{var}: ")

    _set_env("GROQ_API_KEY")
    from langchain_groq import ChatGroq

    model = ChatGroq(model="llama3-8b-8192")
    ```

=== "Ollama"

    ```shell
    pip install -U langchain-ollama
    ```
    ```python
    from langchain_ollama import ChatOllama

    model = ChatOllama(model="llama3")
    ```
