---
name: php
description: >
  Use for basic modern PHP 8.x work: strict types, PSR-12, layered architecture,
  typed DTOs, constructor DI, prepared statements, PSR-3 logging, PHPUnit, and
  platform rules for WooCommerce/OpenCart/PrestaShop. Триггеры: PHP, PSR-12,
  strict_types, PHPStan, Composer, WordPress, WooCommerce, OpenCart, PrestaShop.
  Непрофильный язык — только базовые принципы.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: language
  triggers: PHP, PSR-12, strict_types, PHPStan, Composer, WordPress, WooCommerce, OpenCart, PrestaShop
  role: specialist
  scope: implementation
  output-format: code
  related-skills: security, git-commits
---

# PHP (basic)

Базовые принципы современного PHP 8.x (без фреймворк-специфики, кроме платформенных
правил ниже).

## Когда применять

- PHP-код: контроллеры, сервисы, репозитории, DTO.
- WordPress/WooCommerce/OpenCart/PrestaShop — платформенные правила.
- Ревью на типизацию, PSR, безопасность.

## Стандарты кода

- Каждый файл: `<?php` + `declare(strict_types=1);`.
- Закрывающий `?>` запрещён.
- PSR-1/PSR-12; отступы 4 пробела; строка ≤ 120.
- Именование: классы `PascalCase`, методы/переменные `camelCase`,
  константы `UPPER_CASE`.

## Архитектура (не CMS)

| Слой | Ответственность |
|---|---|
| Controller | HTTP-вход, валидация, вызов сервиса |
| Service | бизнес-логика, транзакции, оркестрация |
| Repository | только БД (PDO/Query Builder/ORM) |
| DTO/Entity | структуры данных |

```php
final class UserService
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly MailerInterface $mailer,
    ) {}

    public function register(string $email, string $password): User
    {
        if ($this->users->existsByEmail($email)) {
            throw new UserAlreadyExistsException($email);
        }
        $user = $this->users->create($email, password_hash($password, PASSWORD_ARGON2ID));
        $this->mailer->sendWelcome($user->getId());
        return $user;
    }
}
```

- Зависимости — только через конструктор; без `global` и синглтонов.
- Сервис не знает про `$_GET`/`$_POST`/Request.
- Мутации — в транзакции; доменные исключения вместо `false`/пустых массивов.
- Repository — только запросы, без бизнес-логики и N+1 в циклах.

## Типизация

- Type hints для аргументов, возвратов, свойств; union/`void`/nullable.
- `mixed` — только при крайней необходимости.
- `readonly`-свойства и value-objects где уместно.

## Безопасность (критично)

- Не доверять `$_GET`/`$_POST`/сырому вводу.
- SQL — только **prepared statements**; интерполяция запрещена.
- XSS — экранировать вывод (`htmlspecialchars`/шаблонизатор).
- Пароли — `password_hash()` (`PASSWORD_ARGON2ID`/`PASSWORD_BCRYPT`).
- Оператор `@` строго запрещён.
- WooCommerce/WordPress: `wp_verify_nonce` + `current_user_can`.

## Ошибки и логи

- Доменные исключения; не возвращать коды ошибок.
- PSR-3 логгер; события `snake_case` прошедшего времени + контекст.

```php
$logger->info('user_registered', ['user_id' => $user->getId()]);
$logger->error('payment_failed', ['payment_id' => $paymentId, 'reason' => $e->getMessage()]);
```

- Запрещено: `echo`/`print_r`/`var_dump`/`error_log` с произвольными строками в прод.

## Тестирование (unit)

- PHPUnit + моки (PHPUnit MockObject/Prophecy); без реальной БД.
- Тестируем бизнес-логику сервисов, валидаторы, расчёты, side-effects (моками).
- Не тестируем простой CRUD, фреймворк/CMS, шаблоны, конфиг.
- PHPStan level 9 перед сдачей.

## Платформенные правила

- **WooCommerce/WordPress:** только Actions/Filters; не править ядро/плагины;
  CRUD WooCommerce (`wc_get_product()`, `$product->save()`), не `update_post_meta`;
  `wp_verify_nonce` + `current_user_can`; `WC_Logger`; WPCS.
- **OpenCart:** Controller → Model → View (Twig) → Language; всё через Events;
  OCMOD/vqmod минимизировать; ресурсы через `$this->registry`; строки — из
  языковых файлов.
- **PrestaShop:** новые модули — Symfony-контроллеры; legacy — ObjectModel;
  хуки в `install()`; БД — `Db::getInstance()` + `DbQuery`; переменные в шаблоны
  только из контроллера.

## Документация

- PHPDoc обязателен для сложных/публичных методов сервисов и неочевидных
  side-effects; тривиальные геттеры/`__construct` — нет.

## Запрещённые паттерны

| ❌ | ✅ |
|---|---|
| Нет `declare(strict_types=1)` | в каждом файле |
| Закрывающий `?>` | без него |
| `global`/синглтоны | DI через конструктор |
| Бизнес-логика в контроллере | Service |
| SQL-интерполяция | prepared statements |
| `@` оператор | явная обработка ошибок |
| `var_dump`/`echo` в проде | PSR-3 логгер |
| незахешированные пароли | `password_hash()` |
| N+1 в циклах | eager/батчи |

## Чек-лист

- [ ] `declare(strict_types=1);`, нет `?>`; PSR-12.
- [ ] Все методы/свойства типизированы.
- [ ] DI через конструктор; логика в Service, не в Controller.
- [ ] Repository без логики; prepared statements.
- [ ] `password_hash()`; нет `@`.
- [ ] Доменные исключения; PSR-3 логи `snake_case` + контекст.
- [ ] Unit-тесты сервисов; PHPStan level 9.
- [ ] Платформенные правила соблюдены.
