# Тестирование — указатель

Unit/integration-тесты на pytest вынесены в отдельный навык **`python-testing`**.
BDD/Gherkin/Page Objects — навык **`pytest-bdd`**.

Политика монорепозитория: **BDD — основное сквозное покрытие** всей логики;
unit-тесты берут только логику вне BDD и критичные ветки. Unit не дублирует BDD.

Кратко:

- pytest-функции и фикстуры, **не** `unittest.TestCase`.
- Зависимости — `Mock`/`AsyncMock`; в unit нет реальной БД/HTTP/`TestClient`.
- Тесты проверяют **поведение**, а не моки; ассерты конкретные.
- Полная изоляция; флаки чинятся, а не перезапускаются.

Подробности — в навыке `python-testing`:

| Тема | Reference |
|---|---|
| Структура, фикстуры, parametrize, покрытие | `python-testing/references/pytest.md` |
| Mock/AsyncMock/patch, стратегия моков | `python-testing/references/mocking.md` |
| pytest-asyncio, async-фикстуры | `python-testing/references/async.md` |
| Антипаттерны, флаки, качество | `python-testing/references/quality.md` |

Фреймворк-специфика: `django/references/testing.md`,
`fastapi/references/testing.md`, `faststream/references/testing.md`,
`aiogram/references/testing.md`.
