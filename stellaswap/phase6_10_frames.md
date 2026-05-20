# StellaSwap — Фаза 6: 10 рамок системного злоупотребления

**Дата:** 2026-05-20

---

## Рамка 1: Desynchronization (Рассинхронизация)

**Применима?** Да.

**Гипотеза:**  
Virtual pool's `currentLiquidity` может рассинхронизироваться с реальной картиной farming позиций через `increaseLiquidity` без farming notification. Реальная ликвидность позиции растёт, virtual pool не знает.

**Более серьёзный вариант:** Через `detachIncentive` в момент, когда поставщики ликвидности мигрируют из одного farming в другой. Новый virtual pool не имеет истории тиков. Старые позиции (в старом virtual pool) накопили `innerRewardGrowth` относительно старого pool. Если позиция перезаходит в НОВЫЙ incentive — её snapshot сбрасывается. Если остаётся в старом (detached) — rewards заморожены.

**Что нужно в коде:** Проверить, что при `detachIncentive` все существующие farmers могут корректно выйти и получить правильные rewards. Проверить `getInnerRewardsGrowth` на detached virtual pool (тики больше не обновляются cross()).

**Что убивает гипотезу:** Если rewards рассчитываются строго на основе снэпшотов и виртуальный пул не нуждается в обновлении для корректного расчёта — детач безопасен.

---

## Рамка 2: Double Meaning (Двойной смысл)

**Применима?** Да.

**Гипотеза:** L2 NFT в FarmingCenter имеет двойной смысл:
1. Для пользователя — "receipt" депозита
2. По ERC721 механике — "полные права контроля над позицией, farming rewards и underlying NFT"

`approve(L2TokenId, operator)` в пользовательском понимании = "разрешить оператору управлять". В контрактном понимании = "дать оператору право withdraw ВСЁ, включая rewards".

**Что нужно в коде:** Проверить, есть ли UI/документация предупреждение о том, что approve L2 NFT = полная передача контроля.

**Что убивает гипотезу:** Если пользователи никогда не делают approve L2 NFT — вектор неактивен. Если нет aggregator/marketplace для L2 NFTs — риск теоретический.

---

## Рамка 3: Lifecycle Confusion (Путаница жизненного цикла)

**Применима?** Да — сильно.

**Гипотеза:** Expired LimitFarming продолжает получать `cross()` уведомления через FarmingCenter proxy mode. После `desiredEndTimestamp`:
- `_increaseCumulative` возвращает NOT_EXIST
- Но `cross()` не проверяет время — продолжает обновлять `globalTick` в expired virtual pool

Позиции, застрявшие в истёкшем LimitFarming (не вызвавшие `exitFarming`), получают `globalTick` обновления post-expiry. При их `exitFarming`, расчёт `globalSecondsPerLiquidityCumulative` основан на правильно ограниченном периоде (endTimestamp), но `globalTick` мог сместиться за это время. 

В `LimitFarming.exitFarming` rewards основаны на `RewardMath`, который использует `seconsdPerLiquidityInsideX128`. Если `globalTick` менялся после endTimestamp (через cross()), это влияет на `getInnerSecondsPerLiquidity` при exitFarming.

**Что нужно в коде:** Трассировка `AlgebraLimitFarming.exitFarming` — точная формула reward. Проверить, зависит ли расчёт от `globalTick` состояния expired virtual pool.

**Что убивает гипотезу:** Если LimitFarming rewards рассчитываются строго на основе времени в [startTime, endTime] и не зависят от текущего `globalTick` virtual pool — вектор закрыт.

---

## Рамка 4: Conservation Violation (Нарушение сохранения)

**Применима?** Потенциально.

**Гипотеза:** В EternalFarming, `rewards[owner][token]` аккумулируется без ограничения. При очень длительном периоде без claim, теоретически может overflow. Комментарий `// user must claim before overflow` подтверждает осознание этого разработчиками.

При overflow: `rewards[owner][token]` = overflow_value (меньше реального). User теряет rewards, которые они заработали.

Более практичная conservation violation: `incentive.totalReward` (глобальный счётчик заработанного) vs `totalClaimable` (сумма всех `rewards[user]`). При overflow в rewards маппинге, sum(rewards[user]) < incentive.totalReward. Этот "разрыв" никуда не денется — overflow потерян.

**Что нужно в коде:** При каком totalReward/liquidity/time комбинации overflow возможен? При STELLA reward rate и типичных liquidity значениях — вычислить.

**Что убивает гипотезу:** При разумных параметрах (reward rate, liquidity, time) overflow недостижим практически.

---

## Рамка 5: Authority Substitution (Замена полномочий)

**Применима?** Да.

**Гипотеза:** `onlyIncentiveMaker` модификатор — единственная защита для критических admin операций (createEternalFarming, setRates, detachIncentive). Если `incentiveMaker` — EOA, а не multisig, компрометация одного ключа = полный контроль над farming параметрами.

Атака: 
1. Получить private key incentiveMaker
2. `setRates(0, 0)` → распределение stops
3. Frontrun own `setRates(0,0)` с claim всех своих накопленных rewards
4. `detachIncentive` → другие фермеры теряют доступ к rewards через normal flow
5. `addRewards` на подконтрольный incentive (с другим beneficiary через addRewards пустая — он уже внутри virtualPool)

**Что нужно в коде:** Адрес `incentiveMaker` в deployed AlgebraEternalFarming (`0xd4b2B7DC9Bc1C47852851f4C5Dc345eabA1A5279`). Проверить через Moonscan: EOA или контракт?

**Что убивает гипотезу:** Если incentiveMaker — multisig с timelock.

---

## Рамка 6: Path Substitution (Замена пути)

**Применима?** Умеренно.

**Гипотеза:** Router V3 `exactInput` принимает `path` как bytes. Если frontend кодирует некорректный path (устаревший pool address) или если атакующий может подменить path в signed transaction — swap идёт через иной пул.

Более конкретно: Router V3 не проверяет, что пулы в path созданы через AlgebraFactory. Он просто вычисляет pool address из path данных. Если pool address в path указывает на произвольный контракт, который реализует IAlgebraPool interface — swap будет исполнен через этот контракт.

**Что нужно:** Проверить, вычисляет ли Router pool address детерминированно через deployer (нет возможности подставить произвольный адрес) или принимает произвольный pool address из path.

**Что убивает гипотезу:** Если Router V3 вычисляет pool address через `PoolAddress.computeAddress(deployer, key)` — path substitution невозможна (адрес детерминирован фабрикой).

---

## Рамка 7: Failure Harvesting (Сбор ценности из отказов)

**Применима?** Да.

**Гипотеза:** При bridged token depeg в V2 пуле:
1. Bridged token (USDC.multi) теряет стоимость
2. Арбитражёры дренируют реальные токены из пула
3. LP holders застряли с worthless LP tokens
4. DualFarms продолжает принимать эти LP tokens и выдавать STELLA

Failure → useful state: пул "мёртв" экономически, но ещё функционирует контрактно. Кто-то может дёшево купить эти LP tokens на вторичном рынке, застейкать, получить STELLA rewards, продать STELLA.

Конкретно: Нужно проверить, есть ли ещё active farms с Multichain (.multi) или Nomad (.mad) LP tokens. Если есть — это активный failure harvesting vector.

**Что нужно в коде:** DualFarms.poolInfo — список pid и lpToken адресов. Проверить, включают ли .multi или .mad tokens.

**Что убивает гипотезу:** Если StellaSwap уже удалил такие farms. Если .multi tokens имеют 0 TVL в пулах.

---

## Рамка 8: Observer Exploit (Эксплуатация наблюдателей)

**Применима?** Да.

**Гипотеза:** Indexer / The Graph subgraph кэширует APR на основе текущего `rewardRate` без учёта `rewardReserve`. Отображаемый APR может быть завышен (реальный reserve исчерпан). Пользователи, видящие высокий APR в UI, входят в farming — получают 0 rewards.

Более конкретно: если `rewardRate > 0` но `rewardReserve = 0`, virtual pool не распределяет ничего, но frontend показывает APR = `rewardRate * annualization`.

**Что нужно:** Проверить APR формулу в StellaSwap frontend/subgraph. Включает ли она проверку rewardReserve?

**Что убивает гипотезу:** Если frontend читает как rewardRate, так и rewardReserve для расчёта реального APR.

---

## Рамка 9: Economic Inversion (Экономическая инверсия)

**Применима?** Да.

**Гипотеза:** StableSwap rampA — admin инициирует снижение A. LP providers, которые ДОЛЖНЫ защищать пул (оставаться LPs), вместо этого получают стимул выйти быстрее всех (банковский набег). Чем больше LP уходит, тем меньше A нужно снижать (пул уже дренирован), но тем хуже для оставшихся.

Incentive alignment: 
- Честные LP пытаются защитить пул → теряют больше от arb
- Быстрые/информированные LP уходят первыми → сохраняют стоимость
- Admin, знающий о rampA, может первым выйти

Это inverting честного поведения: "будь LP" становится убыточной стратегией при pending rampA.

**Что убивает гипотезу:** Если rampA gradual и не создаёт немедленного significant imbalance — набег нерационален. Если A меняется медленно (дни) — LP успевают выйти без паники.

---

## Рамка 10: Boundary Collapse (Схлопывание границ)

**Применима?** Да.

**Гипотеза:** FarmingCenter — граница между NonfungiblePositionManager (LP position world) и EternalFarming (reward world). Через FarmingCenter:
- NFT deposit создаёт L2 NFT (новый identity)
- Original tokenId = ID в NonfungiblePositionManager
- L2TokenId = ID в FarmingCenter ERC721 system

Граница схлопывается, когда кто-то использует original tokenId в farming call (FarmingCenter принимает original tokenId в `enterFarming`, `exitFarming`) И L2TokenId для authorization (`checkAuthorizationForToken(deposit.L2TokenId)`).

Потенциальная путаница: если `deposits[tokenId].L2TokenId` когда-либо совпадает с реальным tokenId другой позиции (из NonfungiblePositionManager), возникает ID collision между двумя identity spaces.

**Когда это теоретически возможно:** `L2TokenId = _nextId` starts at 1 and increments. `tokenId` (original NFT) = `_nextId` of NonfungiblePositionManager, тоже starts at 1. При определённых условиях (маленькие deployments) L2TokenId = original tokenId другой позиции.

**Практически:** Authorization check использует L2TokenId для `_isApprovedOrOwner` — проверяет в FarmingCenter's ERC721 (L2 NFTs). Original NFT и L2 NFT — в РАЗНЫХ ERC721 контрактах. Collision безвредна — они в разных пространствах.

**Что убивает гипотезу:** Контракты используют разные ERC721 контракты для ID spaces. Collision impossible при правильной implementation.

---

## Сводка рамок

| Рамка | Применима | Сила | Гипотеза |
|---|---|---|---|
| 1. Desynchronization | Да | Medium | Virtual pool state lag после increaseLiquidity или detach |
| 2. Double Meaning | Да | Medium | L2 NFT approve = полная передача контроля |
| 3. Lifecycle Confusion | Да | Medium-High | Expired LimitFarming cross() post-expiry |
| 4. Conservation Violation | Теоретически | Low | rewards overflow |
| 5. Authority Substitution | Да | High (if EOA) | incentiveMaker compromise |
| 6. Path Substitution | Умеренно | Low | Router path injection (вероятно закрыто) |
| 7. Failure Harvesting | Да | Medium | Worthless LP → real STELLA rewards |
| 8. Observer Exploit | Да | Medium | Stale APR display (rate=0, reserve=0) |
| 9. Economic Inversion | Да | Medium | rampA bank run incentive |
| 10. Boundary Collapse | Теоретически | Low | L2 NFT ID collision (закрыто) |
