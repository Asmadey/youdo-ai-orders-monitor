# Youdo AI Orders Monitor

Мониторинг новых заказов в категории «Разработка ПО» на Youdo.com с фильтрацией по AI-ключевикам, дедупликацией и отправкой уведомлений в Telegram. Работает через replay сигнатуры мобильного API (KeyVersion=1) — без браузера, без логина, без LLM.

## Возможности

### Получение заказов
- Запрос к Youdo Mobile API (`api.youdo.com/v110/tasks/tasks`) — категория IT/Software Development (Category=4194304)
- Радиус 50км от Москвы, 50 заказов на страницу
- Replay захваченной сигнатуры (HMAC-SHA256, KeyVersion=1) — сигнатура детерминирована, не содержит timestamp/nonce, работает бессрочно
- Без авторизации: публичные данные о заказах доступны через мобильный API

### AI-фильтрация
- **35+ длинных ключевых слов** (substring match): искусственный интеллект, нейросеть, chatgpt, openai, machine learning, n8n, automation, langchain, RAG, LLM и др.
- **18 коротких ключевых слов** (word boundary): ИИ, AI, ML, бот, agent, prompt, промт, copilot и др.
- Word boundary для кириллицы через lookarounds (Python `\b` не работает с кириллицей)
- Проверка по title + description каждого заказа

### Дедупликация
- `seen_ids.json` — список ID обработанных заказов (max 1000, ring buffer)
- Только новые заказы проходят в уведомления
- JSONL-лог всех найденных AI-заказов (`youdo_orders.jsonl`)

### Telegram-уведомления
- HTML-форматирование: название, описание (до 300 символов), цена, дата, локация, автор, ссылка
- Flood control: truncation до 4000 символов (лимит Telegram 4096)
- Error alerts с cooldown (1 час) — не спамит при падении API

### Надёжность
- Чистый deterministic pipeline — без LLM, без браузера, без Playwright
- Error alert cooldown: повторные алерты об одной ошибке не чаще раза в час
- Graceful degradation: при ошибке API → alert в Telegram, при отсутствии новых заказов → тихий выход
- Логирование: structured logging в stderr, JSONL-лог заказов

## Технологии

| Технология | Назначение |
|---|---|
| **Python 3.10+** | Основной язык |
| **requests** | HTTP-запросы к Youdo Mobile API и Telegram Bot API |
| **python-dotenv** | Загрузка переменных окружения (`.env`) |

## Архитектура

Pipeline выполняется за один проход — без цикла, без состояния между запусками. Предназначен для запуска по cron каждые 5 минут.

```
youdo_check.py  (точка входа)

┌─────────────────────────────────────────────────────────────┐
│  Step 1: fetch_tasks()                                       │
│  ├── GET api.youdo.com/v110/tasks/tasks                      │
│  │   ├── Category=4194304 (IT/Software Development)          │
│  │   ├── lat=55.75, lng=37.62, radius=50km (Moscow)          │
│  │   ├── Signature: replay captured HMAC-SHA256 (KeyVer=1)   │
│  │   └── User-Agent: iOS app (iPhone16_2)                    │
│  └── → ResultObject.Items[] (up to 50 tasks)                 │
│                                                              │
│  Step 2: Filter AI orders                                    │
│  ├── For each task: skip if Id in seen_ids                   │
│  ├── is_ai_match(Name + Description)                         │
│  │   ├── AI_RE_LONG: 35+ long keywords (substring)           │
│  │   └── AI_RE_SHORT: 18 short keywords (word boundary)      │
│  └── → new_ai_orders list                                    │
│                                                              │
│  Step 3: Send & Log                                          │
│  ├── format_task_message() → HTML with price, date, link     │
│  ├── send_telegram() → Bot API (truncation at 4000 chars)    │
│  ├── append_jsonl() → youdo_orders.jsonl (full record)       │
│  ├── Update seen_ids (append, trim to 1000)                  │
│  └── save_seen_ids()                                         │
└─────────────────────────────────────────────────────────────┘
```

### Ключевая деталь: Replay сигнатуры

Youdo Mobile API использует HMAC-SHA256 сигнатуру для авторизации запросов. Сигнатура детерминирована — зависит только от URL и секретного ключа (зашитого в iOS-приложении), без timestamp или nonce. Это означает:

- Один захваченный `Signature` работает для одного URL **бессрочно**
- Не нужен MITM-прокси или перехват на каждый запуск
- Не нужен логин или OAuth-токен

Захват сигнатуры выполняется один раз через MITM-прокси (Charles/mitmproxy) при перехвате запроса мобильного приложения.

## Требования

- **Python 3.10+**
- **Захваченная сигнатура Youdo API** (уже встроена в скрипт)
- **Telegram bot token** — для уведомлений

## Установка

```bash
# 1. Клонировать репозиторий
git clone https://github.com/Asmadey/youdo-ai-orders-monitor.git
cd youdo-ai-orders-monitor

# 2. Создать виртуальное окружение
python3 -m venv .venv
source .venv/bin/activate

# 3. Установить зависимости
pip install -r requirements.txt

# 4. Создать .env из примера
cp .env.example .env
# Заполнить TELEGRAM_BOT_TOKEN, TG_CHAT_ID
```

### Переменные окружения (`.env`)

| Переменная | Описание | По умолчанию |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | Токен Telegram-бота от [@BotFather](https://t.me/BotFather) | — |
| `TG_CHAT_ID` | ID чата для уведомлений | `128204572` |

## Запуск

```bash
# Разовый запуск
python3 youdo_check.py

# Cron (каждые 5 минут)
*/5 * * * * cd /path/to/youdo-ai-orders-monitor && python3 youdo_check.py >> logs/cron.log 2>&1
```

## Конфигурация

### Категория и гео

Параметры запроса задаются в `YOUDO_URL` в начале скрипта:

```python
YOUDO_URL = (
    "https://api.youdo.com/v110/tasks/tasks"
    "?status=Opened"
    "&Category=4194304"      # IT / Software Development
    "&lat=55.753215"         # Moscow
    "&lng=37.622504"
    "&radius=50.0"           # 50 km
    "&page=1"
    "&pageSize=50"
)
```

Для мониторинга другой категории измените `Category` и координаты.

### AI-ключевые слова

Два списка в начале скрипта:

```python
AI_KEYWORDS_LONG = [
    "искусственный интеллект", "нейросет", "chatgpt", "openai",
    "machine learning", "n8n", "automation", "langchain",
    "retrieval augmented generation", "fine-tuning", ...
]

AI_KEYWORDS_SHORT = [
    "ии", "ai", "ml", "бот", "bot", "agent", "агент",
    "rag", "llm", "ocr", "nlp", "prompt", "промт", ...
]
```

Long — substring match (уникальные слова). Short — word boundary через lookarounds (короткие слова, защита от false positives вроде «ai» в «again»).

## Структура проекта

| Файл | Назначение |
|---|---|
| `youdo_check.py` | Полный pipeline: fetch → filter → dedup → send → log |
| `.env.example` | Шаблон переменных окружения |
| `requirements.txt` | Python-зависимости |
| `pyproject.toml` | Метаданные пакета, ruff config |

## Runtime-артефакты

Эти файлы создаются во время работы (не входят в репозиторий):

| Артефакт | Назначение |
|---|---|
| `youdo_seen_ids.json` | Список ID обработанных заказов (ring buffer, max 1000) |
| `youdo_orders.jsonl` | Лог всех найденных AI-заказов (по строкам JSON) |
| `youdo_error_alerts.json` | Cooldown-стейт для error alerts (timestamp → last alert) |

## Youdo Mobile API

### Endpoint

```
GET https://api.youdo.com/v110/tasks/tasks
```

### Заголовки

| Заголовок | Назначение |
|---|---|
| `Signature` | HMAC-SHA256 сигнатура (replay, KeyVersion=1) |
| `KeyVersion` | Версия ключа (`1`) |
| `X-VisitorId` | UUID посетителя |
| `X-DeviceId` | UUID устройства |
| `User-Agent` | Строка iOS-приложения (версия, модель, UUID) |
| `msid` | Session ID |

### Ответ

```json
{
  "IsSuccess": true,
  "ResultObject": {
    "Items": [
      {
        "Id": 123456,
        "Name": "Создать чат-бота для Telegram",
        "Description": "Нужен AI-бот...",
        "Budget": {"Min": 5000, "Max": 30000},
        "Location": {"Address": "Москва"},
        "CreatorName": "Иван",
        "CategoryName": "Разработка ПО",
        "DatePublish": 1721629200000
      }
    ],
    "TotalCount": 42
  }
}
```

## Лицензия

MIT