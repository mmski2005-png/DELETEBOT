# StellaSwap — Фазы 8, 9, 10: Опровержение, Приоритизация, Финальный вывод

**Дата:** 2026-05-20

---

# ФАЗА 8: Убийство идей

## Story 1: L2 NFT Full Position Takeover — проверка

**Проверяемые защиты:**

| Защита | Статус |
|---|---|
| Access control на exitFarming | WEAK — достаточно `_isApprovedOrOwner` для L2 NFT |
| Separate reward recipient | MISSING — rewards и NFT оба идут к msg.sender/to |
| Warning in UI/docs | UNKNOWN — нужно проверить |
| No marketplace for L2 NFTs | UNKNOWN — если нет — вектор менее достижим |

**Вердикт: SUSPICIOUS**  
Механизм подтверждён кодом. Вопрос только в достижимости (кто реально делает approve L2 NFT). Если существуют aggregators с approve requests — реальный риск.

---

## Story 2: Expired LimitFarming Ghost — проверка

**Проверяемые защиты:**

| Защита | Статус |
|---|---|
| NOT_EXIST отключает pool | BLOCKED by proxy mode — FarmingCenter скрывает NOT_EXIST |
| LimitVirtualPool time check in cross() | MISSING — cross() не проверяет время |
| Reward calculation ignores post-expiry | PARTIALLY BLOCKED — _increaseCumulative не аккумулирует после endTimestamp. Но globalTick продолжает обновляться |
| incentiveMaker вызывает detachIncentive после expiry | ASSUMED — ручной cleanup нужен |

**Точный механизм: нужна дополнительная трассировка AlgebraLimitFarming.exitFarming**

В LimitFarming rewards рассчитываются через `RewardMath.computeRewardAmount` основанный на `secondsPerLiquidityInsideX128`. Это `getInnerSecondsPerLiquidity` от LimitVirtualPool. Если `globalTick` в expired virtual pool изменился через cross() post-expiry, это ВЛИЯЕТ на `getInnerSecondsPerLiquidity` calculation для positions.

```
getInnerSecondsPerLiquidity(bottom, top):
  if globalTick < bottom: return lowerOuter - upperOuter
  elif globalTick < top: return global - lowerOuter - upperOuter
  else: return upperOuter - lowerOuter
```

Если `globalTick` переместился out of range после expiry (из-за post-expiry cross()), пользователь, который был in-range, получит calculation as if globalTick is now below their range. Это изменит их `innerSeconds`.

**Конкретно:** Если during farming period globalTick был in [T1, T2], а post-expiry cross() переместил globalTick ниже T1, `getInnerSecondsPerLiquidity` вернёт другое значение при exitFarming.

**Вердикт: SUSPICIOUS — NEEDS EVIDENCE**  
Механизм правдоподобен. Требует подтверждения через AlgebraLimitFarming.exitFarming код (который мы не читали полностью).

---

## Story 3: Worthless LP Farming — проверка

**Проверяемые защиты:**

| Защита | Статус |
|---|---|
| LP token value check in DualFarms | MISSING — не проверяется |
| Admin can remove pool | PRESENT — setPool/emergencyWithdraw доступны |
| Historical response time | UNKNOWN — как быстро StellaSwap реагировал на Nomad 2022? |
| Multichain hack response | UNKNOWN — были ли удалены .multi farms? |

**Вердикт: NEEDS EVIDENCE**  
Механизм теоретически корректен. Исторически произошло (Nomad 2022). Вопрос: есть ли сейчас активные farms с compromised LP tokens?

---

## Story 4: Stale Rewards Display — проверка

**Проверяемые защиты:**

| Защита | Статус |
|---|---|
| FarmingCenter.collectRewards вызывает increaseCumulative | PRESENT (строка 179) |
| getRewardInfo помечен как potentially outdated | PRESENT (comment in code) |
| Frontend делает static call collectRewards | UNKNOWN |

**Вердикт: LIKELY BLOCKED for fund loss**  
При exitFarming все rewards корректно рассчитываются. Только UI display может быть неточным. Нет прямой потери funds.

---

# ФАЗА 9: Приоритизация

## Матрица оценок

| Story | Impact (1-5) | Reachability (1-5) | Novelty (1-5) | Evidence (1-5) | Testability (1-5) | Cross-component (1-5) | Economic (1-5) | Итог |
|---|---|---|---|---|---|---|---|---|
| S1: L2 NFT Drain | 5 | 3 | 4 | 4 | 5 | 4 | 5 | Strong Lead |
| S2: Ghost Campaign | 2 | 4 | 4 | 3 | 4 | 5 | 2 | Interesting But Weak |
| S3: Worthless LP | 4 | 4 | 2 | 2 | 5 | 3 | 5 | Needs More Data |
| S4: Stale Rewards | 1 | 5 | 2 | 4 | 5 | 3 | 2 | Likely Blocked |
| H07 rampA sandwich | 3 | 3 | 3 | 3 | 4 | 3 | 4 | Interesting But Weak |
| H11 cross-chain stuck | 4 | 4 | 3 | 1 | 3 | 5 | 4 | Needs More Data |
| Frame 5: incentiveMaker EOA | 5 | 2 | 1 | 2 | 5 | 2 | 5 | Needs More Data |

## Финальная классификация

**Critical Lead:** Нет подтверждённых

**Strong Lead:**
- **S1 (L2 NFT Drain):** Механизм подтверждён кодом. Требует верификации наличия aggregators/protocols с approve requests.

**Interesting But Weak:**
- **S2 (Ghost Campaign):** Реальный gas waste. Возможный reward miscalculation. Требует полного кода AlgebraLimitFarming.exitFarming.
- **H07 (rampA sandwich):** Реальный вектор, но требует admin coordination / mempool visibility.

**Needs More Data:**
- **S3 (Worthless LP):** Высокий impact если active. Нужен on-chain query DualFarms pools.
- **H11 (Cross-chain stuck):** Высокий UX risk. Нужен Squid/Axelar refund code.
- **Frame 5 (incentiveMaker EOA):** Catastrophic if EOA. Trivial to verify.

**Likely Blocked:**
- **S4 (Stale Rewards):** exitFarming корректно обновляет. Только UI issue.
- **H03 (L2 NFT approval — no aggregator):** Если нет aggregators с approve requests.
- **H24 (Fake pool key):** Закрыто incentiveId hash invariant.

---

# ФАЗА 10: Финальный вывод

## 1. Mental Model

StellaSwap — гибридный DEX с тремя типами пулов (V2, StableSwap, Algebra CL), поверх которых построена двухуровневая farming система (DualFarms для V2/Stable, EternalFarming/LimitFarming для CL). Concentrated liquidity farming строится на "виртуальной копии" (VirtualPool), которая синхронизируется с реальным пулом через свап-хуки. Farming rewards начисляются per-unit-virtual-liquidity-time и рассчитываются через снэпшот-дельту при выходе.

Система существенно зависит от мостовых токенов (~10 bridge providers), каждый из которых — отдельный failure domain. Прецедент Nomad 2022 показал, что bridge failure создаёт каскадный effect на LP и farming.

## 2. Source Reliability

| Источник | Верифицированность |
|---|---|
| AlgebraV1 GitHub (farmings, core) | HIGH — читали исходный код |
| Moonscan deployed contracts | HIGH — verified code matches |
| StellaSwap docs | MEDIUM — частичная, некоторые страницы 404 |
| DualFarms V2 code | MEDIUM — только ABI + struct description |
| Cross-chain swap (Squid/Axelar) | LOW — механизм refund UNKNOWN |
| incentiveMaker address | LOW — не проверяли |
| Active DualFarms pool list | LOW — не запрашивали on-chain |

## 3. Non-Obvious Assumptions

1. **L2 NFT approve не = "delegated full control"** — пользователи думают, что это обычный ERC721 approve
2. **FarmingCenter скрывает NOT_EXIST от pool** — proxy mode продлевает жизнь expired incentives на pool уровне
3. **increaseLiquidity разрешён для любого на любую позицию** — нет ownership check
4. **rewardRate без rewardReserve = 0 actual yield** — UI может не проверять reserve
5. **DualFarms не привязан к реальной стоимости LP** — farming продолжается при compromised bridge
6. **Cross-chain swap не атомарен** — multi-step async с независимыми failure modes

## 4. Weird Hypotheses (топ 10)

1. L2 NFT approved operator извлекает всю farming position и rewards
2. Expired LimitFarming продолжает обновлять globalTick, влияя на post-expiry exitFarming rewards
3. DualFarms с worthless bridged LP tokens продолжает эмитировать реальный STELLA
4. increaseLiquidity на staked позицию создаёт phantom farming desync
5. setRates(huge_rate) + первый свап дренирует весь reward reserve в одну транзакцию
6. rampA + immediate withdrawAdminFees = admin captures value pre-destabilization  
7. Cross-chain swap failure оставляет axlUSDC на Moonbeam вместо refund
8. incentiveMaker — single EOA без multisig → полный farming takeover при компрометации
9. StableSwap flashloan guard removable через upgrade (если контракт upgradeable)
10. APR display = rewardRate without reserve check → users entering exhausted farms

## 5. Semantic Gaps

**Главные расхождения:**

| Действие | Human meaning | Contract meaning | Gap |
|---|---|---|---|
| approve L2 NFT | "delegate management" | "give full control over position + rewards" | Критический |
| enterFarming | "stake at current liquidity" | "snapshot once" | Важный (increaseLiquidity desync) |
| cross-chain swap | "atomic exchange" | "multi-step async" | Высокий UX risk |
| APR display | "current yield" | "rate-only, ignore reserve" | Misleading |
| farming "active" | "earning rewards" | "pool considers incentive active" | Может быть expired |

## 6. Strongest Leads

### Lead 1: L2 NFT Approval → Complete Position Takeover

**Почему важно:** Полная потеря farming position + rewards без каких-либо on-chain защит (кроме пользователя не давая approve).  
**Weird state:** Authorized operator = god over position; user expected limited delegation.  
**Components расходятся:** User model vs FarmingCenter authorization model.  
**Safe test:** Задеплоить test scenario на fork: mint position → enterFarming → approve L2 NFT attacker → attacker calls exitFarming + withdrawToken. Verify reward and position destination.

### Lead 2: Worthless LP Token Farming

**Почему важно:** Активный economic exploit при bridge failure. Исторически реализован (Nomad 2022).  
**Weird state:** LP token = ~0 value; STELLA rewards = real value.  
**Components расходятся:** DualFarms contract vs market reality.  
**Safe test:** On-chain query: `DualFarmsV2.poolLength()`, `poolInfo(pid).lpToken` для каждого pid. Проверить lpToken адреса против списка bridged tokens (Multichain, Celer, Nomad).

### Lead 3: incentiveMaker EOA status

**Почему важно:** Если EOA — одна скомпрометированная подпись = захват всего farming. setRates, detachIncentive, createEternalFarming — всё в руках одного адреса.  
**Weird state:** Centralized single point of failure for critical protocol operations.  
**Safe test:** Проверить `AlgebraEternalFarming.incentiveMaker` адрес через Moonscan. Если code size = 0 → EOA.

## 7. Disproved Ideas

| Идея | Механизм блокировки |
|---|---|
| Double eternal farming per incentive | `require(farmsForToken[incentiveId].liquidity == 0)` |
| Fake pool key in collectRewards | incentiveId = keccak256(key) — fake pool даёт другой hash |
| Arbitrary ERC721 to FarmingCenter | `require(msg.sender == nonfungiblePositionManager)` |
| L2 NFT ID collision с original NFT | Разные ERC721 контракты — разные ID spaces |
| rewards overflow practical | При разумных reward amounts — недостижимо |
| FarmingCenter.cross() by arbitrary caller | `_virtualPoolAddresses[caller]` = (0,0) → call to address(0) = no-op |

## 8. Suggested Experiments

### Эксперимент 1: L2 NFT Approval Drain
- **Цель:** Подтвердить полный draining через approve
- **Setup:** Moonbeam fork, deployed FarmingCenter + EternalFarming
- **Действия:** mint position → enterFarming → approve L2 NFT to attacker → attacker.exitFarming + withdrawToken
- **Ожидаемый результат If True:** attacker получает rewards + original NFT
- **Ожидаемый результат If False:** какая-то проверка блокирует
- **Данные сохранить:** tx trace, balance deltas

### Эксперимент 2: On-chain DualFarms Pool Audit
- **Цель:** Найти farms с compromised LP tokens
- **Setup:** Read-only query
- **Действия:** вызвать `poolLength()`, `poolInfo(i).lpToken` для i=0..N. Проверить lpToken против: 0x818ec0A7Fe18Ff94269904fCED6AE3DaE6d6dC0b (USDC.multi), 0x8f552a71EFE5eeFc207Bf75485b356A0b3f01eC9 (USDC.mad), и другие .multi/.mad tokens
- **Ожидаемый результат If True:** найдём active farms с compromised LP
- **Ожидаемый результат If False:** все compromised LP удалены из farms

### Эксперимент 3: incentiveMaker EOA Check
- **Цель:** Проверить, является ли incentiveMaker multisig
- **Setup:** Read-only Moonscan query
- **Действия:** вызвать `AlgebraEternalFarming.incentiveMaker` (нужно найти getter или read storage slot)
- **Ожидаемый результат If EOA:** critical centralization risk confirmed
- **Ожидаемый результат If Multisig:** risk significantly reduced

### Эксперимент 4: Expired LimitFarming Virtual Pool Check
- **Цель:** Найти pools с expired limit virtual pool в proxy mode
- **Setup:** Read-only query
- **Действия:** вызвать `FarmingCenter.virtualPoolAddresses(poolAddress)` для всех Pulsar pools. Для каждого limitVirtualPool: вызвать `LimitVirtualPool.desiredEndTimestamp()`. Сравнить с block.timestamp.
- **Если True:** pools с expired limit + active eternal → ghost campaign подтверждена
- **Данные:** список пар (poolAddress, limitVP, endTimestamp)

### Эксперимент 5: rewardReserve vs displayed APR
- **Цель:** Найти incentives с истощённым reserve
- **Setup:** Read-only query активных EternalVirtualPools
- **Действия:** для каждого `incentive.virtualPoolAddress`: вызвать `EternalVirtualPool.rewardReserve0`, `rewardRate0`, `rewardReserve1`, `rewardRate1`
- **Сравнить с:** displayed APR в StellaSwap UI
- **Если True:** найдём discrepancy between displayed vs actual

### Эксперимент 6: LimitFarming exitFarming reward trace
- **Цель:** Подтвердить/опровергнуть H21 (post-expiry globalTick влияет на rewards)
- **Setup:** Читать AlgebraLimitFarming.exitFarming код полностью
- **Действия:** трассировать reward calculation формулу от exitFarming → RewardMath → getInnerSecondsPerLiquidity → зависит ли от globalTick
- **Если True:** post-expiry cross() изменяет globalTick → reward miscalculation

## 9. Final Judgment

**Story 1 (L2 NFT Drain):**  
"Strong lead; next test should be: verify existence of protocols that request L2 NFT approve on StellaSwap Moonbeam"

**Story 2 (Ghost Campaign):**  
"Suspicious; needs evidence from: AlgebraLimitFarming.exitFarming full code trace — specifically RewardMath dependency on VirtualPool.globalTick"

**Story 3 (Worthless LP Farming):**  
"Suspicious; needs evidence from: DualFarms active pool list with lpToken addresses — compare against bridged tokens with known issues"

**Story 4 (Stale Rewards):**  
"No confirmed issue; the following invariants appear to block the weird states: FarmingCenter.collectRewards calls increaseCumulative first; exitFarming calls applyLiquidityDeltaToPosition which triggers full update"

**incentiveMaker EOA:**  
"Suspicious; needs evidence from: on-chain read of incentiveMaker address → code size check"

**Cross-chain swap refund:**  
"Suspicious; needs evidence from: Squid/Axelar GMP failure handling documentation and on-chain contract — specifically what happens to bridged tokens when Moonbeam DEX leg fails"

---

**Ключевой принцип из prompt.md:**  
Лучшие находки должны выглядеть странно в начале и очевидно задним числом.

- L2 NFT approval → full position takeover: сначала кажется "просто ERC721 approve". Задним числом очевидно: approve + exitFarming + withdrawToken = total control.
- Worthless LP → real rewards: сначала кажется "контракт работает корректно". Задним числом очевидно: экономическая модель предполагает корреляцию между LP value и farming eligibility, которой нет.
