# Playbook: Создание кодового агента

Пошаговое руководство по созданию агента, который генерирует код файлов и структуру проекта.

---

## Что получим

Агент, который:
- Генерирует код на Python (и других языках)
- Создает структуру директорий
- Записывает файлы в указанную директорию
- Проверяет синтаксис созданного кода
- Работает через HTTP API и ACP

---

## Архитектура кодового агента

```
Пользовательский запрос (например: "создай Flask API с CRUD")
        ↓
[reasoning_tool] — анализ задачи и планирование
        ↓
[generate_structure_tool] — генерация структуры проекта
        ↓
[write_file_tool] — запись файлов с кодом
        ↓
[run_command_tool] — проверка синтаксиса (python -m py_compile)
        ↓
[final_answer_tool] — отчет о созданных файлах
```

---

## Шаг 1. Создание структуры проекта

```bash
cd sgr-agent-core
mkdir -p examples/sgr_code_agent/tools
```

---

## Шаг 2. Создание инструментов для работы с файлами

### 2.1. WriteFileTool — запись файлов

Создайте файл `examples/sgr_code_agent/tools/write_file_tool.py`:

```python
"""Tool for writing files."""

from pathlib import Path
from typing import Any

from pydantic import BaseModel, Field

from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext
from sgr_agent_core.agent_definition import AgentConfig


class WriteFileInput(BaseModel):
    """Input for WriteFileTool."""

    file_path: str = Field(
        description="Relative path to the file to write (e.g., 'src/main.py')"
    )
    content: str = Field(
        description="Content to write to the file"
    )
    append: bool = Field(
        default=False,
        description="If True, append to file instead of overwriting"
    )


class WriteFileTool(BaseTool):
    """Tool for writing files to the working directory.

    Creates parent directories if they don't exist.
    Respects working_directory boundary for security.
    """

    tool_name = "write_file_tool"
    reasoning: str
    file_path: str
    content: str
    append: bool = False

    def model_post_init(self, __context: Any) -> None:
        self.args_model = WriteFileInput(
            file_path=self.file_path,
            content=self.content,
            append=self.append
        )

    async def __call__(
        self,
        context: AgentContext,
        config: AgentConfig,
    ) -> str:
        """Write file to working directory."""
        # Get working directory from config or use default
        working_dir = Path(
            getattr(config, "working_directory", ".")
        ).resolve()

        # Resolve target path and ensure it's within working directory
        target_path = (working_dir / self.file_path).resolve()

        # Security check: prevent directory traversal
        try:
            target_path.relative_to(working_dir)
        except ValueError:
            return (
                f"Error: Path '{self.file_path}' is outside "
                f"working directory '{working_dir}'"
            )

        # Create parent directories
        target_path.parent.mkdir(parents=True, exist_ok=True)

        # Write file
        mode = "a" if self.append else "w"
        encoding = "utf-8"

        try:
            with open(target_path, mode, encoding=encoding) as f:
                f.write(self.content)

            action = "appended to" if self.append else "wrote"
            size = len(self.content.encode(encoding))
            return (
                f"Successfully {action} file: {target_path}\n"
                f"Size: {size} bytes\n"
                f"Lines: {self.content.count(chr(10)) + 1}"
            )
        except Exception as e:
            return f"Error writing file: {type(e).__name__}: {e}"
```

### 2.2. CreateDirectoryTool — создание директорий

Создайте файл `examples/sgr_code_agent/tools/create_directory_tool.py`:

```python
"""Tool for creating directories."""

from pathlib import Path
from typing import Any

from pydantic import BaseModel, Field

from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext
from sgr_agent_core.agent_definition import AgentConfig


class CreateDirectoryInput(BaseModel):
    """Input for CreateDirectoryTool."""

    dir_path: str = Field(
        description="Relative path to directory to create (e.g., 'src/utils')"
    )
    exist_ok: bool = Field(
        default=True,
        description="If True, don't raise error if directory exists"
    )


class CreateDirectoryTool(BaseTool):
    """Tool for creating directories within working directory."""

    tool_name = "create_directory_tool"
    reasoning: str
    dir_path: str
    exist_ok: bool = True

    def model_post_init(self, __context: Any) -> None:
        self.args_model = CreateDirectoryInput(
            dir_path=self.dir_path,
            exist_ok=self.exist_ok
        )

    async def __call__(
        self,
        context: AgentContext,
        config: AgentConfig,
    ) -> str:
        """Create directory in working directory."""
        working_dir = Path(
            getattr(config, "working_directory", ".")
        ).resolve()

        target_path = (working_dir / self.dir_path).resolve()

        # Security check
        try:
            target_path.relative_to(working_dir)
        except ValueError:
            return (
                f"Error: Path '{self.dir_path}' is outside "
                f"working directory '{working_dir}'"
            )

        try:
            target_path.mkdir(parents=True, exist_ok=self.exist_ok)
            return f"Created directory: {target_path}"
        except Exception as e:
            return f"Error creating directory: {type(e).__name__}: {e}"
```

### 2.3. ListFilesTool — просмотр созданных файлов

Создайте файл `examples/sgr_code_agent/tools/list_files_tool.py`:

```python
"""Tool for listing created files."""

from pathlib import Path
from typing import Any

from pydantic import BaseModel, Field

from sgr_agent_core.base_tool import BaseTool
from sgr_agent_core.models import AgentContext
from sgr_agent_core.agent_definition import AgentConfig


class ListFilesInput(BaseModel):
    """Input for ListFilesTool."""

    pattern: str = Field(
        default="**/*",
        description="Glob pattern to match files (e.g., '*.py')"
    )


class ListFilesTool(BaseTool):
    """Tool for listing files in working directory."""

    tool_name = "list_files_tool"
    reasoning: str
    pattern: str = "**/*"

    def model_post_init(self, __context: Any) -> None:
        self.args_model = ListFilesInput(pattern=self.pattern)

    async def __call__(
        self,
        context: AgentContext,
        config: AgentConfig,
    ) -> str:
        """List files matching pattern."""
        working_dir = Path(
            getattr(config, "working_directory", ".")
        ).resolve()

        try:
            files = sorted(working_dir.glob(self.pattern))
            files = [f for f in files if f.is_file()]

            if not files:
                return f"No files matching '{self.pattern}' in {working_dir}"

            result = [f"Files in {working_dir}:", ""]
            for f in files:
                rel_path = f.relative_to(working_dir)
                size = f.stat().st_size
                result.append(f"  {rel_path} ({size} bytes)")

            return "\n".join(result)
        except Exception as e:
            return f"Error listing files: {type(e).__name__}: {e}"
```

### 2.4. __init__.py для тулы

Создайте файл `examples/sgr_code_agent/tools/__init__.py`:

```python
"""Tools for code agent."""

from .create_directory_tool import CreateDirectoryTool
from .list_files_tool import ListFilesTool
from .write_file_tool import WriteFileTool

__all__ = [
    "CreateDirectoryTool",
    "ListFilesTool",
    "WriteFileTool",
]
```

---

## Шаг 3. Создание кодового агента

Создайте файл `examples/sgr_code_agent/sgr_code_agent.py`:

```python
"""Code generation agent."""

from typing import Type

from openai import AsyncOpenAI, pydantic_function_tool
from openai.types.chat import ChatCompletionFunctionToolParam

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.agents.sgr_tool_calling_agent import SGRToolCallingAgent
from sgr_agent_core.tools import (
    BaseTool,
    ClarificationTool,
    FinalAnswerTool,
    ReasoningTool,
    RunCommandTool,
)

from .tools import CreateDirectoryTool, ListFilesTool, WriteFileTool


class SGRCodeAgent(SGRToolCallingAgent):
    """Agent for code generation.

    Capabilities:
    - Generate project structure
    - Write code files
    - Create directories
    - Run commands (syntax check, tests)
    - List created files

    Usage:
        agent = SGRCodeAgent(
            task_messages=[{"role": "user", "content": "Create a Flask API"}],
            openai_client=client,
            agent_config=config,
            toolkit=[],
            working_directory="./generated"
        )
    """

    name: str = "sgr_code_agent"

    def __init__(
        self,
        task_messages: list,
        openai_client: AsyncOpenAI,
        agent_config: AgentConfig,
        toolkit: list[Type[BaseTool]],
        def_name: str | None = None,
        working_directory: str | None = None,
        **kwargs: dict,
    ):
        code_tools = [
            WriteFileTool,
            CreateDirectoryTool,
            ListFilesTool,
            RunCommandTool,
        ]
        # Merge code tools with provided toolkit
        merged_toolkit = code_tools + [t for t in toolkit if t not in code_tools]

        super().__init__(
            task_messages=task_messages,
            openai_client=openai_client,
            agent_config=agent_config,
            toolkit=merged_toolkit,
            def_name=def_name,
            **kwargs,
        )
        if working_directory is None:
            working_directory = getattr(agent_config, "working_directory", "./generated")
        self.working_directory = working_directory

    async def _prepare_tools(self) -> list[ChatCompletionFunctionToolParam]:
        """Prepare tools with iteration limits."""
        tools = set(self.toolkit)

        if self._context.iteration >= self.config.execution.max_iterations:
            # Force finalization on last iteration
            tools = {
                ReasoningTool,
                ListFilesTool,
                FinalAnswerTool,
            }

        if self._context.clarifications_used >= self.config.execution.max_clarifications:
            tools -= {ClarificationTool}

        return [pydantic_function_tool(tool, name=tool.tool_name) for tool in tools]
```

---

## Шаг 4. Конфигурация агента

Создайте файл `examples/sgr_code_agent/config.yaml.example`:

```yaml
# SGR Code Agent Configuration
# Copy to config.yaml and fill in your API keys

llm:
  api_key: "your-openai-api-key-here"
  base_url: "https://api.openai.com/v1"
  model: "gpt-4.1-mini"
  max_tokens: 8000
  temperature: 0.3  # Lower temperature for more deterministic code

execution:
  max_clarifications: 2
  max_iterations: 15
  logs_dir: "examples/sgr_code_agent/logs"
  # Allow safe shell commands for syntax checking
  safe_shell_commands:
    - "python"
    - "python3"
    - "pip"
    - "pytest"
    - "flake8"
    - "black"
    - "mypy"

# ACP Configuration
acp:
  agent: sgr_code_agent

# Tool Configuration
tools:
  # Core system tools
  reasoning_tool: {}
  final_answer_tool: {}
  clarification_tool: {}
  run_command_tool:
    timeout: 30

  # File operation tools
  write_file_tool:
    base_class: "tools.WriteFileTool"
  create_directory_tool:
    base_class: "tools.CreateDirectoryTool"
  list_files_tool:
    base_class: "tools.ListFilesTool"

# Agent Definition
agents:
  sgr_code_agent:
    base_class: "sgr_code_agent.SGRCodeAgent"
    working_directory: "./generated"
    execution:
      max_iterations: 15
      max_clarifications: 2
      logs_dir: "examples/sgr_code_agent/logs"
    tools:
      - "reasoning_tool"
      - "clarification_tool"
      - "final_answer_tool"
      - "write_file_tool"
      - "create_directory_tool"
      - "list_files_tool"
      - "run_command_tool"
```

Скопируйте и настройте:

```bash
cp examples/sgr_code_agent/config.yaml.example examples/sgr_code_agent/config.yaml
# Отредактируйте config.yaml - добавьте свои API ключи
```

---

## Шаг 5. Подготовка рабочей директории

```bash
mkdir -p examples/sgr_code_agent/logs
mkdir -p generated  # Директория для создаваемых файлов
```

---

## Шаг 6. Установка пакета в режиме разработки

```bash
cd sgr-agent-core
pip install -e .
```

**Проверка что все импортируется:**
```bash
python -c "from examples.sgr_code_agent.sgr_code_agent import SGRCodeAgent; print('OK')"
python -c "from examples.sgr_code_agent.tools import WriteFileTool; print('OK')"
```

---

## Шаг 7. Запуск сервера

```bash
sgr -c examples/sgr_code_agent/config.yaml
```

**Ожидаемый вывод:**
```
INFO:     Started server process [...]
INFO:     Uvicorn running on http://0.0.0.0:8010
```

---

## Шаг 8. Тестирование кодового агента

### 8.1. Простой тест — создание Python скрипта

```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_code_agent",
    "messages": [
      {"role": "user", "content": "Создай Python скрипт, который вычисляет числа Фибоначчи. Файл: fib.py"}
    ],
    "stream": true
  }'
```

**Ожидаемое поведение:**
- Агент создает файл `generated/fib.py`
- Проверяет синтаксис через `python -m py_compile`
- Возвращает отчет о созданном файле

### 8.2. Создание структуры проекта

```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_code_agent",
    "messages": [
      {"role": "user", "content": "Создай структуру Python проекта для CLI тулзы, которая конвертирует JSON в CSV. Используй best practices: src/, tests/, requirements.txt, README.md"}
    ],
    "stream": true
  }'
```

**Ожидаемое поведение:**
- Создана директория `generated/src/`
- Создана директория `generated/tests/`
- Созданы файлы:
  - `src/__init__.py`
  - `src/converter.py`
  - `src/cli.py`
  - `tests/__init__.py`
  - `tests/test_converter.py`
  - `requirements.txt`
  - `README.md`

### 8.3. Создание Flask API

```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_code_agent",
    "messages": [
      {"role": "user", "content": "Создай Flask REST API с CRUD операциями для модели User (name, email). Используй SQLAlchemy и миграции Alembic."}
    ],
    "stream": true
  }'
```

### 8.4. Создание с проверкой синтаксиса

```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_code_agent",
    "messages": [
      {"role": "user", "content": "Создай Python модуль для работы с API OpenAI. Включи типизацию с Pydantic. После создания проверь синтаксис командой python -m py_compile."}
    ],
    "stream": true
  }'
```

---

## Шаг 9. Проверка созданных файлов

### 9.1. Просмотр структуры

```bash
ls -laR generated/
```

### 9.2. Проверка конкретного файла

```bash
cat generated/fib.py
cat generated/src/converter.py
```

### 9.3. Проверка синтаксиса

```bash
python -m py_compile generated/fib.py && echo "Syntax OK"
```

### 9.4. Просмотр логов агента

```bash
ls -la examples/sgr_code_agent/logs/
cat examples/sgr_code_agent/logs/$(ls -t examples/sgr_code_agent/logs/ | head -1)
```

---

## Шаг 10. Интеграция с ACP (Obsidian, Claude Desktop)

### 10.1. Запуск в ACP режиме

```bash
sgracp -c $(pwd)/examples/sgr_code_agent/config.yaml
```

### 10.2. Настройка в Obsidian

1. Установите плагин Agent Client через BRAT:
   - BRAT repo: `https://github.com/RAIT-09/obsidian-agent-client`

2. Добавьте Custom Agent:
   - Name: `Code Generator`
   - Command: `sgracp`
   - Args: `-c /absolute/path/to/sgr-agent-core/examples/sgr_code_agent/config.yaml`

3. Тестовые запросы:
   - "Создай Python скрипт для парсинга CSV"
   - "Сгенерируй FastAPI проект с авторизацией"

---

## Диагностика проблем

### Проблема: "ModuleNotFoundError: No module named 'examples'"

**Причина:** Пакет не установлен в режиме разработки

**Решение:**
```bash
cd sgr-agent-core
pip install -e .
```

### Проблема: "Error: Path is outside working directory"

**Причина:** Агент пытается записать файл за пределами разрешенной директории

**Решение:**
- Проверьте `working_directory` в config.yaml
- Убедитесь что запросы используют относительные пути

### Проблема: Файлы не создаются

**Причина:**
- Нет прав на запись
- Директория не существует

**Решение:**
```bash
mkdir -p generated
chmod 755 generated
```

### Проблема: Агент не использует write_file_tool

**Причина:** Неправильный system prompt или неправильно зарегистрированы тулы

**Решение:**
- Проверьте что `write_file_tool` есть в списке tools в config.yaml
- Убедитесь что `base_class` указан правильно

---

## Расширение функциональности

### Добавление ReadFileTool

Если агенту нужно читать уже созданные файлы:

```python
# В sgr_code_agent.py добавьте в code_tools:
from examples.sgr_file_agent.tools import ReadFileTool

code_tools = [
    WriteFileTool,
    CreateDirectoryTool,
    ListFilesTool,
    ReadFileTool,  # Добавляем чтение
    RunCommandTool,
]
```

### Добавление шаблонов проектов

Создайте `examples/sgr_code_agent/templates/` с шаблонами и тулы для их использования:

```python
class ApplyTemplateTool(BaseTool):
    """Apply project template."""
    # ... реализация загрузки и применения шаблона
```

### Интеграция с Git

Добавьте тул для инициализации git репозитория:

```python
class GitInitTool(BaseTool):
    """Initialize git repository."""
    # ... реализация git init + начальный commit
```

---

## Примеры промптов для кодового агента

### Создание CLI приложения

```
Создай Python CLI приложение с argparse для управления задачами (todo list).
Поддерживай команды: add, list, done, delete.
Храни данные в JSON файле.
```

### Создание веб-скрапера

```
Создай веб-скрапер на BeautifulSoup + requests, который парсит заголовки
новостей с сайта. Включи обработку ошибок и rate limiting.
```

### Создание Telegram бота

```
Создай Telegram бота на python-telegram-bot с командами /start, /help,
и эхо-ответом на текстовые сообщения. Включи обработку ошибок.
```

### Создание тестов

```
Создай pytest тесты для функции calculate_discount(price, discount_percent).
Включи тесты на граничные случаи и ошибочные входные данные.
```

---

## Чек-лист проверки

- [ ] Структура `examples/sgr_code_agent/` создана
- [ ] Тулы `write_file_tool`, `create_directory_tool`, `list_files_tool` созданы
- [ ] Агент `SGRCodeAgent` реализован
- [ ] `config.yaml` заполнен (ключи, working_directory)
- [ ] Пакет установлен: `pip install -e .`
- [ ] Директория `generated/` создана
- [ ] Сервер запущен без ошибок
- [ ] Тестовый запрос создает файл
- [ ] Синтаксис созданного кода валиден
- [ ] Логи пишутся в `examples/sgr_code_agent/logs/`
- [ ] ACP режим работает (опционально)

---

## Структура созданного примера

```
examples/sgr_code_agent/
├── config.yaml              # Конфигурация агента
├── config.yaml.example      # Шаблон конфигурации
├── sgr_code_agent.py        # Реализация агента
├── tools/
│   ├── __init__.py
│   ├── write_file_tool.py       # Запись файлов
│   ├── create_directory_tool.py # Создание директорий
│   ├── list_files_tool.py       # Просмотр файлов
│   └── read_file_tool.py        # Чтение файлов (опционально)
├── logs/                    # Логи выполнения
└── templates/               # Шаблоны проектов (опционально)

generated/                   # Выходная директория
├── project1/
├── project2/
└── ...
```
