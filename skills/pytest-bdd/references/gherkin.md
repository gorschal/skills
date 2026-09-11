# Gherkin

Feature-файл — контракт на бизнес-языке. Читается продактом, не разработчиком.

## Структура

```gherkin
@payments @critical
Feature: Создание платежа
  As a merchant
  I want to create a payment
  So that I can receive money from customer

  Background:
    Given я авторизован как мерчант

  @smoke
  Scenario: Успешное создание платежа
    When я создаю платёж на сумму "100.00" в валюте "USD"
    Then статус платежа равен "pending"
    And я получаю payment_id
```

- `Feature` — возможность; `Scenario` — конкретное поведение.
- Один feature-файл — один `Feature`.
- Параметры — в двойных кавычках.
- `Background` — общие предусловия, **только `Given`**.

## Scenario Outline

```gherkin
Scenario Outline: Валидация суммы
  When я создаю платёж на сумму "<amount>"
  Then результат равен "<result>"

  @valid
  Examples:
    | amount | result  |
    | 100.00 | success |

  @invalid
  Examples:
    | amount | result |
    | -1     | error  |
    | 0      | error  |
```

- Вариации — через `Examples`, не копипастой сценариев.
- Несколько таблиц `Examples` можно тегировать; фильтр `-k`/`-m` выбирает подмножество.
- Параметры `<var>` подставляются и в docstring/datatable.

## Rules

```gherkin
Feature: Обработка платежей

  @valid
  Rule: Корректные платежи
    Example: Успешный платёж
      Given ...
```

- `Rule` группирует сценарии/примеры; теги правила наследуются.
- `Example` — алиас `Scenario`.

## Declarative vs imperative

```gherkin
# ❌ imperative — детали UI
When я нажимаю кнопку с id "submit-btn"
And я жду 2 секунды
And я ввожу "user@example.com" в input[name=email]

# ✅ declarative — намерение
When я регистрирую пользователя "user@example.com"
```

Правило: описывать **что**, а не **как**. Селекторы/URL/таймауты — в Page Object.

## Правила хорошего сценария

- Один сценарий — одно поведение (не «и то, и это»).
- 3–7 шагов в среднем, ≤ ~12–15.
- Бизнес-термины, понятные заказчику.
- Без условной логики (`if`) — вариации через `Examples`.
- `Background` — короткий и только `Given`; иначе вынести в фикстуру.

## Теги

| Тег            | Назначение             |
| -------------- | ---------------------- |
| `@smoke`       | критический путь       |
| `@regression`  | полная регрессия       |
| `@critical`    | блокирующий функционал |
| `@api` / `@ui` | тип теста              |
| `@wip`         | в разработке, не в CI  |

Теги → pytest-маркеры; регистрировать в `markers` при `--strict-markers`.

## Антипаттерны

| ❌                               | ✅                            |
| -------------------------------- | ----------------------------- |
| Селекторы/URL/таймауты в Gherkin | В Page Object                 |
| `if`/циклы в сценарии            | `Examples`/отдельные сценарии |
| Сценарий на 30 шагов             | Разбить на несколько          |
| `When`/`Then` в `Background`     | Только `Given`                |
| Копипаста сценариев              | `Scenario Outline`            |
| Технические имена шагов          | Бизнес-язык                   |

## Чек-лист

- [ ] `.feature` читается без знания кода.
- [ ] Один feature — один `Feature`; один сценарий — одно поведение.
- [ ] Параметры в двойных кавычках.
- [ ] `Background` только с `Given`.
- [ ] Вариации через `Scenario Outline` + `Examples`.
- [ ] Теги расставлены и зарегистрированы.
