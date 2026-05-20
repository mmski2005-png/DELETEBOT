# StellaSwap — Фаза 1: Восстановление системы с первых принципов

**Дата:** 2026-05-20  
**Сеть:** Moonbeam Mainnet (Chain ID: 1284)  
**Объект анализа:** app.stellaswap.com — пулы ликвидности

---

## Таблица источников

| Источник | Статус | Примечания |
|---|---|---|
| Code / contracts (Factory V1, Router V1, StableSwap, AlgebraFactory, AlgebraPool, EternalFarming, FarmingCenter, DualFarms V2, NonfungiblePositionManager) | PROVIDED (через Moonscan verified code + GitHub AlgebraV1) | Частичное: ABI и структура функций получены, полный байткод не декомпилирован |
| Docs (docs.stellaswap.com) | PROVIDED (частично) | Часть страниц 404, базовая архитектура получена |
| UI assumptions (app.stellaswap.com/pools) | UNTRUSTED / INFERRED | UI не является источником истины; структуры пулов получены косвенно |
| Backend / indexer | MISSING | Subgraph API (thegraph) не опрошен напрямую |
| Events (on-chain emitted events) | NEEDS VERIFICATION | Не декодированы transaction traces |
| API | MISSING | API endpoints не исследованы |
| Off-chain workers | INFERRED | Keeper-механизм EternalFarming через setRates предполагается, не подтверждён |
| Admin / governance actions | INFERRED | setOwner, setOperator, setFarmingAddress — функции существуют, история вызовов не проверена |
| Economic incentives | INFERRED | Механизм STELLA emission известен по структуре кода |
| Cross-chain / external integrations | PROVIDED (структурно) | XCM, Wormhole, Axelar, Squid, Frax Ferry — партнёрства подтверждены, внутренняя логика MISSING |

---

## 1. Объекты системы

### 1.1 Пул V2 (Uniswap V2-fork)

- **Где живёт:** контракт-пара, созданный StellaSwapV2Factory (`0x68a384d826d3678f78bb9fb1533c7e9577dacc0e`)
- **Как создаётся:** `createPair(tokenA, tokenB)` — детерминированный адрес через CREATE2
- **Что хранит:** резервы двух токенов (reserve0, reserve1), supply LP-токенов, feeTo, swapFee, devFee
- **Как изменяется:** через `swap()`, `mint()`, `burn()` в контракте пары
- **Жизненный цикл:** создаётся раз, существует бессрочно, не уничтожается
- **ID:** адрес контракта пары; LP-токен с ERC20 адресом
- **Владелец:** у пары нет owner; Factory имеет `feeToSetter`

### 1.2 Пул StableSwap (Curve-style)

- **Где живёт:** отдельный контракт (пример: 4pool Wormhole `0xb1bc9f56103175193519ae1540a0a4572b1566f6`)
- **Как создаётся:** задеплоен вручную (не через фабрику, требует подтверждения)
- **Что хранит:** балансы N токенов, параметр A (amplification), accumulated admin fees, LP-токен supply, флаги pause
- **Механизм:** invariant Curve D = A·n^n·Σxi + D^(n+1)/(n^n·Πxi)
- **Admin-функции:** rampA/stopRampA (изменение A), setSwapFee, setAdminFee, withdrawAdminFees, pause/unpause
- **Флэш-лоан:** присутствует в контракте, но отключён guard'ом `"Flashloan is not enabled yet"`
- **Жизненный цикл:** deploy → активен → (опционально) pause → unpause
- **ID:** адрес контракта пула; LP-токен ERC20

### 1.3 Позиция Pulsar (Algebra concentrated liquidity)

- **Где живёт:** NonfungiblePositionManager (`0x1FF2ADAa387dD27c22b31086E658108588eDa03a`)
- **Как создаётся:** `mint(params)` — записывает tickLower, tickUpper, liquidity, token0, token1, fee growth snapshots
- **Как изменяется:** `increaseLiquidity()`, `decreaseLiquidity()`, `collect()` (сбор fees)
- **Как уничтожается:** `burn()` после полного вывода ликвидности, но NFT-ID остаётся
- **ID:** tokenId (uint256 ERC721)
- **Владелец:** holder NFT-токена; при стейкинге в FarmingCenter — FarmingCenter становится holder
- **Жизненный цикл:** mint → (optional) depositToFarming → exitFarming → decreaseLiquidity → burn

### 1.4 AlgebraPool (пул концентрированной ликвидности)

- **Где живёт:** создаётся AlgebraPoolDeployer (`0x965A857955d868fd98482E9439b1aF297623fb94`) по заказу AlgebraFactory
- **Что хранит:** globalState (price sqrtPriceX96, tick, fee, timepointIndex, locked), ticks mapping, positions mapping, liquidity, activeIncentive
- **Ключевой механизм:** при свапе пересекает тики; при каждом пересечении вызывает `activeIncentive.cross(tick, zeroToOne)` на виртуальном пуле
- **activeIncentive:** ОДИН адрес виртуального пула на пул в любой момент времени
- **Жизненный цикл:** deploy → активен бессрочно, нет destroy; можно pause через Factory

### 1.5 Позиция в FarmingCenter (staked NFT)

- **Где живёт:** FarmingCenter (`0x0D4F8A55a5B2583189468ca3b0A32d972f90e6e5`)
- **Как создаётся:** пользователь переводит NFT позиции в FarmingCenter через ERC721 `safeTransferFrom`; хранится в `deposits[tokenId]` = {owner, numberOfFarms, inLimitFarming}
- **Как изменяется:** `enterFarming()` (начать фармить), `exitFarming()` (выйти), `collectRewards()`, `claimReward()`
- **Как уничтожается:** `withdrawToken()` — возвращает NFT владельцу при numberOfFarms == 0
- **ID:** tokenId (тот же, что у NFT позиции)
- **Владелец:** в deposits.owner записан оригинальный owner

### 1.6 Incentive (программа наград EternalFarming)

- **Где живёт:** AlgebraEternalFarming (`0xd4b2B7DC9Bc1C47852851f4C5Dc345eabA1A5279`)
- **Как создаётся:** `createEternalFarming(key, rewardAmount, bonusRewardAmount, pool, startTime, endTime, minimalPositionWidth, tierMultipliers)` — только оператором
- **Что хранит:** mapping incentives[key] = {totalReward, bonusReward, virtualPoolAddress, rewardRate, bonusRewardRate, startTime, endTime}
- **Виртуальный пул:** для каждого incentive создаётся отдельный VirtualPool — копия AlgebraPool для трекинга активной ликвидности
- **Жизненный цикл:** create → активен (фермеры входят/выходят) → deactivated (если pool помечает NOT_EXIST)

### 1.7 LP-токен

- **Где живёт:** ERC20 контракт (для V2 и StableSwap), или NFT ERC721 (для Pulsar)
- **Как создаётся:** V2 — mint при add liquidity; StableSwap — mint при addLiquidity; Pulsar — NFT mint
- **Жизненный цикл:** mint → transfer → (stake в ферму) → unstake → burn

### 1.8 Токены мостов (bridged assets)

- **Типы:** xcTokens (XCM, нативные parachain assets), .multi (Multichain), .mad (Nomad — DEPRECATED), wh (Wormhole), ce (Celer), ATOM/UST/LUNA (Axelar), Frax (Frax Ferry)
- **Особенность:** каждый тип имеет СВОЙ custodian/bridge; потеря пега у одного не затрагивает другие
- **Статус Nomad:** DEPRECATED, уязвимость эксплуатирована в августе 2022, фонды частично потеряны
- **ID:** ERC20-адрес на Moonbeam; backing assets — на исходной цепи

---

## 2. Акторы системы

### 2.1 Обычный пользователь (User)

- **Может делать:** свап через Router V1/V3, add/remove liquidity в V2/StableSwap/Pulsar, стейкинг LP в Dual Farms, deposit NFT позиции в FarmingCenter, claim rewards, cross-chain swap через Squid+Axelar
- **Что видит:** UI показывает цены, TVL, APR, балансы — всё UNTRUSTED как источник истины
- **Чему доверяет:** UI (UNTRUSTED), backend API (MISSING), собственным кошелькам
- **Что может пропустить:** изменение activeIncentive в пуле, изменение rampA в StableSwap, изменение rewardRate в farming
- **Что может сделать дважды:** enterFarming одного tokenId (защищено проверкой — "Farming already exists")
- **Что может сделать поздно:** exitFarming после endTime incentive; claim после deactivation
- **Получает:** LP fees, STELLA rewards, бонусные токены
- **Теряет:** при depeg bridged assets, при IL, при изменении параметров пула

### 2.2 Контракт (Smart Contract)

- **Router V1 (Uniswap V2-style):** маршрутизирует свапы через V2 пары; использует `getAmountsOut`, `getAmountsIn`; применяет deadline; не имеет собственного state
- **Router V3 (Pulsar):** маршрутизирует через Algebra пулы; `exactInput`/`exactOutput`; path encoding с fee; поддерживает `sweepTokenWithFee` — может взять процент от sweep
- **FarmingCenter:** держит NFT позиции пользователей; счётчик numberOfFarms защищает от двойного стейкинга; ERC721-обёртка
- **VirtualPool:** лёгкая копия AlgebraPool только для farming; получает уведомления о тик-кроссингах; UNKNOWN — полный код виртуального пула на Moonbeam не верифицирован напрямую

### 2.3 Backend / Indexer

- **Статус:** MISSING — API не опрошен
- **Предположительно:** The Graph subgraph (`stellaswap/pulsar`, `stellaswap/pulsar-farming`)
- **Риск:** indexer может отставать от on-chain state; cached TVL/APR могут не отражать реальность
- **Что может считать:** суммарные объёмы, TVL, APR расчёты, статусы позиций

### 2.4 Admin (Team / Multisig)

- **AlgebraFactory:** setOwner, setOperator, setFarmingAddress, setVaultAddress, setBaseFeeConfiguration, pause/unpause, forbidPause
- **StableSwap:** rampA/stopRampA, setSwapFee, setAdminFee, withdrawAdminFees, pause/unpause
- **DualFarms V2:** add/set pools, setEmissionRate, setTeamPercent/setTreasuryPercent, startFarming
- **EternalFarming:** createEternalFarming, setRates (изменение rewardRate и bonusRewardRate) — ТОЛЬКО operator
- **Риск централизации:** NEEDS VERIFICATION — не известно, является ли admin multisig или EOA
- **Что может сделать:** изменить параметры пула в любой момент, изменить скорость наград, поставить pause

### 2.5 Oracle

- **В Algebra:** используется TWAP-оракул через timepoints (хранилище) в AlgebraPool; не Chainlink
- **В DualFarms:** нет ценового оракула — rewards рассчитываются только в единицах STELLA per second
- **В StableSwap:** нет оракула — used invariant-based pricing
- **Статус:** INFERRED — точный механизм TWAP в AlgebraV1 NEEDS VERIFICATION

### 2.6 Bridge / Relayer

- **XCM (Polkadot):** xcTokens поступают через XCMP; Moonbeam обрабатывает как precompile
- **Axelar:** ATOM, UST, LUNA; Axelar GMP; подтверждения на стороне Axelar validators
- **Wormhole:** 4pool Wormhole; wormhole guardians 2/3 multisig подтверждают трансферы
- **Squid+Axelar:** cross-chain swap; маршрутизатор на стороне Squid, исполнение через Axelar GMP
- **Frax Ferry:** кастомный bridge для Frax
- **Multichain:** DEPRECATED/HACKED в 2023 — статус токенов .multi NEEDS VERIFICATION
- **Что может сделать:** задержать поставку токена, не поставить, задвоить

### 2.7 Governance

- **Статус:** INFERRED / MISSING — нет данных о on-chain governance контракте
- **Предположительно:** централизованный admin/multisig без timelock

### 2.8 Solver/Keeper

- **EternalFarming:** setRates() вызывается оператором — вручную или через keeper
- **VirtualPool:** получает cross() колбэк от AlgebraPool при каждом тик-кроссинге автоматически
- **DualFarms:** updatePool() вызывается при каждом deposit/withdraw — permissionless trigger

---

## 3. Порядок событий

### Строго последовательные (order matters):

- `mint position NFT` → `deposit NFT to FarmingCenter` → `enterFarming` (нарушение порядка невозможно — FarmingCenter держит NFT)
- `decreaseLiquidity` → `collect fees` → `burn NFT` (burn требует нулевую ликвидность)
- `exitFarming` → `claimReward` (или collectRewards сначала) → `withdrawToken` (только если numberOfFarms == 0)
- `rampA start` → (время N) → `stopRampA` или `rampA complete` (StableSwap)

### Может происходить в любом порядке:

- `swap` и `addLiquidity` — независимы
- `collectRewards` и `exitFarming` — обе извлекают наработанные rewards, но exitFarming также убирает позицию
- Несколько пользователей входят/выходят из farming одновременно

### Может не произойти:

- `exitFarming` — позиция может остаться в farming indefinitely
- `stopRampA` — rampA может достичь целевого значения без явной остановки
- `claimReward` — токены могут накапливаться и не выводиться

### Может произойти частично:

- Cross-chain swap через Squid+Axelar: swap может исполниться на исходной цепи, но не дойти до Moonbeam (и наоборот) — refund механизм UNKNOWN
- addLiquidity в StableSwap: может принять не все 4 токена (unbalanced add)
- exactOutput своп: `amountIn` может быть меньше `amountInMaximum` — разница не сохраняется контрактом (refund в Router)

### Зависит от времени/блока:

- `endTime` в EternalFarming — после этой отметки новых rewards не начисляется, но по факту: `endTime` в Algebra EternalFarming — это NOT a hard cutoff для уже активных позиций (NEEDS VERIFICATION)
- `deadline` в Router V1/V3 — транзакции откатываются после deadline timestamp
- `rampA` в StableSwap — linearly interpolates A parameter over time
- Harvest interval в DualFarms V2 — `nextHarvestUntil` блокирует harvest до истечения cooldown
- `startTimestamp` в DualFarms V2 — `startFarming()` должен быть вызван перед первым deposit

### Может быть увидено одним компонентом, но не другим:

- Событие `activeIncentive` изменяется в пуле: Pulsar UI не обновится мгновенно
- Deactivation виртуального пула: EternalFarming узнаёт только через следующий swap/cross; позиции продолжают считать себя активными
- StableSwap rampA: изменение A происходит постепенно — indexer может кешировать старое значение A

---

## 4. Память системы

### Что хранит on-chain:

- V2: reserves, LP totalSupply, feeTo, swapFee per pair
- StableSwap: balances[N], A (amplification), swapFee, adminFee, adminFeeAccumulated, paused
- AlgebraPool: globalState (price, tick, fee, timepointIndex, locked), ticks mapping (liquidityNet, feeGrowthOutside), positions mapping, timepoints (TWAP storage)
- NonfungiblePositionManager: positions mapping {token0, token1, tickLower, tickUpper, liquidity, feeGrowth snapshots, tokensOwed}
- FarmingCenter: deposits mapping {owner, numberOfFarms, inLimitFarming}, l2Nfts mapping
- EternalFarming: incentives mapping {rewardToken, bonusToken, virtualPool, rates, startTime, endTime, totalReward...}, farms mapping {liquidity, innerRewardGrowth0/1, tickLower, tickUpper, tierMultiplier}
- DualFarms V2: poolInfo (allocPoint, lastRewardTimestamp, accStella..., totalLp, depositFeeBP, harvestInterval), userInfo (amount, rewardDebt, nextHarvestUntil, rewardLockedUp)

### Что система забывает:

- История свапов (только события, не state)
- Предыдущие значения reserves (только текущие)
- Завершённые incentives — удалены или обнулены в mapping
- Деактивированные LP-позиции после burn NFT

### Выводится только из событий (Events):

- Полная история транзакций
- Объёмы торгов
- Кто когда добавил/вывел ликвидность
- Точная последовательность тик-кроссингов
- История изменений rewardRate в EternalFarming

### Выводится только backend/indexer:

- Текущий APR/APY (требует внешней цены STELLA)
- TVL в USD (требует price feeds)
- Список пулов с метаданными (NFT позиции по диапазонам)
- Farming campaigns — активные/завершённые

### Существует только в UI:

- "Estimated rewards" — UI-расчёт, не on-chain факт
- Pool rankings by TVL/volume
- Price charts

### Что может отличаться между chain state и backend state:

- Текущая активная incentive в пуле (activeIncentive изменяется on-chain, indexer может не обновить)
- Реальные rewards для позиции (зависит от реального TWAP виртуального пула)
- Статус bridged токенов (если bridge заморожен или hacked)

---

## 5. Экономика системы

### Кто платит и кому:

| Действие | Платит | Получает |
|---|---|---|
| Swap V2 | trader (fee из input) | LP holders (через reserves), feeTo (если set) |
| Swap Pulsar | trader (динамическая fee) | LP holders (через feeGrowth), vault (communityFee) |
| Swap StableSwap | trader (0-5% configurable) | LP holders + admin (adminFee % от swapFee) |
| Cross-chain swap | trader (gas на источнике + Axelar/Squid fee) | Axelar validators, Squid protocol |
| Add liquidity V2 | LP (депонирует токены) | LP получает LP-токены, теоретически fees в будущем |
| Add liquidity Pulsar | LP (депонирует токены в range) | LP получает NFT позицию + fees только за активные тики |
| Farming DualFarms | protocol (mint STELLA) | LP stakers (STELLA + бонус через rewarder) |
| Farming EternalFarming | кто создал incentive (задепонировал rewardToken) | NFT позиции в активных тиках |
| Harvest DualFarms | protocol (mint STELLA) | team%, treasury%, investor%, user (остаток) |

### Авансирует средства:

- Team/Treasury для создания incentives в EternalFarming
- Пользователь при addLiquidity

### Получает refund:

- Пользователь при exactInput swap — Router V3 возвращает неиспользованный inputAmount (через callback)
- Пользователь при removeLiquidity — получает обратно underlying tokens

### Кому выгодны задержки:

- Атакующий MEV при frontrunning свапов
- Arbitrageur — задержка rampA даёт время на арбитраж против пула
- Admin — может рампировать A, пока пользователи не заметили и не вышли

### Кому выгодны повторы:

- DualFarms harvestMany — можно батчить harvests через multicall
- FarmingCenter multicall — можно атомарно enterFarming + collectRewards + exitFarming

### Кому выгодно частичное исполнение:

- exactOutput путь позволяет использовать меньше amountInMax; разница возвращается
- Unbalanced addLiquidity в StableSwap: вносить только те токены, которые выгоднее при текущем A

### Кому выгоден fallback path:

- Router V3 sweepTokenWithFee — если у router остались токены, вызывающий может их забрать + fee
- emergencyWithdraw в DualFarms — выход без rewards, но немедленный; выгоден при depeg underlyings

### Кто теряет при сбое:

- LP при bridge exploit (Nomad 2022 — прецедент)
- LP при резком изменении A в StableSwap (IL в stableswap пуле)
- Фермер при deactivation incentive (VirtualPool становится неактивным, но позиция остаётся locked в FarmingCenter до exitFarming)
- User при failed cross-chain swap если refund механизм не сработал

---

## Ключевые UNKNOWNS требующие проверки

| ID | Неизвестное | Что нужно |
|---|---|---|
| U1 | Является ли admin StellaSwap multisig с timelock? | Проверить deployer txs на Moonscan, историю setOwner |
| U2 | Что происходит с позицией при deactivation incentive (VirtualPool NOT_EXIST)? | Код VirtualPool на Moonbeam (не верифицирован напрямую) |
| U3 | Может ли одна позиция быть в EternalFarming и LimitFarming одновременно? | FarmingCenter deposits.numberOfFarms логика, `inLimitFarming` флаг |
| U4 | Верифицирован ли контракт VirtualPool на Moonscan? | Прямой запрос к FarmingCenter → incentive key → virtualPoolAddress |
| U5 | Статус токенов Multichain (.multi) после взлома Multichain 2023 | Ликвидность в V2 пулах с .multi токенами |
| U6 | Как Squid обрабатывает failed cross-chain swap — куда идут средства? | Squid API / Axelar GMP docs |
| U7 | Есть ли slippage guard на уровне cross-chain swap в контракте или только в UI? | Squid/Axelar contracts |
| U8 | Кто контролирует setOperator в AlgebraFactory? | Onchain tx history |
| U9 | Включён ли flash loan в StableSwap пулах на mainnet? | Прямой вызов функции / проверка guard |
| U10 | TWAP оракул в AlgebraPool — используется ли внешними протоколами (lending?)? | Поиск интеграций |

---

## Итог Фазы 1

StellaSwap — это **гибридный DEX на Moonbeam** с тремя типами пулов:
1. **V2 AMM** (Uniswap V2 fork) — классические xy=k пулы
2. **StableSwap** (Curve-style) — пулы для равноценных активов с изменяемым параметром A
3. **Pulsar** (Algebra V1 concentrated liquidity) — позиции в диапазонах, представленные как NFT

Поверх Pulsar работает **двухуровневая система farming**: FarmingCenter держит NFT позиции пользователей, а виртуальный пул (VirtualPool) получает уведомления о тик-кроссингах от реального AlgebraPool, отслеживая активное время ликвидности для начисления rewards.

Система имеет **высокую зависимость от мостовых токенов** многочисленных протоколов (XCM, Wormhole, Axelar, Celer, Frax Ferry), из которых исторический прецедент (Nomad 2022) показал: депег одного класса bridged assets может уничтожить значительную часть TVL системы.

**Критические неизвестные:** состояние admin-контроля (EOA vs multisig), поведение VirtualPool при deactivation, обработка failed cross-chain swaps.
