# StellaSwap — Фаза 4: Семантические разрывы

**Дата:** 2026-05-20

---

## Действие 1: `enterFarming(key, tokenId, tokensLocked)`

### Human meaning
"Я стейкаю мою LP позицию и начинаю зарабатывать farming rewards. Чем больше ликвидности — тем больше наград."

### Contract meaning
Снимает снэпшот `positions(tokenId).liquidity` в момент вызова. Умножает на multiplier из `tokensLocked`. Записывает в `farms[tokenId][incentiveId].liquidity`. Добавляет в virtual pool.

### Backend / indexer meaning
Индексирует событие FarmEntered. Показывает пользователю estimated APR на основе текущего farm.liquidity snapshot и rewardRate.

### Next component meaning (VirtualPool)
`currentLiquidity += farm.liquidity` если current tick в range. Будущие swaps будут делить rewards между этим значением и другими фармерами.

### Economic meaning
Пользователь получает долю rewards пропорционально `farm.liquidity` (snapshot), а не текущей реальной ликвидности. Если ликвидность реального pool меняется (через `increaseLiquidity`), экономика фарминга остаётся привязана к СТАРОМУ снэпшоту.

### Разрывы
| Тип разрыва | Описание |
|---|---|
| Same asset, different liquidity | Human thinks "my liquidity" = current; Contract uses "snapshot at enter" |
| Same success, different finality | User thinks farming is finalized; contract only captured state at T0 |
| Same balance, different owner | LP fees accumulate on REAL position; farming rewards on SNAPSHOT |

**Можно ли увеличить разрыв?** Да: вызов `increaseLiquidity` на стейкнутой позиции увеличивает LP fees без обновления farming snapshot. Разрыв растёт до exitFarming.  
**Кто заметит первым?** Indexer заметит рост `positions(tokenId).liquidity` при сравнении с `farms[tokenId].liquidity`.  
**Кто никогда не заметит?** On-chain virtual pool — у него нет механизма сравнения.  
**Минимальный тест:** Вызвать `increaseLiquidity(tokenId)` для стейкнутого tokenId. Сравнить `nonfungiblePositionManager.positions(tokenId).liquidity` с `eternalFarming.farms[tokenId][incentiveId].liquidity`.

---

## Действие 2: `exitFarming(key, tokenId, isLimit=false)`

### Human meaning
"Прекращаю farming, получаю все накопленные rewards."

### Contract meaning
`delete farms[tokenId][incentiveId]`. `rewards[msg.sender][rewardToken] += computed_reward`. Rewards не выплачиваются немедленно — только записываются в `rewards` mapping. Требуется отдельный `claimReward`.

### Backend / indexer meaning
Событие FarmEnded → обновляет статус позиции. Может кэшировать reward amount.

### Next component meaning (FarmingCenter)
`deposit.numberOfFarms -= 1`. `deposit.owner = msg.sender`. Если numberOfFarms == 0, позиция доступна для `withdrawToken`.

### Economic meaning
Rewards начислены `msg.sender`. Если msg.sender — approved operator, а не original depositor, rewards не достанутся владельцу. Original depositor теряет всё если не знал об approval.

### Разрывы
| Тип разрыва | Описание |
|---|---|
| Same user, different authority | "My deposit" != "who gets rewards" if approval was given |
| Same success, different finality | Human: "I got rewards". Contract: "rewards in pending state, not transferred" |
| Same action, different accounting | exitFarming triggers reward computation at CURRENT time; if last swap was 12hr ago, rewards since last swap only computed NOW |

**Расхождение:** Пользователь может считать, что rewards "за всё время". Фактически: rewards за время до последнего свапа или до exitFarming (whichever triggered `_increaseCumulative` позже).

---

## Действие 3: `swap` через Router V3

### Human meaning
"Я меняю X токена A на Y токена B по котировке, которую показал UI."

### Contract meaning
Исполняет swap через AlgebraPool. Применяет slippage check (amountOutMin). Использует `path` bytes — последовательность токенов и pool addresses.

### Backend / indexer meaning
The Graph индексирует swap. Обновляет TVL, объём, price impact.

### Next component meaning (AlgebraPool → VirtualPool)
При пересечении тика: `activeIncentive.cross(tick, direction)`. Обновляет virtual pool tick structure.

### Economic meaning
Фермеры в диапазоне получают LP fees + virtual pool обновляется (farming rewards меняются). MEV боты видят pending транзакцию и могут sandwish её.

### Разрывы
| Тип разрыва | Описание |
|---|---|
| Same quote, different execution | UI quote vs actual execution price can differ due to other pending txs |
| Same route, different execution path | Router chooses pool based on encoded path; если path устарел (cached in frontend) — может идти через пул с худшей ценой |
| Same timestamp, different clocks | blockTimestamp (uint32 truncated) vs system time |
| Same success, different finality | UI shows "swap complete" → indexer may show different state if reorg occurs on Moonbeam |

---

## Действие 4: `addRewards` (EternalFarming) + `setRates`

### Human meaning
"Добавляю награды в farming программу. Фермеры начнут получать больше."

### Contract meaning
`addRewards`: переводит tokens на контракт; `virtualPool.addRewards` → увеличивает `rewardReserve`. `setRates`: вызывает `_increaseCumulative` (распределяет старые rewards), затем устанавливает новые ставки.

### Backend / indexer meaning
Событие RewardsAdded и RewardsRatesChanged → APR calculator обновляет displayed APR.

### Next component meaning
Фермеры с следующей транзакцией (swap или collectRewards) получат rewards по НОВОЙ ставке.

### Economic meaning
Добавленные rewards распределяются только с момента следующего `_increaseCumulative`. Если admin добавляет большой reserve, но ставка rate = 0 — rewards никогда не распределятся пока rate не поднимут.

### Разрывы
| Тип | Описание |
|---|---|
| Same reserve, different distribution | Reserve может быть большим, но при rate=0 фермеры ничего не получают |
| Same success, different timing | "Rewards added" != "rewards available now" — нужны свапы для triggerring |
| Same rate, different reserve | High rate + low reserve = rewards exhausted fast; low rate + high reserve = slow but long |

---

## Действие 5: `rampA` в StableSwap

### Human meaning
"Плавно улучшаем/ухудшаем стабильность пула."

### Contract meaning
Записывает `initialA`, `futureA`, `initialATime`, `futureATime`. Функция `_A()` возвращает линейно интерполированное значение.

### Backend / indexer meaning
Индексирует событие RampA. May or may not update cached A value immediately.

### Next component meaning (swap pricing)
Каждый последующий swap использует новое значение A. Те же входные суммы дадут разный output.

### Economic meaning
LP, не знающие о rampA, могут столкнуться с неожиданно высоким slippage при выходе. Арбитражёры торгуют против изменяющегося инварианта.

### Разрывы
| Тип | Описание |
|---|---|
| Same swap params, different result | Swap с теми же параметрами даст разный output до и после rampA начала |
| Same failure, different accounting | LP "normal exit" becomes "exit with impermanent loss" due to A shift |
| Same A value in UI, different A on-chain | Indexer может кэшировать начальное A; реально A уже изменилось |

---

## Действие 6: `safeTransferFrom(nfpm, farmingCenter, tokenId)` (deposit position)

### Human meaning
"Отправляю мою позицию в FarmingCenter для farming."

### Contract meaning
`onERC721Received` создаёт `deposits[tokenId]` с `owner = from`, L2TokenId = nextId. Минтит L2 NFT пользователю.

### Backend / indexer meaning
Событие DepositTransferred. Индексирует owner.

### Next component meaning
L2 NFT представляет ownership позиции в FarmingCenter. Стандартные ERC721 операции (approve, transfer) применимы к L2 NFT.

### Economic meaning
Владелец L2 NFT контролирует: farming enrollment, reward collection, position withdrawal. Approve L2 NFT = передача полного контроля.

### Разрывы
| Тип | Описание |
|---|---|
| Same asset, different owner | "Position owner" ≠ "L2 NFT owner" if L2 was transferred |
| Same approval, different consequence | approve(L2, operator) in NFT context = "can transfer L2"; in FarmingCenter context = "can steal my farming rewards and position" |

---

## Действие 7: Cross-chain swap через StellaSwap bridge UI

### Human meaning
"Обменять tokens с одной цепи на другую в один клик."

### Contract meaning (source chain)
Исполняет swap + bridge через Axelar GMP. Отправляет tokens на Axelar gateway.

### Axelar meaning
Создаёт cross-chain message. Validators подтверждают. Затем исполняет на Moonbeam.

### Contract meaning (Moonbeam)
Получает bridged tokens. Пытается сделать swap на StellaSwap. Если slippage check fails — ??? (refund механизм UNKNOWN).

### Economic meaning
Пользователь платит fees на source chain + Axelar fees + StellaSwap fees. Если что-то идёт не так на любом этапе — может получить промежуточный токен.

### Разрывы
| Тип | Описание |
|---|---|
| Same "swap" different atomicity | User thinks atomic; actually multi-step async |
| Same failure, different accounting | "Failed swap" on one chain = success on bridge, failure on Moonbeam DEX |
| Same refund, different lifecycle | Auto-refund on failed bridge vs manual refund on failed DEX leg |
| Same success, different finality | Source chain confirmed ≠ Moonbeam execution confirmed |

---

## Сводка наиболее важных разрывов

1. **enterFarming liquidity snapshot vs actual position liquidity** — farm records snapshot, actual grows. Backend/indexer may not reflect this.

2. **exitFarming rewards → caller vs owner** — human assumes "owner gets rewards"; contract routes to `msg.sender` who has L2 NFT approval.

3. **StableSwap A rampA** — same A value in indexer cache vs increasing/decreasing on-chain A.

4. **Cross-chain swap atomicity** — user sees "one action"; under the hood 2-3 separate transactions across chains with independent failure modes.

5. **EternalFarming reserve vs rate** — high displayed APR (based on rate) while actual distributable rewards = 0 (empty reserve).
