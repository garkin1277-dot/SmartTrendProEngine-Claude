# SMART TREND PRO ENGINE — DEVELOPMENT LOG

Постоянный журнал проекта. Пополняется по каждому модулю на всех стадиях
Module Lifecycle: Design → Implementation → Testing → Optimization → Freeze.

**Правило ведения:** запись создаётся один раз на модуль и дополняется по
мере прохождения стадий. После Freeze запись не редактируется, кроме как
через Amendment Procedure (см. раздел ниже).

---

## MODULE: SYSTEM CORE

**Layer:** CORE (первый слой в Data Flow: CORE → DATA → DECISION → SIGNAL → EXECUTION → INTERFACE → FUTURE)
**Status:** FROZEN (v1.0.0-core) — 2026-07-21
**Files:** `SmartTrendProEngine_v1_SystemCore.pine` (production), `SmartTrendProEngine_v1_SystemCore_TESTING.pine` (temporary QA build, не входит в контракт)

### Design
Определён контракт `SystemCore` — единственный источник системного состояния
движка. Разделены зоны ответственности: что обязано храниться в CORE
(системное состояние, флаги, готовность данных, fail-safe, module readiness)
и что запрещено (любая торговая логика — сигналы, фильтры, риск, AI, Smart
Money, визуализация).

### Implementation
- UDT `SystemCore` с группами полей: Engine Information, Runtime Information,
  Bar State, Data Availability, Strategy State, Global Execution, Fail Safe,
  Module Readiness.
- Единая функция `populateSystemCore()` — единственная точка записи в
  `SystemCore`.
- Inputs ограничены системными тумблерами (`Enable Strategy/LONG/SHORT/Experimental`).

### Testing — находки

1. **`isLastBar` через `barstate.islast` не компилируется.**
   Pine Script требует `calc_on_every_tick = true` для использования
   `barstate.islast` в `strategy()`. Включать этот режим в проде не стали —
   он даёт пересчёт на каждом тике и риск перерисовки, что противоречит
   принципу стабильности EXECUTION-слоя.
   **Решение:** `isLastBar := bar_index == last_bar_index`. Семантика поля
   не меняется, `calc_on_every_tick` остаётся `false`.

2. **Debug-таблица показывает только последний обработанный бар.**
   Не баг — особенность `table` в Pine: объект переписывается на каждом
   баре, визуальная прокрутка графика не меняет его содержимое. Для
   побарной исторической проверки использован `plot(..., display =
   display.data_window)` — значения по конкретному бару читаются через
   Data Window при наведении курсора.

3. **`isFirstBar = true` ⇒ `isCorePopulated = true`, `isInitializationCompleted = false`.**
   Подтверждено на реальном первом баре истории (BTCUSDT.P, 4h, OKX/Bybit).
   Это ключевое поведение: ядро физически заполнено данными до того, как
   считается "готовым" для потребления другими слоями — на первом баре
   DECISION/SIGNAL обязаны оставаться неактивными.

4. **`isRealtime` не означает "текущий формирующийся бар".**
   Означает "бар, закрывшийся уже после старта текущей сессии скрипта".
   Все бары, загруженные как история до запуска скрипта, остаются
   `isHistorical = true` до перезагрузки страницы. Поведение подтверждено,
   не является дефектом.

5. **Pine требует минимум один visual-output вызов в `strategy()`.**
   Компилятор не даст собрать `strategy()` без хотя бы одного
   `plot/bgcolor/table/...`, даже если модулю нечего показывать.
   **Первая версия решения оказалась ошибочной и была исправлена по
   факту тестирования компиляции:** `plot(..., display =
   display.data_window)` **в одиночку не удовлетворяет требование
   компилятора** — при попытке скомпилировать с одним только
   `display.data_window` TradingView возвращает ошибку `A strategy must
   contain at least one of the following: any "strategy.*()" function
   that creates orders, any "plot*()" function, ...`. Подтверждено
   практической компиляцией (не предположение).
   Попытка убрать `display` полностью устраняет ошибку компиляции, но
   создаёт новое нарушение: без явного `display` используется
   `display.all` по умолчанию, и при `overlay = true` плейсхолдер
   рисуется линией поверх цены на графике — это противоречит Visual
   Design Rules проекта.
   **Финальное решение, подтверждённое компиляцией и визуальной
   проверкой на графике (ETHUSDT.P, 4h, Bybit):**
   `plot(core.isCorePopulated ? 1 : 0, title =
   "STPE_CORE_COMPILER_PLACEHOLDER", display = display.data_window +
   display.status_line)`. Компилируется без ошибок; на графике ничего
   не рисуется; значение доступно через Data Window и через панель
   значений скрипта (status line). Это техническая заглушка ради
   компиляции, не реализация VISUAL ENGINE. **Подлежит пересмотру**, когда
   появится VISUAL ENGINE — либо заменяется реальным выводом того слоя,
   либо остаётся как есть, если у VISUAL ENGINE появится собственный
   независимый visual-output.

6. **`isLastBar` структурно не достигает `true` на непрерывно торгуемых (24/7)
   инструментах.**
   Подтверждено практической проверкой на реальном времени (ETHUSDT.P, 4h,
   Bybit): на последнем фактически обработанном баре (`barIndexCurrent`
   совпадает с последним значением в debug-таблице) `isLastBar` всё равно
   возвращает `false`.
   **Причина:** `isLastBar := bar_index == last_bar_index` (см. находку
   №1). `last_bar_index` в Pine всегда указывает на текущий, в том числе
   ещё не закрывшийся бар. Поскольку `calc_on_every_tick = false`,
   `populateSystemCore()` выполняется только на закрытых барах. На
   инструменте без торговых сессий (крипто, 24/7) формирующийся бар есть
   всегда, поэтому последний закрытый и обработанный бар — это
   `last_bar_index - 1`, а не `last_bar_index`. Формула структурно не
   может дать `true` в этих условиях.
   **Статус: принято как ожидаемое поведение, не дефект.** `isLastBar`
   в текущей реализации не применим для условия "это последний закрытый
   бар на непрерывно торгуемом инструменте" — оговорка зафиксирована в
   контракте. Семантика поля остаётся без изменений (`bar_index ==
   last_bar_index`), решение о доработке (если какому-то будущему модулю
   потребуется именно "последний закрытый бар") откладывается до
   Amendment Procedure по факту реальной необходимости, а не превентивно.

### Testing — покрытие (все пункты подтверждены)

| Группа | Проверено |
|---|---|
| Fail Safe (`isDataInvalid`/`isFailSafeTriggered`/`isNeutralPassAvailable`) | ✅ |
| Strategy State (все тумблеры, все комбинации) | ✅ |
| Global Execution (`hasPosition`/`positionDirection`/`positionSize`/`isFlat`) | ✅ на реальных сделках long/short |
| Module Readiness (`isCorePopulated` vs `isInitializationCompleted`) | ✅ на первом баре истории |
| Bar State (`isFirstBar`/`isLastBar`/`barIndexCurrent`) | ⚠️ с оговоркой — см. находку №6 |
| Runtime (`isRealtime`/`isHistorical`) | ✅ |

### Optimization
_(в процессе)_
- [x] Убрать QA-harness полностью из production-файла (debug table, Data Window plots, жёлтая подсветка, тест-хуки входа в позицию). **Подтверждено построчной сверкой** `SmartTrendProEngine_v1_SystemCore.pine`: ни одного из перечисленных артефактов не найдено. Единственный `plot()` в файле — `STPE_CORE_COMPILER_PLACEHOLDER`, классифицирован как Compiler Compliance Requirement (не Testing/Visual/Experimental Artifact, не QA Harness) и не подлежит удалению.
- [x] Проверить производительность `populateSystemCore()` на большом датасете (5000+ баров, `max_bars_back`). **Подтверждено практическим тестом:** ETHUSDT.P, 4h, Bybit, датасет 12 161 бар (01.01.2021 03:00 → реальное время, 20.07.2026 включительно) — компиляция без предупреждений/timeout, точечная сверка полей `core` на первом (`bar_index=0`), среднем (`bar_index=5372`) и последнем обработанном (`bar_index=12160`) барах — все поля соответствуют ожидаемому поведению контракта (с учётом принятой оговорки по `isLastBar`, находка №6).

**Freeze Gate (обязательные условия для перехода к Freeze):**
- [x] QA-harness отсутствует в production-файле — verified
- [x] `populateSystemCore()` performance validated on 5000+ bars — verified (12 161 bars, ETHUSDT.P 4h Bybit, no compiler warnings/timeout)
- [x] No compiler warnings/errors on full validation run — verified
- [x] Known accepted limitation documented: `isLastBar` does not reach `true` on continuously-traded (24/7) instruments under `calc_on_every_tick = false` — see finding #6

### Freeze
- [x] Дата заморозки: 2026-07-21
- [x] Итоговая версия контракта зафиксирована как **v1.0.0-core**
- [x] Project Freeze Status обновлён (см. таблицу ниже)

**Оговорка при заморозке:** `isLastBar` замораживается в текущей семантике
(`bar_index == last_bar_index`) с документированным известным ограничением
(находка №6) — не является дефектом, доработка откладывается до Amendment
Procedure по факту реальной потребности со стороны последующих слоёв.

---

## MODULE: LONG SETTINGS

**Layer:** CORE LAYER (второй модуль слоя, после SYSTEM CORE)
**Status:** DESIGN
**Files:** `SmartTrendProEngine_v1_LongSettings.pine` (не создан)

### Design

**1. Universal Boundary Rule**

Если настройка относится исключительно к одной функциональной области, она
всегда принадлежит соответствующей группе настроек — независимо от типа
(bool/int/float/string), сложности или того, является ли она
enable/disable-флагом. Принадлежность определяется областью
ответственности настройки, а не её формой.

**2. Group Architecture**

#### 2.1 Long General Settings

- **Purpose:** Глобальная конфигурация LONG-системы как независимой
  торговой подсистемы. Определяет режим работы LONG в целом, а не
  поведение отдельных его частей.
- **Architectural Constraints:**
  Может содержать только: глобальные настройки LONG-системы; настройки
  включения/отключения функциональных частей LONG; настройки режима
  работы LONG.
  Не может содержать: настройки отдельных фильтров, риска, позиции,
  сигналов, визуализации, алертов; параметры любой другой из 7 групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычисляемых значений, никакой условной логики внутри группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Boundary Notes:** Общий enable-флаг конкретной функциональной области
  (например, `Enable Long Trend Filter`) относится к своей
  специализированной группе, а не к General, даже если выглядит
  "глобальным". Критерий: если настройка управляет поведением конкретной
  функциональной области — она принадлежит этой области, даже если это
  просто on/off. General Settings содержит только то, что управляет
  LONG-системой целиком и не может быть отнесено ни к одной
  специализированной группе (например: включена ли LONG-система вообще).

#### 2.2 Long Filter Settings

- **Purpose:** Пользовательская конфигурация всех фильтров LONG-системы
  (Trend/Volatility/Momentum/Market/Confirmation — согласно замороженной
  Filter Architecture). Определяет, какие фильтры включены и с какими
  ручными параметрами они работают.
- **Architectural Constraints:**
  Может содержать только: enable/disable-флаги по каждому фильтру или
  группе фильтров LONG; ручные параметры фильтров (числовые значения,
  режимы работы, варианты алгоритмов, источники данных, таймфреймы и
  другие декларативные настройки фильтра).
  Не может содержать: саму логику вычисления фильтра (LONG FILTER ENGINE /
  DECISION LAYER); параметры риска, позиции, сигналов, визуализации,
  алертов; параметры General/других групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычислений, никакой условной логики между фильтрами внутри группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Internal Filter Independence Rule:** Каждый фильтр внутри группы —
  независимая конфигурационная сущность. Запрещается: изменение
  параметров одного фильтра другим; условная доступность параметров
  одного фильтра на основании параметров другого; автоматическая
  синхронизация значений между фильтрами; скрытые зависимости между
  фильтрами. Комбинирование фильтров допускается исключительно в LONG
  FILTER ENGINE (DECISION LAYER).
- **Boundary Notes:** `Enable Trend Filter`, `Enable Volatility Filter` и
  т.п. принадлежат этой группе, а не General. Расчёт значения фильтра
  принадлежит DECISION LAYER — Filter Settings хранит только вход
  (параметры), не вывод (результат вычисления).

#### 2.3 Long Risk Settings

- **Purpose:** Пользовательская конфигурация всех механизмов
  риск-менеджмента LONG-системы. Определяет ручные настройки
  риск-механизмов и предоставляет входные параметры для их последующего
  использования нижележащими слоями.
- **Architectural Constraints:**
  Может содержать только: enable/disable-флаги отдельных механизмов
  риск-менеджмента (ATR-based stop, fixed stop, trailing stop и т.п.);
  ручные параметры риска (числовые значения, режимы расчёта, варианты
  алгоритмов, источники данных для расчёта риска).
  Не может содержать: сам расчёт риска, стоп-лосса, тейк-профита (risk-
  модуль нижележащего слоя); параметры фильтров, позиции, сигналов,
  визуализации, алертов; параметры General/других групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычислений и никакой условной логики между риск-механизмами внутри
  группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Internal Risk Mechanism Independence Rule:** Каждый риск-механизм —
  независимая конфигурационная сущность. Запрещается: изменение
  параметров одного механизма другим; условная доступность параметров
  одного механизма на основании параметров другого; автоматическая
  синхронизация значений между механизмами; скрытые зависимости между
  ними. Комбинирование/выбор активного механизма риска — исключительно в
  соответствующем risk-модуле нижележащего слоя.
- **Boundary Notes:** `Enable ATR Stop Loss` и подобные флаги принадлежат
  этой группе, а не General. Расчёт стоп-лосса/тейк-профита принадлежит
  risk-модулю — Risk Settings хранит только вход, не результат.
- **Boundary Rule: Risk vs Position:** Если настройка определяет
  допустимый уровень риска сделки — Long Risk Settings. Если настройка
  определяет способ открытия, удержания, изменения или закрытия позиции —
  Long Position Settings. Принадлежность определяется объектом
  управления, а не индустриальными привычками именования.
  Разбор примеров: Stop Loss distance/method → Risk; Take Profit
  distance/method → Risk; Risk % per trade → Risk; Break Even trigger →
  Risk; Position Size → Position; Pyramid Entries / Scale In / Scale Out /
  Partial Take Profit → Position.

#### 2.4 Long Position Settings

- **Purpose:** Пользовательская конфигурация всех механизмов управления
  позицией LONG-системы: как открывается, изменяется и закрывается
  позиция. Саму логику исполнения не реализует.
- **Architectural Constraints:**
  Может содержать только: enable/disable-флаги отдельных механизмов
  управления позицией (Pyramid Entries, Scale In, Scale Out, Partial Take
  Profit и т.п.); ручные параметры позиции (размер/объём, число входов,
  шаг наращивания, условия частичного закрытия, режимы расчёта, источники
  данных).
  Не может содержать: сам расчёт исполнения позиции (EXECUTION LAYER);
  параметры риска (Long Risk Settings); параметры фильтров, сигналов,
  визуализации, алертов; параметры General/других групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычислений и никакой условной логики между механизмами внутри группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Internal Position Mechanism Independence Rule:** Каждый механизм
  управления позицией — независимая конфигурационная сущность.
  Запрещается: изменение параметров одного механизма другим; условная
  доступность параметров одного механизма на основании параметров
  другого; автоматическая синхронизация значений между механизмами;
  скрытые зависимости между ними. Комбинирование механизмов — только в
  EXECUTION LAYER.
- **Boundary Notes:** `Enable Re-entry Logic`, `Enable Scale In` и
  подобные флаги принадлежат этой группе, а не General. Position Size —
  Position (величина риска, если используется как источник расчёта, —
  вход из Risk Settings, но само поле размера позиции живёт здесь);
  Pyramid Entries / Scale In / Scale Out / Partial Take Profit — Position;
  Stop Loss / Take Profit / Break Even — НЕ принадлежат этой группе (см.
  Long Risk Settings). Расчёт фактического объёма ордера — EXECUTION
  LAYER.

#### 2.5 Long Signal Settings

- **Purpose:** Пользовательская конфигурация правил формирования и
  подтверждения торговых сигналов LONG-системы. Саму генерацию/вычисление
  сигнала не выполняет.
- **Architectural Constraints:**
  Может содержать только: enable/disable-флаги отдельных механизмов
  сигнальной логики (Enable Signal Confirmation, Enable Multi-Timeframe
  Confirmation и т.п.); ручные параметры сигналов (число подтверждений,
  задержка подтверждения, режимы согласования с фильтрами, источники
  данных, таймфреймы подтверждения).
  Не может содержать: саму логику генерации/вычисления сигнала (SIGNAL
  LAYER); параметры риска, позиции, фильтров, визуализации, алертов;
  параметры General/других групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычислений и никакой условной логики между механизмами внутри группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Internal Signal Mechanism Independence Rule:** Каждый механизм
  сигнальной логики — независимая конфигурационная сущность. Запрещается:
  изменение параметров одного механизма другим; условная доступность
  параметров одного механизма на основании параметров другого;
  автоматическая синхронизация значений между механизмами; скрытые
  зависимости между ними. Комбинирование механизмов подтверждения — только
  в SIGNAL LAYER.
- **Boundary Notes:** `Enable Signal Confirmation` и подобные флаги
  принадлежат этой группе, а не General.
- **Boundary Rule: Filter vs Signal:** Filter Settings конфигурирует
  отдельные независимые условия рынка (тренд, волатильность, моментум,
  market state, confirmation-индикатор как самостоятельная единица).
  Signal Settings конфигурирует правила согласования результатов фильтров
  и других условий в единое решение о сигнале (сколько подтверждений
  нужно, на каком таймфрейме, с какой задержкой), но не сами условия.
  Итоговое вычисление и комбинирование — целиком в DECISION LAYER / SIGNAL
  LAYER.

#### 2.6 Long Visual Settings

- **Purpose:** Пользовательская конфигурация визуального отображения
  LONG-системы на графике. Саму отрисовку не выполняет.
- **Architectural Constraints:**
  Может содержать только: enable/disable-флаги отдельных визуальных
  элементов LONG (Enable Long Dashboard, Enable Long Entry Markers, Enable
  Long Stop/Take Lines и т.п.); ручные параметры отображения (цвета,
  прозрачность, размер/толщина, позиция на графике, формат подписи).
  Не может содержать: саму логику отрисовки (VISUAL ENGINE / PLOTS
  ENGINE); параметры риска, позиции, фильтров, сигналов, алертов;
  параметры General/других групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычислений и никакой условной логики между визуальными элементами
  внутри группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Internal Visual Element Independence Rule:** Каждый визуальный элемент
  — независимая конфигурационная сущность. Запрещается: изменение
  параметров одного элемента другим; условная доступность параметров
  одного элемента на основании параметров другого; автоматическая
  синхронизация значений между элементами; скрытые зависимости между
  ними. Комбинирование визуальных элементов на графике — только в VISUAL
  ENGINE / PLOTS ENGINE.
- **Boundary Notes:** `Enable Long Dashboard` и подобные флаги принадлежат
  этой группе, а не General.
- **Visual Ownership Rule:** Любые пользовательские настройки, влияющие
  исключительно на визуальное представление элементов LONG-системы,
  принадлежат исключительно группе Long Visual Settings — независимо от
  того, какой функциональной области принадлежит отображаемый элемент.
  Распространяется на: цвета, стили отображения, размеры элементов,
  параметры отображения, настройки видимости, другие визуальные
  параметры. Другие группы LONG SETTINGS не могут содержать визуальные
  настройки ни при каких условиях.
- **Boundary Rule: Visual vs All Other Groups:** Если настройка влияет
  только на внешний вид элемента — она всегда принадлежит Long Visual
  Settings. Принадлежность определяется характером самой настройки
  (presentation), а не происхождением отображаемого элемента.
  Примеры: цвет отображения Trend Filter → Visual; показывать ли LONG
  Dashboard → Visual; цвет линии Stop Loss → Visual; размер метки LONG
  Signal → Visual; цвет отображения Partial Take Profit → Visual. Ни одна
  другая группа LONG SETTINGS не может содержать визуальные поля.

#### 2.7 Long Alert Settings

- **Purpose:** Пользовательская конфигурация алертов LONG-системы. Саму
  генерацию и отправку alert-события не выполняет.
- **Architectural Constraints:**
  Может содержать только: enable/disable-флаги отдельных типов алертов
  LONG (Enable Long Entry Alert, Enable Long Exit Alert, Enable Long
  Stop/Take Alert и т.п.); ручные параметры алертов (текст/шаблон
  сообщения, частота срабатывания, формат, дополнительные декларативные
  настройки уведомления).
  Не может содержать: саму логику формирования и отправки alert-события
  (соответствующий alert-модуль EXECUTION/INTERFACE LAYER); параметры
  риска, позиции, фильтров, сигналов, визуализации; параметры
  General/других групп.
- **Allowed Setting Types:** Только декларативные inputs. Никаких
  вычислений и никакой условной логики между типами алертов внутри
  группы.
- **Group Independence:** Не зависит от остальных 6 групп и не влияет на
  них.
- **Internal Alert Type Independence Rule:** Каждый тип алерта —
  независимая конфигурационная сущность. Запрещается: изменение
  параметров одного типа алерта другим; условная доступность параметров
  одного типа алерта на основании параметров другого; автоматическая
  синхронизация значений между типами алертов; скрытые зависимости между
  ними. Комбинирование/логика триггера алерта — только в соответствующем
  alert-модуле нижележащего слоя.
- **Boundary Notes:** `Enable Long Alerts`, `Enable Long Entry Alert` и
  подобные флаги принадлежат этой группе, а не General.
- **Boundary Rule: Alert vs остальные группы:** Long Alert Settings хранит
  только "включён ли алерт" и "как он выглядит" (оформление уведомления),
  но не "при каком условии он срабатывает" — само условие определяется
  той группой/слоем, к которой относится событие (например, событие входа
  — SIGNAL LAYER, событие срабатывания стопа — risk-модуль EXECUTION
  LAYER). Alert Settings не дублирует и не хранит эти условия — только
  параметры уведомления о событии, которое произошло в другом слое.

**3. Additional Architectural Rules (сводно)**
- Internal Filter Independence Rule (2.2)
- Internal Risk Mechanism Independence Rule (2.3)
- Boundary Rule: Risk vs Position (2.3)
- Internal Position Mechanism Independence Rule (2.4)
- Internal Signal Mechanism Independence Rule (2.5)
- Boundary Rule: Filter vs Signal (2.5)
- Internal Visual Element Independence Rule (2.6)
- Visual Ownership Rule (2.6)
- Boundary Rule: Visual vs All Other Groups (2.6)
- Internal Alert Type Independence Rule (2.7)
- Boundary Rule: Alert vs остальные группы (2.7)

**4. Architecture Compliance Review**

| Критерий | Статус |
|---|---|
| Specification compliance | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle | ✅ |

\* Полная симметричная проверка выполняется после завершения Level 1 для
SHORT SETTINGS — нормальное ограничение порядка разработки, не
архитектурный недостаток.

Отдельно отмечено и закрыто в ходе Review: изначально неоднозначная
формулировка Boundary Notes у Long Visual Settings ("по умолчанию
остаются в Visual Settings, откладывается до Level 2") заменена на
окончательное правило — Visual Ownership Rule (см. 2.6) — до перехода к
Level 2.

**5. Design Progress**
- Level 1 (Group Architecture): **COMPLETE**
- Level 2 (Field Architecture): **IN PROGRESS** — см. пункты 6–9

**6. Level 2 Rules (applied during Long Filter Settings design)**

- Level 2 Meta Rule
- General Settings Minimalism Rule
- Configuration vs Runtime Rule
- Candidate Field Validation Rule
- Minimal Field Count Principle
- Specialized Group Completeness Rule
- Filter Subgroup Minimal Contract Rule
- Mandatory vs Optional Field Rule
- Parameter Container Rule
- Algorithm Agnosticism Rule
- Design Recursion Boundary Rule (revised)
- Responsibility Boundary Rule
- Filter Pattern Observation Rule
- Data Ownership Boundary Rule
- Shared vs Local Computation Rule
- Visual Element Independence Validation Rule (added during Long Visual
  Settings design)
- Standard term: `ARCHITECTURALLY COMPLETE (for current Design level)`

**7. Level 2 — Long General Settings**

**Candidate Review — Enable LONG System**

`Enable LONG System` — **REJECTED**.

Причина: master-toggle LONG-системы уже принадлежит SYSTEM CORE:

```text
SYSTEM CORE
├─ i_longEnabled
│  └─ пользовательский input: Enable LONG System
└─ core.isLongEnabled
   └─ runtime-состояние:
      i_strategyEnabled AND i_longEnabled
```

LONG SETTINGS не создаёт дублирующий input с тем же назначением, именем
или иной формулировкой. Такое поле нарушило бы Module Responsibility
Principle и создало бы два конкурирующих владельца одного master-toggle.

Это не Amendment: SYSTEM CORE, его контракт, inputs и поведение не
изменяются. Решение относится исключительно к незамороженному модулю
LONG SETTINGS.

**Previously reviewed candidates — retained without change**

- Long Operating Mode — **REJECTED**: undefined responsibility, AUTO PARAMETERS risk.
- Trading Style — **REJECTED**: AUTO PARAMETERS violation.
- Entry Mode — **REJECTED**: принадлежит Long Position Settings.
- LONG Profile / LONG Preset — **REJECTED**: mass automatic parameter modification.
- Market Type — **UNCLEAR**: не одобрено в текущей версии; может быть
  повторно рассмотрено только как новый кандидат через Candidate Field
  Validation Rule после отдельного обоснования области ответственности.

**Level 2 Result**

| Показатель | Значение |
|---|---|
| Approved Fields | 0 |
| Rejected Candidate | Enable LONG System |
| Причина отсутствия полей | Глобальный master-toggle LONG уже является ответственностью SYSTEM CORE |

**Compliance Review**

Long General Settings прошёл Level 2 с результатом **0 утверждённых полей**.
Пустой состав является осознанным архитектурным решением, а не признаком
незавершённости.

Применены:

- General Settings Minimalism Rule;
- Minimal Field Count Principle;
- Module Responsibility Principle;
- Configuration vs Runtime Rule.

**Status: LONG GENERAL SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

**Уточнение Minimal Field Count Principle**

Архитектурно корректная группа может содержать любое количество полей,
**включая ноль**, если это является результатом применения архитектурных
ограничений. Запрещается добавлять поля ради симметрии, видимой полноты
или заполнения группы.

**8. Level 2 — Long Filter Settings**

Maximum Responsibility: пользовательская конфигурация всех LONG-фильтров
(Trend/Volatility/Momentum/Market/Confirmation — Frozen Filter
Architecture, все пять MANDATORY).

*Trend Filters* — Fields: Enable Trend Filter (bool, MANDATORY), Trend
Algorithm (string, algorithm-agnostic), Trend Timeframe (timeframe,
default `""`), Trend Source (source, default `close`). Container: Trend
Parameters (Algorithm-dependent, structure not yet defined, depends on
Trend Filter Algorithm Contract).

*Volatility Filters* — тот же паттерн полей: Enable Volatility Filter
(MANDATORY), Volatility Algorithm, Volatility Timeframe, Volatility
Source, Container Volatility Parameters (Algorithm-dependent).

*Momentum Filters* — независимо проверено через Candidate Review (не
скопировано по паттерну): тот же состав — Enable Momentum Filter
(MANDATORY), Momentum Algorithm, Momentum Timeframe, Momentum Source,
Container Momentum Parameters (Algorithm-dependent).

*Market Filters* — отличается от паттерна остальных трёх по Data
Ownership Boundary Rule / Shared vs Local Computation Rule: Enable Market
Filter (MANDATORY), Market Regime Usage (string/string[], regime-agnostic,
зависит от будущего MARKET REGIME ENGINE Regime Taxonomy Contract).
Market Timeframe/Market Data Source/Market Condition Algorithm/Market
Parameters — REJECTED/REMOVED (принадлежат MARKET REGIME ENGINE, DATA
LAYER — общесистемное состояние, а не локальный расчёт фильтра).
Trading Session Filter — NOT APPROVED (граница с Execution Layer не
определена). News Filter — REJECTED.

*Confirmation Filters* — Enable Confirmation Filter (MANDATORY),
Confirmation Algorithm, Confirmation Timeframe, Confirmation Source,
Container Confirmation Parameters (Algorithm-dependent). Confirmation
Count / Number of Confirmations Required — REJECTED (Boundary Rule: Filter
vs Signal — принадлежит Long Signal Settings).

**Architecture Compliance Review (Level 2, Long Filter Settings):**

| Критерий | Статус |
|---|---|
| Frozen Filter Architecture (все 5 подгрупп) | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle (Filter↔Signal, Filter↔Data, Filter↔Risk/Position/Execution) | ✅ |
| Internal Filter Independence Rule | ✅ |
| Data Ownership Boundary | ✅ |

\* Полная симметричная проверка — после проектирования SHORT SETTINGS.

**Status: LONG FILTER SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

**9. Design Progress (updated)**
- Level 1 (Group Architecture): **COMPLETE**
- Level 2 (Field Architecture):
  - Long General Settings: **ARCHITECTURALLY COMPLETE**
  - Long Filter Settings: **ARCHITECTURALLY COMPLETE**
  - Long Risk Settings: **ARCHITECTURALLY COMPLETE**
  - Long Position Settings: **ARCHITECTURALLY COMPLETE**
  - Long Signal Settings: **ARCHITECTURALLY COMPLETE**
  - Long Visual Settings: **ARCHITECTURALLY COMPLETE**
  - Long Alert Settings: **ARCHITECTURALLY COMPLETE**

Current Design Status: **LEVEL 2 DESIGN — COMPLETE (7 / 7 LONG SETTINGS
groups)**. LONG SETTINGS Level 2 Design завершён. Следующий этап
жизненного цикла — Implementation (когда предусмотрено дорожной картой).

**10. Level 2 — Long Risk Settings**

**Boundary Definition:** Long Risk Settings отвечает на вопрос "какой
уровень риска принимает LONG-сделка и как этот риск ограничивается во
времени" (объект управления — допустимая потеря, уровень защитного
выхода, риск/вознаграждение, динамика защитного уровня). Не отвечает за
размер/структуру позиции (Long Position Settings), условия входа (Long
Signal Settings), фактическое исполнение (EXECUTION LAYER).

**Risk vs Position Object Rule:** разграничение определяется объектом
изменения, а не названием функции — если изменяется уровень допустимого
убытка, это Risk; если изменяется сама позиция как сущность, это
Position. Следствие: Stop Loss/Trailing Stop/Break Even → Risk; Position
Size → Position.

**Partial Exit Clarification Rule:** термин "Partial Exit" неоднозначен.
Partial Take Profit (частичное закрытие по достижении цели прибыли) —
остаётся Position Settings (решение Level 1, не пересмотрено).
Risk-Based Partial Exit (частичное закрытие по риск-триггеру) — DEFERRED,
отдельный будущий Candidate Review, не создаёт конфликта с уже принятым
решением.

**Internal Risk Mechanism Independence Rule (applied):** Stop Loss, Take
Profit, Break Even, Trailing Stop — независимые механизмы, каждый со
своим Enable-полем; общий `Enable Risk Management` не заменяет
индивидуальные enable-поля.

**Candidate Architecture Review — Approved:**

| Candidate | Type | Status |
|---|---|---|
| Enable Risk Management | Field | ✅ MANDATORY |
| Risk per Trade | Field | ✅ APPROVED |
| Enable Stop Loss | Field | ✅ APPROVED |
| Stop Loss Method | Field (algorithm-agnostic) | ✅ APPROVED |
| Stop Loss Parameters | Container (Algorithm-dependent) | ✅ APPROVED |
| Enable Take Profit | Field | ✅ APPROVED |
| Take Profit Method | Field (algorithm-agnostic) | ✅ APPROVED |
| Take Profit Parameters | Container (Algorithm-dependent) | ✅ APPROVED |
| Maximum Exposure | Field | ✅ APPROVED |
| Enable Break Even | Field | ✅ APPROVED |
| Break Even Trigger Method | Field (algorithm-agnostic) | ✅ APPROVED |
| Break Even Trigger Parameters | Container (Algorithm-dependent) | ✅ APPROVED |
| Enable Trailing Stop | Field | ✅ APPROVED |
| Trailing Stop Method | Field (algorithm-agnostic) | ✅ APPROVED |
| Trailing Stop Parameters | Container (Algorithm-dependent) | ✅ APPROVED |
| Risk-Based Partial Exit | — | ⏸ DEFERRED |
| Risk Profile / Preset / Auto Risk Mode / Smart Risk Selector | — | ❌ REJECTED (Manual Parameters Principle / AUTO PARAMETERS) |

**Field Architecture (summary):** все Method-поля (Stop Loss/Take
Profit/Break Even Trigger/Trailing Stop) — `string`, algorithm-agnostic,
Allowed Values/Default определяются соответствующим ещё не спроектированным
Method Contract. Все Enable-поля — `bool`. `Risk per Trade` (`float`,
default `1.0`, % капитала). `Maximum Exposure` (`float`, default `5.0`, %
капитала). Все Parameters — Containers (Algorithm-dependent, structure not
yet defined).

**Architecture Compliance Review:**

| Критерий | Статус |
|---|---|
| Frozen Risk Architecture | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle (Risk↔Position/Signal/Execution/General) | ✅ |
| Internal Risk Mechanism Independence | ✅ |
| Algorithm Agnosticism Rule | ✅ |
| Parameter Container Rule | ✅ |
| Configuration vs Runtime Separation | ✅ |

\* Полная симметричная проверка — после проектирования SHORT SETTINGS.

**Status: LONG RISK SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

**11. Level 2 — Long Position Settings**

**Boundary Definition:** Long Position Settings отвечает на вопрос «как
существует, изменяется и завершается LONG-позиция как торговая сущность
после её открытия» (объект управления — размер позиции, структура
позиции, количество входов, правила изменения объёма, правила полного или
частичного закрытия). Не отвечает за допустимый уровень риска и защитные
механизмы (Long Risk Settings), условия возникновения торгового сигнала
(Long Signal Settings), фактическое исполнение ордеров (EXECUTION LAYER),
вычисление условий изменения позиции (нижележащие слои).

**Boundary Rule: Risk vs Position (Level 1, п.2.3 — применено):** если
изменяется уровень допустимого убытка/условие риска — Risk; если
изменяется сама позиция как структурная сущность — Position. Прецедент:
Stop Loss/Take Profit/Break Even/Trailing Stop полностью закрывают
позицию физически, но остаются Risk, поскольку определяют риск-условие, а
не структуру позиции.

**Internal Position Mechanism Independence Rule (Level 1, п.2.4 —
применено):** каждый механизм управления позицией (Pyramid Entries, Scale
In, Scale Out) владеет собственной конфигурацией полностью независимо,
включая собственные количественные лимиты; Position не координирует и не
распределяет ресурсы между механизмами.

**Candidate Architecture Review:**

| Candidate | Type | Status |
|---|---|---|
| Position Size | Field | ✅ PROVISIONALLY APPROVED |
| Partial Take Profit | Field (Enable + Method + Container) | ✅ APPROVED |
| Pyramid Entries | Field (Enable + Maximum Entries + Method + Container) | ✅ APPROVED |
| Scale In | Field (Enable + Maximum Entries + Method + Container) | ✅ APPROVED |
| Scale Out | Field (Enable + Method + Container) | ✅ APPROVED |
| Risk-Based Partial Exit | — | ⏸ DEFERRED — конфликт двух прецедентов: Partial Exit Clarification Rule (частичность → Position) vs Boundary Rule: Risk vs Position (риск-триггер → Risk) |
| Entry Mode | — | ❌ REJECTED — Minimal Field Count Principle, дублирование Pyramid/Scale In |
| Multiple Entries | — | ❌ REJECTED — вычисляемое свойство (`Maximum * Entries > 1`), нарушает Configuration vs Runtime Rule |
| Add-on Entries | — | ❌ REJECTED — дублирует Pyramid/Scale In под другим именем |
| Maximum Entries (Position-level) | — | ❌ REJECTED — создаёт скрытую координацию между механизмами, нарушает Internal Position Mechanism Independence Rule |
| Re-entry Logic | — | ↪ OUT OF BOUNDARY → Long Signal Settings |
| Signal Reconfirmation | — | ↪ OUT OF BOUNDARY → Long Signal Settings |
| Ограничения на повторные входы (условные/временные) | — | ↪ OUT OF BOUNDARY → Long Signal Settings |

**Negative Candidate Review:**

| Candidate | Status | Причина |
|---|---|---|
| Position Merge | ❌ REJECTED | Конфликтует с зафиксированной single-position моделью SYSTEM CORE (`hasPosition`/`positionDirection`/`positionSize`) |
| Position Split | ❌ REJECTED | Конфликтует с той же моделью SYSTEM CORE; дублирует Scale Out |
| Position Lock | ❌ REJECTED | Относится к SYSTEM CORE Strategy State / EXECUTION LAYER, не к структурному свойству позиции |
| Universal Position Mode | ❌ REJECTED | Нарушение Principle 3 — AUTO PARAMETERS permanently prohibited |
| Auto Position Management | ❌ REJECTED | Нарушение Principle 3 — AUTO PARAMETERS permanently prohibited |

**Field Architecture (summary):**

- **Position Size** — mandatory. `PROVISIONALLY APPROVED Field
  Architecture`: `Position Size Method`, `Position Size Parameters`.
  Position Size Method Contract, Allowed Values и Default Values будут
  определены отдельно при проектировании Position Size Method Contract.
- **Partial Take Profit** — `Enable Partial Take Profit` (`bool`) +
  `Partial Take Profit Method` (`string`) + `Partial Take Profit
  Parameters` (Container).
- **Pyramid Entries** (полностью самодостаточный механизм) — `Enable
  Pyramid Entries` (`bool`) + `Maximum Pyramid Entries` (`int`) + `Pyramid
  Entries Method` (`string`) + `Pyramid Entries Parameters` (Container).
- **Scale In** (полностью самодостаточный механизм) — `Enable Scale In`
  (`bool`) + `Maximum Scale In Entries` (`int`) + `Scale In Method`
  (`string`) + `Scale In Parameters` (Container).
- **Scale Out** — `Enable Scale Out` (`bool`) + `Scale Out Method`
  (`string`) + `Scale Out Parameters` (Container).

**Architecture Compliance Review:**

| Критерий | Статус |
|---|---|
| Frozen Position Architecture (Level 1) | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle (Position↔Risk/Signal/Execution/General) | ✅ |
| Internal Position Mechanism Independence Rule (Level 1, applied) | ✅ |
| Boundary Rule: Risk vs Position (Level 1, applied) | ✅ |
| Parameter Container Rule | ✅ |
| Configuration vs Runtime Rule | ✅ |
| Minimal Field Count Principle | ✅ |

\* Полная симметричная проверка — после проектирования SHORT SETTINGS.

**Status: LONG POSITION SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

**12. Level 2 — Long Signal Settings**

**Forward Reference Review (выполнен перед Boundary Definition):**

| Source | Candidate | Status | Reason |
|---|---|---|---|
| Filter | Confirmation Count / Number of Confirmations Required | REJECTED (Filter) → Forward | `Boundary Rule: Filter vs Signal` (Level 1, п.2.5) |
| Position | Re-entry Logic | OUT OF BOUNDARY → Forward | Position отвечает «можно ли физически», не «когда разрешено» |
| Position | Signal Reconfirmation | OUT OF BOUNDARY → Forward | Обработка повторного сигнала после открытия позиции |
| Position | Ограничения на повторные входы (условные/временные) | OUT OF BOUNDARY → Forward | Временные/условные ограничения — область Signal |

Рассмотрено и сознательно не включено: `Market Type` (General, UNCLEAR —
без назначенного адресата), `Risk-Based Partial Exit` (Risk/Position,
DEFERRED — оспаривается между Risk и Position, Signal не участвует),
`Trading Session Filter` (Filter, NOT APPROVED — граница с EXECUTION
LAYER, не с Signal), `News Filter` (Filter, REJECTED — форварда нет).

**Boundary Definition:** Long Signal Settings отвечает на вопрос «когда
LONG-сигнал считается достаточно подтверждённым для выполнения
разрешённого торгового действия». На текущем этапе архитектуры такими
действиями являются Initial Entry, Additional Entry, Re-entry — как
примеры, а не исчерпывающий контракт. Саму генерацию/вычисление сигнала
группа не выполняет (SIGNAL LAYER); она конфигурирует правила
согласования.

**Signal ↔ Position:** Signal отвечает только за принятие решения о
допустимости торгового действия; Position отвечает за то, как изменится
позиция после этого решения.

Не отвечает за: сами условия рынка как независимые единицы (Long Filter
Settings), уровень риска (Long Risk Settings), физическую структуру
позиции (Long Position Settings), итоговое вычисление сигнала
(DECISION/SIGNAL LAYER).

**Boundary Rule: Filter vs Signal (Level 1, п.2.5 — применено):** Filter
Settings конфигурирует отдельные независимые условия рынка; Signal
Settings конфигурирует правила согласования результатов фильтров и
других условий в единое решение о сигнале, но не сами условия.

**Internal Signal Mechanism Independence Rule (Level 1, п.2.5 —
применено):** каждый механизм сигнальной логики — независимая
конфигурационная сущность; запрещены скрытые зависимости, условная
доступность параметров одного механизма от другого, автоматическая
синхронизация значений между механизмами.

**Candidate Architecture Review:**

| Candidate | Type | Status |
|---|---|---|
| Signal Confirmation (Initial Entry) | Native Level 1 (Enable + Method/Count + Delay + Filter Agreement Mode + Container) | ✅ APPROVED |
| Multi-Timeframe Confirmation | Native Level 1 (Enable + Confirmation Timeframe) | ✅ APPROVED |
| Confirmation Count (forward, Filter) | Параметр внутри Signal Confirmation | ✅ APPROVED — поглощён, не отдельное поле |
| Signal Reconfirmation (forward, Position) | — | ✅ RESOLVED — не отдельная сущность, поглощён Re-entry Confirmation |
| Conditional Re-entry Restrictions (forward, Position) | — | ✅ RESOLVED — параметры Re-entry Confirmation, не отдельное поле |
| Re-entry Logic (forward, Position) | Candidate → `Re-entry Confirmation` | ✅ **PROVISIONALLY APPROVED** |

**Re-entry Confirmation — PROVISIONALLY APPROVED.** Рассматривается как
самостоятельный механизм (Вариант А), поскольку отвечает на другой вопрос
принятия решения, чем Initial Entry Confirmation («разрешено ли
открыть новую позицию» vs «разрешено ли повторное торговое действие после
уже существующей истории позиции») — тот же критерий, которым ранее были
разведены Risk↔Position и Filter↔Signal. Field Architecture не
фиксируется до проектирования отдельного **Re-entry Confirmation
Mechanism Contract** — по аналогии с Position Size Method Contract.

**Negative Candidate Review:**

| Candidate | Status | Причина |
|---|---|---|
| Signal Mode (Conservative/Balanced/Aggressive) | ❌ REJECTED | Нарушение Principle 3 (AUTO PARAMETERS) / Manual Parameters Principle |
| Signal Profile / Signal Preset | ❌ REJECTED | Массовое автоматическое изменение параметров |
| Auto Signal Confirmation / Adaptive Confirmation Count | ❌ REJECTED | Нарушение Principle 3 — AUTO PARAMETERS permanently prohibited |
| Signal Strength / Confidence Score | ❌ REJECTED | Вычисляемое значение — SIGNAL LAYER, нарушает Configuration vs Runtime Rule |
| Universal Signal Mode | ❌ REJECTED | Скрытая синхронизация нескольких независимых механизмов, нарушает Manual Parameters Principle |
| Master Signal Enable | ❌ REJECTED | Дублирует ответственность существующих Enable-флагов, нарушает Internal Signal Mechanism Independence Rule и Minimal Field Count Principle |
| Signal Priority / Signal Weight | ❌ REJECTED | Логика принятия решения — DECISION/SIGNAL LAYER, нарушает Configuration vs Runtime Rule |

**Field Architecture (summary):**

- **Signal Confirmation** (Initial Entry) — `Enable Signal Confirmation`
  (`bool`) + `Confirmation Count` (`int`) + `Confirmation Delay`
  (parameter) + `Filter Agreement Mode` (`string`) + `Confirmation Data
  Source` (Container).
- **Multi-Timeframe Confirmation** — `Enable Multi-Timeframe Confirmation`
  (`bool`) + `Confirmation Timeframe` (timeframe).
- **Re-entry Confirmation** — `PROVISIONALLY APPROVED`. Field Architecture
  определяется отдельно при проектировании Re-entry Confirmation
  Mechanism Contract.

**Architecture Compliance Review:**

| Критерий | Статус |
|---|---|
| Frozen Signal Architecture (Level 1) | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle (Signal↔Filter/Risk/Position/Execution) | ✅ |
| Internal Signal Mechanism Independence Rule (Level 1, applied) | ✅ |
| Boundary Rule: Filter vs Signal (Level 1, applied) | ✅ |
| Configuration vs Runtime Rule | ✅ |
| Minimal Field Count Principle | ✅ |

\* Полная симметричная проверка — после проектирования SHORT SETTINGS.

**Status: LONG SIGNAL SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

**13. Level 2 — Long Visual Settings**

**Visual Surface Inventory (выполнен перед Boundary Definition):**

KNOWN Surface:

| Механизм | Surface |
|---|---|
| Stop Loss / Take Profit / Break Even / Trailing Stop | Ценовой уровень |
| Partial Take Profit | Ценовой уровень |
| Pyramid Entries / Scale In / Scale Out | Маркер исполнения |
| Signal Confirmation (Initial Entry) | Маркер сигнала |
| LONG Dashboard | Нативная панель Visual |
| Trend / Volatility / Momentum / Market / Confirmation Filter — Decision Surface | Статус-индикатор (Passed/Failed/Neutral/Disabled), инвариантен относительно алгоритма — Filter Settings хранит вход, DECISION LAYER всегда производит единый результат (Level 1, п.2.2) |

UNKNOWN — Chart Representation / Surface, требует Algorithm/Mechanism
Contract или отдельного Candidate Review:

| Механизм | Причина |
|---|---|
| Filter — Chart Representation (overlay) | Algorithm-dependent геометрия, не инвариантна относительно алгоритма фильтра |
| Position Size | Неясно, самостоятельная ли это визуальная сущность |
| Multi-Timeframe Confirmation | Неясно, отдельный объект или атрибут Signal-маркера |
| Re-entry Confirmation | DEFERRED до Re-entry Confirmation Mechanism Contract (Signal) |

**Boundary Definition:** Long Visual Settings отвечает на вопрос «как
визуально представлен уже утверждённый элемент LONG-системы, имеющий
определённую Visual Surface». Сама логику вычисления/генерации
отображаемых данных группа не выполняет (VISUAL ENGINE / PLOTS ENGINE);
она конфигурирует только декларативные параметры представления —
включён ли элемент, цвет, стиль, размер, прозрачность, позиция.

**Не отвечает за:** саму отрисовку и вычисление отображаемых значений
(VISUAL ENGINE / PLOTS ENGINE); функциональная ответственность источника
элемента полностью остаётся за соответствующей группой (Filter, Risk,
Position, Signal) — Visual Settings владеет исключительно параметрами
представления.

**Visual Ownership Rule (Level 1, п.2.6 — применено):** любая настройка,
влияющая исключительно на внешний вид элемента, принадлежит Visual
независимо от происхождения элемента; ни одна другая группа не может
содержать визуальные поля.

**Internal Visual Element Independence Rule (Level 1, п.2.6 —
применено):** каждый визуальный элемент — независимая конфигурационная
сущность, без скрытых зависимостей между элементами; комбинирование —
только в VISUAL ENGINE.

**Visual Element Independence Validation Rule (Level 2, новое правило —
введено при проектировании Long Visual Settings):** визуальная сущность
считается самостоятельным кандидатом только если она может быть
независимо включена или отключена без изменения архитектурного владельца
представления. Если визуальный объект — составная часть другого
утверждённого визуального элемента и не обладает собственной независимой
визуальной жизнью, он не образует отдельного кандидата и поглощается
владельцем.

**Candidate Architecture Review:**

| Candidate | Source | Status |
|---|---|---|
| Enable Long Dashboard | Native (2.6) | ✅ APPROVED |
| Filter Status Row (×5, Decision Surface) | Filter | ✅ RESOLVED — часть LONG Dashboard, не отдельный кандидат (Visual Element Independence Validation Rule: не существует как отдельная визуальная сущность без Dashboard) |
| Enable Long Stop Loss Line + Color/Style | Risk | ✅ APPROVED |
| Enable Long Take Profit Line + Color/Style | Risk | ✅ APPROVED |
| Enable Long Break Even Line + Color/Style | Risk | ✅ APPROVED |
| Enable Long Trailing Stop Line + Color/Style | Risk | ✅ APPROVED |
| Enable Long Partial Take Profit Line + Color/Style | Position | ✅ APPROVED |
| Enable Long Pyramid Entry Marker + Style/Size | Position | ✅ APPROVED |
| Enable Long Scale In Marker + Style/Size | Position | ✅ APPROVED |
| Enable Long Scale Out Marker + Style/Size | Position | ✅ APPROVED |
| Enable Long Signal Marker + Style/Size | Signal | ✅ APPROVED |

**Negative Candidate Review:**

| Candidate | Status | Причина |
|---|---|---|
| Master Visual Enable / Show All Elements | ❌ REJECTED | Дублирует индивидуальные Enable-флаги, нарушает Internal Visual Element Independence Rule и Minimal Field Count Principle |
| Visual Mode (Minimal/Standard/Full) | ❌ REJECTED | Одновременно меняет набор отображаемых элементов, нарушает Manual Parameters Principle |
| Universal Visual Style | ❌ REJECTED | Глобальная стилизация нескольких независимых визуальных элементов одновременно, нарушает Internal Visual Element Independence Rule и Manual Parameters Principle |
| Visual Profile / Visual Preset | ❌ REJECTED | Массовое автоматическое изменение параметров |
| Auto Color Adjustment / Adaptive Visual Styling | ❌ REJECTED | Нарушение Principle 3 — AUTO PARAMETERS permanently prohibited |
| Chart Theme (Dark/Light) | ❌ REJECTED | Свойство платформы TradingView, вне архитектурной юрисдикции проекта |

**Field Architecture (summary):**

- **Enable Long Dashboard** (`bool`) — включает панель; внутренний состав
  (включая Filter Status Row) — реализация VISUAL ENGINE, не отдельные
  Long Visual Settings поля.
- **Stop Loss / Take Profit / Break Even / Trailing Stop Line** (каждый —
  полностью самодостаточный элемент) — `Enable` (`bool`) + `Color` +
  `Style`.
- **Partial Take Profit Line** — `Enable` (`bool`) + `Color` + `Style`.
- **Pyramid Entry / Scale In / Scale Out Marker** (каждый — полностью
  самодостаточный элемент) — `Enable` (`bool`) + `Style` + `Size`.
- **Signal Marker** — `Enable` (`bool`) + `Style` + `Size`.

**Architecture Compliance Review:**

| Критерий | Статус |
|---|---|
| Frozen Visual Architecture (Level 1) | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle (Visual↔Filter/Risk/Position/Signal) | ✅ |
| Internal Visual Element Independence Rule (Level 1, applied) | ✅ |
| Visual Ownership Rule (Level 1, applied) | ✅ |
| Visual Element Independence Validation Rule (Level 2, applied) | ✅ |
| Minimal Field Count Principle | ✅ |

\* Полная симметричная проверка — после проектирования SHORT SETTINGS.

**Status: LONG VISUAL SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

**14. Level 2 — Long Alert Settings**

**Alert Event Source Inventory (выполнен перед Boundary Definition):**

| Source | Event | Status |
|---|---|---|
| Signal — Signal Confirmation (Initial Entry) | Entry Executed | ✅ KNOWN EVENT SOURCE |
| Signal — Re-entry Confirmation | Re-entry/Additional Entry Executed | ⏸ DEFERRED — pending Re-entry Confirmation Mechanism Contract |
| Risk — Stop Loss / Take Profit / Break Even / Trailing Stop | Level Triggered | ✅ KNOWN EVENT SOURCE |
| Risk — Risk-Based Partial Exit | Partial Exit by risk trigger | ⏸ DEFERRED — сам механизм ещё не утверждён |
| Position — Partial Take Profit | Partial Close Executed | ✅ KNOWN EVENT SOURCE |
| Position — Pyramid Entries | Volume Added (Pyramid) | ✅ KNOWN EVENT SOURCE |
| Position — Scale In | Volume Added (Scale In) | ✅ KNOWN EVENT SOURCE |
| Position — Scale Out | Volume Reduced | ✅ KNOWN EVENT SOURCE |
| Filter (все 5) | — | ❌ NON-EVENT SOURCE / OUT OF ALERT BOUNDARY — condition, не action |
| Visual | — | ❌ NON-EVENT SOURCE — presentation, не событие |
| General | — | ❌ NON-EVENT SOURCE — 0 approved fields |

**Boundary Definition:** Long Alert Settings отвечает на вопрос «как
пользователь уведомляется (off-chart) о произошедшем событии, источником
которого является другая утверждённая группа/слой». Условие срабатывания
события Alert не хранит и не дублирует — оно принадлежит группе-источнику.
Саму генерацию и отправку alert-события Alert не выполняет
(EXECUTION/INTERFACE LAYER).

**Boundary Rule: Alert vs остальные группы (Level 1, п.2.7 — применено):**
Alert хранит только «включён ли алерт» и «как оформлено уведомление», не
условие срабатывания.

**Visual ↔ Alert (Boundary clarification, не новое правило Level 2):**
Visual владеет только on-chart presentation; Alert владеет только
off-chart notification presentation — разделение по каналу, не по факту
«это представление». Уже подразумевалось независимыми Purpose `2.6`/`2.7`,
здесь впервые сделано явным.

**Internal Alert Type Independence Rule (Level 1, п.2.7 — применено):**
каждый тип алерта — независимая конфигурационная сущность, без скрытых
зависимостей между типами алертов; Independent Mechanism → Independent
Runtime Event → Independent Alert Type (прямое следствие Internal
Position Mechanism Independence Rule для Pyramid Entries/Scale In, не
отдельное решение).

**Candidate Architecture Review:**

| Candidate | Source Event | Status |
|---|---|---|
| Enable Long Entry Alert | Signal — Entry Executed | ✅ APPROVED |
| Enable Long Stop Loss Alert | Risk — Stop Loss Triggered | ✅ APPROVED |
| Enable Long Take Profit Alert | Risk — Take Profit Triggered | ✅ APPROVED |
| Enable Long Break Even Alert | Risk — Break Even Triggered | ✅ APPROVED |
| Enable Long Trailing Stop Alert | Risk — Trailing Stop Triggered | ✅ APPROVED |
| Enable Long Partial Take Profit Alert | Position — Partial Close Executed | ✅ APPROVED |
| Enable Long Pyramid Entry Alert | Position — Volume Added (Pyramid) | ✅ APPROVED |
| Enable Long Scale In Alert | Position — Volume Added (Scale In) | ✅ APPROVED |
| Enable Long Scale Out Alert | Position — Volume Reduced | ✅ APPROVED |

**Negative Candidate Review:**

| Candidate | Status | Причина |
|---|---|---|
| Master Alert Enable / Enable All Alerts | ❌ REJECTED | Дублирует индивидуальные Enable-флаги, нарушает Internal Alert Type Independence Rule и Minimal Field Count Principle |
| Alert Mode / Alert Profile / Alert Preset | ❌ REJECTED | Массовое автоматическое изменение параметров — нарушение Principle 3 (AUTO PARAMETERS) |
| Universal Alert Template / Universal Message Format | ❌ REJECTED | Глобальный формат для нескольких независимых типов алертов одновременно, нарушает Internal Alert Type Independence Rule |
| Adaptive Alert Frequency / Auto Alert Throttling | ❌ REJECTED | Нарушение Principle 3 — AUTO PARAMETERS permanently prohibited |
| Alert Priority / Alert Severity Score | ❌ REJECTED | Вычисляемое значение — EXECUTION/INTERFACE LAYER, нарушает Configuration vs Runtime Rule |
| Alert Delivery Channel (Email/SMS/Push/Webhook) | ❌ REJECTED | Свойство платформы TradingView, вне архитектурной юрисдикции проекта |
| Conditional Alert (алерт только при доп. условии) | ❌ REJECTED | Потребовало бы хранения условия срабатывания, нарушение Boundary Rule: Alert vs остальные группы |

**Field Architecture (summary):**

Каждый тип алерта — полностью самодостаточная сущность (`Internal Alert
Type Independence Rule`): `Enable Long [Event] Alert` (`bool`) + `Alert
Message Template` (`string`), девять раз — по одному на каждый KNOWN
EVENT SOURCE из Candidate Architecture Review.

**Architecture Compliance Review:**

| Критерий | Статус |
|---|---|
| Frozen Alert Architecture (Level 1) | ✅ |
| LONG/SHORT Independence | ✅* |
| Manual Parameters Principle | ✅ |
| Module Responsibility Principle (Alert↔Signal/Risk/Position) | ✅ |
| Internal Alert Type Independence Rule (Level 1, applied) | ✅ |
| Boundary Rule: Alert vs остальные группы (Level 1, applied) | ✅ |
| Visual ↔ Alert Boundary Clarification (applied) | ✅ |
| Configuration vs Runtime Rule | ✅ |
| Minimal Field Count Principle | ✅ |

\* Полная симметричная проверка — после проектирования SHORT SETTINGS.

**Status: LONG ALERT SETTINGS — ARCHITECTURALLY COMPLETE (for current
Design level)**

---

**LONG SETTINGS — Level 2 Design: COMPLETE (7 / 7 groups).** Эволюция
методологии по группам: General → Candidate Review; Filter →
Pattern/Container Analysis; Risk → Boundary/Object Analysis; Position →
Mechanism Analysis; Signal → Forward Reference Review; Visual → Visual
Surface Inventory; Alert → Alert Event Source Inventory. Каждая группа
получила собственную процедуру первого шага, определяемую её предметной
областью, без искусственного навязывания единого шаблона. Следующий этап
жизненного цикла модуля LONG SETTINGS — Implementation, когда это будет
предусмотрено дорожной картой.

### Implementation
Ожидает завершения Level 2 Design. Файл не создан.

### Testing
Ожидает Implementation.

### Optimization
Ожидает Testing.

### Freeze
Ожидает Optimization.

---

## Development Log Entries (Verification Records)

Отдельная категория записей, не привязанных к конкретной стадии Module
Lifecycle. Фиксирует факт независимой ревизии и связанное с ней
инженерное решение — не Documentation Synchronization Change (документация
не приводится в соответствие с кодом) и не архитектурное изменение
(контракт модуля не меняется). Нумерация: `LOG-00001`, `LOG-00002`, ...
последовательно, без пропусков.

### LOG-00001 — SYSTEM CORE Baseline Verification

**Type:** Development Log Entry (Verification)
**Module:** SYSTEM CORE
**Version:** v1.0.0-core
**Date:** 2026-07-27

**Summary**
Completed an independent static architectural review of
`SmartTrendProEngine_v1_SystemCore_12.pine`. The review confirmed that the
implementation is consistent with the frozen architectural specification
(`v1.0.0-core`) and the corresponding Development Log decisions.

**Verification Results**
- SYSTEM CORE structure matches the frozen contract.
- All 23 UDT fields and 8 logical groups are present.
- `populateSystemCore()` matches the approved implementation.
- QA Harness artifacts are absent.
- Compiler placeholder uses the approved implementation:
  `display.data_window + display.status_line`.
- Header documentation is synchronized with the frozen project status.
- No architectural or structural discrepancies were identified during
  static review.

**Scope**
This verification confirms architectural conformity only. Runtime
behaviour, TradingView compilation, and execution characteristics remain
outside the scope of this review.

**Decision**
`SmartTrendProEngine_v1_SystemCore_12.pine` is accepted as the baseline
implementation of SYSTEM CORE for subsequent implementation of CORE LAYER
modules (`LONG SETTINGS` and `SHORT SETTINGS`).

---

## Amendment Procedure (для справки)

Если после заморозки модуля в последующих слоях обнаружится нехватка
системного состояния, модуль временно выводится из Freeze через отдельную
запись в этом логе с пометкой `[AMENDMENT]`, указанием причины и ссылкой на
модуль, который выявил нехватку. Обычное расширение функциональности
внутри уже существующих групп полей разрешено и Amendment не требует.

---

## Change Classification for Frozen Modules (для справки)

| Тип изменения | Amendment Procedure | Изменение версии | Изменение статуса Freeze |
|---|---|---|---|
| Functional Change | Да | Да | Возможен пересмотр Freeze |
| Architectural Contract Change | Да | Да | Да |
| Documentation Synchronization Change | Нет | Нет | Нет |

Documentation Synchronization Change применяется, когда правка затрагивает
только комментарии/аннотации в production-файле (устаревшая терминология
слоёв, уточнение описаний, снятие двусмысленных формулировок) и не меняет
UDT, логику, поведение, inputs или контракт модуля. Не требует Amendment
Procedure, не меняет версию и статус Freeze.

---

## Project Freeze Status

| Модуль | Статус |
|---|---|
| SYSTEM CORE | **FROZEN (v1.0.0-core)** — 2026-07-21 |
| LONG SETTINGS | **DESIGN** (Level 1 complete, Level 2 DESIGN COMPLETE — 7/7 groups) — 2026-07-21 |
| SHORT SETTINGS | Not started |
| LONG FILTER SETTINGS | Not started |
| SHORT FILTER SETTINGS | Not started |