# SGR Agent Core мастер-класс - AI Conf 2026

Этот репозиторий содержит материалы мастер-класса по `sgr-agent-core`

Основной фокус:
- архитектура SGR агентов
- работа с YAML конфигами
- практика Deep Research агента
- практика файлового агента

Репозиторий с кодовой базой фреймворка находится отдельно  
[`vamplabai/sgr-agent-core`](https://github.com/vamplabai/sgr-agent-core)

## Что внутри

- `masterclass.md` - основная версия слайдов в Markdown
- `masterclass.pptx` - исходная презентация
- `plan.md` - подробный план выступления с таймингом и демо командами
- `speech.md` - текст сопровождения для спикера
- `practice-01-deep-research.ipynb` - практика по запуску Deep Research
- `practice-02-file-agent.ipynb` - практика по сборке файлового агента
- `practice-02-file-agent.md` - та же практика в текстовом формате
- `assets/` - изображения и схемы для слайдов

## Для кого

- инженеры и разработчики которые хотят быстро поднять OpenAI-compatible агентный API
- команды которые хотят перейти от промптов к конфигурации и воспроизводимым пайплайнам
- участники воркшопа которым нужен готовый учебный маршрут

## Что нужно для практики

- Python `3.11+` или Docker
- Git
- API ключ для LLM с поддержкой structured output
- опционально ключ Tavily для web поиска

## Быстрый старт Deep Research

```bash
git clone https://github.com/vamplabai/sgr-agent-core.git
cd sgr-agent-core
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -e .
cp examples/sgr_deep_research/config.yaml.example examples/sgr_deep_research/config.yaml
sgr -c examples/sgr_deep_research/config.yaml
```

После старта сервера можно отправить тестовый запрос:

```bash
curl -N -X POST "http://localhost:8010/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "sgr_tool_calling_agent",
    "messages": [{"role": "user", "content": "Сделай исследование по теме RAG"}],
    "stream": true
  }'
```

## Быстрый старт файлового агента

Практика полностью описана в `practice-02-file-agent.md` и `practice-02-file-agent.ipynb`

Короткий путь:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install sgr-agent-core openai pydantic
mkdir -pv sgr-file-agent/{tools,logs}
cd sgr-file-agent
```

Далее:
- добавить код тулов и агента из `practice-02-file-agent.md`
- создать `config.yaml`
- запустить `sgr -c config.yaml --host 127.0.0.1 --port 8015`

## ACP режим

Для интеграции с ACP хостами можно использовать `sgracp`

Пример:

```bash
sgracp -c sgr-file-agent/config.yaml
```

## Важно про структуру этого репозитория

- это репозиторий материалов мастер-класса
- рабочие директории `sgr-agent-core/` и `sgr-file-agent/` добавлены в `.gitignore`
- поэтому они могут присутствовать локально у спикера, но не обязаны быть в Git истории

## Рекомендуемый порядок изучения

1. `masterclass.md`
2. `plan.md`
3. `practice-01-deep-research.ipynb`
4. `practice-02-file-agent.ipynb` или `practice-02-file-agent.md`
5. `speech.md`

