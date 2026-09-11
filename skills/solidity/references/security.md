# Безопасность Solidity

Контракты необратимы, средства под угрозой. Безопасность — приоритет над газом и
красотой. Ниже — обязательный минимум; общий чек-лист — навык `security`.

## Access control

- Каждая state-changing `external`/`public` — под модификатором.
- Простое: `Ownable2Step`; ролевая модель: `AccessControl`.
- Критичные действия (апгрейд, вывод средств) — Multisig + Timelock.

## Checks-Effects-Interactions (CEI)

```solidity
function withdraw(uint256 amount) external nonReentrant {
    require(balances[msg.sender] >= amount, "Insufficient"); // Checks
    balances[msg.sender] -= amount;                          // Effects
    (bool ok,) = msg.sender.call{value: amount}("");         // Interactions
    require(ok, "Transfer failed");
    emit Withdrawn(msg.sender, amount);
}
```

- Reentrancy: `nonReentrant` + CEI. Осторожно с cross-function/cross-contract.
- Read-only reentrancy: не полагаться на внешний view-вызов как на источник истины.

## Внешние вызовы и токены

- ETH: только `call{value: amount}("")` + проверка `success`.
- ERC-20: только `SafeERC20` (`safeTransfer`/`safeTransferFrom`).
- `.transfer()`/`.send()` запрещены (2300 gas).
- `delegatecall` — только через allowlist проверенных реализаций.
- Проверять `extcodesize`/возврат; не игнорировать результат вызова.

## Оракулы и цены

- Spot-цена без защиты — запрещена (манипуляции, flash loans).
- Chainlink/проверенный оракул + stale/heartbeat проверки; по возможности TWAP.
- Учитывать decimals и порядок цен.

## Случайность и время

- `block.timestamp` — только с допуском ±30–60 сек, **не** для рандома.
- Рандом — Chainlink VRF (или аналог).
- `blockhash` — не источник случайности.

## Типичные уязвимости (SWC/OWASP)

| Категория | Профилактика |
|---|---|
| Reentrancy (SWC-107) | CEI + `nonReentrant` |
| Access control (SWC-105/106) | модификаторы, Multisig/Timelock |
| Unchecked call (SWC-104) | проверять возврат `call`/`SafeERC20` |
| tx.origin (SWC-115) | `msg.sender` |
| Delegatecall (SWC-112) | allowlist, storage layout |
| Randomness (SWC-120) | Chainlink VRF |
| Front-running | commit-reveal, slippage limits |
| DoS (SWC-113/128) | Pull-over-Push, без unbounded циклов |
| Storage collision (proxy) | layout, слоты ERC-1967 |
| Signature replay | nonce, chainId (EIP-712/191) |
| Overflow | checked по умолчанию (0.8+); `unchecked` — осознанно |

## Запрещённые паттерны

| ❌ | ✅ |
|---|---|
| `tx.origin` | `msg.sender` |
| `.transfer()`/`.send()` | `call{value:}("")` + проверка |
| `selfdestruct` | `pause`/`emergencyWithdraw` |
| spot-цена без защиты | Chainlink/TWAP + stale check |
| цикл > 50 элементов | Pull-over-Push |
| `block.timestamp` как рандом | Chainlink VRF |
| повторный `initialize` | модификатор `initializer` |
| изменение storage layout | только добавление в конец |
| приватные данные в storage/event | не хранить |
| `delegatecall` на произвольный адрес | allowlist |

## Чек-лист

- [ ] Access control на всех state-changing функциях.
- [ ] CEI + `nonReentrant`; события emit'ятся.
- [ ] `SafeERC20`/безопасный ETH; проверка возврата вызовов.
- [ ] Оракул защищён от stale/manipulation.
- [ ] Нет `tx.origin`, `.transfer()`, spot-цен, небезопасного рандома.
- [ ] Апгрейд — Timelock/Multisig; storage layout корректен.
