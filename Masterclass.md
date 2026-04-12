# SGR Agent Core — Мастер-класс

_Автоматическая конвертация из PPTX в Markdown._

- Исходный файл: `sgr-conference-masterclass.pptx`
- Слайдов: 14

---

## Рыков Павел — Росгосстрах

SGR Agent Core — Мастер-класс

---

## Программа мастер-класса

0. Вступление
1. Что такое SGR Agent Core
2. Архитектура API сервера
3. Конфигурация YAML и типы агентов
4. Подготовка окружения
5. Практика: Deep Research
6. Практика: Файловый агент
7. Метрики, тесты, Langfuse
8. Домашнее задание
9. Завершение и Roadmap

---

## 0. Вступление

Что понадобится:
- Python 3.11+ и/или Docker
- Git
- API-ключ с моделью поддерживающей Sturctured Output
 
Материалы:
- https://github.com/vamplabai/sgr-agent-core
 
По итогу получим:
- локальный файловый агент
- HTTP API
- ACP интегрированный в Obsidian

---

## 1. Что такое SGR Agent Core

![](assets/1.png)

---

## 2. Архитектура

SGR Agent Core работает в двух режимах

**API Mode** — HTTP сервер для интеграций
- OpenAI-совместимый эндпоинт /v1/chat/completions
- Интеграции: Open WebUI, LibreChat, любой OpenAI-клиент
- Stateless (без состояния) и Stateful (с сессиями)

**ACP Mode** — Agent Client Protocol
- Stdio transport для локальных агентов
- Интеграции: Obsidian, Claude Desktop, любой ACP-хост
- Stateless-only, контекст через threads

---

## 2.1. ACP Mode — Agent Client Protocol

Что такое ACP:
- Протокол для локальных агентов (stdio transport)
- Агент запускается как подпроцесс, общается через JSON-RPC
- Поддержка tools, resources, prompts

Интеграции:
- Obsidian + Copilot plugin
- Claude Desktop
- Cursor и другие IDE

QR-код на спецификацию ACP
[QR: https://spec.modelcontextprotocol.io/]

---

## 2.2. API Mode — REST сервер

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

---

## 2.3. Таксономия — Тулы и Агенты

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

---

## 3. YAML-конфигурации

Наследование настроек (приоритет снизу вверх):

```
defaults (в коде)  ← базовые значения
     ↓
envы               ← переменные окружения
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

---

## 3.1. Регистрация тулов

Схема определения тулов в конфиге:

```
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
```
agent:
  tools:
    - "web_search_tool"
    - "extract_page_content_tool"
    - "create_report_tool"
```

---

## 3.2. MCP интеграция

MCP (Model Context Protocol) — внешние серверы с тулами:

```
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

---

## 3.3. Определение агентов

Схема определения агента:

```
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
```
acp:
  agent: sgr_tool_calling_agent
```

---

## 3.4. Архитектура — связь компонентов

Общая схема взаимодействия:

![](assets/3.png)

**Агент** содержит SGR Pipeline с тремя фазами:
- Reasoning Tool — анализ и выбор стратегии
- Planning Tool — построение плана
- Acting Tools — выполнение действий через тулы

**Tools Layer** — регистрированные в конфиге тулы

**MCP Servers** — внешние серверы с дополнительными тулами

---

## 4. Что нужно для старта

**Шаг 1 — Окружение:**
- Python 3.11+ или Docker
- Git для клонирования примеров

**Шаг 2 — API-ключи:**
- OpenAI-совместимый API (обязательно)
- Tavily API key (опционально, для Deep Research)

**Шаг 3 — Конфигурация:**
- Скопировать config.yaml.example → config.yaml
- Прописать ключи в секции llm: и tools:

---

## 4.1. Установка

```bash
git clone https://github.com/vamplabai/sgr-agent-core.git
cd sgr-agent-core
pip install -e .
```

---

## 4.2. Токены и где их взять

**OpenAI-совместимый API:**
- OpenAI: https://platform.openai.com/api-keys
- Другие провайдеры с OpenAI-compatible API

**Tavily (для поиска):**
- https://tavily.com — 1000 запросов/месяц бесплатно

**Альтернативный вариант для мастер-класса:**
- API: https://api.rpa.icu
- Ключ: https://t.me/evilfreelancer
- Модель: gpt-oss:120b

---

## 4.3. Пример настройки LLM для api.rpa.icu

```yaml
llm:
  base_url: "https://api.rpa.icu/v1"
  api_key: "https://t.me/evilfreelancer"
  model: "gpt-oss:120b"
```

---

## 5. Практика: Deep Research — Roadmap

**СЛЕВА — Что делаем (пошагово):**

![](assets/51.png)

**СПРАВА — Что получим в итоге:**

![](assets/52.png)

Цель мастер-класса — запустить рабочий Deep Research агент из пакета sgr-agent-core

---

## Слайд 9. SGR Agent Core — Мастер-класс

Перерыв 10-15 минут

---

## Слайд 10. 6. Практика: Файловый агент

1. Пишем тулы: read, write, grep, glob
2. Настраиваем YAML-конфигурую
3. Запуск API-сервера
4. Выполение cURL-зарпроса
5. Запуск ACP stdio (задача со*)
6. Подключение APC к Obsidian (задача со*)

Что получим -->>

---

## Слайд 11. 7. Langfuse и тесты

Уровни тестирования: тулы -> агенты -> API
Langfuse для трассировки шагов модели (просто показать)

---

## Слайд 12. 8. Домашнее задание

(расписать подробнее задачу)
1. файловый агент на своём vault с 3+ тестами
2. SGR ACP интеграция с VS Code

->>>>
Что проверить: воспроизводимость экспериментов, метрики Langfuse, Swagger-документация API

---

## Слайд 13. 9. Завершение и Roadmap

(добавить визуала, в виде роадмапа чарта)
Резюме: сервер -> модель -> ACP -> тестировать
Roadmap: тулы записи, MCP серверы, прод-политики
Шпаргалка по командам: клонирование, запуск, тесты
Q&A и обсуждение

---

## Слайд 14. SGR Agent Core — Мастер-класс

Спасибо за внимание!

---
