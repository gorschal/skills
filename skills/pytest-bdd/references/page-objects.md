# Page Objects и API Clients

Вся техническая работа с UI/API — здесь. Шаги остаются тонкими.

## BasePage (UI)

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class BasePage:
    def __init__(self, driver, timeout: int = 10) -> None:
        self.driver = driver
        self.wait = WebDriverWait(driver, timeout)

    def click(self, locator: tuple) -> None:
        self.wait.until(EC.element_to_be_clickable(locator)).click()

    def type_text(self, locator: tuple, text: str) -> None:
        el = self.wait.until(EC.presence_of_element_located(locator))
        el.clear()
        el.send_keys(text)

    def get_text(self, locator: tuple) -> str:
        return self.wait.until(EC.presence_of_element_located(locator)).text
```

## Конкретная страница

```python
from selenium.webdriver.common.by import By

class LoginPage(BasePage):
    URL = "/login"
    EMAIL = (By.NAME, "email")
    PASSWORD = (By.NAME, "password")
    SUBMIT = (By.CSS_SELECTOR, "button[type=submit]")
    ERROR = (By.CSS_SELECTOR, ".error")

    def open(self) -> "LoginPage":
        self.driver.get(f"{settings.base_url}{self.URL}")
        return self

    def login(self, email: str, password: str) -> "LoginPage":
        self.type_text(self.EMAIL, email)
        self.type_text(self.PASSWORD, password)
        self.click(self.SUBMIT)
        return self

    def get_error(self) -> str:
        return self.get_text(self.ERROR)
```

Правила:

- Локаторы — **константы класса**, не в методах/шагах.
- Методы возвращают `self` для fluent-интерфейса, где уместно.
- Никаких `time.sleep()` — только `WebDriverWait` + Expected Conditions.
- Один метод — одно действие страницы.

## BaseAPIClient

```python
import httpx

class BaseAPIClient:
    def __init__(self, base_url: str, timeout: float = 10.0) -> None:
        self.client = httpx.Client(base_url=base_url, timeout=timeout)
        self.token: str | None = None

    def _headers(self) -> dict[str, str]:
        return {"Authorization": f"Bearer {self.token}"} if self.token else {}

    def get(self, path: str, **kwargs) -> httpx.Response:
        response = self.client.get(path, headers=self._headers(), **kwargs)
        logger.info("api_request", method="GET", path=path, status=response.status_code)
        return response

    def close(self) -> None:
        self.client.close()
```

```python
class AuthClient(BaseAPIClient):
    def login(self, email: str, password: str) -> dict:
        response = self.client.post("/auth/login", json={"email": email, "password": password})
        response.raise_for_status()
        self.token = response.json()["access_token"]
        return response.json()
```

- Клиенты наследуют `BaseAPIClient` и логируют запросы/ответы.
- Авторизация — через токен в заголовке.
- Таймауты заданы; `raise_for_status` там, где уместно.

## Playwright-вариант

При Playwright — те же принципы: локаторы в Page Object, `expect`/auto-wait
вместо `sleep`, фикстуры браузера/контекста из pytest-playwright.

## Антипаттерны

| ❌                               | ✅                        |
| -------------------------------- | ------------------------- |
| Локаторы в step-функции          | Константы Page Object     |
| `time.sleep(2)`                  | `WebDriverWait`/auto-wait |
| Page Object без методов-действий | Метод на действие         |
| HTTP-вызовы в шаге               | Метод API Client          |
| Ассерты в Page Object (для UI)   | Ассерты в шагах (`then`)  |
| Хардкод base_url                 | Конфиг/фикстура           |

## Чек-лист

- [ ] Локаторы — константы класса.
- [ ] Нет `time.sleep()`; только явные ожидания.
- [ ] Методы Page Object — действия страницы.
- [ ] API-клиенты логируют запросы/ответы, задают таймауты.
- [ ] `base_url` из конфига.
- [ ] Шаги не содержат Selenium/httpx.
