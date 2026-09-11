# Архитектура Python-приложения

Слоистая архитектура: presentation → service → repository. Цель — тестируемость
бизнес-логики без БД и HTTP, явные зависимости, контролируемые транзакции.

## Слои и границы

| Слой                                   | Знает про                      | Не знает про                      | Тестируется                     |
| -------------------------------------- | ------------------------------ | --------------------------------- | ------------------------------- |
| Presentation (router/view/handler/CLI) | вход/выход, схемы, DI          | SQL, транзакции, доменную логику  | почти не тестируется            |
| Service                                | домен, репозитории, транзакции | HTTP/`request`/`response`, брокер | unit-тестами с моками           |
| Repository                             | источник данных, SQL/ORM       | бизнес-правила, коммиты           | интеграционно (вне репо)        |
| Schema/DTO                             | форма и валидация данных       | побочные эффекты                  | через Pydantic-тесты (вне репо) |

Правило: **один вызов сервиса из presentation**; сервис оркестрирует
репозитории и другие сервисы.

## Структура проекта (src-layout)

```
project/
├── src/
│   └── app/
│       ├── main.py                 # composition root: сборка зависимостей
│       ├── api/                    # presentation
│       │   ├── deps.py             # DI-фабрики
│       │   └── v1/
│       ├── services/               # бизнес-логика
│       ├── repositories/           # доступ к данным
│       ├── schemas/                # Pydantic/DTO
│       ├── domain/                 # сущности, value objects
│       └── core/                   # config, database, exceptions, logger
├── tests/
├── pyproject.toml
└── .env.example
```

`main.py` (composition root) — единственное место, где собираются конкретные
реализации. Остальной код зависит от абстракций (`Protocol`).

## Dependency Injection через конструктор

```python
from typing import Protocol

class UserRepository(Protocol):
    def get(self, user_id: int) -> "User | None": ...
    def save(self, user: "User") -> None: ...

class UserService:
    def __init__(
        self,
        user_repo: UserRepository,             # обязательная зависимость
        notifier: "Notifier | None" = None,    # вторичная — fallback допустим
    ) -> None:
        self.user_repo = user_repo
        self.notifier = notifier or NullNotifier()
```

- **Репозитории — только явно, без fallback.** `self.repo = repo or User.objects`
  запрещено: скрывает связь и ломает тесты.
- **Вторичные зависимости** (утилиты, клиенты) могут иметь безопасный fallback.
- `Protocol` вместо наследования — репозиторий легко подменить `Mock`/`AsyncMock`.

## Транзакции

Транзакция — граница бизнес-операции, а не запроса.

```python
class OrderService:
    def __init__(self, session: Session, orders: OrderRepository, payments: PaymentRepository) -> None:
        self.session = session
        self.orders = orders
        self.payments = payments

    def checkout(self, order_id: int) -> None:
        with self.session.begin():          # одна транзакция на операцию
            order = self.orders.get(order_id)
            self.payments.charge(order)
            self.orders.mark_paid(order)
        # побочные эффекты — ПОСЛЕ успешного коммита
        self.notifier.send_receipt(order)
```

- Репозиторий **никогда** не вызывает `commit`/`rollback`.
- Несколько операций — в одном `begin()`/`atomic()`.
- Письма, задачи, публикация событий — после коммита (outbox/`on_commit`),
  иначе получите «письмо об откате».

## Побочные эффекты и идемпотентность

- Внешние вызовы (email, платёжка, брокер) — вне транзакции БД.
- Критичные обработчики делайте идемпотентными: уникальный ключ операции,
  проверка «уже обработано», upsert.
- Публикация события — через outbox-таблицу в той же транзакции, отправка —
  отдельным воркером (гарантия «ровно один раз» на уровне БД).

## Конфигурация и composition root

```python
def build_user_service(session: Session) -> UserService:
    repo = SqlUserRepository(session)
    return UserService(user_repo=repo, notifier=EmailNotifier())
```

Веб-фреймворк подключает это через DI-механизм (Django — фабрика + явный вызов;
FastAPI — `Depends`). Ручное создание сервисов «на месте» запрещено.

## Владение данными в монорепозитории

В полиглот-монорепозитории (Django + FastAPI + ...) **источник истины по схеме
БД — один сервис** (обычно Django). Остальные — потребители.

- Миграции и изменение схемы — только во владельце. Дублировать модели/миграции
  в других фреймворках запрещено.
- Сервис-потребитель работает с существующими таблицами: маппинг через
  SQLAlchemy ORM поверх таблиц или Core; собственных миграций нет.
- Если потребителю нужна запись — согласовать контракт с владельцем
  (события/API), чтобы не разъехались инварианты.
- Выбор «ORM vs Core» и направление владения — **политика проекта**, фиксируется
  ADR; по умолчанию: владелец — Django, FastAPI — read-only потребитель.

## Антипаттерны

| ❌                                     | Почему плохо                          | ✅                            |
| -------------------------------------- | ------------------------------------- | ----------------------------- |
| Бизнес-логика во view/handler          | нельзя переиспользовать и тестировать | вынести в service             |
| SQL в presentation                     | связность, инъекции, дублирование     | repository                    |
| `commit()` в repository                | транзакция размазана                  | транзакция в service          |
| `request` внутри service               | связанность с транспортом             | передавать данные (`user_id`) |
| Глобальные синглтоны-сервисы           | скрытые зависимости                   | DI через конструктор          |
| Циклические импорты service↔repository | хрупкость                             | Protocol в отдельном модуле   |

## Чек-лист

- [ ] Presentation вызывает ровно один сервис.
- [ ] Репозитории внедряются явно, без fallback.
- [ ] Транзакция — в сервисе, одна на бизнес-операцию.
- [ ] Побочные эффекты после коммита.
- [ ] Внешние вызовы идемпотентны на критичных путях.
- [ ] Composition root один; остальной код зависит от абстракций.
