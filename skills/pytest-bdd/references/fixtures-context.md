# Фикстуры и контекст

Состояние между шагами — через `target_fixture` или `context`. Ресурсы — через
pytest-фикстуры с очисткой.

## Контекст сценария

```python
import pytest

@pytest.fixture
def context() -> dict:
    """Общий контекст сценария (токены, id, временные данные)."""
    return {}
```

Использовать для «мелочей», которые не тянут отдельную фикстуру. Предпочитать
`target_fixture` для именованных сущностей.

## Браузер (Selenium)

```python
@pytest.fixture(scope="function")
def browser():
    options = Options()
    options.add_argument("--headless=new")
    driver = webdriver.Chrome(options=options)
    yield driver
    driver.quit()
```

Playwright: использовать фикстуры `pytest-playwright` (`page`, `context`, `browser`).

## API-клиент

```python
@pytest.fixture(scope="function")
def api_client() -> APIClient:
    client = APIClient(base_url=settings.base_url)
    yield client
    client.close()
```

## Scope и очистка

- `scope="function"` по умолчанию — максимальная изоляция.
- Дорогие ресурсы (`scope="session"`) — только если безопасно для изоляции.
- Очистка — `yield` + teardown; данные, созданные сценарием, удаляются.

## target_fixture vs context

| Нужно                                 | Использовать               |
| ------------------------------------- | -------------------------- |
| Именованная сущность для других шагов | `target_fixture="article"` |
| Набор временных значений              | `context`                  |
| Переиспользуемый ресурс               | pytest-фикстура            |

## Переиспользование фикстур из unit-тестов

Фикстуры unit-тестов доступны в шагах через DI:

```python
@pytest.fixture
def article() -> Article:
    return Article(is_beautiful=True)

@given("у меня есть красивая статья")
def beautiful(article: Article) -> None:
    pass
```

Значение фикстуры вычисляется один раз в рамках scope и кэшируется.

## Порядок и изоляция

- Не полагаться на порядок шагов вне сценария.
- Не использовать глобальные переменные модуля.
- Один сценарий не оставляет данных, влияющих на другой.

## Антипаттерны

| ❌                                          | ✅                         |
| ------------------------------------------- | -------------------------- |
| Глобальные переменные                       | `context`/`target_fixture` |
| `scope="session"` для изменяемого состояния | `scope="function"`         |
| Нет teardown для созданных данных           | `yield` + очистка          |
| Ресурс создаётся в шаге                     | Фикстура                   |
| `time.sleep()` в фикстуре                   | явные ожидания             |

## Чек-лист

- [ ] Состояние — через `target_fixture`/`context`.
- [ ] `scope="function"` по умолчанию.
- [ ] Очистка ресурсов через `yield`.
- [ ] Нет глобальных переменных.
- [ ] Фикстуры переиспользуемы и изолированы.
