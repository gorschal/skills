---
name: solidity
description: >
  Use when writing, testing, reviewing or deploying Solidity smart contracts with
  Foundry (forge, cast, anvil, chisel): access control, checks-effects-interactions,
  reentrancy, external calls, oracles, gas optimization, upgradeability (UUPS),
  fuzz/invariant tests, Slither. Триггеры: Solidity, Foundry, forge, cast, anvil,
  chisel, smart contract, EVM, ERC20, OpenZeppelin, reentrancy, slither, gas,
  upgradeable, UUPS, NatSpec. Безопасность — раздел ниже.
license: Proprietary
compatibility: opencode
metadata:
  version: "1.1.0"
  domain: blockchain
  triggers: Solidity, Foundry, forge, cast, anvil, smart contract, EVM, ERC20, OpenZeppelin, reentrancy, slither, gas, UUPS
  role: specialist
  scope: implementation
  output-format: code
  related-skills: security, python
---

# Solidity + Foundry

Solidity 0.8.24+ и Foundry (`forge`, `cast`, `anvil`, `chisel`) для смарт-контрактов.
Безопасность — критична: контракты необратимы, деньги под угрозой.

## Когда применять

- Написание/ревью контрактов, тестов (Forge), деплой-скриптов.
- Безопасность: access control, reentrancy, оракулы.
- Газ-оптимизация, апгрейдабельность, аудит.

## Ключевые принципы

1. **Безопасность прежде газа и красоты.**
2. **Checks-Effects-Interactions** строго: проверки → состояние → внешние вызовы.
3. **Access control** на каждой state-changing `external`/`public` функции.
4. **Каждое изменение состояния** — `emit` события.
5. **Внешние вызовы** — только безопасными средствами (`call`+проверка, `SafeERC20`).
6. **NatSpec** для всех публичных API.
7. **Тесты обязательны**: unit + fuzz (+ инварианты), негативные сценарии.

## Foundry

```
src/  tests/  scripts/  deployments/  lib/
foundry.toml  docker-compose.yaml  deploy-bootstrap.sh
```

```toml
[profile.default]
src = "src"
out = "out"
test = "tests"
script = "scripts"
libs = ["lib"]
solc = "0.8.24"
optimizer = true
optimizer_runs = 200
fuzz = { runs = 256 }
```

| Действие           | Команда                                                                                   |
| ------------------ | ----------------------------------------------------------------------------------------- |
| Сборка             | `forge build`                                                                             |
| Тесты              | `forge test`                                                                              |
| Тесты + газ        | `forge test --gas-report`                                                                 |
| Один тест          | `forge test --match-contract XTest --match-test test_X -vvvv`                             |
| Покрытие           | `forge coverage`                                                                          |
| Fuzz (10k)         | `forge test --fuzz-runs 10000`                                                            |
| Газ-снапшот        | `forge snapshot`                                                                          |
| Формат             | `forge fmt`                                                                               |
| Storage layout     | `forge inspect <Contract> storage-layout`                                                 |
| Локальная нода     | `anvil` (или `anvil --fork-url <RPC>`)                                                    |
| Деплой (dry-run)   | `forge script scripts/DeployPolygonStandardPayment.s.sol --rpc-url <RPC> --sender <ADDR>` |
| Деплой + broadcast | `forge script ... --broadcast --private-key <PK>`                                         |

Тесты: файл `Contract.t.sol` в `tests/`, контракт `ContractTest is Test`,
`assertEq`/`assertTrue`, `vm.label(...)`, fuzz + `bound`/`vm.assume`, инварианты
`StdInvariant`. Anvil-форк и forge — через `docker-compose.yaml` (`rpc-proxy`,
`anvil`, `forge`, `deploy`); `deploy-bootstrap.sh` — автономный бутстрап.

Подробно: [references/foundry.md](references/foundry.md).

## Стиль и NatSpec

- `// SPDX-License-Identifier` + `pragma solidity ^0.8.24;` в каждом файле.
- `PascalCase` (контракты), `camelCase` (функции/переменные), `UPPER_CASE` (константы).
- NatSpec обязателен для `public`/`external`, событий, модификаторов, важных return.
- Формат — только `forge fmt`; импорты — именованные (`{ERC20} from "..."`).

```solidity
/// @title SecureToken
/// @notice ERC-20 с паузой
contract SecureToken is ERC20, ReentrancyGuard { /* ... */ }
```

## Безопасность (критично)

| Правило        | Требование                                                 |
| -------------- | ---------------------------------------------------------- |
| Access control | модификатор на каждой state-changing функции               |
| CEI            | проверки → эффекты → взаимодействия                        |
| Reentrancy     | `nonReentrant` + CEI                                       |
| События        | каждое изменение состояния emit'ит                         |
| Ошибки         | `require`/`revert` с понятным сообщением или custom errors |

```solidity
function withdraw(uint256 amount) external nonReentrant {
    require(balances[msg.sender] >= amount, "Insufficient");
    balances[msg.sender] -= amount;                    // Effects
    (bool ok,) = msg.sender.call{value: amount}("");   // Interactions
    require(ok, "Transfer failed");
    emit Withdrawn(msg.sender, amount);
}
```

- Access control: `Ownable2Step`/`AccessControl`; критичное — Multisig + Timelock.
- ETH — только `call{value:}("")` + проверка `success`; ERC-20 — `SafeERC20`.
- `.transfer()`/`.send()` запрещены; `delegatecall` — только через allowlist.
- Оракулы: Chainlink/TWAP + проверки stale/heartbeat; spot-цена без защиты запрещена.
- `tx.origin` запрещён; `block.timestamp` — не для рандома (только Chainlink VRF).
- Массовые выплаты — Pull-over-Push; `selfdestruct` — запрещён без крайней необходимости.
- Приватные данные в storage/event — не хранить (всё публично).

Подробно: [references/security.md](references/security.md).

## Газ

- `external` вместо `public`; `calldata` вместо `memory` (read-only).
- Кэшировать `storage`/`.length` перед циклами; `immutable`/`constant`.
- Storage packing; `unchecked` для безопасной арифметики.
- Контроль регрессий: `forge snapshot`.

Подробно: [references/gas-upgrade.md](references/gas-upgrade.md).

## Апгрейдабельность

- Предпочтительно **UUPS**; `constructor` реализации → `_disableInitializers()`.
- Инициализация — `initialize()` + модификатор `initializer`.
- **Storage layout**: новые переменные — только в конец; порядок/типы не менять.
- Проверка: `forge inspect <Contract> storage-layout`.
- Апгрейд — `onlyOwner` + Timelock/Multisig.

Подробно: [references/gas-upgrade.md](references/gas-upgrade.md).

## Тестирование и аудит

- Unit + fuzz для критичной логики; инварианты для протоколов; негативные сценарии
  (`vm.expectRevert`); покрытие ≥ 80% (ядро — 90%+).
- Fork-тесты против mainnet/sepolia — полезно.
- Перед деплоем: `slither .` без High/Critical, `forge coverage`, `forge snapshot`,
  проверка storage layout.

Подробно: [references/testing-audit.md](references/testing-audit.md).

## Запрещённые паттерны

| ❌                                   | ✅                            |
| ------------------------------------ | ----------------------------- |
| `tx.origin`                          | `msg.sender`                  |
| `.transfer()`/`.send()`              | `call{value:}("")` + проверка |
| `selfdestruct`                       | `pause`/`emergencyWithdraw`   |
| spot-цена без защиты                 | Chainlink/TWAP + stale check  |
| цикл > 50 элементов                  | Pull-over-Push                |
| `block.timestamp` как рандом         | Chainlink VRF                 |
| повторный `initialize`               | модификатор `initializer`     |
| изменение storage layout             | только добавление в конец     |
| `delegatecall` на произвольный адрес | allowlist                     |

## Чек-лист перед деплоем

- [ ] `forge test` зелёный; fuzz на критике (≥ 10k runs).
- [ ] Покрытие ≥ 80%; `forge snapshot` без регрессии газа.
- [ ] `slither` без High/Critical; storage layout проверен.
- [ ] Access control на всех state-changing функциях; события emit'ятся.
- [ ] CEI + `nonReentrant`; `SafeERC20`/безопасный ETH.
- [ ] Оракул защищён от stale/manipulation; апгрейд — Timelock/Multisig.
- [ ] NatSpec актуален; `forge fmt` применён; секреты только в `.env`.

## Справочники

| Тема                    | Reference                                                  | Когда                              |
| ----------------------- | ---------------------------------------------------------- | ---------------------------------- |
| Foundry, тесты, команды | [references/foundry.md](references/foundry.md)             | forge/cast/anvil, структура, тесты |
| Безопасность            | [references/security.md](references/security.md)           | access control, CEI, оракулы, табу |
| Газ и апгрейды          | [references/gas-upgrade.md](references/gas-upgrade.md)     | оптимизация, UUPS, storage layout  |
| Тесты и аудит           | [references/testing-audit.md](references/testing-audit.md) | fuzz/invariant, slither, деплой    |

## Связанные навыки

- `security` — общие принципы безопасности.
- `docker` — контейнеризация скриптов/сервисов.
- `git-commits` — атомарные коммиты.
