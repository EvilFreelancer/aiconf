# SGR Agent Core — Мастер-класс (AI Conf)

> **Цель мастер-класса:** провести аудиторию по пути от бизнес-запроса («хочу агента по локальным файлам») до работающего OpenAI-совместимого сервиса.
> **Фокус:** конфигурация, метрики, воспроизводимость — а не «магические промпты».

---

## Что понадобится

- Python 3.11+ и/или Docker
- Git
- API-ключ с моделью поддерживающей Structured Output

---

## Материалы и ссылки

| Ресурс | Ссылка |
|--------|--------|
| Репозиторий | `https://github.com/vamplabai/sgr-agent-core` |
| Документация | `https://vamplabai.github.io/sgr-agent-core/` |
| Obsidian Agent Client | `https://github.com/RAIT-09/obsidian-agent-client` |

---

## Тайминг и структура

| Блок | Содержание | Минуты |
|------|------------|--------|
| **0. Вступление** | Приветствие, цели, обещанный результат | ~5 |
| **1. Что такое SGR Agent Core** | Schema Guided Reasoning, отличие от обычного агента | ~10 |
| **2. Архитектура API сервера** | FastAPI, stateless/stateful, жизненный цикл, ACP | ~15 |
| **3. Конфигурация YAML и типы агентов** | Иерархия конфига, регистрация тулов, MCP | ~15 |
| **4. Подготовка окружения** | Установка, токены, настройка LLM | ~10 |
| **5. Практика: Deep Research** | Запуск готового агента, план → поиск → ответ | ~15 |
| **Перерыв** |  | 10-15 |
| **6. Практика: Файловый агент** | Файловые тулы, 4 режима работы, ACP + Obsidian | ~30-40 |
| **7. Метрики, тесты, Langfuse** | Уровни тестирования, observability | ~20 |
| **8. Домашнее задание** | Задание на самостоятельную работу | ~5 |
| **9. Завершение и Roadmap** | Резюме, roadmap, вопросы | ~5-10 |

**Итого:** ~130-140 минут (с перерывом).

---

## Блок 0. Вступление (~5 мин)

### Learning objectives
- Понять, зачем нужен SGR Agent Core в «зоопарке агентов»
- Зафиксировать обещанный результат мастер-класса
- Понять формат: теория → live demo → практика

### Что понадобится
- Python 3.11+ и/или Docker
- Git
- API-ключ с моделью поддерживающей Structured Output

### Материалы
- https://github.com/vamplabai/sgr-agent-core

### По итогу получим
- локальный файловый агент
- HTTP API
- ACP интегрированный в Obsidian

### Слайды
- Кто мы и что за проект
- Цель и обещанный результат
- Структура занятия по блокам и таймингу

### Speaker notes
> «Сегодня мы не будем писать промпты по 500 строк. Вместо этого — конфигурация, метрики и воспроизводимость. К концу занятия у вас будет локальный файловый агент с HTTP API, который можно подключить к Obsidian.»

---

## Блок 1. Что такое SGR Agent Core (~10 мин)

### Learning objectives
- Объяснить, что такое Schema Guided Reasoning
- Показать разницу между SGR-агентом и обычным чат-ботом с tool calling
- Показать стабильность и валидируемость результатов

### Что объясняем
- **Schema Guided Reasoning** — явное описание шагов рассуждения
- Чем SGR-агент отличается от обычного чат-бота с tool calling
- Как SGR помогает добиваться стабильных и валидаируемых результатов

### Код (прямое использование из Python)
```python
import asyncio
from openai import AsyncOpenAI
from sgr_agent_core import AgentConfig
from sgr_agent_core.agents.sgr_agent import SGRAgent
from sgr_agent_core.tools import FinalAnswerTool


async def main() -> None:
    client = AsyncOpenAI(api_key="YOUR_OPENAI_API_KEY")

    agent = SGRAgent(
        task_messages=[
            {"role": "user", "content": "Собери краткий обзор по теме RAG систем"},
        ],
        openai_client=client,
        agent_config=AgentConfig(),
        toolkit=[FinalAnswerTool],
    )

    result = await agent.execute()
    print(result)


if __name__ == "__main__":
    asyncio.run(main())
```

### Слайды
- Определение SGR и визуальная схема фаз (рассуждение → действия)
- Сравнение классического агента и SGR-агента
- Мини-пример кода: создание и запуск агента

### Speaker notes
> «SGR — это не просто ещё один фреймворк. Это попытка сделать рассуждения агента детерминированными и интерпретируемыми. Каждый шаг — это структурированный выбор, который можно валидировать.»

---

## Блок 2. Архитектура API сервера (~15 мин)

### Learning objectives
- Понять архитектуру FastAPI-сервера
- Различать stateless и stateful режимы
- Понять жизненный цикл агента
- Понять разницу между API Mode и ACP Mode

### 2.1. Общая архитектура

SGR Agent Core работает в двух режимах:

**API Mode** — HTTP сервер для интеграций
- OpenAI-совместимый эндпоинт /v1/chat/completions
- Интеграции: Open WebUI, LibreChat, любой OpenAI-клиент
- Stateless (без состояния) и Stateful (с сессиями)

**ACP Mode** — Agent Client Protocol
- Stdio transport для локальных агентов
- Интеграции: Obsidian, Claude Desktop, любой ACP-хост
- Stateless-only, контекст через threads

### 2.2. API Mode — REST сервер

FastAPI сервер с хранилищем агентов

Жизненный цикл агента:
1. Создание — загрузка конфига, инициализация тулов
2. Выполнение — обработка запроса, SGR пайплайн
3. Очистка — освобождение ресурсов

Режимы работы:
- Stateless — каждый запрос независимый
- Stateful — сохраняется контекст диалога

Интеграции:
- Open WebUI, LibreChat, ChatGPT-Next-Web
- Любой клиент с OpenAI-compatible API

### 2.3. ACP Mode — Agent Client Protocol

Что такое ACP:
- Протокол для локальных агентов (stdio transport)
- Агент запускается как подпроцесс, общается через JSON-RPC
- Поддержка tools, resources, prompts

Интеграции:
- Obsidian + Copilot plugin
- Claude Desktop
- Cursor и другие IDE

### Команды
**Базовый стриминговый запрос:**
```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_agent",
    "messages": [
      {"role": "user", "content": "Исследуй рынок RAG систем и сделай краткий вывод"}
    ],
    "stream": true
  }'
```

**Stateful-запрос по `agent_id`:**
```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"sgr_agent_12345678-1234-1234-1234-123456789012\",
    \"messages\": [
      {\"role\": \"user\", \"content\": \"Вот уточнение к предыдущему вопросу\"}
    ],
    \"stream\": true
  }"
```

### Слайды
- Sequence-диаграмма: клиент → сервер → тул → ответ
- Две дорожки: stateless и stateful
- OpenAI-совместимый протокол и совместимые клиенты
- QR-код на спецификацию ACP

### Speaker notes
> «Stateful-режим удобен, когда вы хотите продолжить диалог без пересылки всей истории. Но помните: агент живёт в памяти сервера и имеет таймаут.»

---

## Блок 3. Конфигурация YAML и типы агентов (~15 мин)

### Learning objectives
- Понять иерархию конфигурации
- Уметь создавать минимальный `config.yaml`
- Различать типы агентов
- Уметь регистрировать тул и подключать MCP

### 3.1. Иерархия конфигурации

Наследование настроек (приоритет снизу вверх):
```
defaults (в коде)  ← базовые значения
     ↓
env                ← переменные окружения
     ↓
config.yaml        ← файл конфигурации
     ↓
CLI args           ← аргументы командной строки
```

Структура YAML-файла:
```
llm:              # настройки LLM
execution:        # лимиты и директории
tools:            # регистрация тулов
mcp:              # MCP серверы
agents:           # определение агентов
acp:              # ACP настройки
```

### 3.2. Регистрация тулов

Схема определения тулов в конфиге:
```yaml
tools:
  # Простая регистрация (только base_class)
  reasoning_tool: {}
  final_answer_tool: {}

  # С конфигурацией
  web_search_tool:
    engine: "tavily"
    api_key: "${TAVILY_API_KEY}"
    max_results: 10

  # Кастомный тул из модуля
  read_file_tool:
    base_class: "tools.ReadFileTool"
```

Тулы подключаются к агенту по имени:
```yaml
agent:
  tools:
    - "web_search_tool"
    - "extract_page_content_tool"
    - "create_report_tool"
```

### 3.3. MCP интеграция

MCP (Model Context Protocol) — внешние серверы с тулами:
```yaml
mcp:
  mcpServers:
    # Пример: DeepWiki MCP сервер
    deepwiki:
      url: "https://mcp.deepwiki.com/mcp"

    # Локальный stdio сервер
    filesystem:
      command: "npx"
      args: ["-y", "@modelcontextprotocol/server-filesystem"]
```

Агент автоматически получает тулы от MCP серверов.

### 3.4. Определение агентов

Схема определения агента:
```yaml
agents:
  sgr_tool_calling_agent:
    base_class: "agents.ResearchSGRToolCallingAgent"
    llm:
      model: "gpt-4.1-mini"
      temperature: 0.4
    tools:
      - "web_search_tool"
      - "extract_page_content_tool"
      - "reasoning_tool"
      - "create_report_tool"
      - "final_answer_tool"
```

Выбор агента для ACP:
```yaml
acp:
  agent: sgr_tool_calling_agent
```

### Таксономия — Тулы и Агенты

**Системные тулы:**
- `reasoning_tool` — анализ ситуации и планирование следующего шага
- `generate_plan_tool` — генерация исследовательского плана
- `adapt_plan_tool` — адаптация плана на основе новых данных
- `clarification_tool` — запрос уточнения при неоднозначности
- `final_answer_tool` — финализация задачи с ответом агента
- `answer_tool` — промежуточный ответ с продолжением диалога
- `web_search_tool` — поиск (Tavily, Brave, Perplexity)
- `extract_page_content_tool` — извлечение контента из URL
- `create_report_tool` — создание файла-отчета с цитированием
- `run_command_tool` — выполнение shell-команд (safe/unsafe режимы)

**Типы агентов:**
- `sgr_agent` / `sgr_tool_calling_agent` — полный SGR пайплайн с reasoning
- `tool_calling_agent` — агент с тулколлингом, без SGR
- `dialog_agent` — диалоговый агент с промежуточными результатами
- `iron_agent` — упрощенный агент без tool calling

### Слайды
- Схема иерархии конфигурации
- Таблица типов агентов и их ролей
- Упрощённый пример `config.yaml` с выделением ключевых блоков

### Speaker notes
> «Ключевая идея — минимальные переопределения. Базовый конфиг задаёт LLM и execution, а конкретный агент только выбирает тулы и класс.»

---

## Блок 4. Подготовка окружения (~10 мин)

### Learning objectives
- Уметь запускать SGR двумя способами: Docker и pip
- Понять разницу между сервером и библиотекой

### 4.1. Установка

```bash
git clone https://github.com/vamplabai/sgr-agent-core.git
cd sgr-agent-core
pip install -e .
```

**Docker:**
```bash
git clone https://github.com/vamplabai/sgr-agent-core.git
cd sgr-agent-core

sudo mkdir -p logs reports
sudo chmod 777 logs reports

cp examples/sgr_deep_research/config.yaml.example examples/sgr_deep_research/config.yaml
nano examples/sgr_deep_research/config.yaml

docker run --rm -i \
  --name sgr-agent \
  -p 8010:8010 \
  -v "$(pwd)/examples/sgr_deep_research:/app/examples/sgr_deep_research:ro" \
  -v "$(pwd)/logs:/app/logs" \
  -v "$(pwd)/reports:/app/reports" \
  ghcr.io/vamplabai/sgr-agent-core:latest \
  --config-file /app/examples/sgr_deep_research/config.yaml \
  --host 0.0.0.0 \
  --port 8010
```

**pip install:**
```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install sgr-agent-core

cp config.yaml.example config.yaml
sgr -c config.yaml
```

### 4.2. Токены и где их взять

**OpenAI-совместимый API:**
- OpenAI: https://platform.openai.com/api-keys
- Другие провайдеры с OpenAI-compatible API

**Tavily (для поиска):**
- https://tavily.com — 1000 запросов/месяц бесплатно

**Альтернативный вариант для мастер-класса:**
- API: https://api.rpa.icu
- Ключ: https://t.me/evilfreelancer
- Модель: gpt-oss:120b

### 4.3. Пример настройки LLM для api.rpa.icu

```yaml
llm:
  base_url: "https://api.rpa.icu/v1"
  api_key: "https://t.me/evilfreelancer"
  model: "gpt-oss:120b"
```

### Чек-лист перед блоком
- [ ] Участники установили Python 3.11+ или Docker
- [ ] У всех есть API-ключ
- [ ] Проверена связь с интернетом

### Слайды
- Чеклист подготовки окружения
- Два режима запуска: Docker и локальная библиотека
- QR-код или ссылка на инструкции

### Speaker notes
> «Если у вас Docker — запуск за 2 минуты. Если pip — больше контроля, можно копаться в коде.»

---

## Блок 5. Практика: Deep Research (~15 мин)

### Learning objectives
- Запустить готового агента deep research
- Понять пайплайн: план → поиск → извлечение → ответ
- Найти логи и отчёты

### Что объясняем
- Набор тулов для deep research: `GeneratePlan`, `AdaptPlan`, `WebSearch`, `ExtractPageContent`, `FinalAnswer`
- Как агент строит план, адаптирует его и выдаёт итоговый отчёт
- Где смотреть логи (`logs/`) и отчёты (`reports/`)

### Команды
```bash
cd sgr-agent-core
cp examples/sgr_deep_research/config.yaml.example examples/sgr_deep_research/config.yaml
nano examples/sgr_deep_research/config.yaml

sgr -c examples/sgr_deep_research/config.yaml
```

**Запрос к агенту:**
```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_tool_calling_agent",
    "messages": [
      {"role": "user", "content": "Сделай глубокое исследование рынка RAG решений"}
    ],
    "stream": true
  }'
```

### Слайды
- Схема пайплайна deep research
- Пример потока событий с вызовами тулов
- Короткий блок про метрики и бенчмарки

### Speaker notes
> «Обратите внимание на логи — там виден каждый шаг. Это не чёрный ящик, а прозрачный пайплайн, поверх которого можно строить метрики.»

---

## Перерыв (10-15 минут)

---

## Блок 6. Практика: Файловый агент (~30-40 мин)

### Learning objectives
- Перейти от research-сценария к локальному файловому агенту
- Понять границы безопасности (read-only)
- Освоить 4 режима работы агента

### 6.1. Файловые тулы

**Базовые тулы для работы с файлами:**

| Тул | Назначение | Параметры |
|-----|-----------|-----------|
| `read_file_tool` | Чтение содержимого файла | `file_path`, `offset`, `limit` |
| `write_file_tool` | Запись/дозапись в файл | `file_path`, `content` |
| `list_dir_tool` | Список файлов в директории | `dir_path`, `recursive` |
| `grep_tool` | Поиск по содержимому файла | `pattern`, `file_path` |
| `find_tool` | Быстрый поиск файлов по имени | `pattern`, `dir_path` |

### 6.2. Конфиг тулов

```yaml
tools:
  # Системные тулы
  clarification_tool: {}
  final_answer_tool: {}

  # Файловые тулы (кастомные)
  read_file_tool:
    base_class: "tools.ReadFileTool"
  write_file_tool:
    base_class: "tools.WriteFileTool"
  list_dir_tool:
    base_class: "tools.ListDirTool"
  grep_tool:
    base_class: "tools.GrepTool"
  find_tool:
    base_class: "tools.FindTool"
```

Каждый тул наследуется от `BaseTool` и определяет:
- Pydantic-модель входных параметров
- Метод `__call__` с логикой выполнения

### 6.3. Код агента

**Структура SGRFileAgent:**

```python
class SGRFileAgent(SGRToolCallingAgent):
    """Агент для работы с файловой системой."""

    def __init__(
        self,
        config: AgentConfig,
        working_directory: str = ".",
        **kwargs
    ):
        super().__init__(config, **kwargs)
        self.working_directory = Path(working_directory).resolve()

        # Проверяем и создаем рабочую директорию
        if not self.working_directory.exists():
            raise ValueError(
                f"Working directory does not exist: {self.working_directory}"
            )

    async def execute(self, context: AgentContext) -> str:
        """Выполнение с проверкой безопасности путей."""
        # Все операции ограничены working_directory
        # Предотвращаем выход за пределы рабочей директории
        return await super().execute(context)
```

**Ключевые особенности:**
- `working_directory` — рабочая директория агента (ограничение песочницы)
- Проверка путей — все операции внутри working_directory
- Наследование от `SGRToolCallingAgent` — SGR пайплайн + тулколлинг

### 6.4. Конфиг агента

```yaml
agents:
  sgr_file_agent:
    base_class: "sgr_file_agent.SGRFileAgent"
    working_directory: "."
    execution:
      max_iterations: 3
      max_clarifications: 1
      logs_dir: "logs/file_agent"
    tools:
      - "clarification_tool"
      - "final_answer_tool"
      - "get_system_paths_tool"
      - "list_dir_tool"
      - "read_file_tool"
      - "grep_tool"
      - "find_tool"
```

### 6.5. Четыре режима работы агента

| Режим | Команда / Код | Когда использовать |
|-------|---------------|-------------------|
| **1. HTTP сервер** | `sgr --config-file examples/sgr_file_agent/config.yaml` | Интеграция с любыми OpenAI-совместимыми клиентами |
| **2. Stateful** | `model: "sgr_file_agent_<agent_id>"` в curl | Продолжение диалога без пересылки истории |
| **3. Библиотека** | `from sgr_agent_core.agents.sgr_agent import SGRAgent` | Встраивание в свой Python-код |
| **4. ACP (stdio)** | `sgracp --config /abs/path/to/config.yaml` | Интеграция с Obsidian, IDE и другими MCP-клиентами |

**HTTP API с gpt-oss 120b:**
```yaml
llm:
  api_key: "YOUR_KEY"
  base_url: "https://your-openai-compatible-host/v1"
  model: "gpt-oss:120b"
  max_tokens: 4000
  temperature: 0.3
```

**Запуск сервера:**
```bash
sgr --config-file examples/sgr_file_agent/config.yaml
```

**Запрос к файловому агенту:**
```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_file_agent",
    "messages": [
      {"role": "user", "content": "Найди в заметках все упоминания SGR за последний месяц"}
    ],
    "stream": true
  }'
```

**Проверка `sgracp`:**
```bash
which sgracp
sgracp --config /absolute/path/to/sgr-agent-core/examples/sgr_file_agent/config.yaml
```

**Минимальный блок `acp` в yaml:**
```yaml
acp:
  agent: sgr_file_agent
```

**Установка плагина через BRAT:**
- BRAT: `https://github.com/RAIT-09/obsidian-agent-client`
- Настройка Custom Agents: [документация](https://rait-09.github.io/obsidian-agent-client/agent-setup/custom-agents.html)

**Пример выполнения запроса "Найди все TODO в проекте":**

1. **reasoning_tool** — анализирует задачу и планирует шаги
2. **find_tool** — находит файлы проекта
3. **grep_tool** — ищет TODO-комменты в найденных файлах
4. **final_answer_tool** — формирует отчет с результатами

### Слайды
- Схема потока: запрос → тулы ФС → финальный ответ
- Таблица 4 режимов работы
- Диаграмма: Obsidian → Agent Client → sgracp → vault

### Speaker notes
> «Работа с файлами — read-only по умолчанию. working_directory — это корень, за который агент не выйдет. Это важно для безопасности.»

> «ACP — это мост между SGR и Obsidian. Вы говорите агенту «найди в моих заметках», и он ищет в вашем vault.»

---

## Блок 7. Метрики, тесты, Langfuse (~20 мин)

### Learning objectives
- Понять уровни тестирования агента
- Уметь писать unit-тест на отдельный тул
- Понять, как подключить Langfuse для трейсинга

### Что объясняем
- Уровни тестирования: unit (отдельный тул) → интеграция (с фикстурой каталога)
- Логи в `logs/` и отчёты в `reports/`
- Регрессии при смене модели (mini → gpt-oss 120b)
- **Langfuse** для трассировки вызовов, задержек и сравнения прогонов

### Пример теста тула
```python
import pytest

from sgr_agent_core.agent_definition import AgentConfig
from sgr_agent_core.models import AgentContext

from examples.sgr_file_agent.tools.read_file_tool import ReadFileTool


@pytest.mark.asyncio
async def test_read_file_tool_smoke(tmp_path) -> None:
    f = tmp_path / "note.md"
    f.write_text("# hello", encoding="utf-8")
    tool = ReadFileTool(
        reasoning="read vault file",
        file_path=str(f),
    )
    context = AgentContext()
    config = AgentConfig()

    result = await tool(context, config)

    assert "hello" in result
```

### Метрики качества агента
| Метрика | Что измеряем |
|---------|-------------|
| Полнота ответа | Нашёл ли агент всё, что просили |
| Число лишних вызовов тулов | Эффективность рассуждений |
| Время выполнения | Latency, время на итерацию |
| Регрессия при смене модели | Сохраняется ли качество |

### Слайды
- Список метрик: полнота, число вызовов, время
- Схема observability: SGR логи + Langfuse
- Что автоматизировать в CI для агента

### Speaker notes
> «Тестируйте не только «правильный» путь. Проверяйте, что агент graceful обрабатывает отсутствующие файлы, пустые директории, странные запросы.»

---

## Блок 8. Домашнее задание (~5 мин)

### Задание 1: Настройка Langfuse

Цель: настроить трассировку выполнения агентов

Что сделать:
1. Зарегистрироваться на https://langfuse.com
2. Получить API-ключи (publicKey, secretKey)
3. Добавить в config.yaml секцию observability:
```yaml
observability:
  langfuse:
    enabled: true
    public_key: "${LANGFUSE_PUBLIC_KEY}"
    secret_key: "${LANGFUSE_SECRET_KEY}"
    host: "https://cloud.langfuse.com"
```
4. Запустить агента и проверить трейсы в UI

Что проверить: каждый шаг агента виден в трейсах

### Задание 2: ACP интеграция с VS Code

Цель: подключить файловый агент к VS Code через ACP

Что сделать:
1. Запустить SGR Agent Core в ACP режиме:
```bash
sgracp -c sgr-file-agent/config.yaml
```
2. Настроить подключение в VS Code (через расширение с поддержкой ACP)
3. Проверить команды:
   - "Найди все TODO в проекте"
   - "Покажи структуру папки src"
   - "Прочитай файл README.md"

Что проверить: агент отвечает на запросы из редактора

### Варианты
1. **Файловый агент (рекомендуется):** закрепите папку с заметками, прогоните 3–5 запросов через API или ACP, сохраните конфиг
2. **Confluence-бот (альтернатива):** если в треке есть Confluence — повторите сценарий с Confluence API

### Что проверить
- [ ] Агент отвечает на запросы о содержимом папки
- [ ] Логи пишутся в `logs/`
- [ ] Конфиг сохранён и работает после перезапуска

### Speaker notes
> «Лучший способ закрепить — запустить на своих данных. Не на демо-папке, а на своих заметках.»

---

## Блок 9. Завершение и Roadmap (~5–10 мин)

### Learning objectives
- Систематизировать пройденный материал
- Понять следующие шаги для развития файлового агента

### Что обсуждаем
- Резюме ключевых идей: конфигурация → сервер → модель → ACP → Obsidian
- Следующие шаги:
  - Тулы записи (write, append, mkdir)
  - Отдельный конфиг под production
  - MCP-интеграции
  - Корпоративные политики доступа
- Идеи для самостоятельной работы:
  - Повторить цепочку на своём vault
  - Измерить latency на разных моделях
  - Написать 3 теста для своего агента

### Q&A и ресурсы

**Что обсудим:**
- Вопросы по материалам мастер-класса
- Сложности при настройке окружения
- Идеи для собственных агентов

**Ресурсы:**
- Репозиторий: https://github.com/vamplabai/sgr-agent-core
- Документация: https://vamplabai.github.io/sgr-agent-core/
- API для мастер-класса: https://api.rpa.icu (телеграм @evilfreelancer)

### Шпаргалка по командам
```bash
# Клонирование
git clone https://github.com/vamplabai/sgr-agent-core.git

# Установка
cd sgr-agent-core && pip install -e .

# Подготовка конфигурации
cd examples/sgr_deep_research
cp config.yaml.example config.yaml
# отредактируй config.yaml

# Запуск OpenAI-совместимого API
sgr -c config.yaml
```

### Слайды
- Краткое резюме мастер-класса
- Дорожная карта: файловый агент → API → ACP → экосистема
- Задания для самостоятельной работы

### Speaker notes
> «Мы прошли путь от бизнес-запроса до работающего сервиса. Теперь у вас есть инструмент, который можно развивать: добавлять тулы, подключать к разным UI, измерять качество.»

---

## Приложение A. Чек-лист спикера

### Перед мастер-классом
- [ ] Проверен доступ к API (OpenAI / gpt-oss)
- [ ] Docker-образ скачан или `pip install` выполнен
- [ ] Конфиги `config.yaml` подготовлены и протестированы
- [ ] Папка с демо-заметками готова
- [ ] Obsidian с плагином Agent Client настроен (если показываете ACP)
- [ ] Langfuse подключён (если показываете observability)
- [ ] Резервный план на случай проблем с сетью

### Во время мастер-класса
- [ ] Записи экрана / live coding доступны аудитории
- [ ] Команды копируются из плана (не набираются вручную)
- [ ] Время контролируется по таймингу

---

## Приложение B. Шпаргалка по командам

```bash
# Клонирование
git clone https://github.com/vamplabai/sgr-agent-core.git
cd sgr-agent-core

# Deep Research
cp examples/sgr_deep_research/config.yaml.example examples/sgr_deep_research/config.yaml
sgr -c examples/sgr_deep_research/config.yaml

# Файловый агент
cp examples/sgr_file_agent/config.yaml.example examples/sgr_file_agent/config.yaml
sgr -c examples/sgr_file_agent/config.yaml

# ACP
sgracp --config /abs/path/to/config.yaml

# Тесты
pytest examples/sgr_file_agent/tests/
```

---

*План актуален на 12.04.2026. Последнее обновление: синхронизация с Masterclass.md*
