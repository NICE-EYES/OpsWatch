# OpsWatch

**Мониторинг, резервное копирование и уведомления в Telegram для Windows.**
*Monitoring, backups and Telegram notifications for Windows — [English below](#english).*

---

## Возможности

- **Мониторинг источников** — встроенные коннекторы и собственные плагины на Python (папка `plugins`).
- **Приём событий по вебхуку** — Prometheus Alertmanager, Zabbix, PowerShell-скрипты и любые другие системы (`POST /api/ingest/<TOKEN>`).
- **Уведомления** — Telegram-бот, всплывающие уведомления Windows, эскалации.
- **Резервное копирование** — копии хранятся в `data\backups`; большие файлы отправляются ссылкой на скачивание.
- **Многопользовательский режим** — регистрация сотрудников, подтверждение администратором, роли и права.
- **Веб-панель** — доступна из браузера по адресу `http://<сервер>:8765`.
- **Служба Windows** — сервер работает без входа пользователя в систему.
- **SQLite или PostgreSQL** — автоматическое обновление схемы базы и перенос данных.
- **Два языка** — русский и английский интерфейс и уведомления.

## Состав репозитория

| Путь | Назначение |
|---|---|
| `OpsWatch.exe` | Настольное приложение (сервер или клиент) |
| `OpsWatchServer.exe` | Консольный сервер / служба Windows |
| `_internal/` | Среда выполнения и библиотеки (нужна обоим `.exe`) |
| `.env.example` | Образец файла настроек |
| `examples/` | Примеры интеграций и коннектора |
| `plugins/` | Папка для собственных коннекторов |
| `README.txt` | Краткая инструкция, поставляется вместе с программой |
| `LICENSE.txt` | Лицензия MIT |

Папка `data/` (база, ключ шифрования, журналы, бэкапы) создаётся при первом запуске и в репозиторий не попадает.

## Быстрый старт

1. Скачайте репозиторий (**Code → Download ZIP**) и распакуйте, например, в `C:\OpsWatch`.
2. Запустите `OpsWatch.exe`.
3. Выберите режим:
   - **«Этот компьютер — сервер»** — на компьютере, где будет работать OpsWatch;
   - **«Подключиться к серверу»** — на компьютерах сотрудников (например, `http://192.168.1.10:8765`).
4. Войдите под `admin1` / `admin1` и **сразу смените пароль** в профиле.
5. Сотрудники нажимают «Регистрация», администратор подтверждает их на вкладке «Пользователи» и выдаёт роль.

> Порт по умолчанию — **8765**. Разрешите его в брандмауэре Windows для доступа из сети.

## Telegram

- **Администратор:** Настройки → Telegram → токен бота от [@BotFather](https://t.me/BotFather).
- **Пользователь:** Профиль → «Привязать Telegram» → открыть бота и нажать «Старт».
- **Уведомления Windows:** Профиль → «Уведомления Windows».

## Служба Windows

Командная строка от имени администратора в папке программы:

```bat
OpsWatchServer.exe install
OpsWatchServer.exe start
```

| Команда | Действие |
|---|---|
| `OpsWatchServer.exe install` | Установить службу |
| `OpsWatchServer.exe start` | Запустить службу |
| `OpsWatchServer.exe remove` | Удалить службу |
| `OpsWatchServer.exe run` | Запустить сервер в консоли |
| `OpsWatchServer.exe migrate-db <URL> --write-env` | Перенести данные в PostgreSQL |

## Настройки (`.env`)

Скопируйте `.env.example` в `.env` рядом с программой:

| Переменная | По умолчанию | Описание |
|---|---|---|
| `OPSWATCH_HOST` | `0.0.0.0` | Адрес, на котором слушает сервер |
| `OPSWATCH_PORT` | `8765` | Порт веб-панели и API |
| `OPSWATCH_ADMIN_USERNAME` | `admin1` | Логин администратора по умолчанию |
| `OPSWATCH_ADMIN_PASSWORD` | `admin1` | Пароль администратора по умолчанию |
| `OPSWATCH_TELEGRAM_TOKEN` | — | Токен Telegram-бота |
| `OPSWATCH_TELEGRAM_API_URL` | — | Свой адрес Telegram Bot API (прокси / локальный сервер) |
| `OPSWATCH_PUBLIC_URL` | — | Внешний адрес сервера для ссылок в уведомлениях |
| `OPSWATCH_DATABASE_URL` | — | Строка подключения к PostgreSQL (пусто — SQLite) |
| `OPSWATCH_SECRET_KEY` | — | Ключ шифрования (пусто — `data\secret.key`) |
| `OPSWATCH_DATA_DIR` | — | Папка данных (пусто — `data` рядом с программой) |
| `OPSWATCH_LOG_LEVEL` | `INFO` | Уровень журналирования |
| `OPSWATCH_LANGUAGE` | — | Язык по умолчанию: `ru` или `en` |

## Данные

```
data\opswatch.db   — настройки, пользователи и журнал
data\secret.key    — ключ шифрования паролей (сохраните его в надёжном месте!)
data\backups\      — резервные копии
data\logs\         — журналы работы
```

## Обновление и PostgreSQL

Новую версию распакуйте поверх старой — папка `data` и файл `.env` сохранятся, схема базы обновится автоматически.

Перенос из SQLite в PostgreSQL (сервер должен быть остановлен):

```bat
OpsWatchServer.exe migrate-db postgresql://user:password@host/opswatch --write-env
```

## Интеграции

Каждый источник типа «вебхук» получает свой токен. События отправляются JSON-запросом:

```http
POST http://<сервер>:8765/api/ingest/<TOKEN>
Content-Type: application/json

{
  "title": "Диск D: заполнен на 95%",
  "message": "Проверка из планировщика заданий",
  "severity": "critical",
  "category": "monitoring",
  "external_id": "disk-d-SRV01"
}
```

Готовые примеры в папке `examples`:

- `alertmanager.yml` — Prometheus Alertmanager;
- `zabbix_webhook.js` — тип оповещения «Webhook» в Zabbix;
- `generic_webhook.ps1` — PowerShell (например, из планировщика заданий).

## Собственные коннекторы

Положите `.py`-файл с классом коннектора в папку `plugins` и перезапустите OpsWatch — новый тип источника появится на вкладке «Источники».

```python
from opswatch.connectors import Connector, ConnectorError, Field, PollResult, register

@register
class MyConnector(Connector):
    type = "my_check"
    title = "Моя проверка"
    category = "monitoring"
    default_interval = 300
    fields = [Field("path", "Путь", required=True)]

    async def poll(self) -> PollResult:
        ...
```

Полный пример — `examples/plugins/disk_space.py` (контроль свободного места на диске). Подробное описание API — [docs/connectors.md](docs/connectors.md).

## Системные требования

- Windows 10 / 11 или Windows Server 2016+ (x64)
- Установка Python не требуется

## Лицензия

[MIT](LICENSE.txt)

---

<a id="english"></a>

## English

OpsWatch is a Windows application for monitoring, backups and Telegram notifications with a built-in web panel.

**Quick start**

1. Download the repository (**Code → Download ZIP**), unzip it (e.g. to `C:\OpsWatch`) and run `OpsWatch.exe`.
2. Choose **"This computer is the server"** or **"Connect to a server"**.
3. Sign in with `admin1` / `admin1` and change the password in the profile.
4. Switch the language to English on the sign-in screen or in the profile.

**Windows service** (elevated command prompt in the program folder):

```bat
OpsWatchServer.exe install
OpsWatchServer.exe start
```

The web panel is available at `http://<server>:8765`.

- All data is kept in the `data` folder next to the program — keep `data\secret.key` safe.
- Settings: copy `.env.example` to `.env` (see the table above).
- Move data to PostgreSQL: `OpsWatchServer.exe migrate-db postgresql://user:password@host/opswatch --write-env`
- Default language: `OPSWATCH_LANGUAGE=en` in `.env`.
- Integration examples are in `examples`, custom connectors go into `plugins`.

License: [MIT](LICENSE.txt).
