# Практика 2 - Файловый агент

Пошаговый сценарий по аналогии с первой практикой.
Здесь полный код тулов, полный код агента и полный код конфигурации.

---

## 1. Подготовка окружения

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install sgr-agent-core openai pydantic
```

---

## 2. Создание папки проекта

```bash
mkdir -pv sgr-file-agent/{tools,logs}
cd sgr-file-agent
```

Целевая структура

```text
sgr-file-agent/
├── config.yaml
├── sgr_file_agent.py
├── logs/
└── tools/
    ├── __init__.py
    ├── write_file_tool.py
    ├── list_dir_tool.py
    ├── grep_tool.py
    └── find_tool.py
```

---

## 3. Код тулов

### 3.1 `tools/write_file_tool.py`

```python
"""Tool for writing files."""

from pathlib import Path

from pydantic import Field

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext


class WriteFileTool(BaseTool):
    """Write utf-8 files inside working directory."""

    tool_name = "write_file_tool"
    reasoning: str = Field(description="Why this file operation is needed")
    file_path: str = Field(description="Relative path to file")
    content: str = Field(description="File content")
    append: bool = Field(default=False, description="Append instead of overwrite")

    async def __call__(self, context: AgentContext, config: AgentConfig) -> str:
        """Write file and return short report."""
        working_dir = Path(getattr(config, "working_directory", ".")).resolve()
        target_file = (working_dir / self.file_path).resolve()

        try:
            target_file.relative_to(working_dir)
        except ValueError:
            return f"Error - path outside working directory - {self.file_path}"

        target_file.parent.mkdir(parents=True, exist_ok=True)
        mode = "a" if self.append else "w"

        try:
            with open(target_file, mode, encoding="utf-8") as output:
                output.write(self.content)
        except Exception as exc:
            return f"Error writing file - {type(exc).__name__} - {exc}"

        action = "appended" if self.append else "written"
        bytes_count = len(self.content.encode("utf-8"))
        lines_count = self.content.count("\n") + 1 if self.content else 0
        return (
            f"File {action} successfully\n"
            f"- path: {target_file}\n"
            f"- bytes: {bytes_count}\n"
            f"- lines: {lines_count}"
        )
```

### 3.2 `tools/list_dir_tool.py`

```python
"""Tool for listing directory contents."""

from pathlib import Path

from pydantic import Field

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext


class ListDirTool(BaseTool):
    """List files and directories in working directory."""

    tool_name = "list_dir_tool"
    reasoning: str = Field(description="Why directory listing is needed")
    dir_path: str = Field(default=".", description="Relative path to directory")
    recursive: bool = Field(default=False, description="Enable recursive walk")
    include_hidden: bool = Field(default=False, description="Include hidden entries")

    async def __call__(self, context: AgentContext, config: AgentConfig) -> str:
        """List directory entries within working directory boundary."""
        working_dir = Path(getattr(config, "working_directory", ".")).resolve()
        target_dir = (working_dir / self.dir_path).resolve()

        try:
            target_dir.relative_to(working_dir)
        except ValueError:
            return f"Error - path outside working directory - {self.dir_path}"

        if not target_dir.exists():
            return f"Error - directory not found - {target_dir}"
        if not target_dir.is_dir():
            return f"Error - not a directory - {target_dir}"

        iterator = target_dir.rglob("*") if self.recursive else target_dir.iterdir()
        lines = [f"Directory listing for {target_dir}"]

        for item in sorted(iterator):
            rel_path = item.relative_to(working_dir)
            if not self.include_hidden and any(part.startswith(".") for part in rel_path.parts):
                continue
            kind = "dir" if item.is_dir() else "file"
            suffix = "/" if item.is_dir() else ""
            lines.append(f"- {kind} - {rel_path}{suffix}")

        if len(lines) == 1:
            return f"Directory is empty - {target_dir}"
        return "\n".join(lines)
```

### 3.3 `tools/grep_tool.py`

```python
"""Tool for searching text in files."""

import re
from pathlib import Path

from pydantic import Field

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext


class GrepTool(BaseTool):
    """Search across files in working directory."""

    tool_name = "grep_tool"
    reasoning: str = Field(description="Why regex search is needed")
    pattern: str = Field(description="Regex pattern")
    file_glob: str = Field(default="**/*", description="Glob for target files")
    case_sensitive: bool = Field(default=False, description="Use case-sensitive search")
    max_matches: int = Field(default=200, ge=1, le=2000, description="Limit total matches")

    async def __call__(self, context: AgentContext, config: AgentConfig) -> str:
        """Find regex matches with file and line references."""
        working_dir = Path(getattr(config, "working_directory", ".")).resolve()
        flags = 0 if self.case_sensitive else re.IGNORECASE

        try:
            regex = re.compile(self.pattern, flags)
        except re.error as exc:
            return f"Error - invalid regex - {exc}"

        matches = []
        for file_path in sorted(working_dir.glob(self.file_glob)):
            if not file_path.is_file():
                continue

            try:
                file_path.resolve().relative_to(working_dir)
            except ValueError:
                continue

            try:
                lines = file_path.read_text(encoding="utf-8").splitlines()
            except Exception:
                continue

            for line_number, line in enumerate(lines, start=1):
                if regex.search(line):
                    rel_path = file_path.relative_to(working_dir)
                    matches.append(f"{rel_path}:{line_number}:{line}")
                    if len(matches) >= self.max_matches:
                        break
            if len(matches) >= self.max_matches:
                break

        if not matches:
            return (
                "No matches found\n"
                f"- pattern: {self.pattern}\n"
                f"- file_glob: {self.file_glob}"
            )

        header = [
            "Matches",
            f"- pattern: {self.pattern}",
            f"- file_glob: {self.file_glob}",
            f"- total: {len(matches)}",
            "",
        ]
        return "\n".join(header + matches)
```

### 3.4 `tools/find_tool.py`

```python
"""Tool for finding files by name."""

from pathlib import Path

from pydantic import Field

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext


class FindTool(BaseTool):
    """Find files by glob pattern."""

    tool_name = "find_tool"
    reasoning: str = Field(description="Why file discovery is needed")
    pattern: str = Field(description="Glob pattern for file names")
    dir_path: str = Field(default=".", description="Relative directory to scan")
    max_results: int = Field(default=200, ge=1, le=5000, description="Result limit")

    async def __call__(self, context: AgentContext, config: AgentConfig) -> str:
        """Find files relative to working directory."""
        working_dir = Path(getattr(config, "working_directory", ".")).resolve()
        target_dir = (working_dir / self.dir_path).resolve()

        try:
            target_dir.relative_to(working_dir)
        except ValueError:
            return f"Error - path outside working directory - {self.dir_path}"

        if not target_dir.exists():
            return f"Error - directory not found - {target_dir}"
        if not target_dir.is_dir():
            return f"Error - not a directory - {target_dir}"

        results = []
        for file_path in sorted(target_dir.rglob(self.pattern)):
            if file_path.is_file():
                results.append(str(file_path.relative_to(working_dir)))
            if len(results) >= self.max_results:
                break

        if not results:
            return (
                "No files found\n"
                f"- pattern: {self.pattern}\n"
                f"- directory: {target_dir}"
            )

        return "\n".join(
            [
                "Found files",
                f"- pattern: {self.pattern}",
                f"- directory: {target_dir}",
                f"- total: {len(results)}",
                "",
                *[f"- {path}" for path in results],
            ]
        )
```

### 3.5 `tools/__init__.py`

```python
"""Tools package for SGR file agent."""

from .find_tool import FindTool
from .grep_tool import GrepTool
from .list_dir_tool import ListDirTool
from .write_file_tool import WriteFileTool

__all__ = [
    "FindTool",
    "GrepTool",
    "ListDirTool",
    "WriteFileTool",
]
```

---

## 4. Код агента

### `sgr_file_agent.py`

```python
"""SGR file agent with tool filtering by iteration limits."""

from typing import Type

from openai import AsyncOpenAI, pydantic_function_tool
from openai.types.chat import ChatCompletionFunctionToolParam

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.agents.tool_calling_agent import ToolCallingAgent
from sgr_agent_core.tools import (
    BaseTool,
    ClarificationTool,
    FinalAnswerTool,
)

from tools import FindTool, GrepTool, ListDirTool, WriteFileTool


class SGRFileAgent(ToolCallingAgent):
    """Agent for local file-system tasks."""

    name: str = "sgr_file_agent"

    def __init__(
        self,
        task_messages: list,
        openai_client: AsyncOpenAI,
        agent_config: AgentConfig,
        toolkit: list[Type[BaseTool]],
        def_name: str | None = None,
        **kwargs: dict,
    ):
        file_tools = [
            WriteFileTool,
            ListDirTool,
            GrepTool,
            FindTool,
        ]
        merged_toolkit = file_tools + [tool for tool in toolkit if tool not in file_tools]

        super().__init__(
            task_messages=task_messages,
            openai_client=openai_client,
            agent_config=agent_config,
            toolkit=merged_toolkit,
            def_name=def_name,
            **kwargs,
        )

    async def _prepare_tools(self) -> list[ChatCompletionFunctionToolParam]:
        """Limit tools when the iteration budget is almost exhausted."""
        tools = set(self.toolkit)

        if self._context.iteration >= self.config.execution.max_iterations:
            tools = {
                ListDirTool,
                FinalAnswerTool,
            }

        if self._context.clarifications_used >= self.config.execution.max_clarifications:
            tools -= {ClarificationTool}

        return [pydantic_function_tool(tool, name=tool.tool_name) for tool in tools]
```

---

## 5. Код конфигурации

### `config.yaml`

```yaml
llm:
  api_key: "https://t.me/evilfreelancer"
  base_url: "https://api.rpa.icu/v1"
  model: "gpt-oss:120b"
  max_tokens: 8000
  temperature: 0.2

execution:
  max_clarifications: 2
  max_iterations: 8
  logs_dir: "logs"

tools:
  reasoning_tool: {}
  clarification_tool: {}
  final_answer_tool: {}

  write_file_tool:
    base_class: "tools.WriteFileTool"
  list_dir_tool:
    base_class: "tools.ListDirTool"
  grep_tool:
    base_class: "tools.GrepTool"
  find_tool:
    base_class: "tools.FindTool"

agents:
  sgr_file_agent:
    base_class: "sgr_file_agent.SGRFileAgent"
    working_directory: "."
    execution:
      max_iterations: 8
      max_clarifications: 2
      logs_dir: "logs"
    tools:
      - "reasoning_tool"
      - "clarification_tool"
      - "final_answer_tool"
      - "write_file_tool"
      - "list_dir_tool"
      - "grep_tool"
      - "find_tool"
```

---

## 6. Проверка импортов

```bash
python -c "from sgr_file_agent import SGRFileAgent; print('OK')"
python -c "from tools import WriteFileTool, ListDirTool, GrepTool, FindTool; print('OK')"
```

---

## 7. Запуск API

```bash
sgr -c config.yaml --host 127.0.0.1 --port 8015
```

Ожидаемая строка в логах

```text
Uvicorn running on http://127.0.0.1:8015
```

---

## 8. Тестовые запросы

### 8.1 Список файлов

```bash
curl -N -X POST "http://localhost:8015/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_file_agent",
    "messages": [
      {"role": "user", "content": "Покажи структуру текущего проекта, исключи скрытые файлы"}
    ],
    "stream": true
  }'
```

### 8.2 Поиск TODO

```bash
curl -N -X POST "http://localhost:8015/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_file_agent",
    "messages": [
      {"role": "user", "content": "Найди все строки с class в python файлах внутри tools"}
    ],
    "stream": true
  }'
```

### 8.3 Поиск файлов по имени

```bash
curl -N -X POST "http://localhost:8015/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_file_agent",
    "messages": [
      {"role": "user", "content": "Найди все файлы с расширением py"}
    ],
    "stream": true
  }'
```

### 8.4 Создание файла

```bash
curl -N -X POST "http://localhost:8015/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_file_agent",
    "messages": [
      {"role": "user", "content": "Создай файл notes/out.txt с текстом hello"}
    ],
    "stream": true
  }'
```

---

## 9. Проверка логов

```bash
ls -la logs
```
