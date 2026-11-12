# Собственные коннекторы OpsWatch

Коннектор — это Python-класс, который OpsWatch периодически опрашивает. Он сообщает о проблемах (события), о том, что проблема ушла (восстановление), и передаёт метрики.

## Установка

1. Положите `.py`-файл с классом коннектора в папку `plugins` рядом с `OpsWatch.exe`.
2. Перезапустите OpsWatch (или службу `OpsWatchServer`).
3. Новый тип источника появится на вкладке **«Источники»**.

Полный рабочий пример: [`examples/plugins/disk_space.py`](../examples/plugins/disk_space.py).

## Структура коннектора

```python
from opswatch.connectors import Connector, ConnectorError, Field, PollResult, register


@register
class MyConnector(Connector):
    type = "my_check"                  # уникальный идентификатор типа
    title = "Моя проверка"             # название в интерфейсе
    category = "monitoring"            # категория источника
    description = "Что проверяет коннектор"
    default_interval = 300             # интервал опроса по умолчанию, секунд
    fields = [
        Field("path", "Путь", required=True, placeholder="D:\\"),
        Field("limit", "Порог, %", type="number", default=90),
    ]

    async def poll(self) -> PollResult:
        ...
```

| Атрибут | Назначение |
|---|---|
| `@register` | Регистрирует класс как новый тип источника |
| `type` | Уникальный идентификатор (латиница, без пробелов) |
| `title` | Название типа источника в интерфейсе |
| `category` | Категория, например `monitoring` |
| `description` | Краткое описание в интерфейсе |
| `default_interval` | Интервал опроса по умолчанию в секундах |
| `fields` | Поля настройки, которые пользователь заполняет при создании источника |

### Поля настройки — `Field`

```python
Field(name, label, required=False, type=..., default=..., placeholder=...)
```

- `name` — ключ параметра, по которому значение читается в коде;
- `label` — подпись поля в форме;
- `required=True` — поле обязательно;
- `type="number"` — числовое поле;
- `default` — значение по умолчанию;
- `placeholder` — подсказка в пустом поле.

## Метод `poll()`

Асинхронный метод, который вызывается по расписанию. Возвращает `PollResult`.

### Полезные методы базового класса

| Метод / свойство | Назначение |
|---|---|
| `self.option(name, default)` | Значение поля настройки |
| `self.int_option(name, default)` | Значение поля как целое число |
| `self.fingerprint(key)` | Устойчивый идентификатор проблемы. По нему OpsWatch связывает повторы и восстановление |
| `self.event(title=, message=, severity=, type=, fingerprint=)` | Создаёт событие (проблему) |
| `self.resolved(fingerprint)` | Сообщает, что проблема с этим идентификатором устранена |
| `self.ctx.name` | Имя источника, заданное пользователем |

Уровни `severity`: `warning`, `critical`.

### Результат — `PollResult`

```python
PollResult(events=[...], metrics={...}, message="...")
```

- `events` — список событий и/или восстановлений;
- `metrics` — словарь числовых метрик для графиков;
- `message` — краткий статус источника.

### Ошибки

Если проверку выполнить нельзя (путь недоступен, сервис не отвечает), выбросьте `ConnectorError` — источник будет помечен как недоступный:

```python
raise ConnectorError(f"Путь недоступен: {exc}") from exc
```

## Пример: свободное место на диске

```python
import shutil

from opswatch.connectors import Connector, ConnectorError, Field, PollResult, register


@register
class DiskSpaceConnector(Connector):
    type = "disk_space"
    title = "Свободное место на диске"
    category = "monitoring"
    description = "Предупреждает, когда на диске или в сетевой папке заканчивается место"
    default_interval = 300
    fields = [
        Field("path", "Диск или папка", required=True, placeholder="D:\\"),
        Field("warn_percent", "Предупреждение, % занято", type="number", default=85),
        Field("critical_percent", "Критично, % занято", type="number", default=95),
    ]

    async def poll(self) -> PollResult:
        path = str(self.option("path", "")).strip()
        try:
            usage = shutil.disk_usage(path)
        except OSError as exc:
            raise ConnectorError(f"Путь недоступен: {exc}") from exc
        used = round(usage.used / usage.total * 100, 1)
        fingerprint = self.fingerprint("usage")
        metrics = {"used_percent": used, "free_gb": round(usage.free / 1024**3, 1)}
        if used >= self.int_option("critical_percent", 95):
            severity = "critical"
        elif used >= self.int_option("warn_percent", 85):
            severity = "warning"
        else:
            return PollResult(events=[self.resolved(fingerprint)], metrics=metrics, message=f"Занято {used}%")
        event = self.event(
            title=f"{self.ctx.name}: занято {used}%",
            message=f"Свободно {metrics['free_gb']} ГБ на {path}",
            severity=severity,
            type="disk.usage",
            fingerprint=fingerprint,
        )
        return PollResult(events=[event], metrics=metrics, message=f"Занято {used}%")
```

## Советы

- Код в `poll()` асинхронный: долгие блокирующие операции выносите в поток (`asyncio.to_thread`).
- Используйте один и тот же `fingerprint` для одной проблемы, иначе каждое срабатывание будет новым событием.
- Возвращайте `self.resolved(...)`, когда всё в порядке, — тогда OpsWatch закроет событие и пришлёт уведомление о восстановлении.
