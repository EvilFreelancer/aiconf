# SGR Agent Core — Мастер-класс (AI Conf)

> **Цель масте-класса:** провести аудиторию по пути от бизнес-запроса («хочу агента по локальным файлам») до работающего OpenAI-совместимого сервиса.  
> **Фокус:** конфигурация, метрики, воспроизводимость — а не «магические промпты».

---

## Что понадобится

- Python 3.11+, Docker (опционально)
- Git
- API-ключ OpenAI или доступ к OpenAI-совместимому эндпоинту
- (Опционально) Tavily API-ключ для Deep Research
- (Опционально) Langfuse аккаунт для observability

---

## Материалы и ссылки

| Ресурс | Ссылка |
|--------|--------|
| Репозиторий | `https://github.com/vamplabai/sgr-agent-core` |
| Docker-образ | `ghcr.io/vamplabai/sgr-agent-core:latest` |
| PyPI-пакет | `pip install sgr-agent-core` |
| Obsidian Agent Client | `https://github.com/RAIT-09/obsidian-agent-client` |
| Документация Agent Client | `https://rait-09.github.io/obsidian-agent-client/agent-setup/custom-agents.html` |

---

## Тайминг и структура (согласование 16.03.2026)

| Блок | Содержание | Минуты |
|------|------------|--------|
| **0. Вступление** | Приветствие, цели, обещанный результат | ~5 |
| **1. Теория** | SGR Agent Core, Schema Guided Reasoning, архитектура API, stateless/stateful, конфигурация YAML, типы агентов | ~30 |
| **2. Практика 1** | Подготовка окружения, клонирование, первый запуск | ~5–10 |
| **3. Практика 2** | Deep Research, запуск готового агента из `examples/sgr_deep_research` | ~15 |
| **4. Практика 3** | Файловый агент, файловые тулы, 4 режима работы (HTTP, stateful, библиотека, ACP), gpt-oss 120b | ~30–40 |
| **5. Практика 4** | Метрики, тесты, Langfuse / observability | ~20 |
| **6. Домашка** | Задание на самостоятельную работу | ~5 |
| **7. Заключение** | Резюме, roadmap, вопросы | ~5–10 |

**Итого:** 115–120 минут.

---

## Блок 0. Вступление и сценарий (~5 мин)

### Learning objectives
- Понять, зачем нужен SGR Agent Core в «зоопарке агентов»
- Зафиксировать обещанный результат мастер-класса
- Понять формат: теория → live demo → практика

### Что показываем
- Короткое представление спикеров
- Цель: от бизнес-запроса до кастомного файлового агента
- Формат: теория + живой код + практика участников

### Команды
```bash
tree -L 2 .
```

### Speaker notes
> «Сегодня мы не будем писать промпты по 500 строк. Вместо этого — конфигурация, метрики и воспроизводимость. К концу занятия у вас будет локальный файловый агент с HTTP API, который можно подключить к Obsidian.»

### Слайды
- Кто мы и что за проект
- Цель и обещанный результат
- Структура занятия по блокам и таймингу

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

### Speaker notes
> «SGR — это не просто ещё один фреймворк. Это попытка сделать рассуждения агента детерминированными и интерпретируемыми. Каждый шаг — это структурированный выбор, который можно валидировать.»

### Слайды
- Определение SGR и визуальная схема фаз (рассуждение → действия)
- Сравнение классического агента и SGR-агента
- Мини-пример кода: создание и запуск агента

---

## Блок 2. Архитектура API сервера (~10 мин)

### Learning objectives
- Понять архитектуру FastAPI-сервера
- Различать stateless и stateful режимы
- Понять жизненный цикл агента

### Что объясняем
- FastAPI сервер и хранилище агентов
- Жизненный цикл агента: создание → выполнение → завершение
- Stateless (клиент шлёт полный контекст) vs Stateful (диалог по `agent_id`)

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

### Speaker notes
> «Stateful-режим удобен, когда вы хотите продолжить диалог без пересылки всей истории. Но помните: агент живёт в памяти сервера и имеет таймаут.»

### Слайды
- Sequence-диаграмма: клиент → сервер → тул → ответ
- Две дорожки: stateless и stateful
- OpenAI-совместимый протокол и совместимые клиенты

---

## Блок 3. Конфигурация YAML и типы агентов (~10 мин)

### Learning objectives
- Понять иерархию конфигурации
- Уметь создавать минимальный `config.yaml`
- Различать типы агентов

### Что объясняем
- **GlobalConfig** + **AgentDefinition**: один базовый `config.yaml` задаёт общие параметры
- Наследование: `defaults` → `env` → `config.yaml` → `agents.yaml`
- Типы агентов:
  - `SGRAgent` — базовый, с явными шагами рассуждения
  - `ToolCallingAgent` — классический tool calling
  - `SGRToolCallingAgent` — гибрид: SGR + tool calling

### Пример config.yaml
```yaml
llm:
  api_key: "your-openai-api-key-here"
  base_url: "https://api.openai.com/v1"
  model: "gpt-4o-mini"
  max_tokens: 8000
  temperature: 0.4

execution:
  max_clarifications: 3
  max_iterations: 10
  mcp_context_limit: 15000
  logs_dir: "logs"

tools:
  web_search_tool:
    tavily_api_key: "your-tavily-api-key-here"
    tavily_api_base_url: "https://api.tavily.com"
    max_searches: 4
    max_results: 10
  extract_page_content_tool:
    tavily_api_key: "your-tavily-api-key-here"
    tavily_api_base_url: "https://api.tavily.com"
    content_limit: 1500
  final_answer_tool:
  clarification_tool:
  reasoning_tool:

agents:
  sgr_tool_calling_agent_no_reporting:
    base_class: "agents.ResearchSGRToolCallingAgentNoReporting"
    llm:
      model: "gpt-4o-mini"
      temperature: 0.4
    tools:
      - "web_search_tool"
      - "extract_page_content_tool"
      - "final_answer_tool"
      - "clarification_tool"
      - "reasoning_tool"
```

### Speaker notes
> «Ключевая идея — минимальные переопределения. Базовый конфиг задаёт LLM и execution, а конкретный агент только выбирает тулы и класс.»

### Слайды
- Схема иерархии конфигурации
- Таблица типов агентов и их ролей
- Упрощённый пример `config.yaml` с выделением ключевых блоков

---

## Блок 4. Подготовка окружения (~5–10 мин)

### Чек-лист перед блоком
- [ ] Участники установили Python 3.11+ или Docker
- [ ] У всех есть API-ключ
- [ ] Проверена связь с интернетом

### Learning objectives
- Уметь запускать SGR двумя способами: Docker и pip
- Понять разницу между сервером и библиотекой

### Команды
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

### Speaker notes
> «Если у вас Docker — запуск за 2 минуты. Если pip — больше контроля, можно копаться в коде.»

### Слайды
- Чеклист подготовки окружения
- Два режима запуска: Docker и локальная библиотека
- QR-код или ссылка на инструкции

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

### Speaker notes
> «Обратите внимание на логи — там виден каждый шаг. Это не чёрный ящик, а прозрачный пайплайн, поверх которого можно строить метрики.»

### Слайды
- Схема пайплайна deep research
- Пример потока событий с вызовами тулов
- Короткий блок про метрики и бенчмарки

---

## Блок 6. Практика: Файловый агент (~30–40 мин)

### Learning objectives
- Перейти от research-сценария к локальному файловому агенту
- Понять границы безопасности (read-only)
- Освоить 4 режима работы агента

### 6.1 Файловые тулы из примера `sgr_file_agent`

**Набор тулов:**
- `GetSystemPathsTool` — стандартные пути: home, documents, downloads, desktop
- `ListDirectoryTool` — листинг с опцией рекурсии
- `ReadFileTool` — чтение с диапазоном строк
- `SearchInFilesTool` — поиск текста в файлах (grep-подобный)
- `FindFilesFastTool` — поиск файлов по паттерну/размеру/дате

**Команды:**
```bash
cd sgr-agent-core
cp examples/sgr_file_agent/config.yaml.example examples/sgr_file_agent/config.yaml
```

### 6.2 Четыре режима работы агента

| Режим | Команда / Код | Когда использовать |
|-------|---------------|-------------------|
| **1. HTTP сервер** | `sgr --config-file examples/sgr_file_agent/config.yaml` | Интеграция с любыми OpenAI-совместимыми клиентами |
| **2. Stateful** | `model: "sgr_file_agent_<agent_id>"` в curl | Продолжение диалога без пересылки истории |
| **3. Библиотека** | `from sgr_agent_core.agents.sgr_agent import SGRAgent` | Встраивание в свой Python-код |
| **4. ACP (stdio)** | `sgracp --config /abs/path/to/config.yaml` | Интеграция с Obsidian, IDE и другими MCP-клиентами |

### 6.3 HTTP API с gpt-oss 120b

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

### 6.4 ACP и Obsidian Agent Client

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

### Speaker notes
> «Работа с файлами — read-only по умолчанию. working_directory — это корень, за который агент не выйдет. Это важно для безопасности.»

> «ACP — это мост между SGR и Obsidian. Вы говорите агенту «найди в моих заметках», и он ищет в вашем vault.»

### Слайды
- Схема потока: запрос → тулы ФС → финальный ответ
- Таблица 4 режимов работы
- Диаграмма: Obsidian → Agent Client → sgracp → vault

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

### Speaker notes
> «Тестируйте не только «правильный» путь. Проверяйте, что агент graceful обрабатывает отсутствующие файлы, пустые директории, странные запросы.»

### Слайды
- Список метрик: полнота, число вызовов, время
- Схема observability: SGR логи + Langfuse
- Что автоматизировать в CI для агента

---

## Блок 8. Домашнее задание (~5 мин)

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

### Команды (по желанию)
```bash
git diff
```

### Speaker notes
> «Мы прошли путь от бизнес-запроса до работающего сервиса. Теперь у вас есть инструмент, который можно развивать: добавлять тулы, подключать к разным UI, измерять качество.»

### Слайды
- Краткое резюме мастер-класса
- Дорожная карта: файловый агент → API → ACP → экосистема
- Задания для самостоятельной работы

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

*План актуален на 16.03.2026. Последнее обновление: 08.04.2026.*
