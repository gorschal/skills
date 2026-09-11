# Тестирование и аудит

## Уровни тестов (Foundry)

- **Unit** — каждая функция, happy path + негатив (`vm.expectRevert`).
- **Fuzz** — критичная логика на случайных входах (`bound`/`vm.assume`).
- **Invariant** — протоколы с инвариантами (`StdInvariant`, `targetContract`).
- **Fork** — против mainnet/sepolia (`vm.createSelectFork`) для интеграций.

```solidity
function testFuzz_Deposit(uint256 amount) public {
    amount = bound(amount, 1, 1000 ether);
    vm.deal(user, amount);
    vm.prank(user);
    bank.deposit{value: amount}();
    assertEq(bank.balances(user), amount);
}

function invariant_TotalEqualsBalances() public {
    assertEq(vault.totalDeposits(), vault.sumBalances());
}
```

## Покрытие и газ

```bash
forge coverage
forge test --fuzz-runs 10000
forge snapshot          # .gas-snapshot в git, контроль регрессий
```

- Покрытие ≥ 80% (ядро — 90%+).
- Fuzz на критике — ≥ 10k runs.
- Снапшот газа — не допускать значимой регрессии.

## Статический анализ

```bash
slither .
```

- Перед деплоем — **без High/Critical**.
- Дополнительно: `forge fmt --check`, `forge build --sizes` (лимит размера).

## Аудит перед mainnet

- [ ] Внешний аудит (для контрактов с деньгами).
- [ ] Bug bounty/тестнет-период.
- [ ] Проверка прав (owner/roles) и плана апгрейда.
- [ ] Инцидент-план: `pause`/`emergencyWithdraw`.

## Чек-лист перед деплоем

- [ ] `forge test` зелёный; fuzz ≥ 10k на критике.
- [ ] Инварианты для протокольной логики.
- [ ] Негативные сценарии покрыты.
- [ ] Покрытие ≥ 80%; `forge snapshot` без регрессии.
- [ ] `slither` без High/Critical; storage layout проверен.
- [ ] Fork-тесты для интеграций с внешними протоколами.
- [ ] NatSpec актуален; секреты только в `.env`.

## Справочники

- Foundry Book: https://book.getfoundry.sh
- Slither: https://github.com/crytic/slither
- SWC Registry: https://swcregistry.io
- OpenZeppelin Contracts: https://docs.openzeppelin.com/contracts
