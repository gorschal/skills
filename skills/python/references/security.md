# Безопасность (Python-уровень)

Языковая специфика; расширенный чек-лист — навык `security`. Фреймворк-настройки —
`django`/`fastapi` (`references/security.md`).

- Секреты — `pydantic-settings`/`SecretStr`; не в коде, git, логах, ответах.
- Валидация входа на границе (Pydantic/формы); `extra="forbid"` на командах.
- SQL — параметризация/ORM; f-строки в SQL запрещены.
- Пароли — `argon2`/`bcrypt`; токены — `secrets` (не `random`).
- Без `eval`/`exec`/`pickle`/`marshal`/`yaml.load` на недоверенных данных.
- Зависимости — `pip-audit`/`safety`; линт — `ruff --select S`, `bandit`.
- Общий обработчик не раскрывает стектрейс; секреты/PII не логируются.

```bash
uv run pip-audit
uv run bandit -r src
uv run ruff check --select S .
```

## Чек-лист

- [ ] Секретов нет в коде/git/логах; `.env` игнорируется.
- [ ] Вход валидируется; SQL параметризован.
- [ ] Пароли хешируются современным алгоритмом.
- [ ] Нет `eval`/`exec`/`pickle` на вводе.
- [ ] `pip-audit`/`bandit`/`ruff S` без критичных находок.
- [ ] Стектрейс не уходит клиенту.
