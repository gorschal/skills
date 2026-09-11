# Мокирование

Мок — инструмент изоляции от внешнего мира, а не самоцель. Главный принцип:
**тестируй поведение, а не моки**.

## Инструменты

```python
from unittest.mock import Mock, AsyncMock, MagicMock, patch, call

# sync
mock = Mock()
mock.get.return_value = {"id": 1}

# async
amock = AsyncMock()
amock.get.return_value = {"id": 1}

# patch
with patch("app.services.clock.now") as mock_now:
    mock_now.return_value = datetime(2024, 1, 1, tzinfo=UTC)
```

- Async-методы — `AsyncMock` (`assert_awaited_once_with`).
- `patch` — на границе (внешний модуль/функция), не на внутренностях.
- Предпочитать инъекцию зависимости (`__init__`) глобальному `patch`.

## Стратегия моков

- Мокать **внешнее**: сеть (`httpx`), БД (репозиторий), время, файлы, очереди.
- Не мокать **внутреннюю** логику — иначе тест проверяет реализацию.
- Реальные зависимости там, где возможно (чистые функции, домен).
- Один тест — минимум моков; «10 моков на тест» — сигнал плохого дизайна.

```python
# ✅ внешняя зависимость замокана, поведение проверяется
async def test_get_user_returns_data(service, repo):
    repo.get.return_value = {"id": 1, "name": "Alice"}
    user = await service.get_user(1)
    assert user.name == "Alice"
```

## Полные моки

```python
# ❌ неполный: код упадёт на user.email
repo.get.return_value = {"id": 1, "name": "Alice"}

# ✅ полный контракт (или фабрика)
def make_user(**overrides) -> dict:
    return {"id": 1, "name": "Alice", "email": "a@b.c", "active": True, **overrides}
```

Неполные моки проходят тест, но ломают прод. Использовать фабрики с дефолтами.

## Side effects

```python
repo.get.side_effect = [None, {"id": 1}]      # последовательность
repo.save.side_effect = DatabaseError("boom") # исключение
```

## Время и недетерминизм

- Замораживать время (`freezegun`/`time-machine`) или инъектировать `clock`.
- Не полагаться на реальные таймеры/`sleep`; использовать фейки.

## Проверка вызовов

```python
repo.get.assert_awaited_once_with(1)
repo.save.assert_called_once()
```

Проверка вызовов — **в дополнение** к проверке результата, не вместо неё.

## Антипаттерны

| ❌ | ✅ |
|---|---|
| `assert mock.called` без проверки результата | ассерт на поведение |
| Мокать всё (включая домен) | мокать только внешнее |
| Неполный мок | фабрика/полный контракт |
| `patch` на внутренний метод | инъекция зависимости |
| Реальные сеть/БД/время | мок/фейк/freeze |
| `MagicMock` для async-метода | `AsyncMock` |

## Чек-лист

- [ ] Мокается только внешнее; поведение проверяется.
- [ ] Моки полные (фабрики).
- [ ] Async — `AsyncMock`.
- [ ] Время/недетерминизм контролируется.
- [ ] Проверка вызовов — в дополнение к результату.
