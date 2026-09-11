# Газ и апгрейдабельность

## Оптимизация газа

- `external` вместо `public` для внешних вызовов.
- `calldata` вместо `memory` для read-only массивов/строк.
- Кэшировать `storage` и `.length` перед циклами.
- `immutable`/`constant` везде, где возможно (не занимают storage).
- Storage packing: переменные одного типа/размера — в один слот.
- `unchecked` для арифметики, где переполнение невозможно.
- События дешевле storage — использовать для истории.
- Не читать storage в цикле — вынести в память.
- Контроль регрессий: `forge snapshot` (`.gas-snapshot` в git).

```solidity
// кэш длины + calldata
function sum(uint256[] calldata xs) external pure returns (uint256 total) {
    uint256 len = xs.length;
    for (uint256 i; i < len;) {
        total += xs[i];
        unchecked { ++i; }
    }
}
```

## Апгрейдабельность (UUPS)

- Предпочтительный паттерн — **UUPS** (логика апгрейда в реализации).
- `constructor` реализации — `_disableInitializers()`.
- Инициализация — только `initialize()` + модификатор `initializer`.
- Апгрейд — `onlyOwner` (или роль) + Timelock/Multisig.
- Хранение — слоты ERC-1967 (`__gap` для будущих полей).

```solidity
contract VaultV1 is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    function initialize() external initializer {
        __Ownable_init(msg.sender);
        __UUPSUpgradeable_init();
    }

    function _authorizeUpgrade(address) internal override onlyOwner {}
}
```

## Storage layout (критично)

- Новые переменные — **только в конец**. Никогда не менять порядок/тип существующих.
- Не удалять переменные без `__gap`/пересчёта слотов.
- Проверять: `forge inspect <Contract> storage-layout`.
- Наследование меняет layout — учитывать порядок базовых контрактов.
- Апгрейд-скрипт — с проверкой совместимости layout (OZ Upgrades / ручной diff).

## Чек-лист

- [ ] `external`/`calldata`/`immutable`/`constant` где возможно.
- [ ] Кэш storage и длины в циклах; `unchecked` только безопасно.
- [ ] `forge snapshot` — нет регрессии газа.
- [ ] UUPS + `_disableInitializers()`; `initialize` защищён `initializer`.
- [ ] Storage layout: только добавление в конец; проверен `forge inspect`.
- [ ] Апгрейд — Timelock/Multisig.
