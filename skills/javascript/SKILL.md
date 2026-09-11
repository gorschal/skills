---
name: javascript
description: >
  Use for basic modern JavaScript/Node.js work: ES2023+ syntax, async/await,
  ESM modules, error handling, Node/browser APIs. Триггеры: JavaScript,
  ES2023, Node.js, async await, Promise, ESM, module, JSDoc, eslint, .js/.mjs/.cjs.
  Непрофильный язык — только базовые принципы; фреймворки (React/Vue) вне scope.
license: MIT
compatibility: opencode
metadata:
  version: "1.0.0"
  domain: language
  triggers: JavaScript, ES2023, Node.js, async await, Promise, ESM, module, JSDoc, eslint
  role: specialist
  scope: implementation
  output-format: code
  related-skills: python, git-commits
---

# JavaScript (basic)

Базовые принципы современного JavaScript/Node.js (ES2023+). Непрофильный язык:
фреймворки (React/Vue/Next) вне scope.

## Когда применять

- Vanilla JS/Node: модули, async, утилиты, скрипты.
- Ревью `.js`/`.mjs`/`.cjs` на корректность и базовые практики.

## Ключевые принципы

1. **`const` по умолчанию**, `let` — если переприсваивается; `var` запрещён.
2. **ES2023+**; `?.` и `??` вместо ручных проверок.
3. **`async/await`** для асинхронности; без callback-стиля.
4. **ESM** (`import`/`export`) для новых проектов; не смешивать с CJS.
5. **Обрабатывать ошибки** в async (`try/catch`, проверка `response.ok`).
6. **Не блокировать** event loop (без sync I/O в Node).
7. **Не мутировать** параметры функций.

## Async/await

```js
async function fetchUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
  } catch (err) {
    console.error("fetchUser failed:", err);
    return null;
  }
}
```

- Всегда обрабатывать rejections; не оставлять «висящие» промисы.
- Параллельно — `Promise.all`; с ошибками — `Promise.allSettled`.
- `await` в цикле — только если порядок важен; иначе `Promise.all`.

## Null-безопасность

```js
const city = user?.address?.city ?? "Unknown";   // ✅
const city = user.address.city || "Unknown";      // ❌ бросит при undefined
```

## Модули (ESM)

```js
// utils/math.mjs
export const add = (a, b) => a + b;

// consumer.mjs
import { add } from "./utils/math.mjs";
```

- ESM для новых проектов; именованные экспорты (не default-only для библиотек).
- Не смешивать `require()` и `import` в одном модуле.

## Node / browser (кратко)

- Node: `fs/promises` вместо sync; streams для больших данных; `AbortController`
  для таймаутов/отмены.
- Browser: `fetch`, `AbortController`, `IntersectionObserver`, `localStorage`
  осознанно; Web Workers для тяжёлых вычислений.

## Запрещённые паттерны

| ❌ | ✅ |
|---|---|
| `var` | `const`/`let` |
| callback-стиль | `async/await` |
| смешение ESM и CJS | один стиль |
| sync I/O в Node | `fs/promises` |
| `await` в цикле без нужды | `Promise.all` |
| необработанные rejection | `try/catch`/`allSettled` |
| мутация параметров | новые объекты |

## Чек-лист

- [ ] `const`/`let`, без `var`; `?.`/`??`.
- [ ] Async через `await`; ошибки обработаны.
- [ ] ESM; модули не смешивают стили.
- [ ] Нет sync I/O и блокировок.
- [ ] Публичные API с JSDoc.
- [ ] Линт (`eslint`) проходит.
