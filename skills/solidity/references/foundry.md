# Foundry

`forge` (сборка/тесты), `cast` (RPC/утилиты), `anvil` (локальная нода/форк),
`chisel` (REPL). Зависимости — git-субмодули в `lib/`.

## Структура проекта

```
.
├── src/                     # контракты
│   ├── PolygonStandardPayment.sol
│   └── TreasuryConverter.sol
├── tests/                   # тесты (Contract.t.sol)
├── scripts/                 # *.s.sol: деплой, админ, симуляции
├── deployments/             # JSON-артефакты деплоя (создаются скриптами)
├── lib/                     # git-субмодули (forge-std, openzeppelin-contracts)
├── deploy-bootstrap.sh      # автономный бутстрап (блок, холдер DAI, деплой)
├── docker-compose.yaml      # rpc-proxy, anvil (форк), forge, deploy
├── foundry.toml
├── .env / .env.example
└── README.md
```

> Пути — `tests/` и `scripts/` (plural). Указать в `foundry.toml`:
> `test = "tests"`, `script = "scripts"`.

## foundry.toml

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
verbosity = 3

[fmt]
line_length = 100
tab_width = 4
bracket_spacing = false
int_types = "long"
quote_style = "double"
```

## Команды

| Действие | Команда |
|---|---|
| Сборка | `forge build` |
| Тесты | `forge test` |
| Тесты + газ | `forge test --gas-report` |
| Один тест | `forge test --match-contract XTest --match-test test_X -vvvv` |
| Покрытие | `forge coverage` |
| Fuzz (10k) | `forge test --fuzz-runs 10000` |
| Газ-снапшот | `forge snapshot` |
| Формат | `forge fmt` |
| Storage layout | `forge inspect <Contract> storage-layout` |
| Локальная нода | `anvil` |
| Форк сети | `anvil --fork-url <RPC>` |
| Деплой (dry-run) | `forge script scripts/DeployPolygonStandardPayment.s.sol --rpc-url <RPC> --sender <ADDR>` |
| Деплой + broadcast | `forge script ... --broadcast --private-key <PK>` |

`cast` — `cast call`, `cast send`, `cast decode-abi`, `cast wallet`.
`chisel` — интерактивная проверка Solidity.

## Docker: anvil-форк и forge

`docker-compose.yaml` поднимает изолированную среду: `rpc-proxy`, `anvil` (форк
mainnet), `forge`, `deploy`. Позволяет гонять тесты/деплой против форка без
внешнего RPC.

```bash
docker compose up -d anvil        # форк-нода
docker compose run --rm forge test
docker compose run --rm deploy    # forge script ... --broadcast против форка
```

- `deploy-bootstrap.sh` — автономный сценарий: подготовка блока/холдера DAI,
  деплой, проверка. Использовать для воспроизводимого локального/CI-прогона.
- Секреты (RPC, private key) — через `.env` (в `.dockerignore`/`.gitignore`).

## Тесты

```solidity
// tests/TestPolygonStandardPayment.t.sol
contract TestPolygonStandardPayment is Test {
    PolygonStandardPayment payment;
    address user = address(0xA11CE);

    function setUp() public {
        payment = new PolygonStandardPayment();
        vm.label(user, "user");
    }

    function test_Deposit() public {
        vm.deal(user, 1 ether);
        vm.prank(user);
        payment.deposit{value: 1 ether}();
        assertEq(payment.balances(user), 1 ether);
    }

    function testFuzz_Deposit(uint256 amount) public {
        amount = bound(amount, 1, 1000 ether);
        vm.deal(user, amount);
        vm.prank(user);
        payment.deposit{value: amount}();
        assertEq(payment.balances(user), amount);
    }

    function test_RevertWhen_Zero() public {
        vm.expectRevert("Zero amount");
        payment.deposit{value: 0}();
    }
}
```

- Asserts: `assertEq`, `assertTrue`, `assertGt`, `assertLt`, ...
- `vm.label(addr, "name")` — читаемые трейсы (обязательно).
- Fuzz: `bound`/`vm.assume`; инварианты: `StdInvariant` + `targetContract`.
- Негативные сценарии: `vm.expectRevert`.

## Скрипты

```
scripts/
├── DeployPolygonStandardPayment.s.sol   # деплой
├── Admin.s.sol                          # админ-операции
├── ProcessPayment.s.sol                 # обработка платежа
├── ReadState.s.sol                      # чтение состояния
└── SimulatePayment.s.sol                # симуляция
```

```solidity
contract DeployPolygonStandardPayment is Script {
    function run() external returns (PolygonStandardPayment payment) {
        vm.startBroadcast();
        payment = new PolygonStandardPayment();
        vm.stopBroadcast();
    }
}
```

- Артефакты деплоя (адреса/ABI) — в `deployments/*.json`; писать из скрипта.
- Секреты — из окружения (`--private-key`/`--ledger`/`cast wallet`), не в коде.

## Чек-лист

- [ ] `forge build`/`forge test` зелёные (`tests/`).
- [ ] Fuzz на критике; инварианты где нужно.
- [ ] `vm.label` в тестах; негативные сценарии.
- [ ] `forge fmt` применён.
- [ ] Скрипты в `scripts/`; артефакты — в `deployments/`.
- [ ] Секреты не в коде; форк-прогон через `docker compose`.
