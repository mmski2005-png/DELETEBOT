# StellaSwap — Фаза 2: Законы системы

**Дата:** 2026-05-20  
**Источник:** AlgebraV1 GitHub (verified), Moonscan verified contracts, StellaSwap docs

---

## Что такое "закон" в данном контексте

Закон — неявное правило, без которого дизайн перестаёт быть корректным. Нарушение закона не обязательно означает баг — оно означает, что поведение системы расходится с тем, на что полагаются её компоненты.

---

## Законы системы

### L1: Liquidity snapshot is immutable during farming

**Формулировка:** Ликвидность, зафиксированная в `Farm.liquidity` при `enterFarming`, не изменяется до `exitFarming`. Именно на неё рассчитываются все farming rewards для данной позиции.

**Какие компоненты полагаются:**
- AlgebraEternalFarming (формула reward = `growthDelta * farm.liquidity`)
- EternalVirtualPool (currentLiquidity = сумма farm.liquidity всех активных позиций)
- Пользователь, ожидающий что его доля наград пропорциональна его ликвидности

**Где enforced:** Только в `enterFarming` (записывается snapshot). Нет механизма обновления farm.liquidity без exit+re-enter.

**Где только assumed:** NonfungiblePositionManager не уведомляет farming систему об `increaseLiquidity` вызовах.

**Что странного при нарушении:** Вызов `increaseLiquidity` на стейкнутой позиции (NFT в FarmingCenter) обновляет реальную позицию в AlgebraPool, но не обновляет `farm.liquidity`. Реальная позиция зарабатывает LP-fees на новой ликвидности, но виртуальный пул продолжает отслеживать старую сумму. Farm.liquidity и actual position.liquidity расходятся.

---

### L2: Virtual pool liquidity mirrors sum of all active farm liquidities

**Формулировка:** `currentLiquidity` в EternalVirtualPool равен сумме `farm.liquidity` всех позиций, чьи тиковые диапазоны включают текущий тик.

**Где enforced:** `applyLiquidityDeltaToPosition` вызывается при `enterFarming` (+delta) и `exitFarming` (-delta), обновляя `currentLiquidity`.

**Где только assumed:** Никто не проверяет глобальный инвариант. Каждая операция обновляет дельту, предполагая, что предыдущие операции были корректны.

**Что странного при нарушении:** Если `currentLiquidity` занижен, `totalRewardGrowth` растёт быстрее — оставшиеся фермеры получают больше наград, чем ожидалось. Если завышен — получают меньше.

---

### L3: Every position can be in at most one eternal farming incentive simultaneously

**Формулировка:** Для данного `(tokenId, incentiveId)` пара не может существовать дважды. `farm.liquidity == 0` означает "не в farming".

**Где enforced:** `require(farmsForToken[incentiveId].liquidity == 0, 'token already farmed')` в AlgebraEternalFarming.enterFarming.

**Где только assumed:** FarmingCenter проверяет `numberOfFarms` счётчик, но этот счётчик считает ВСЕ farmings (eternal + limit), не отдельно eternal. Отдельная проверка на дублирование eternal — в AlgebraEternalFarming.

**Что странного при нарушении:** Если позиция могла бы дважды войти в один incentive, её ликвидность учлась бы дважды в virtual pool, получая двойную долю наград.

---

### L4: Farm rewards are credited to msg.sender of exitFarming/collectRewards

**Формулировка:** Farming rewards зачисляются на адрес, который вызывает `exitFarming` или `collectRewards` через FarmingCenter (не на оригинального депозитора).

**Где enforced:** `_farming.exitFarming(key, tokenId, msg.sender)` — третий аргумент — получатель. В FarmingCenter, `msg.sender` должен быть `_isApprovedOrOwner(msg.sender, deposit.L2TokenId)`.

**Где только assumed:** Пользователь предполагает, что награды достанутся ему, не задумываясь об одобрениях L2 NFT.

**Что странного при нарушении:** Если пользователь одобрил (`approve`) L2 NFT другому адресу, тот адрес может вызвать exitFarming и получить ВСЕ накопленные награды + может вызвать withdrawToken и забрать original position NFT.

---

### L5: Tick cross notifications propagate simultaneously to real and virtual pool

**Формулировка:** Когда реальный AlgebraPool пересекает тик, virtual pool получает `cross()` уведомление в ТОЙ ЖЕ транзакции, обновляя `currentLiquidity` и `globalTick` синхронно.

**Где enforced:** AlgebraPool.swap вызывает `IAlgebraVirtualPool(activeIncentive).cross(step.nextTick, zeroToOne)` при каждом пересечении инициализированного тика.

**Где только assumed:** Если `activeIncentive` — адрес FarmingCenter, cross() проксируется к ОБОИМ virtual pools. Если один из них нет — вызов address(0) успешен, но ничего не делает.

**Что странного при нарушении:** Если cross() на virtual pool пропустить (например, из-за gas exhaustion или конкретного сценария), virtual pool считает ликвидность активной/неактивной неверно, начисляя награды не тем позициям.

---

### L6: Pool deactivates incentive when NOT_EXIST is returned

**Формулировка:** Когда AlgebraPool вызывает `activeIncentive.increaseCumulative()` и получает `NOT_EXIST`, он устанавливает `activeIncentive = address(0)`, прекращая все farming callbacks.

**Где enforced:** AlgebraPool.swap строки 754-755.

**Где только assumed:** Если `activeIncentive = FarmingCenter` (proxy mode), FarmingCenter НИКОГДА не вернёт NOT_EXIST, если хотя бы один из virtual pools возвращает ACTIVE. Expired Limit farming (чей `_increaseCumulative` возвращает NOT_EXIST) игнорируется.

**Что странного при нарушении:** При истечении LimitFarming в proxy-mode, пул не деактивирует FarmingCenter. FarmingCenter продолжает получать cross() и increaseCumulative() вызовы, форвардируя к истёкшему LimitVirtualPool. Истёкший LimitVirtualPool продолжает обрабатывать cross() вызовы (нет time-check в cross()), хотя `_increaseCumulative` возвращает NOT_EXIST. Это создаёт бесполезные gas costs при каждом свапе, пока incentiveMaker вручную не вызовет detachIncentive.

---

### L7: Reward distribution only occurs when currentLiquidity > 0

**Формулировка:** Награды EternalFarming накапливаются ТОЛЬКО при наличии активной farming ликвидности в диапазоне. В противном случае время утекает без распределения наград (timeOutside).

**Где enforced:** EternalVirtualPool._increaseCumulative, строки 66-89: `if (_currentLiquidity > 0) { distribute } else { timeOutside += timeDelta }`.

**Где только assumed:** Пользователь ожидает, что "незаработанные" награды никуда не денутся. На самом деле они остаются в `rewardReserve` и будут распределены ПОЗЖЕ при наличии ликвидности (rate * time дренирует reserve).

**Что странного при нарушении:** Если пул кратковременно теряет всю farming ликвидность (все фермеры вышли из диапазона), следующий фермер, вошедший в тот же диапазон, начинает "накапливать" старый reserve с текущей скоростью — ничего не теряется, но ожидания пользователей о своей доле наград могут расходиться с реальностью.

---

### L8: increaseLiquidity requires NFT ownership/approval

**Формулировка:** Только владелец или одобренный оператор NFT позиции может изменять ликвидность.

**Где enforced:** `decreaseLiquidity` и `collect` имеют модификатор `isAuthorizedForToken`. `burn` имеет `isAuthorizedForToken`.

**Где только assumed:** `increaseLiquidity` НЕ имеет `isAuthorizedForToken`!

**Что странного при нарушении:** Любой внешний аккаунт может вызвать `increaseLiquidity` на любую позицию (включая застейкнутые в FarmingCenter). Реальная позиция обновляется, farm.liquidity снэпшот остаётся старым. (Закон L1 нарушается одновременно.)

---

### L9: Only legitimate Algebra pools can trigger virtual pool state changes

**Формулировка:** Только pool, зарегистрированный через `setIncentive(FarmingCenter)`, может изменять состояние virtual pools через cross() и increaseCumulative().

**Где enforced:** `onlyFromPool` модификатор в AlgebraVirtualPoolBase проверяет `msg.sender == farmingCenterAddress || msg.sender == pool`. `pool` устанавливается в конструкторе virtual pool.

**Где только assumed:** FarmingCenter.cross() и FarmingCenter.increaseCumulative() не проверяют, является ли msg.sender легитимным пулом. Они просто смотрят `_virtualPoolAddresses[msg.sender]`.

**Что странного при нарушении:** Любой аккаунт может вызвать FarmingCenter.cross() или FarmingCenter.increaseCumulative(). Если у них нет зарегистрированных virtual pools — no-op через address(0) вызовы. Если у них ЕСТЬ (что невозможно без incentiveMaker) — могли бы манипулировать state.

---

### L10: setRates updates cumulative before changing rates

**Формулировка:** При изменении скорости наград (`setRates`), rewards за прошедший период сначала начисляются по старой ставке, затем применяется новая.

**Где enforced:** `setRates` вызывает `_increaseCumulative(uint32(block.timestamp))` до изменения ставки.

**Где только assumed:** Правильность зависит от того, что `_increaseCumulative` всегда корректно обновляет cumulative до текущего момента.

**Что странного при нарушении:** Если бы ставка менялась без предварительного обновления cumulative, период между последним свапом (обновлением) и `setRates` мог бы начисляться по новой ставке вместо старой.

---

### L11: One virtual pool per pool per farming type

**Формулировка:** Для одного AlgebraPool может быть активен ровно один eternal virtual pool и один limit virtual pool одновременно.

**Где enforced:** `createEternalFarming` проверяет `require(_incentive == address(0), 'Farming already exists')`. `createLimitFarming` проверяет что предыдущий limit farming истёк.

**Где только assumed:** Два разных incentiveMaker вызова (или последовательные) не могут создать конфликтующие farmings для одного пула.

**Что странного при нарушении:** Если два virtual pools записали ликвидность одной позиции в одном тиковом диапазоне, currentLiquidity в пуле реальном и суммарное по virtual pools расходились бы.

---

### L12: StableSwap A parameter changes are time-gated

**Формулировка:** Изменение параметра усиления A происходит линейно во времени через `rampA`. Нельзя изменить A мгновенно.

**Где enforced:** `rampA(futureA, futureATime)` устанавливает `initialATime`, `futureATime`, `initialA`, `futureA`. Функция `_A()` интерполирует текущее значение.

**Где только assumed:** Что пользователи заметят изменение А через события или UI до существенного сдвига цены в пуле.

**Что странного при нарушении:** Если бы Admin мог изменить A мгновенно (или через stopRampA + немедленный новый rampA с крайними значениями), это создало бы мгновенный дисбаланс между рыночной ценой и ценой пула. Быстрый арбитраж мог бы опустошить пул за счёт LP.

---

### L13: LP tokens in V2 pools accurately represent share of reserves

**Формулировка:** `LP_amount / LP_totalSupply * reserve` = доля LP в пуле. Резервы V2 пула точно отражают баланс токенов в контракте.

**Где enforced:** StellaSwapV2Factory/Pair — стандартный Uniswap V2 механизм.

**Где только assumed:** Что оба токена в паре имеют реальную reclaimable стоимость. Если один токен является bridged-asset, его фактическая стоимость зависит от внешнего bridge.

**Что странного при нарушении:** Прецедент Nomad 2022: ETH.mad/USDC.mad стали worth ~0 при exploit бриджа. LP токены для пар с этими активами стали фактически бесполезными несмотря на корректные формулы пула. Рыночная цена LP токена и on-chain резервная стоимость кардинально расходились.

---

### L14: Farming rewards distribution is proportional to time-weighted in-range liquidity

**Формулировка:** Доля фермера в наградах = его `farm.liquidity` / `totalVirtualLiquidity` × время активности.

**Где enforced:** Вся архитектура EternalVirtualPool + getInnerRewardsGrowth + farm.liquidity расчёты.

**Где только assumed:** Что multiplier-токены не создают произвольного перекоса (ограничен MAX_MULTIPLIER = 5x). Что виртуальная ликвидность всегда корректно отражает реальную картину.

**Что странного при нарушении:** Если один фермер имеет multiplier 5x и другие — 1x, первый получает в 5 раз больше. Но реальная его ликвидность (активы в пуле) НЕ увеличилась. Multiplied liquidity в virtual pool "разбавляет" награды других, не внося пропорциональной реальной ценности.

---

### L15: The FarmingCenter is the exclusive controller of virtual pool state

**Формулировка:** Только FarmingCenter может вызывать операции на virtual pools через `onlyFromPool` (который принимает FarmingCenter или pool). Только AlgebraFarming может вызывать `applyLiquidityDeltaToPosition` (через `onlyFarming`).

**Где enforced:** `onlyFromPool`: `msg.sender == farmingCenterAddress || msg.sender == pool`. `onlyFarming`: `msg.sender == farmingAddress`.

**Где только assumed:** Что pool, зарегистрированный в конструкторе, соответствует реальному AlgebraPool. Что farmingCenterAddress в конструкторе VirtualPool — легитимный FarmingCenter.

**Что странного при нарушении:** Если бы кто-то мог создать VirtualPool со своими `farmingCenterAddress` и `farmingAddress`, он мог бы самостоятельно управлять farming state этого virtual pool. Но VirtualPool деплоится внутри `createEternalFarming`/`createLimitFarming` — только incentiveMaker может это сделать.

---

### L16: claimReward drains rewards[owner] — cannot claim more than earned

**Формулировка:** `rewards[owner][token]` аккумулирует начисленные награды. При `claimReward` они вычитаются. Нельзя получить больше, чем накоплено.

**Где enforced:** `_claimReward`: `reward = rewards[from][rewardToken]; ... rewards[from][rewardToken] = reward - amountRequested;`.

**Где только assumed:** Что `rewards[owner]` корректно аккумулируется в exitFarming и collectRewards без возможности overflow или двойного зачисления.

**Что странного при нарушении:** Комментарий `// user must claim before overflow` присутствует в коде рядом с `rewardBalances[key.rewardToken] += reward`. Если `rewards[owner][token] + newReward > type(uint256).max`, произойдёт overflow. Через много эпох без claim — теоретически достижимо, но крайне маловероятно при разумных reward amounts.

---

### L17: StableSwap flashloan is disabled

**Формулировка:** В текущей конфигурации StableSwap пулов на Moonbeam flash loan функционал недоступен.

**Где enforced:** Guard `"Flashloan is not enabled yet"` в функции `flashLoan`.

**Где только assumed:** Что guard не может быть обойдён и что `flashLoan` не вызывается косвенно.

**Что странного при нарушении:** Если guard будет убран (через upgrade/proxy) без соответствующей проверки безопасности, flash loan мог бы использоваться для манипуляции ценой в stable pool за один блок.

---

### L18: V2 swap requires exact path through factory-registered pairs

**Формулировка:** Router V1 маршрутизирует только через пары, созданные StellaSwapV2Factory. Нельзя подставить произвольный контракт как "пару".

**Где enforced:** `getAmountsOut` вычисляет `pairFor(tokenA, tokenB)` детерминированно через Factory адрес и initCodeHash.

**Где только assumed:** Что Factory хранит только легитимные пары. Что `setMigrator` не может установить адрес, влияющий на pair lookup.

**Что странного при нарушении:** Если бы routing принимал произвольные пары, можно было бы создать fake pair с нужными ценами и направить через неё свап, извлекая токены пользователя.

---

### L19: cross-chain swap outcome matches user's UI expectation

**Формулировка:** Пользователь, инициировавший cross-chain swap, получает ожидаемый токен на целевой цепи. Если swap не выполнится — средства возвращаются.

**Где enforced:** UNKNOWN. Механизм возврата средств при failed Axelar GMP execution — NEEDS VERIFICATION.

**Где только assumed:** Squid/Axelar гарантирует либо успешное исполнение, либо возврат на исходный адрес.

**Что странного при нарушении:** Если swap исполнился на исходной цепи (tokens bridged), но Moonbeam execution failed (slippage exceeded), пользователь получает промежуточный bridged token (не исходный, не целевой). Пользователь считает себя обманутым.

---

### L20: exitFarming correctly handles the case where ticks were cleared

**Формулировка:** В `exitFarming` перед расчётом наград вызывается `applyLiquidityDeltaToPosition(0, ...)` — это гарантирует, что тиковые данные актуальны, даже если тики были очищены при других операциях.

**Где enforced:** AlgebraEternalFarming.exitFarming, строки 179-180: `virtualPool.applyLiquidityDeltaToPosition(uint32(block.timestamp), farm.tickLower, farm.tickUpper, 0, tick)`.

**Где только assumed:** Что `applyLiquidityDeltaToPosition(delta=0)` корректно восстанавливает context для последующего `getInnerRewardsGrowth`. Если тики были удалены (при previous exit с flippedBottom/flippedTop), `getInnerFeeGrowth` для этих тиков вернёт значения на основе нулевых `outerRewardGrowth`.

**Что странного при нарушении:** Если тики были удалены и getInnerRewardsGrowth использует 0-значения, расчёт наград будет некорректным — позиция получит reward как если бы её external growth был 0 с начала существования virtual pool. Это может означать либо overpay либо underpay наград.

---

## Сводная таблица законов

| ID | Закон | Где enforced | Где только assumed | Риск нарушения |
|---|---|---|---|---|
| L1 | Farm.liquidity immutable during staking | enterFarming snapshot | NonfungiblePositionManager.increaseLiquidity | LOW-MEDIUM (нет direct profit) |
| L2 | VirtualPool.currentLiquidity = sum of active farms | applyLiquidityDeltaToPosition | Global invariant не проверяется | HIGH при нарушении |
| L3 | One eternal farming per incentive per position | liquidity == 0 check | - | LOW (enforced) |
| L4 | Rewards → caller of exitFarming | msg.sender | User assumptions about L2 NFT approval | MEDIUM |
| L5 | Cross notifications synchronous | AlgebraPool.swap | - | HIGH при нарушении |
| L6 | NOT_EXIST deactivates pool | AlgebraPool.swap | NOT when FarmingCenter returns ACTIVE | MEDIUM (gas waste) |
| L7 | Rewards need active liquidity | currentLiquidity check | - | LOW (expected behavior) |
| L8 | increaseLiquidity needs auth | NOT ENFORCED | YES assumed | MEDIUM (desync) |
| L9 | Only pools trigger VirtualPool | onlyFromPool | FarmingCenter has no caller check | LOW (address(0) calls) |
| L10 | setRates updates cumulative first | setRates calls _increaseCumulative | - | LOW (enforced) |
| L11 | One virtual pool per type per pool | createEternalFarming check | - | LOW (enforced) |
| L12 | StableSwap A changes time-gated | rampA mechanism | Admin notification | MEDIUM |
| L13 | LP tokens represent real value | Uniswap V2 math | Bridge health | CRITICAL (precedent) |
| L14 | Rewards proportional to liquidity*time | full virtual pool system | Multiplier correctness | MEDIUM |
| L15 | FarmingCenter exclusive VirtualPool controller | onlyFromPool, onlyFarming | Virtual pool constructor args | LOW |
| L16 | Cannot claim more than earned | rewards mapping | No overflow protection | LOW (practical) |
| L17 | Flashloan disabled | hardcoded guard | No upgrade protection verified | MEDIUM |
| L18 | V2 routing through Factory pairs only | Factory pair lookup | Factory integrity | LOW |
| L19 | Cross-chain swap refund guarantee | UNKNOWN | Squid/Axelar SLA | HIGH (UNKNOWN) |
| L20 | exitFarming handles cleared ticks | 0-delta call first | tick deletion edge cases | MEDIUM |
