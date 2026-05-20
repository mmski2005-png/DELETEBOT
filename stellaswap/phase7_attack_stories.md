# StellaSwap — Фаза 7: Attack Stories

**Дата:** 2026-05-20

---

## Story 1: The Delegated Drain

**Name:** L2 NFT Full Position Takeover Through Approval

**Broken Assumption:**  
Дизайнеры предполагали, что `approve(L2 NFT, operator)` — стандартная ERC721 операция с семантикой "разрешить transfer". В реальности в FarmingCenter approval = "передать оператору право на все farming rewards И underlying position NFT".

**Weird State:**  
Пользователь держит позицию в EternalFarming, накопил X STELLA rewards. Его L2 NFT одобрен для оператора (например, через farming aggregator, который попросил approve). Оператор вызывает exitFarming (rewards → operator) + withdrawToken (NFT → operator). Пользователь теряет всё, хотя contraction пользователя с агрегатором предполагал только "помощь в управлении".

**Why Normal Reasoning Misses It:**  
Чтение документации/UI создаёт впечатление, что L2 NFT — это просто "farming receipt". Стандартное ERC721 approve обычно используется для простых transfer операций. Никто не думает, что approve даёт право claim чужие rewards и withdraw underlying asset.

**Minimum Required Powers:**  
- Способность убедить пользователя сделать approve L2 NFT (aggregator, marketplace, protocol)
- ИЛИ: доступ к address с approvalForAll на FarmingCenter ERC721

**Execution Shape:**  
1. Создать агрегатор/helper контракт, который просит approve L2 NFT
2. Дождаться накопления значительных rewards у users
3. Вызвать `FarmingCenter.exitFarming(key, tokenId, false)` → rewards → attacker
4. Вызвать `FarmingCenter.withdrawToken(tokenId, attacker)` → NFT → attacker
5. Вызвать `eternalFarming.claimReward(rewardToken, attacker, 0)` → STELLA → attacker

**Components That Disagree:**  
- FarmingCenter: операция авторизована (caller is approved for L2 NFT)
- Original depositor: ожидал что approve = limited delegation, не total control
- Rewards mapping: правильно зачисляет на msg.sender (attacker)

**Expected Divergence:** Ownership (position NFT → attacker), Credit (farming rewards → attacker), Settlement status (owner considers farming active; attacker already exited)

**Potential Impact:** Полная потеря farming position и rewards для users, одобривших L2 NFT произвольному контракту.

**Evidence Needed:**  
- Код FarmingCenter.exitFarming: подтверждён — `msg.sender` получает rewards  
- Код FarmingCenter.withdrawToken: подтверждён — `to` parameter контролируется вызывающим  
- Список протоколов/aggregators на Moonbeam, которые используют FarmingCenter и просят L2 NFT approve

**Confidence: High** (механизм подтверждён кодом)  
**Reachability: Medium** (требует user error или ненадёжный aggregator)  
**Impact: High** (полная потеря position + rewards)  
**Next Test:** Проверить все aggregator/UI контракты на Moonbeam, которые интегрировались с StellaSwap FarmingCenter. Прочитать их isApproved logic.  
**Expected Result If True:** Обнаружим aggregator с approve request → attack path реален.  
**Expected Result If False:** Никто не просит approve L2 NFT → теоретическая проблема.

---

## Story 2: The Ghost Campaign

**Name:** Expired Limit Farming Continues Consuming Cross Notifications

**Broken Assumption:**  
Дизайнеры предполагали, что при истечении LimitFarming pool автоматически прекращает вызывать cross() на истёкший virtual pool. В реальности при proxy mode (оба farming активны), FarmingCenter всегда возвращает ACTIVE если EternalFarming работает, и pool никогда не деактивирует FarmingCenter.

**Weird State:**  
LimitFarming истёк (endTimestamp прошёл). EternalFarming всё ещё активен. Pool в proxy mode. При каждом свапе:
1. Pool вызывает `FarmingCenter.increaseCumulative(block.timestamp)` → ACTIVE (eternal active)
2. Pool вызывает `FarmingCenter.cross(tick, direction)` при tick crossing
3. FarmingCenter форвардит cross() к ОБОИМ virtual pools — включая expired LimitVirtualPool
4. Expired LimitVirtualPool обновляет globalTick (post-expiry)
5. Если в expired LimitFarming остались незавершённые позиции (didn't exit) — их tick calculations основаны на post-expiry globalTick

**Why Normal Reasoning Misses It:**  
Кажется, что после endTimestamp LimitFarming "просто останавливается". Фактически: proxy mode маскирует NOT_EXIST от pool. Expired virtual pool продолжает получать state updates.

**Minimum Required Powers:**  
- Обычный пользователь — достаточно не вызывать exitFarming для LimitFarming до истечения
- ИЛИ: наблюдение ситуации (validator, keeper) и последующий вход в выгодное состояние

**Execution Shape:**  
1. StellaSwap создаёт pool с EternalFarming + LimitFarming (proxy mode)
2. LimitFarming истекает, но не все позиции вышли
3. Pool продолжает вызывать cross() на expired LimitVirtualPool
4. Expired LimitVirtualPool.globalTick обновляется post-expiry
5. Farmer, выходящий из LimitFarming post-expiry, получает reward расчёт с potentially incorrect inner seconds per liquidity

**Components That Disagree:**  
- AlgebraPool: считает incentive активным (FarmingCenter возвращает ACTIVE)
- LimitVirtualPool: истёк, но продолжает обрабатывать cross()
- `_increaseCumulative`: возвращает NOT_EXIST (не накапливает секунды после endTimestamp)
- Фермер: думает что вышел из completed, finalized incentive

**Expected Divergence:** Settlement status (pool thinks incentive active; limit farming technically expired), Gas accounting (extra gas per swap)

**Potential Impact:**
1. Газ: каждый свап тратит лишние ~20-50k gas на forward к expired virtual pool (умножить на volume)
2. Потенциально: неверный reward для позиций, не вышедших до expiry (зависит от RewardMath)

**Evidence Needed:**  
- `AlgebraLimitFarming.exitFarming` — полный код и reward formula (нужен)
- On-chain: `FarmingCenter.virtualPoolAddresses(pool)` — есть ли limitVirtualPool != address(0) для pools с истёкшим incentive?
- Transaction traces: swap transactions на пулах с proxy mode после expiry

**Confidence: Medium**  
**Reachability: High** (если есть expired LimitFarming с не-withdrawn positions)  
**Impact: Low-Medium** (gas waste; potential reward miscalculation)  
**Next Test:** Запросить `virtualPoolAddresses` для всех Pulsar pools на Moonbeam. Сравнить с активными LimitFarming endTimestamps.

---

## Story 3: The Worthless Farm

**Name:** Bridge-Devalued LP Tokens Generate Real STELLA Rewards

**Broken Assumption:**  
Дизайнеры предполагали, что LP tokens, допущенные к DualFarms, имеют реальную экономическую ценность. Farming reward emission не зависит от рыночной стоимости стейкнутых LP.

**Weird State:**  
Bridge, обеспечивающий один из токенов в V2 паре, взломан или заморожен. Bridged token (A.bridge) торгуется по $0.01. V2 пул GLMR/A.bridge имеет:
- reserveGLMR = 1000
- reserveA.bridge = 100,000 (дренировано arb)
- LP tokens: holders имеют claim на эту смесь
- Рыночная цена LP → ~$0.01 за LP

DualFarms всё ещё принимает GLMR/A.bridge LP tokens для стейкинга. Reward rate STELLA/block не изменился. Атакующий:
1. Покупает GLMR/A.bridge LP на рынке за $0.01 за LP
2. Стейкает в DualFarms
3. Получает STELLA rewards (реальная рыночная стоимость)
4. Продаёт STELLA

**Why Normal Reasoning Misses It:**  
DualFarms не имеет oracle. Он только видит `lpToken.balanceOf(msg.sender)`. Экономическая ценность LP token — вне модели контракта. Farming была настроена когда LP был ценным.

**Minimum Required Powers:**  
- Обычный пользователь
- Доступ к дешёвым LP tokens хакнутого bridge

**Execution Shape:**  
1. Bridge exploit: bridged token (A.bridge) теряет peg
2. V2 пул GLMR/A.bridge: арбитражёры дренируют GLMR, LP holders застряли
3. LP tokens доступны на рынке за ~0  
4. Купить LP дёшево
5. Stake в DualFarms
6. Получать STELLA
7. Продавать STELLA — до тех пор пока StellaSwap team не снимет farming program

**Components That Disagree:**  
- DualFarms: LP token имеет economic value (так было при настройке)
- Рынок: LP token стоит ~0
- STELLA emitter: продолжает mint STELLA для LP holders
- STELLA price: реальная рыночная цена

**Expected Divergence:** Economic value (LP worth ~0; STELLA earned worth real $), Balance (staked LP count unchanged, real value collapses)

**Potential Impact:** STELLA inflation за счёт worthless LP farming. При большом объёме → давление на STELLA цену. Исторический прецедент: Nomad 2022 exploit.

**Evidence Needed:**  
- Список активных DualFarms pools (pid, lpToken) — нужен on-chain query
- Текущие bridged tokens, используемые в V2 парах: Multichain (.multi), Celer (ce*), Nomad (.mad)
- Рыночная стоимость LP tokens с bridged assets

**Confidence: Medium-High**  
**Reachability: High** (если farms с compromised LP ещё активны)  
**Impact: Medium** (зависит от TVL в таких farms)  
**Next Test:** Вызвать `DualFarmsV2.poolLength()` и `DualFarmsV2.poolInfo(pid)` для каждого pid. Проверить lpToken адреса — содержат ли они .multi, .mad, или другие compromised bridge tokens.

---

## Story 4: The Stale Oracle Farming

**Name:** collectRewards Returns Stale Amounts — UI Overestimates Earned Rewards

**Broken Assumption:**  
Дизайнеры знали, что `getRewardInfo` возвращает устаревшие значения (комментарий: "reward amounts can be outdated"). Они предполагали, что пользователи будут использовать static call `collectRewards` в FarmingCenter для актуальных данных. UI должен явно вызывать `increaseCumulative` перед отображением.

**Weird State:**  
Пул не получал swap'ов 48 часов. `EternalVirtualPool.prevTimestamp` = 48 часов назад. `EternalVirtualPool.totalRewardGrowth` не обновлялся. UI читает `getRewardInfo` — видит старое значение. Реальные rewards за 48 часов не отображаются. Пользователь думает, что фармит плохо. Вызывает exitFarming → `_increaseCumulative(block.timestamp)` → все 48 часов distributes rewards → получает больше, чем ожидал.

**Inverse scenario:** UI читает cached high APR (rewardRate * annualization). Пользователь входит. Реальные rewards меньше (rewardReserve уже частично исчерпан). APR = overestimated.

**Components That Disagree:**  
- UI (shows APR from rate)  
- Virtual pool reserve (partially drained)
- User expectations

**Expected Divergence:** Credit (expected vs actual earnings)

**Potential Impact:** Misleading APR → unnecessary gas costs (enter/exit cycling), bad economic decisions.

**Evidence Needed:**  
- StellaSwap frontend APR calculation code
- Real-time comparison: `rewardRate` vs `rewardReserve` for active incentives

**Confidence: Medium**  
**Reachability: High** (every farming user potentially affected)  
**Impact: Low** (no direct fund loss; UX/economic decision quality)  
**Next Test:** Вызвать `EternalVirtualPool.rewardRate0`, `EternalVirtualPool.rewardReserve0`, `EternalVirtualPool.prevTimestamp` для активных incentives. Сравнить с displayed APR в UI.
