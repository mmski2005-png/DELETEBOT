# StellaSwap — Фаза 5: Злоупотребление обычными функциями

**Дата:** 2026-05-20

---

## F1: `increaseLiquidity` (NonfungiblePositionManager)

**Нормальное назначение:** Добавить ликвидность к существующей NFT позиции.  
**Кто может вызвать:** ЛЮБОЙ адрес. Нет `isAuthorizedForToken` проверки.  
**Когда:** В любое время, пока позиция существует.  
**Что должно быть true:** Позиция с данным tokenId существует.  
**Что создаёт при failure:** Ничего — tokens не переводятся, state не меняется.  
**On-chain state changes:** `_positions[tokenId].liquidity += actualLiquidity`. AlgebraPool position `position.liquidity` увеличивается.  
**Off-chain state changes:** Backend/indexer видит увеличение liquidity на позиции.  
**Events:** `IncreaseLiquidity(tokenId, ...)`.  
**Кто реагирует:** FarmingCenter — **не реагирует**. Farming снэпшот остаётся старым.

**Можно ли совместить с другой функцией:**  
→ `enterFarming` (уже застейкано) + `increaseLiquidity` = farming snapshot слишком мал vs реальная ликвидность.  
→ `increaseLiquidity` перед `enterFarming` = legit boost к farming (корректно учитывается при enterFarming).  
→ Атакующий вызывает `increaseLiquidity` на ЧУЖУЮ стейкнутую позицию → чужой farming не улучшается (snapshot lock), но реальная позиция растёт. LP fees от added liquidity идут FarmingCenter, который перераспределяет через `collect`. Это может быть использовано как griefing (diluting чужие LP fees) или donation.

---

## F2: `collectRewards` (FarmingCenter)

**Нормальное назначение:** Начислить накопленные farming rewards без выхода из farming.  
**Кто может вызвать:** Authorized holder L2 NFT.  
**Когда:** Пока позиция в farming.  
**Что должно быть true:** `deposits[tokenId].L2TokenId` должен существовать.  
**On-chain state changes:** Обновляет `farms[tokenId][incentiveId].innerRewardGrowth0/1`. Добавляет в `rewards[msg.sender]`. Вызывает `increaseCumulative` на virtual pool.  
**Events:** RewardsCollected.  
**Кто реагирует:** The user — для последующего `claimReward`.

**Можно ли совместить с другой функцией:**  
→ `collectRewards` + `exitFarming` в multicall: первый обновляет snapshot, второй вычисляет rewards от snapshot до current. Суммарно = полные rewards. Корректно.  
→ Многократный `collectRewards` в одном блоке через multicall: второй и последующие вызовы дадут 0 (timeDelta = 0). Gas waste.  
→ `collectRewards` перед `setRates(0,0)` admin: пользователь захватывает rewards текущей ставки до того, как admin обнулит rate. Frontend-run против admin action.

---

## F3: `setRates` (EternalVirtualPool через EternalFarming)

**Нормальное назначение:** Регулировать скорость distribution наград.  
**Кто может вызвать:** Только incentiveMaker (через `onlyIncentiveMaker` в AlgebraEternalFarming).  
**Когда:** В любое время после создания farming.  
**On-chain state changes:** `_increaseCumulative` вызывается сначала; затем `rewardRate0/1` обновляются.  
**Events:** RewardsRatesChanged.

**Можно ли совместить:**  
→ `setRates(0, 0)` + `setRates(huge_rate, 0)` в следующем блоке: за один блок между двумя вызовами никаких наград не распределяется. При следующем свапе — весь reserve может истечь в один момент. MEV-aware фермеры, видящие setRates в mempool, могут:  
  1. Выйти из farming до setRates(0,0) → claiming rewards at old rate  
  2. Войти с максимальным multiplier перед setRates(huge_rate) → захватить большую долю

---

## F4: `rampA` (StableSwap)

**Нормальное назначение:** Плавно скорректировать amplification coefficient.  
**Кто может вызвать:** Admin (onlyOwner).  
**Когда:** В любое время.  
**On-chain state changes:** `initialA`, `futureA`, `initialATime`, `futureATime` записываются.  
**Events:** RampA.

**Можно ли совместить:**  
→ `rampA` + `withdrawAdminFees` в той же транзакции: при запуске rampA в сторону снижения A, и одновременном withdrawAdminFees, admin извлекает fees прямо перед тем, как пул становится менее стабильным.  
→ Frontrun: зная о rampA транзакции в mempool, бот может сначала выйти из пула (removeLiquidity) по старому A, затем дождаться нового A и войти обратно по новой цене.

---

## F5: `detachIncentive` (AlgebraEternalFarming)

**Нормальное назначение:** Отключить incentive от pool (virtual pool больше не получает swaps).  
**Кто может вызвать:** incentiveMaker.  
**On-chain state changes:** `pool.setIncentive(address(0))` или переключение на другой virtual pool.  
**Что создаёт при failure:** Если incentive уже detached — revert "Farming do not exist".

**Можно ли совместить:**  
→ `detachIncentive` + немедленно `attachIncentive` другого: первый очищает virtual pool; второй подключает новый. Между двумя вызовами — мгновенное отсутствие farming. Свапы в этот момент не обновляют никакой virtual pool.  
→ `detachIncentive` при наличии застейканных позиций: позиции остаются в farms[] mapping, rewards накапливаются на старом virtual pool (который больше не получает cross() уведомлений). Rewards "заморожены" на момент последнего свапа до detach. Фермеры могут выйти и получить rewards, но только те, что были накоплены до detach.

---

## F6: `emergencyWithdraw` (DualFarms V2)

**Нормальное назначение:** Выйти из farming без получения rewards при экстренных ситуациях.  
**Кто может вызвать:** Любой participant.  
**On-chain state changes:** Обнуляет `userInfo[pid][msg.sender]`. Переводит LP tokens. Вызывает rewarder.onEmergencyWithdraw.  
**Что создаёт при failure:** При revert rewarder — может заблокировать emergencyWithdraw.

**Можно ли совместить:**  
→ `emergencyWithdraw` + повторный `deposit` в следующем блоке: получаешь свежий state. Работает как "reset harvest interval" если нужно войти заново без ожидания.  
→ Если `emergencyWithdraw` освобождает LP из фармы при hacked bridge token, пользователь получает worthless LP, но сохраняет возможность продать их на рынке пока они что-то стоят.

---

## F7: `withdrawToken` (FarmingCenter)

**Нормальное назначение:** Вернуть original NFT позицию пользователю после выхода из farming.  
**Кто может вызвать:** Authorized holder L2 NFT (при `numberOfFarms == 0`).  
**On-chain state changes:** Удаляет `deposits[tokenId]`, burns L2 NFT, переводит original NFT.

**Можно ли совместить:**  
→ `exitFarming` + `withdrawToken` + `decreaseLiquidity` + `collect` в multicall: одна транзакция извлекает всё — farming rewards, возвращает NFT, снимает ликвидность, собирает LP fees. Это normal, efficient flow.  
→ Оператор (с L2 NFT approval): `exitFarming` (rewards → operator) + `withdrawToken(tokenId, operator_address)` (NFT → operator). Original owner теряет всё.

---

## F8: `addRewards` (EternalFarming) — permissionless

**Нормальное назначение:** Пополнить reward reserve для incentive.  
**Кто может вызвать:** ЛЮБОЙ. Нет `onlyIncentiveMaker`.  
**On-chain state changes:** `incentive.totalReward += received`. `rewardReserve += received`.

**Можно ли совместить:**  
→ Любой может добавить rewards в существующий incentive. Это permissionless — хорошо для ecosystem grants, но: если мгновенно добавить huge amount И rewardRate высокий, следующий свап опустошает весь reserve. MEV-aware фермеры этим воспользуются.  
→ `addRewards` с rewardToken = bonusRewardToken (одинаковые токены): `incentive.totalReward` и `incentive.bonusReward` обновляются отдельно, но контракт принимает оба в одной tx. Это valid.  
→ `addRewards` с fee-on-transfer token: проверка `balanceAfter > balanceBefore` гарантирует actual received amount. Количество наград = реально полученное. Если token имеет high fee (50%), добавление 100 токенов даёт 50 в reserve. Это корректно учтено.

---

## F9: `claimReward` (AlgebraFarming) — direct call

**Нормальное назначение:** Вывести накопленные farming rewards.  
**Кто может вызвать:** ЛЮБОЙ `from` = msg.sender.  
**On-chain state changes:** Уменьшает `rewards[msg.sender][token]`. Переводит tokens.

**Можно ли совместить:**  
→ `claimReward(token, to, 0)` с amountRequested=0: по логике `if (amountRequested == 0 || amountRequested > reward) { amountRequested = reward }` — это CLAIM ALL. Пользователь может вывести все накопленные rewards сразу без явного указания суммы.  
→ `claimRewardFrom(token, from, to, amount)` через FarmingCenter: FarmingCenter может claim от имени from (msg.sender). В клиентском коде FarmingCenter.claimReward вызывает `_claimRewardFromFarming` с `from = msg.sender`. Пользователь не может claim чужие rewards через этот путь.

---

## F10: `sweepTokenWithFee` (SwapRouter V3)

**Нормальное назначение:** Вернуть unused tokens из router после swap.  
**Кто может вызвать:** ЛЮБОЙ. No restriction.  
**On-chain state changes:** Если у router есть balance токена, переводит `balance - amountMinimum` минус `feeBips` в `feeRecipient`, остаток в `recipient`.  
**Events:** нет специфических sweep events.

**Можно ли совместить:**  
→ Если exactInput swap оставляет tokens в router (например, при path routing через несколько пулов), немедленный `sweepTokenWithFee` в следующей транзакции извлечёт их с fee.  
→ Нормально, что router не хранит tokens между транзакциями. Но если транзакция ревертируется после перевода токенов в router (но до продолжения), токены могут теоретически застрять. `sweepTokenWithFee` — механизм recovery.

---

## F11: `connect/disconnectVirtualPool` через `attachIncentive/detachIncentive`

**Нормальное назначение:** Подключить/отключить virtual pool от реального пула.  
**Кто может вызвать:** incentiveMaker.  
**Что создаёт:** Мгновенное изменение `pool.activeIncentive`. Свапы с этого момента идут к новому virtual pool.

**Можно ли совместить:**  
→ `detachIncentive` + `attachIncentive` (swap между двумя incentives): за один блок все pending swaps в этом блоке могут быть при одном virtual pool, а следующие — при другом. Если pending свапы ещё не исполнены (в mempool), их cross() уведомления пойдут новому virtual pool после attach. Но позиции ещё записаны в СТАРОМ virtual pool. Это приводит к расхождению между virtual pool и farms mapping.

---

## F12: `pause/unpause` (AlgebraFactory или StableSwap)

**Нормальное назначение:** Emergency stop для протокола.  
**Кто может вызвать:** Factory owner (для Algebra), admin (для StableSwap).  
**On-chain state changes:** AlgebraFactory.isPaused флаг. Все новые pool.mint/swap fail when paused.

**Можно ли совместить:**  
→ `pause` + замена virtualPool через detach/attach + `unpause`: в период паузы пользователи не могут выйти из positions через router/swaps. Но они МОГУТ вызвать `exitFarming` напрямую! Farming не заблокировано паузой.  
→ `pause` блокирует только новые операции через pool (swap, mint). Существующие farming positions продолжают "накапливать" rewards пока `_increaseCumulative` может вызываться (через `collectRewards` → FarmingCenter → increaseFromCP — это не требует swap через паузенный пул).

---

## F13: `forbidPause` (AlgebraFactory)

**Нормальное назначение:** Необратимо запретить pausability.  
**Кто может вызвать:** Factory owner.  
**On-chain state changes:** Устанавливает флаг, после которого `pause()` всегда reverts.

**Можно ли совместить:**  
→ `forbidPause` при активной атаке: если атака происходит именно тогда, когда pause мог бы помочь — а forbidPause уже вызван — admin лишён emergency lever. Это trade-off между trustlessness и incident response.

---

## Сводка наиболее опасных комбинаций

| Комбинация | Риск |
|---|---|
| `increaseLiquidity` (anyone) → стейкнутая позиция | Phantom LP fees без farming update |
| L2 NFT approve → `exitFarming` + `withdrawToken` | Полная потеря farming position |
| `detachIncentive` + `attachIncentive` в одном блоке | Cross-pool virtual pool state confusion |
| `setRates(huge)` + massive swap в следующем блоке | Reserve дренирование в одном свапе |
| `rampA(low_value)` + `withdrawAdminFees` в одном tx | Admin capture перед pool destabilization |
| `emergencyWithdraw` при broken rewarder | DoS на normal exit path |
