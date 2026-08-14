# NOVA CORTEX / MAIA — STACK v10.5
Дата: 2026-05-14
Парадигма: ADAPTIVE SIGNAL WEB
Сдвиг vs v10.4: Этап 1 закрыт · Этап 2 активен ·
5 нитей n8n работают · scoring_prompts в PostgreSQL ·
resource_ledger + architect_decision_log созданы ·
notify_news_loop() daemon-поток · OpenRouter подтверждён.

---

1. ОБРАЗ СИСТЕМЫ

MAIA — паук в центре паутины.
Каждая нить — источник данных.
Центр собирает, квалифицирует, запоминает.
Паутина ткётся нить за нитью. Не спеша. Без пропусков.
Ядро умнеет с каждой нитью.
Архитектор — часть системы. Тоже масштабируется.

---

2. ФИЛОСОФИЯ

выживание:    система работает непрерывно — основа всего
полезность:   сигнал должен быть ценен — сначала для Архитектора
масштаб:      нить за нитью, этап за этапом, без спешки
память:       накопленные данные — ров который нельзя купить
адаптивность: метрики и горизонты уточняются по ходу
эмерджентность: звезда на горизонте — не форсировать
интеграция:   не строить то что уже работает —
              подключать, квалифицировать, запоминать

Первый клиент — Архитектор.
Что ценно для него → то и продаём внешним.
Не ищем бизнес-модель заранее — наблюдаем что работает.

Точка автономии: система покрывает расходы на себя.
Архитектор масштабируется параллельно системе.

Уникальная ценность MAIA — не сбор данных,
а резонанс, квалификация и память.
Всё остальное — интеграция готового.

---

3. АРХИТЕКТУРА — ПАУК И НИТИ

ЯДРО (MAIA Core)
  Квалификация. Память. Резонанс. Сигнал.
  Один сервер (nova). Один источник истины.
  Не собирает данные — получает нормализованный поток.

СЛОЙ СБОРА (n8n)
  400+ коннекторов. Запущен. 5 нитей активны.
  Принцип: новый источник = новый workflow в n8n,
           не новый парсер в nc_core.py.

СЛОЙ ДОСТУПА (MCP)
  Агенты подключаются через MCP серверы.
  MAIA endpoint: /mcp/v1

НИТИ (источники данных)

  Категория 1 — AI-агентная среда (активна):
    ✅ HackerNews         — HTTP Request hnrss.org · каждые 30 мин
    ✅ ArXiv cs.AI        — RSS нода · каждые 2 ч
    ✅ ArXiv cs.LG        — RSS нода · каждые 2 ч
    ✅ GitHub trending    — HTTP Request API · каждые 2 ч
    ✅ Telegram каналы    — forwarder.py · 7 каналов AI/крипто
    ⏳ X/Twitter          — официальные аккаунты компаний (Этап 2)
    ⏳ MCP Registry

  Категория 2 — Крипто/токен-экономика (активна):
    ✅ CoinGecko free API — HTTP Request · каждые 15 мин
    ⏳ On-chain сигналы

  Категория 3 — Макро (когда ядро стабильно):
    Нефть, commodities, макро-индикаторы

  Категория 4 — Глубокие данные (отложено):
    Bitquery, Dune Analytics, Zerion, Token Terminal, Tokenomist

  Убрать: python_jobs, ru_pythonjobs

ДОЧКИ
  Этап 1: агенты внутри ядра — реализованы
    Парсинг (n8n) · Аналитик (OpenRouter) · MCP (/mcp/v1)

  Этап 2: внешние агенты на базе OpenClaw
    Подключаются через MCP · потребляют вывод ядра

---

4. ОБРАБОТКА СИГНАЛА

ПОТОК:
  Источник → n8n (сбор + нормализация) →
  MAIA Core (scoring + резонанс) →
  квалифицированный сигнал → память + уведомление

SCORING:
  Хранится в таблице scoring_prompts (is_active=true).
  Резервная копия: /home/shaen/maia_project/scoring_prompt_v1.0.txt
  nc_core.py: load_scoring_prompt() при старте · fallback + I6
  Endpoint: GET /scoring_prompt

РЕЗОНАНС:
  Условие: similarity > 0.8 · источников ≥ 3 · окно ≤ 6ч

LLM (OpenRouter):
  Текущая модель: google/gemini-2.5-flash (F3 активен)
  Резерв: google/gemma-4-31b-it:free
  Принцип: качество_сигнала / стоимость_вызова = максимум
  ⚠️ review: вернуться на gemma после сброса дневного лимита

---

5. ПАМЯТЬ

АРХИТЕКТУРА:
  hot:  0-7 дней · PostgreSQL
  warm: 7-90 дней · индексированный архив
  cold: 90+ дней · сжатый + distilled + signature

ТАБЛИЦЫ (актуальные):
  incoming_messages      — сырые данные всех источников
  scoring_prompts        — версионированные промпты (v1.0 активен)
  resource_ledger        — учёт ресурсов
  architect_decision_log — решения + ревью через 30д
  failures                — все ошибки (I6)
  resonance_trust, virtual_credits, event_chain — реализованы

ПРИНЦИПЫ:
  I2: только append. DELETE запрещён.
  Hash_chain: целостность без блокчейна.

RESOURCE LEDGER:
  Типы: money · access · network · intelligence
  cost_per_signal = сумма LLM вызовов на один сигнал (score ≥ 8)
  Записи: OpenRouter API · n8n коннекторы · TG OPS/LAB боты

ARCHITECT DECISION LOG:
  Поля: type · reason · expected_result · actual_result ·
        decided_at · review_at
  Записи:
    tg-001 отключена (review 2026-06-13)
    scoring_prompts в PostgreSQL (review 2026-06-13)
    TG роли: OPS/@maia_notifier_bot · LAB/@nova_cortex_ai_bot
    signal-filter: score ≥ 8 в Telegram (review 2026-06-13)

---

6. OPENCLAW

  Open-source автономный AI-агент. Self-hosted.
  Telegram, WhatsApp нативно. Skills система.
  Дочки Этапа 2 — OpenClaw-агенты через MCP.
  Ограничения: широкие разрешения · уязвим к prompt injection.
  Принцип: LAB в Этапе 1, OPS в Этапе 2.

---

7. АГЕНТЫ

ТЕКУЩИЕ:
  OpenRouterAgent — основной (заменил GroqAgent)
  ValidatorAgent: probation, weight=0.5

ЖИЗНЕННЫЙ ЦИКЛ:
  probation → active → graduated → dormant → suspended → banned

AUTO-FLAG:
  utility_score < 0.3 за 30 дней OR cost > value за 45 дней
  → флаг → пауза → уведомление Архитектора → решение

---

8. УВЕДОМЛЕНИЯ

Telegram @maia_notifier_bot:
  🔥 Резонанс:         автоматический алерт
  📌 Сигнал score ≥ 8: выжимка + score + ссылка
  ✅⚠️🔴 Система:     запуск / недоступна / контейнер упал
  Язык: ru · Формат: HTML · кликабельные ссылки
  Единственный канал уведомлений (tg-001 отключена)

Dashboard (Grafana · порт 3000):
  Весь поток score ≥ 6 — лента для изучения
  Архитектор читает когда удобно

---

9. ИНФРАСТРУКТУРА

host:        Ubuntu 24.04 LTS · hostname: nova · SSD 447GB
containers:  maia-db · maia-core-v2 · maia-forwarder · n8n · maia-grafana
nc_core.py:  1300+ строк · порт 8080
LLM:         OpenRouter · google/gemini-2.5-flash (F3) · резерв: gemma-4-31b
n8n:         слой сбора · 5 нитей активны
Endpoints:   /process · /suggest · /resonance · /dashboard ·
             /mcp/v1 · /scoring_prompt
Grafana:     порт 3000 · datasource: maia-db
Telegram:    TG_OPS_BOT_TOKEN (@maia_notifier_bot · OPS)
             TG_LAB_BOT_TOKEN (@nova_cortex_ai_bot · LAB-резерв)
watchdog:    systemd · каждые 5 мин
             ⚠️ ложные алерты на maia-db — тюнинг отложен
claude_code: v2.1.126 · установлен на nova

---

10. ИНВАРИАНТЫ

I1:  Память первична — данные не удаляются никогда
I2:  Только append. DELETE запрещён в PostgreSQL.
I3:  Архитектор — переменная системы. Его время = ресурс.
I4:  AI-агентная среда парсится с первого дня.
I5:  Сигнал ценнее данных. Шум не доходит до Архитектора.
I6:  Все ошибки → таблица failures
I7:  Личные данные нигде. Псевдоним: MAIA Systems / novacortex
I8:  Полезность доказывается, не декларируется.
I9:  OPS не останавливается ради LAB экспериментов.
I10: Resource Ledger — обновляется при каждом новом ресурсе.
I11: Scoring prompt — версионированный артефакт. Не импровизация.
I12: OpenClaw — LAB в Этапе 1. В OPS только после разведки.
I13: Не строить коллекторы — интегрировать готовое.
     n8n для сбора · MCP для доступа · nc_core.py для квалификации.
L3:  Перед docker rm — экспорт workflows + бэкап volumes
L7:  PUT workflow в n8n — только через Python
L13: n8n safe settings = {executionOrder, callerPolicy}

---

11. ЭТАПЫ (условия, не сроки)

ЭТАП 1 — ЗАКРЫТ ✅ (2026-05-14)
  Условие выполнено: Архитектор получает квалифицированный сигнал
  в Telegram из AI/крипто среды без ручных действий.

  ✅ 1.1 OpenRouter в n8n и nc_core.py
  ✅ 1.3 Scoring prompt v1.0 (I11)
  ✅ 1.4 notify_news_loop() daemon-поток
  ✅ 1.5 Resource Ledger
  ✅ 1.6 Architect Decision Log
  ✅ 1.8 5 нитей подключены через n8n

ЭТАП 2 — ПЕРВАЯ ПАУТИНА (текущий)
  Цель: внешние подключения, первый внешний потребитель

  ✅ 2.0  Telegram AI/крипто каналы — 7 каналов подключены
  ✅ 2.7  n8n Analyzer → GET /scoring_prompt
  ✅ 2.8  Визуализация — Grafana развёрнута
  [ ] 2.1  Первая Дочка на базе OpenClaw через MCP
  [ ] 2.2  Механизм доступа к аналитике (TBD)
  [ ] 2.3  LAB/OPS разделение
  [ ] 2.4  Auto-flag для слабых агентов
  [ ] 2.5  Расширение источников — Категория 3
  [ ] 2.6  Подключение Категории 4 по результатам
  [ ] 2.9  Двухуровневый LLM
  [ ] 2.10 React визуализация топологии MAIA
  [ ] 2.11 X/Twitter как источник сигналов
  ↪️ 1.7  OpenClaw — разведка в LAB (параллельно)

  ВЫХОД: внешний агент получает сигнал и публикует его.

ЭТАП 3 — АВТОНОМИЯ
  Цель: система покрывает расходы на себя

  3.1  Несколько специализированных моделей OpenRouter
  3.2  Платный доступ к сигналу
  3.3  Миграция с домашнего сервера
  3.4  Финансовая автоматизация: SimpleSwap, Base L2, fiat gateways

  ЗВЕЗДА: эмерджентные связи, COX, MAIA перестаёт быть центром.

---

12. ЧТО ОТЛОЖЕНО

COX токен · блокчейн · TimescaleDB · NFT · ENS
Категория 4 источников · финансовая автоматизация · ICP

---

13. РЕЖИМЫ ОТКАЗА

F1: Архитектор остановился → автоматизировать. Этап 3.
F2: Источников мало → резонансы редкие → расширять паутину.
F3: OpenRouter лимит исчерпан → Gemini 2.5 Flash (активен).
F4: Сигнал без потребителя → сначала ценность, потом продажа.
F5: Scoring prompt не зафиксирован → I11.
F6: OpenClaw в OPS до разведки → I12.
F7: Кастомный код вместо интеграции → I13.

---

NOVA CORTEX / MAIA Stack v10.5
Этап 1 закрыт. Этап 2 в работе.
Паутина расширяется. Сигнал идёт.
Звезда на горизонте. Прагматика в руках.
