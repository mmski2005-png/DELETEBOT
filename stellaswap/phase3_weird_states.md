# StellaSwap — Фаза 3: Невозможные, но валидные состояния

**Дата:** 2026-05-20  
**Метод:** Для каждого закона из Фазы 2 генерируем гипотезы через вопросы "что если"

---

## H01: Farming position phantom liquidity

**Какой закон ломается:** L1 (snapshot immutable), L2 (virtualPool mirrors actual)  
**Странное валидное состояние:**  
Пользователь A застейкал позицию с ликвидностью L. Затем произвольный аккаунт (или сам A) вызывает `NonfungiblePositionManager.increaseLiquidity(tokenId, ΔL)`. Реальная позиция теперь имеет L + ΔL. Virtual pool отслеживает только L * multiplier. Между реальным пулом и virtual pool существует постоянный "призрак" ΔL, невидимый для farming системы.

**Какие компоненты расходятся:**
- AlgebraPool (tracks L + ΔL for fees)
- NonfungiblePositionManager (stores L + ΔL)
- EternalVirtualPool (tracks L * multiplier)
- FarmingCenter.farms (records L * multiplier)

**Что может быть получено неправильно:** Другие фермеры получают неожиданно высокую долю наград, потому что виртуальный пул "видит" меньше суммарной ликвидности, чем реальный. totalRewardGrowth растёт быстрее относительно реальных активов в пуле.  
**Что нужно проверить:** On-chain вызов `increaseLiquidity` на стейкнутый tokenId; сравнение `positions(tokenId).liquidity` с `farms[tokenId][incentiveId].liquidity`.  
**Первичная оценка: Low** (нет прямого profit для атакующего; бенефициары — другие фермеры случайно)

---

## H02: Expired LimitFarming zombie ticks

**Какой закон ломается:** L6 (NOT_EXIST deactivates pool)  
**Странное валидное состояние:**  
Pool имеет активный EternalFarming И истёкший LimitFarming. FarmingCenter в proxy mode. LimitVirtualPool.increaseCumulative возвращает NOT_EXIST. FarmingCenter.increaseCumulative возвращает ACTIVE. AlgebraPool НИКОГДА не деактивирует FarmingCenter. При каждом свапе, пересекающем инициализированные тики LimitVirtualPool, FarmingCenter.cross() вызывает оба pool.cross() — включая истёкший LimitVirtualPool. Тики в LimitVirtualPool обновляются (globalTick меняется), хотя farming давно закончился.

**Какие компоненты расходятся:**
- AlgebraPool (считает farming активным)
- LimitVirtualPool (давно истёк, но продолжает получать cross())
- EternalVirtualPool (корректно активен)
- Фермеры (предполагают, что LimitFarming давно отключён)

**Что может быть получено неправильно:** Gas waste при каждом свапе. Потенциально: если позиции всё ещё находятся в LimitFarming (не вышли после endTime), их `getInnerSecondsPerLiquidity` продолжает обновляться via cross() вне периода farming. Это может влиять на расчёт reward при exitFarming через LimitVirtualPool.  
**Что нужно проверить:** Существование LimitFarming incentives у StellaSwap с прошедшим endTime; on-chain `virtualPoolAddresses(poolAddress)`; количество позиций с `inLimitFarming == true` для истёкших incentives.  
**Первичная оценка: Medium** (real gas cost; потенциальный reward calculation issue при stuck positions)

---

## H03: L2 NFT approval drains farming rewards

**Какой закон ломается:** L4 (rewards go to caller)  
**Странное валидное состояние:**  
Пользователь A депонирует позицию, L2 NFT выдан ему. A случайно или доверчиво одобряет (`approve`) свой L2 NFT для контракта B (например, marketplace, ненадёжный агрегатор). B вызывает `FarmingCenter.exitFarming` — награды начисляются на адрес B, не на A. B вызывает `FarmingCenter.withdrawToken(tokenId, B)` — original position NFT уходит B. A теряет и награды, и позицию.

**Какие компоненты расходятся:**
- FarmingCenter (считает операции легитимными — authorization прошла)
- Пользователь (ожидал, что approve даёт только права на L2 NFT transfer, не на farming claims)
- Economic model (approve != "получить мои farming rewards")

**Что может быть получено неправильно:** Полная потеря farming rewards и underlying position для владельца.  
**Что нужно проверить:** Есть ли маркетплейсы или агрегаторы, которые запрашивают approve на L2 NFT FarmingCenter на StellaSwap? Документация пользователю о рисках approve?  
**Первичная оценка: Medium** (requires user error, но semantic gap реален)

---

## H04: FarmingCenter owner update race condition

**Какой закон ломается:** L4 (rewards accountability)  
**Странное валидное состояние:**  
`deposit.owner = msg.sender` в exitFarming обновляет поле owner. Если один пользователь A (real owner L2 NFT) и оператор B (approved) вызывают exitFarming конкурентно (или последовательно для разных farmings), owner поле перезаписывается. После первого exitFarming: `deposit.owner = B`. После второго (если бы был возможен): `deposit.owner = B или A`. Это не создаёт double-spend, но semantic of "who is the owner" становится нечётким.

**Какие компоненты расходятся:**
- `deposit.owner` (field overwritten by exitFarming)
- Фактический owner L2 NFT (unchanged by exitFarming)
- `rewards[msg.sender]` (always correct — rewards go to actual caller)

**Что может быть получено неправильно:** Информационная непоследовательность; deposit.owner не соответствует L2 NFT owner.  
**Что нужно проверить:** Используется ли deposit.owner где-либо ещё кроме storage? Является ли оно read-only для внешних интеграций?  
**Первичная оценка: Low** (поле owner — информационное; authorization через ERC721 не затрагивается)

---

## H05: Tick clearing breaks subsequent reward calculation

**Какой закон ломается:** L20 (exitFarming handles cleared ticks), L2 (virtual pool mirrors)  
**Странное валидное состояние:**  
Три фермера A, B, C входят с диапазоном [T1, T2]. B и C выходят. При выходе последнего C, тики T1 и T2 flipped → удалены (delete ticks[T1], delete ticks[T2]). Теперь для фермера A остались только его farm данные, но T1 и T2 в virtual pool удалены. При следующем свапе через диапазон, cross() вызывается для T1 или T2, но эти тики неинициализированы — cross() выполняет ничего (check `if (ticks[nextTick].initialized)`). globalTick обновляется, но liquidityDelta не применяется. currentLiquidity virtual pool может не уменьшиться/увеличиться при выходе A из диапазона через свап.

**Какие компоненты расходятся:**
- EternalVirtualPool.currentLiquidity (может быть неверным после tick deletion)
- EternalVirtualPool.totalRewardGrowth (рассчитывается на основе неверного currentLiquidity)
- Farm A (считает, что его ликвидность учтена)

**Что может быть получено неправильно:** Фермер A может получить больше/меньше наград, если virtualPool.currentLiquidity не отражает его реальную активность.  
**Что нужно проверить:** Код applyLiquidityDeltaToPosition при exitFarming — восстанавливает ли он тики перед получением inner growth? Строки 179-197 AlgebraEternalFarming.exitFarming.  
**Первичная оценка: Medium** (нужна конкретная воспроизводимая последовательность)

---

## H06: Virtual pool globalTick divergence from real pool

**Какой закон ломается:** L5 (cross notifications synchronous)  
**Странное валидное состояние:**  
`applyLiquidityDeltaToPosition` устанавливает `globalTick = currentTick` на основе аргумента, переданного из AlgebraFarming. Это значение берётся как `(, int24 tick, , , , , ) = pool.globalState()` В _enterFarming и _exitFarming. Между вызовом `pool.globalState()` и вызовом `applyLiquidityDeltaToPosition` с этим тиком — никаких свапов не происходит (одна транзакция). Но: если при exitFarming вызывается `applyLiquidityDeltaToPosition(0, ...)` первым (строка 179), а ЗАТЕМ ещё раз с `-liquidity` (строка 186), между ними `globalTick` мог бы измениться — НО нет, обе операции в одной транзакции.

**Реальный риск:** `applyLiquidityDeltaToPosition` устанавливает `globalTick = currentTick` принудительно. Если cross() уже обновил `globalTick` в virtual pool иначе (из swap в том же блоке), `applyLiquidityDeltaToPosition` перезапишет его. Между двумя вызовами внутри exitFarming — небольшое окно.

**Что нужно проверить:** Sequence: swap → cross() обновляет globalTick → exitFarming в той же транзакции? Такое невозможно (нет re-entrancy без lock), но в разных транзакциях одного блока — возможно.  
**Первичная оценка: Low** (globalTick overwrite between operations seems benign)

---

## H07: StableSwap rampA sandwich

**Какой закон ломается:** L12 (A changes are time-gated)  
**Странное валидное состояние:**  
Admin вызывает `rampA(newA, futureTime)` где `newA << currentA` (резкое снижение усиления). В течение периода рампа, эффективное A меняется с каждым блоком. Арбитражёр, знающий о rampA транзакции (через mempool или поведение admin), может:
1. До rampA: занять позицию в токенах, которые будут переоценены при снижении A
2. После rampA начала: свапить в направлении нового имбаланса
3. Profit от изменения invariant path LP'ов

Это не требует flash loan — просто timing знания о governance action.

**Какие компоненты расходятся:**
- Admin (знает о будущем изменении A)
- LP держатели (не уведомлены о конкретном timing)
- On-chain price (немедленно меняется после каждого блока rampA)

**Что может быть получено неправильно:** LP несут убытки от предсказуемого арбитража в пользу MEV ботов и информированного admin.  
**Что нужно проверить:** Есть ли timelock на rampA? История вызовов rampA на StableSwap контрактах StellaSwap.  
**Первичная оценка: Medium** (предсказуемый вектор; реализуем при знании timing)

---

## H08: Bridged token desynchronization in V2 pool

**Какой закон ломается:** L13 (LP tokens represent real value)  
**Странное валидное состояние:**  
V2 пул содержит USDC.multi (Multichain bridge) и xcUSDT (XCM). Multichain взломан (прецедент 2023). USDC.multi торгуется по $0.10. Пул имеет 1000 USDC.multi и 1000 xcUSDT. Invariant k = 1,000,000. Arbitrageur дренирует xcUSDT до ~990.049 за ~100 USDC.multi (через свапы). LP получают LP fees в USDC.multi (бесполезных) и теряют реальные xcUSDT. V2 пул продолжает работать корректно по математике, но экономически LP разорены.

**Какие компоненты расходятся:**
- AlgebraPool/V2 Pair (видит токены как эквивалентные ERC20s)
- Реальный мир (USDC.multi почти ничего не стоит)
- LP токены StellaSwap (claim на mix реального и нулевого актива)
- Farming rewards (продолжают начисляться STELLA за LP в разрушенном пуле)

**Что нужно проверить:** Статус Multichain (.multi) токенов в V2 пулах StellaSwap; TVL в этих пулах сейчас.  
**Первичная оценка: High** (prецедент Nomad 2022; Multichain hack 2023)

---

## H09: Farming rewards for worthless LP

**Какой закон ломается:** L13, L14  
**Странное валидное состояние:**  
DualFarms V2 начисляет STELLA rewards для LP, которые представляют destroyed value (например, пара с hacked bridge token). Фермер может иметь LP из почти-нулевых токенов, стейкать их и получать реальные STELLA rewards. Потенциально: атакующий может дешево купить "burned" LP (например, USDC.mad/anything после Nomad), застейкать, получать STELLA rewards.

**Какие компоненты расходятся:**
- DualFarms V2 (LP token → staking → STELLA rewards)
- Реальная стоимость LP (→ 0)
- Стоимость STELLA rewards (→ реальные)

**Что нужно проверить:** Остались ли у StellaSwap активные DualFarms фармы с Nomad/Multichain LP токенами? Проверить pid список DualFarms и связанные lpToken адреса.  
**Первичная оценка: Medium** (если фармы с degen LP ещё активны; прямой profit path)

---

## H10: collectRewards staleness — reward snapshot divergence

**Какой закон ломается:** L1, L10  
**Странное валидное состояние:**  
Нет свапов в пуле на протяжении N часов. `EternalVirtualPool.prevTimestamp` устарел. Фермер вызывает `FarmingCenter.collectRewards` — FarmingCenter вызывает `increaseCumulative(block.timestamp)`. Rewards за N часов аккумулируются. Обновляется farm snapshot. 

НО: если другой фермер вызывает `collectRewards` В ТОЙ ЖЕ транзакции (multicall), second `increaseCumulative` вернёт `timeDelta = 0` (prevTimestamp уже = block.timestamp). Второй фермер получит rewards только до block.timestamp (без дополнительного delta). Это корректно — rewards не дублируются. НО: фермер, который вызывает первым в multicall, обновляет cumulative для всех.

**Реально интересное:** Что если пул не получал swap'ов долго, а фермер вызывает exitFarming (не через FarmingCenter.collectRewards, а напрямую через exitFarming)? В exitFarming вызывается `virtualPool.applyLiquidityDeltaToPosition(block.timestamp, ...)` которое внутри вызывает `_increaseCumulative(block.timestamp)`. Rewards за весь период аккумулируются в этот момент. Корректно.

**Что нужно проверить:** Есть ли разница в total claimed rewards при `collectRewards + exitFarming` vs `exitFarming` alone?  
**Первичная оценка: Low** (правильно работает, но полезно верифицировать)

---

## H11: Cross-chain swap with intermediate asset stuck on Moonbeam

**Какой закон ломается:** L19 (cross-chain swap refund)  
**Странное валидное состояние:**  
Пользователь инициирует cross-chain swap: USDC (Polygon) → GLMR (Moonbeam) через Squid+Axelar. Этап 1: USDC переводится через Axelar на Moonbeam как axlUSDC. Этап 2: Squid пытается свапнуть axlUSDC → GLMR на StellaSwap, но slippage слишком большой (price moved) — транзакция ревертится. Пользователь не получает GLMR, но axlUSDC уже на Moonbeam. 

**Human meaning:** "Верните мне USDC"  
**Contract meaning:** axlUSDC на Moonbeam у пользователя — это USDC на Moonbeam (через Axelar bridge). Возврат на Polygon требует отдельного bridge back действия.  
**Divergence:** Пользователь ожидал "атомарный" обмен или автоматический refund. Реально: промежуточный актив застрял.

**Что нужно проверить:** Поведение Squid при failed execution — документация, on-chain контракт на Moonbeam, исторические failed cross-chain transactions.  
**Первичная оценка: High** (реальный UX риск; зависит от Squid/Axelar implementation)

---

## H12: Minimal position width bypass via multiplier

**Какой закон ломается:** L14 (proportional rewards)  
**Странное валидное состояние:**  
Incentive создан с `minimalPositionWidth = 0` (допустимо). Пользователь создаёт позицию с диапазоном [T, T+1] (минимально возможный). Застейкает с максимальным multiplier (5x). В virtual pool регистрируется liquidity = actualL * 5. Реальная ликвидность узкая (активна только для крошечного ценового диапазона), но farming ликвидность 5x. При редких свапах в этом диапазоне, этот фермер получает огромную долю farming rewards несмотря на минимальный вклад в LP fees.

**Что нужно проверить:** Текущее `minimalPositionWidth` в активных incentives StellaSwap; максимальный multiplier токены и их цена.  
**Первичная оценка: Medium** (farming reward gaming; multiplier system работает "как задумано" но экономически странно)

---

## H13: FarmingCenterVault locked tokens without active farming

**Какой закон ломается:** L1 (farming lifecycle)  
**Странное валидное состояние:**  
Пользователь входит в farming с tokensLocked = X. Farming incentive detachIncentive вызывается admin'ом (отключает виртуальный пул от реального пула). Фермер НЕ может вызвать exitFarming (incentive detached, но farming record всё ещё существует). Токены locked в FarmingCenterVault. Пользователь может вызвать exitFarming только если `incentive.totalReward > 0` (требование в collectRewards, но exitFarming не проверяет totalReward напрямую).

**Что нужно проверить:** Может ли exitFarming быть вызван для detached incentive? Код `_detachIncentive` не удаляет `incentives[incentiveId]`. `exitFarming` в EternalFarming проверяет `require(farm.liquidity != 0, 'farm does not exist')` — это пройдёт, если позиция была в farming. Значит exitFarming должен работать даже после detach.  
**Первичная оценка: Low** (нужна верификация)

---

## H14: Two-block reward frontrunning

**Какой закон ломается:** L10 (setRates updates cumulative)  
**Странное валидное состояние:**  
Block N: incentiveMaker вызывает `addRewards(key, large_amount, 0)` — добавляет большой reserve.
Block N: incentiveMaker вызывает `setRates(key, 0, 0)` → cumulative обновлён до блока N, ставка = 0. Никаких наград не распределяется.
Block N+1: incentiveMaker вызывает `setRates(key, huge_rate, 0)` — огромная скорость.
Block N+1: Атакующий (зная планы) в том же блоке N+1 перед setRates: — входит в farming с позицией P. 
Block N+1: setRates с huge_rate. 
Block N+2: Swap → increaseCumulative → весь reserve уходит в один блок. Атакующий получает огромную долю.

**Что нужно проверить:** Есть ли min/max на setRates? Нет. Может ли реально один farm получить весь reserve в один блок? Если `rewardRate = rewardReserve` и timeDelta = 1 секунда, reward = rewardReserve. YES.  
**Первичная оценка: Medium** (требует знания о timing setRates; admin extraction или MEV сценарий)

---

## H15: enterFarming after pool tick moves out of position range

**Какой закон ломается:** L2 (VirtualPool mirrors actual)  
**Странное валидное состояние:**  
Реальный пул имеет currentTick < tickLower (позиция вне диапазона, не активная). Пользователь всё равно может вызвать `enterFarming` — нет проверки активности текущего тика. В `applyLiquidityDeltaToPosition`:
```solidity
if (currentTick >= bottomTick && currentTick < topTick) {
    currentLiquidity = LiquidityMath.addDelta(currentLiquidity, liquidityDelta);
}
```
Если currentTick < tickLower, `currentLiquidity` НЕ увеличивается. Тики T1 и T2 обновляются (liquidityNet изменяется). Farming position зарегистрирована с liquidity L, но virtual pool's `currentLiquidity` не меняется. Фермер НЕ получает rewards, пока цена не войдёт в [tickLower, tickUpper]. Это корректное поведение.

**Интересно:** Фермер может потратить multiplierTokens (locked) за позицию, которая немедленно не активна в virtual pool.  
**Что нужно проверить:** UI предупреждает ли пользователя о неактивных позициях вне текущего ценового диапазона?  
**Первичная оценка: Low** (корректное поведение, но UX issue)

---

## H16: increaseLiquidity boosts LP fees without farming notification — fee skimming

**Какой закон ломается:** L1, L8  
**Странное валидное состояние:**  
Пользователь A застейкал позицию P (liquidity L). A замечает, что virtual pool только отслеживает L (маленький snapshot). A вызывает `increaseLiquidity(tokenId, massiveΔL)` — реальная позиция становится L + massiveΔL. A получает LP fees на весь диапазон L + massiveΔL, но farming rewards только за L. Farming rewards "дешевле" относительно реального вклада ликвидности, так как другие фермеры продолжают делить reward pool с меньшей конкуренцией (virtual pool не видит ΔL других). При следующем exitFarming: rewards за L (маленькие), LP fees за L + massiveΔL (нормальные). После exit: повторный enterFarming уже с L + massiveΔL — получает корректный snpashot.

**Вопрос:** Есть ли стратегия профита? Только если LP fees от добавленной ликвидности превышают потерянные farming rewards. Зависит от соотношения fee rate vs farming APR. При высоком farming APR: нет смысла добавлять liquidity без re-enter. При высоком LP fee rate: стратегия имеет смысл для опытных LPs.

**Первичная оценка: Low** (нет прямого exploit; это обходной путь управления позицией)

---

## H17: DualFarms harvest interval bypass via depositFor

**Какой закон ломается:** L10 (timing enforcement)  
**Странное валидное состояние:**  
DualFarms V2 имеет `harvestInterval`. Фермер A не может harvest до `nextHarvestUntil`. НО: если A переводит LP на другой адрес B (вне farming), B делает новый deposit → B получает свежий `nextHarvestUntil = 0`. B может немедленно harvest в том же блоке.

**Реальная проверка:** В DualFarms V2 harvest происходит при `deposit()` и `withdraw()`. Если B делает `deposit(0)` (0 LP), пройдёт ли это? Нужно проверить — возможно, require amount > 0. Если deposit требует amount > 0, то B должен иметь реальные LP. Если LP можно передать/занять — возможно создание "fresh harvest window" через новый аккаунт.  
**Что нужно проверить:** Минимальный deposit в DualFarms; permit-based deposits.  
**Первичная оценка: Low** (harvest interval — незначительное ограничение)

---

## H18: StableSwap A change benefits exit before others

**Какой закон ломается:** L12 (A changes are time-gated)  
**Странное валидное состояние:**  
Admin начинает `rampA` от 200 до 1 (extreme reduction). По мере снижения A, пул становится менее стабильным. LP, знающие о rampA в начале, могут:
1. Продолжать держать LP и нести потери от арбитража
2. Выйти немедленно — но все не успеют

Те, кто выходит первым (before significant price impact), получают "справедливую" стоимость. Те, кто выходит последними (после арбитражной дренажки), получают меньше. **"Банковский набег"** на StableSwap пул при объявлении rampA.

**Что нужно проверить:** Есть ли минимальный delay между rampA announcement и первым изменением A? История StellaSwap StableSwap pool rampA вызовов.  
**Первичная оценка: Medium** (coordination game; informed actors vs. uninformed LPs)

---

## H19: V2 swapFee change benefits pending transactions

**Какой закон ломается:** L13 (fairness of V2 pool)  
**Странное валидное состояние:**  
StellaSwapV2Factory имеет `setSwapFee(pair, fee)`. Admin повышает fee. Пользователи с уже подписанными транзакциями получают худший rate (более высокий fee, чем ожидали). Наоборот: если admin снижает fee и пользователь быстро делает swap — платит меньше, чем показывал UI.

**Что нужно проверить:** feeToSetter адрес; timelock на setSwapFee?  
**Первичная оценка: Low** (стандартный admin control; но без timelock — frontrunnable)

---

## H20: EternalFarming with 0 reward reserve but positive rewardRate

**Какой закон ломается:** L7 (reward distribution needs active liquidity and reserve)  
**Странное валидное состояние:**  
`rewardReserve0 = 0`, `rewardRate0 = 1000`. При `_increaseCumulative`, `reward0 = 1000 * timeDelta`. Check: `if (reward0 > _rewardReserve0) reward0 = _rewardReserve0 = 0`. Итог: `totalRewardGrowth0` не меняется. Farming "активен" (структура incentive существует, totalReward > 0), но наград не выдаёт. UI может показывать положительный APR (на основе rewardRate), но реальные claims = 0.

**Что нужно проверить:** EternalFarming UI на StellaSwap — проверяет ли он rewardReserve vs rewardRate? Историческое состояние incentives с исчерпанным reserve.  
**Первичная оценка: Low** (техническое поведение; APR UI может быть misleading)

---

## H21: Proxy mode (both farmings active) — cross() to expired LimitVirtualPool writes globalTick

**Какой закон ломается:** L5, L6  
**Странное валидное состояние:**  
EternalFarming активен. LimitFarming истёк (endTimestamp прошёл). Pool в proxy mode (activeIncentive = FarmingCenter). При свапе, FarmingCenter.cross(nextTick, zeroToOne) вызывается. Внутри — cross() на обоих virtual pools. LimitVirtualPool.cross() обновляет `globalTick = zeroToOne ? nextTick - 1 : nextTick` в истёкшем пуле. Если в истёкшем LimitVirtualPool остались незавершённые positions (`numberOfFarms > 0` для limit farming), их `getInnerSecondsPerLiquidity` теперь зависит от actualLick в истёкшем virtual pool, который продолжает обновляться после endTime.

**Что может быть получено неправильно:** Фермеры в истёкшем LimitFarming, которые ещё не вышли, могут иметь некорректный расчёт rewards при exitFarming.  
**Что нужно проверить:** AlgebraLimitFarming.exitFarming — как он считает rewards (через RewardMath, не через innerRewardGrowth). LimitFarming использует `globalSecondsPerLiquidityCumulative`, который строго ограничен `desiredEndTimestamp`.  
**Первичная оценка: Medium** (нужно подтвердить через code трассировку LimitFarming.exitFarming)

---

## H22: sweepTokenWithFee extracts router dust

**Какой закон ломается:** L18 (router neutrality)  
**Странное валидное состояние:**  
SwapRouter V3 имеет функцию `sweepTokenWithFee(token, amountMinimum, recipient, feeBips, feeRecipient)`. Если свап оставляет dust токены в router, любой может вызвать `sweepTokenWithFee` и извлечь их, оставив `feeBips %` себе как feeRecipient. В случае MEV-атаки: атакующий отслеживает транзакции, оставляющие tokens в router, и sweep в том же блоке.

**Что нужно проверить:** Код Router V3 на Moonscan — есть ли protection на sweepTokenWithFee? Типичные scenarios когда tokens остаются в router.  
**Первичная оценка: Low** (стандартная поведение; tokens в router считаются "owned by no one")

---

## H23: Eternal farming reward overflow (theoretical)

**Какой закон ломается:** L16 (claim == earned)  
**Странное валидное состояние:**  
Комментарий в коде: `// user must claim before overflow`. Если `rewards[owner][token] + earned_rewards > type(uint256).max` — overflow. При стандартных reward amounts это практически недостижимо. НО: если `rewardToken = 1 wei token` с 0 decimals и totalReward = uint256.max, теоретически возможен overflow. Однако `totalReward` ограничен балансом контракта реального токена.

**Первичная оценка: Low** (theoretical; практически невозможно при разумных токенах)

---

## H24: Fake pool address calls FarmingCenter.collectRewards via farming

**Какой закон ломается:** L9, L15  
**Странное валидное состояние:**  
FarmingCenter.collectRewards(key, tokenId) вызывается пользователем, авторизованным для L2 NFT. Внутри: `_virtualPool = _virtualPoolAddresses[address(key.pool)].eternalVirtualPool`. Если `key.pool` указывает на произвольный адрес — `_virtualPoolAddresses` для него будет нулевым, IAlgebraVirtualPool(address(0)) не вызывается (check `if (_virtualPool != address(0))`). Затем eternalFarming.collectRewards вызывается — там проверяется `incentives[incentiveId].totalReward > 0`. Если key сконструирован так, что incentiveId совпадает с реальным incentive, но key.pool — фейковый, pool.globalState() в exitFarming вернёт данные... от фейкового пула.

**Что нужно проверить:** В `exitFarming`: `(, int24 tick, , , , , ) = key.pool.globalState()` — если key.pool фейковый, это возвращает произвольное значение. Это может дать произвольный `tick`, который переопределит globalTick virtual pool. НО: incentiveId = keccak256(abi.encode(key)) — при фейковом pool адресе, incentiveId не совпадёт с реальным incentive, и проверка `incentive.totalReward > 0` провалится.  
**Первичная оценка: Low** (закрыто инвариантом на incentiveId)

---

## H25: FarmingCenter receives arbitrary ERC721 tokens

**Какой закон ломается:** L15 (FarmingCenter exclusive controller)  
**Странное валидное состояние:**  
`onERC721Received` в FarmingCenter проверяет `require(msg.sender == address(nonfungiblePositionManager), 'not an Algebra nft')`. Если отправить NFT от другого контракта — транзакция ревертируется. НО: что если кто-то `safeTransferFrom` NFT с поддельным адресом, где msg.sender — специально созданный контракт, идентичный nonfungiblePositionManager по байткоду? Невозможно без контроля над addresses.

**Первичная оценка: Low** (закрыто require проверкой)

---

## H26: DualFarms rewarder callback gas bomb

**Какой закон ломается:** Внешние интеграции (rewarder callbacks)  
**Странное валидное состояние:**  
DualFarms V2 вызывает внешний rewarder контракт при deposit/withdraw. Если rewarder — вредоносный или broken контракт, его callback может использовать весь gas или ревертить. Это блокирует deposit/withdraw для данного пула. Пользователи, застейкавшие в этом пуле, не могут выйти (кроме emergencyWithdraw).

**Что нужно проверить:** rewarder адреса для активных DualFarms пулов на StellaSwap; код rewarder'ов.  
**Первичная оценка: Medium** (DoS на конкретный pool; emergencyWithdraw — mitigation)

---

## H27: Block timestamp truncation to uint32

**Какой закон ломается:** L10 (timing mechanics)  
**Странное валидное состояние:**  
AlgebraPool и EternalVirtualPool используют `uint32(block.timestamp)`. В 2106 году (Unix timestamp > 4294967295), block.timestamp перестаёт умещаться в uint32. Значение оборачивается. `prevTimestamp = uint32(block.timestamp)` будет маленьким числом. Следующий вызов `_increaseCumulative` даст `timeDelta = newTimestamp - wrappedOldTimestamp` — огромное число. Весь rewardReserve уходит мгновенно.

**Первичная оценка: Low** (2106 год — не в нашем горизонте)

---

## H28: Proxy mode FarmingCenter.cross() calling dead virtualPool address

**Какой закон ломается:** L6, L5  
**Странное валидное состояние:**  
FarmingCenter в proxy mode. EternalVirtualPool удалён из mapping (`eternalVirtualPool = address(0)`), но `_virtualPoolAddresses[pool].limitVirtualPool != address(0)`. При вызове FarmingCenter.cross():
```solidity
IAlgebraVirtualPool(address(0)).cross(nextTick, zeroToOne); // call to address(0) — no-op
IAlgebraVirtualPool(limitVirtualPool).cross(nextTick, zeroToOne); // OK
```
No-op на address(0) не ревертирует. Это может произойти в момент перехода между farming программами.

**Первичная оценка: Low** (transient state; OK by design)

---

## H29: Simultaneous eternal + limit farming — cross-contamination of liquidity

**Какой закон ломается:** L2, L11  
**Странное валидное состояние:**  
Позиция P в ОБОИХ — eternal и limit farming. При enterFarming в eternal: liquidity L_eternal = actualL * mult_E регистрируется в EternalVirtualPool. При enterFarming в limit: liquidity L_limit = actualL * mult_L регистрируется в LimitVirtualPool. При каждом тик-кроссинге оба виртуальных пула получают cross(). Оба независимо обновляют свои currentLiquidity. Два отдельных reward streams для одной реальной позиции. 

**Интересное:** Если mult_E ≠ mult_L (разные tokensLocked вложены), то доля rewards в каждом пуле разная. Пользователь может стратегически выбрать разные multipliers для разных программ.

**Что нужно проверить:** Позволяет ли FarmingCenter одновременно входить в оба с разными tokensLocked? В FarmingCenter.enterFarming, tokensLocked передаётся независимо для каждого вызова.  
**Первичная оценка: Low** (by design; interesting but not exploitable)

---

## H30: adminFee withdrawal from StableSwap after pool imbalance

**Какой закон ломается:** L13, L12  
**Странное валидное состояние:**  
StableSwap admin накапливает `adminFeeAccumulated` (часть trading fees). После bridged token depeg (один из 4 токенов в пуле), пул сильно разбалансирован. Admin вызывает `withdrawAdminFees()` — получает токены pro-rata текущим balances. Если один токен обесценился, admin получает в основном ценные токены (те, которых LP пытаются получить при выходе). 

**Более точно:** `withdrawAdminFees` выводит накопленные admin fees, которые являются ДОЛЯМИ каждого токена. Если пул разбалансирован к этому моменту, admin fees также разбалансированы. Admin мог бы front-run собственный `withdrawAdminFees` в момент, когда распределение наиболее выгодно.

**Первичная оценка: Medium** (admin privilege; potential extraction before user awareness)

---

## Сводка гипотез по приоритету

| ID | Название | Закон | Оценка |
|---|---|---|---|
| H02 | Expired LimitFarming zombie ticks | L6 | Medium |
| H03 | L2 NFT approval drains rewards | L4 | Medium |
| H05 | Tick clearing breaks reward calc | L20, L2 | Medium |
| H07 | StableSwap rampA sandwich | L12 | Medium |
| H08 | Bridged token desync V2 pool | L13 | High |
| H09 | Farming rewards for worthless LP | L13, L14 | Medium |
| H11 | Cross-chain swap stuck intermediate | L19 | High |
| H12 | Minimal position width bypass | L14 | Medium |
| H14 | Two-block reward frontrunning | L10 | Medium |
| H18 | StableSwap rampA bank run | L12 | Medium |
| H21 | Proxy mode cross to expired LP | L5, L6 | Medium |
| H26 | DualFarms rewarder gas bomb | external | Medium |
| H30 | adminFee withdrawal after depeg | L13 | Medium |
