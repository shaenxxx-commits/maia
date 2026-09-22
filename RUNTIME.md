# MAIA — RUNTIME
Обновлён: 2026-09-22
Обновляет: КОНЦЕПТ после аудита / УЧ после значимой сессии

⚠️ Полная картина расхождений документ/реальность за 2026-07 →
AUDIT_2026-07.md (историческая точка, не отражает состояние после
22.09). Этот документ отражает состояние после цикла восстановления
скоринга (09.09-14.09.2026), вектора "Обнуление" (19.09.2026),
замены S2→OpenAlex и первой реализации entity-слоя (21-22.09.2026),
диагностики n8n schedule-trigger и watchdog-механизма (21-22.09.2026).

---

## ИНФРАСТРУКТУРА

Host:       Ubuntu 24.04 LTS · hostname: nova · SSD 447GB
CPU:        12 ядер
RAM:        15GB total · ~9.5GB доступно
Dual boot:  Windows/Ubuntu — ПК регулярно переключается между ОС,
            рестарты хоста — рутинная, частая ситуация, не
            исключение.
Docker:     29.4.2
Claude Code: недоступен — Free план (см. DECISIONS.md: ИНСТРУМЕНТЫ)

✅ ИСПРАВЛЕНО (22.09.2026): прежняя формулировка "Restart policy =
'no' → требуется ручной cold start" была НЕТОЧНОЙ. Docker restart
policy у 4 основных контейнеров (maia-db, maia-core-v2,
maia-forwarder, n8n) действительно "no" — это подтверждено и
остаётся так. НО systemd-таймер maia-watchdog.timer запускает
maia-watchdog.service каждые 5 минут НЕЗАВИСИМО от того, что сам
.service формально помечен "disabled" (disabled означает только
"не стартует сам при загрузке хоста", не "не работает вообще" —
таймер вызывает его по расписанию отдельно).
При обнаружении упавшего контейнера watchdog.sh выполняет
`docker compose up -d --no-deps <name>` автоматически. Подтверждено
прямым логом (journalctl, 22.09.2026, рестарт хоста 09:17-09:18
UTC): watchdog поднял все 4 контейнера последовательно за ~43
секунды (maia-db→maia-core-v2→maia-forwarder→n8n). Реальное время
восстановления системы после рестарта хоста — обычно ~5-10 минут
(зависит от того, когда сработает ближайший тик 5-минутного
таймера + 180с grace period на cold start), не требует ручного
вмешательства Архитектора в подавляющем большинстве случаев.
Ручной cold start (команда ниже) остаётся полезен только если
нужно восстановление быстрее, чем эти 5-10 минут.
  cold start (при необходимости ускорить):
  docker start maia-db maia-core-v2 maia-forwarder n8n

## КОНТЕЙНЕРЫ

12 контейнеров.

maia-db        — PostgreSQL · порт 5432
maia-core-v2   — nc_core.py · порт 8080
maia-forwarder — forwarder.py · Telegram listener
n8n            — слой сбора · порт 5678

LibreChat        — НЕСТАБИЛЕН. Остановлен вручную 14.09.2026
                    (crash-loop, DNS NXDOMAIN на _mongodb._tcp.
                    cluster0.dhsvvox.mongodb.net, MongoDB Atlas).
                    ⚠️ Вновь замечен в статусе "Restarting"
                    22.09.2026 — не расследовано в этой сессии,
                    требует отдельного технического вектора.
                    Имя контейнера — "LibreChat" (с заглавной буквы).
vectordb         — LibreChat зависимость, не остановлен
rag_api          — LibreChat зависимость, не остановлен
chat-mongodb     — LibreChat зависимость, не остановлен
chat-meilisearch — LibreChat зависимость, не остановлен

Недокументированные (статус открыт, требуют решения Архитектора):
  maia-lab-sentinel — чистая заглушка (tail -f /dev/null,
                       Mounts: [], без healthcheck).
  nc_glances        — API реально живой (GET /api/4/cpu отвечает).
                       Назначение не установлено.
  portainer         — UI управления Docker. Exited (2) на момент
                       последней проверки (22.09.2026) — не
                       расследовано.

maia-grafana — удалена (см. DECISIONS.md, 2026-06-14).

## ПУТИ

project:    /home/shaen/maia_project/
sessions:   /home/shaen/maia/sessions/
nc_core:    /home/shaen/maia/sessions/nc_core.py
forwarder:  /home/shaen/maia/sessions/forwarder.py
docs:       /home/shaen/maia_project/docs/

⚠️ Реальные пути nc_core.py разошлись с тем, что указывал
AUDIT_2026-07.md — правки происходили в /home/shaen/maia/sessions/,
не в /home/shaen/maia_project/. Требует сверки при следующей
технической сессии.

## LLM (OpenRouter)

Primary:  nvidia/nemotron-3-super-120b-a12b:free
Fallback: nvidia/nemotron-nano-9b-v2:free

Оплата $10 подтверждена, лимит поднят (переход тира 11.09,
подтверждено данными аккаунта 14.09). Точный потолок нового тира
провайдером формально не установлен. До остановки realtime-потока
(22.09) канал уведомлений устойчиво доставлял до ~50 сообщений/день
на arXiv-only фокусе.

⚠️ Несведённая находка (не проверено в этой сессии): код
OpenRouterAgent в nc_core.py может по-прежнему указывать на
мёртвый slug gpt-oss-20b вместо решённого nemotron-3 — backlog
для следующей технической сессии.

N8N_API_KEY не используется nc_core.py / MAIA Core — релевантен
только для n8n Public API напрямую (см. раздел N8N ниже).

Мёртвые (не использовать): gpt-oss-120b:free, gpt-oss-20b:free,
gemma-4-26b-a4b:free, gemma-4-31b-it:free, gemini-2.5-flash,
llama-3.2-3b:free, mistral-7b:free

## АРХИТЕКТУРА СКОРИНГА

Реальная структура — 6 workflow: MAIA DeepSeek Analyzer v3.0
(центральный, вызывает OpenRouter НАПРЯМУЮ, сам пишет результат
в PostgreSQL) + 5 сборщиков по источникам (CoinGecko среди них
остановлен, см. раздел N8N ниже).

nc_core.py используется только как источник промпта
(GET /scoring_prompt). /process существует в коде, но не
используется этой схемой — см. DECISIONS.md: I13.

Скоринг пишется в поле status. Значения: pending_review,
irrelevant, relevant, processed.

⚠️ notify_news_loop() ОСТАНОВЛЕН (22.09.2026, обратимо) — см.
раздел УВЕДОМЛЕНИЯ ниже. Скоринг продолжает работать и писать в
БД, просто без realtime-рассылки в Telegram.

## N8N — СТАТУС ИСТОЧНИКОВ

Очередь "PG Read pending" — сортировка received_at DESC (с
12.09.2026). Приоритет источников (CASE arxiv/hackernews →
github_trending → остальное) работает поверх сортировки.

⚠️ ИЗВЕСТНЫЙ ВОСПРОИЗВОДИМЫЙ БАГ (найден 21-22.09.2026): hours-
интервальные Schedule Trigger (arxiv_cs_ai, arxiv_cs_lg,
github_trending — все hoursInterval=2) НЕ переустанавливают
таймер после первого успешного срабатывания при старте процесса
n8n — замолкают до явного рестарта контейнера n8n. Воспроизведено
дважды подряд на разных рестартах хоста (20.09, 21-22.09).
minutes-интервал (hackernews, 30 мин) этой проблеме не подвержен.
Естественно устраняется при каждом рестарте контейнера n8n,
который watchdog.sh выполняет автоматически при падении контейнера
(см. раздел ИНФРАСТРУКТУРА) — то есть проблема не требует
отдельного немедленного фикса, но может создавать окна тишины
до нескольких часов между падением и восстановлением конкретно
n8n, если сам n8n не падает физически (просто зависает по
таймеру), а только его schedule-логика.

Источники (статус на 22.09.2026):
  arxiv_cs_ai       — активен. Живые данные с 21.09.2026 14:00:22
                       (первый батч после восстановления schedule):
                       214 строк (90 irrelevant, 75 pending_review,
                       49 relevant).
  arxiv_cs_lg       — ⚠️ ПОЛНОСТЬЮ НЕФУНКЦИОНАЛЕН. За весь батч
                       21.09 — 0 строк ЛЮБОГО статуса (не только с
                       апострофами, как считалось раньше). SQL-баг
                       в узле "PG Insert" (конкатенация строк без
                       экранирования) — известная причина, но
                       охват шире, чем предполагалось при первой
                       находке (19.09). ПРИОРИТЕТНАЯ находка для
                       следующей технической сессии, требует
                       фикса на параметризованный запрос.
  github_trending   — активен, срабатывал синхронно с
                       arxiv-workflow (тот же hours-баг и то же
                       восстановление).
  hackernews        — МЁРТВ с 2026-06-12. Причина установлена:
                       workflow использует hnrss.org/newest?q=...
                       (поисковый эндпоинт), который возвращает
                       0 items с середины июня. Прямой RSS
                       (hnrss.org/newest?points=50) работает.
                       Решение (сменить URL/отключить) — не
                       принято.
  coingecko         — ОСТАНОВЛЕН (19.09.2026). Workflow "CoinGecko
                       Top-10 Volume" (id OQb6SyZXbseEwPVt) —
                       active=false. Часть закрытия крипто-ветки.
  cointelegraph     — ПРОДОЛЖАЕТ РАБОТАТЬ через forwarder.py
                       (Telegram), не через n8n-workflow. Крипто-
                       ветка закрыта только частично — forwarder-
                       путь выведен из периметра Шага 1, переносится
                       на Шаг 3 плана перестройки.

N8N_API_KEY — рабочий, используется для авторизации n8n Public
API напрямую. Ротирован 12.09.2026, новое значение только в .env.
Не используется nc_core.py/MAIA Core.

n8n Public API на этом инстансе: PATCH не поддерживается (405),
рабочий метод — PUT.

## ДРУГИЕ ИСТОЧНИКИ

Telegram forwarder — реально работают 2 из 7 каналов:
  cointelegraph — работает (crypto, см. выше — не остановлен)
  coingecko     — работает (Telegram-путь; n8n-путь того же
                   источника остановлен отдельно, см. раздел N8N)
  AnthropicAI   — НЕ работает. Канал сменил идентичность на
                   "Claude" (username anthropicai).
  LangChainAI, huggingface, whale_alert, coindesk — НЕ работают.
    Причина не установлена. Ошибок доступа в логах нет.
Решение Архитектора: не трогать до Шага 3 плана перестройки
(forwarder.py целиком переписывается в рамках entity/trajectory
модели).
Побочный баг: last_id в buffer_reader() не персистится (не
устранено, не блокирует работу).

## ЭНДПОИНТЫ (maia-core-v2)

/health, /upload, /files, /resonance, /scoring_prompt, /dashboard,
/mcp, /mcp/v1

⚠️ /health отдаёт "telegram":"active" НЕЗАВИСИМО от реального
статуса потока — вводит в заблуждение сейчас, когда
notify_news_loop() намеренно остановлен (22.09). Не исправлено,
backlog для следующей сессии.
⚠️ /process — существует, не вызывается n8n (I13).
⚠️ /challenge — ВЫРЕЗАН ЦЕЛИКОМ (2026-08-02).
/suggest — НЕ СУЩЕСТВУЕТ.
/mcp/v1 — полноценный рабочий JSON-RPC 2.0 MCP сервер.

## БАЗА ДАННЫХ

Host: maia-db:5432 · DB: maia_memory · User: maia_user

Реальная схема — 8 таблиц:
  incoming_messages — 0 строк на момент TRUNCATE (19.09.2026),
                       новый поток с 21.09.2026 (см. раздел N8N
                       для разбивки по источникам). Схема, индексы
                       (idx_unique_norm_text), триггеры —
                       сохранены. Sequence id не сброшена,
                       новые записи продолжают с id~31557+.
                       ⚠️ Несведённая находка: sequence
                       incoming_messages_id_seq наблюдался на
                       уровне ~311к (сильно выше ожидаемого) —
                       вероятная причина (отклонённые дубли жгут
                       nextval()) не подтверждена, не блокирует,
                       требует отдельной узкой проверки.
  scoring_prompts    — v1.0 активен, 1 строка (не тронута)
  resource_ledger    — 10 строк (была 9, +1: email-ресурс
                       maia.systems@proton.me, 21.09.2026, I10)
  architect_decision_log — 6 строк (не тронута)
  failures            — 4 строки. ⚠️ I6 нарушен для сборщиков:
                       ни обрыв schedule-cron, ни SQL-баг cs.LG
                       не попали в failures — ошибка пишется
                       только из workflow "MAIA DeepSeek Analyzer
                       v3.0", не из arxiv/hn/github-воркфлоу
                       напрямую.
  agents               — 3 строки (не тронута)
  event_chain           — 0 строк (не тронута)
  paper_entities         — НОВАЯ (создана 21.09.2026, Шаг 3 плана
                       перестройки). paper_id (PK), arxiv_id (TEXT,
                       мягкая связь + UNIQUE INDEX WHERE NOT NULL —
                       FK намеренно не вводился, БД проекта
                       архитектурно не имеет ни одного FK), title,
                       cited_by_count, venue, first_seen_at,
                       enriched_at. Живых записей — 0 на момент
                       22.09.2026: первая end-to-end запись
                       отложена объективно (статьи текущего потока
                       моложе минимального возраста индексации
                       OpenAlex, 7-14 дней) — ожидается
                       ~начало-середина октября 2026.

⚠️ Архивные данные, ранее хранившиеся в incoming_messages
(python_jobs 717 строк, freelancehunt 160 строк) — снесены вместе
со всей таблицей 19.09.2026, не сохранялись отдельно.

FK — ни одного, включая новую paper_entities (решение сохранено
намеренно). UNIQUE — agents(agent_id), failures(action_hash),
incoming_messages(md5(norm_text)), paper_entities(arxiv_id) WHERE
NOT NULL.

## УВЕДОМЛЕНИЯ

⚠️ ОСТАНОВЛЕНЫ (22.09.2026, обратимо). notify_news_loop() —
вызов запуска потока закомментирован в nc_core.py (не удаление
функции, не boolean-флаг), с явным комментарием даты и причины
в коде. Причина: Архитектор сконцентрирован на entity-слое (Шаг 3
плана перестройки), не на потоке сообщений. Скоринг продолжает
работать, просто без realtime-рассылки.

До остановки — Telegram @maia_notifier_bot, порог score ≥ 8,
подтверждён рабочим end-to-end (14.09.2026), устойчивый поток
до ~50 сообщений/день после перехода на arXiv-only (19.09.2026).

Замена — ежедневный дайджест (Шаг 2 плана перестройки) — ОТЛОЖЕНА
ЦЕЛИКОМ до готовности entity-слоя, не строится поверх текущей
архитектуры.

tg-001 ("Telegram Architect") — удалена окончательно из workflow
Analyzer (19.09.2026, подтверждено GET — 28 нод, было 29,
dangling refs = 0).

## SCORING

Prompt v1.0 активен в scoring_prompts.
Резерв: /home/shaen/maia_project/scoring_prompt_v1.0.txt

## ENTITY-СЛОЙ (Шаг 3 плана перестройки)

Enrichment-источник: OpenAlex (заменил Semantic Scholar,
21.09.2026 — см. DECISIONS.md). Без ключа/формы/одобрения:
100 000 запросов/день, 10 req/sec. mailto-параметр:
maia.systems@proton.me (.env: MAIA_EMAIL) — polite pool, не
барьер доступа.

Метод точного matching (подтверждён живыми тестами):
  GET /works?filter=doi:https://doi.org/10.48550/arxiv.{ID}
    &mailto={email}
Строгий формат обязателен: полный DOI URL, строчные arxiv. Bare
DOI и прямой /works/doi:{...} lookup — не работают.
Title-based поиск ПРИЗНАН НЕНАДЁЖНЫМ (проблема дублей/merge-
записей в OpenAlex) — matching только через DOI.

Таблица paper_entities — см. раздел БАЗА ДАННЫХ выше.

arxiv_id извлечение из живых сигналов: подтверждено на практике —
raw_text содержит паттерн "arXiv:{ID}v{N}" (пример:
"arXiv:2609.20974v1" → "2609.20974"), прямой текстовый разбор.

Статус: логика проверена технически по всем частям (extraction,
DOI, запрос к OpenAlex, синтаксис INSERT). Живых end-to-end
записей — 0, ожидается ~начало-середина октября 2026 (возраст
статей для индексации OpenAlex).

## WATCHDOG

Активен:   watchdog.sh · systemd-таймер (maia-watchdog.timer) ·
           каждые 5 мин · РЕАЛЬНО автоматически поднимает упавшие
           контейнеры (см. раздел ИНФРАСТРУКТУРА — правка неточной
           формулировки про "ручной cold start")
Отключён:  watchdog.py.disabled
Cooldown:  CONTAINER_ALERT_COOLDOWN=900 (15 мин между алертами на
           контейнер)
Grace:     180с после старта (uptime < 180s — skip, cold start grace)
STATE_DIR: /home/shaen/maia_project/.watchdog_state
⚠️ Telegram-уведомления watchdog не настроены (TELEGRAM_BOT_TOKEN/
CHAT_ID пусты в его контексте — "WARN: Telegram not configured,
skipping notify" в каждом срабатывании) — алерты о падении
контейнеров сейчас нигде не видны Архитектору, только в
watchdog.log. Не устранено, не приоритет.

## LIBRECHAT

Версия: v0.8.6-rc1 · Адрес: http://192.168.1.104:3080
MongoDB: Atlas M0 Free · Stockholm
⚠️ НЕСТАБИЛЕН — см. раздел КОНТЕЙНЕРЫ выше.

## GIT / .GITIGNORE

.gitignore создан (12.09.2026). Покрывает: .env и .env.backup*,
*.key, *.session, secrets/*, **/credentials*, backup_*.json/py,
wf_analyzer_*.json, .watchdog_state/, dump_*.sql, *.deprecated_*,
*.log, __pycache__/, *.pyc.

## КОД — nc_core.py

Объём: ~1064 строки (проверено 2026-08-02).
⚠️ Ранее указывалось "1300+ строк" — расхождение не расследовано.

Устранено (Шаг 4, 2026-08-02): entropy_protocol_loop вырезан,
GeminiAgent/DeepSeekAgent удалены, I7 токены/пароль вынесены,
patch_analyzer.py → .deprecated.
Устранено (22.09.2026): вызов notify_news_loop() закомментирован
(не удалён), см. раздел УВЕДОМЛЕНИЯ.

## КРИПТО-ВЕТКА (закрыта частично, статус на 22.09.2026)

Стратегическое решение (см. HANDOFF, DECISIONS.md: СМЕНА ФОКУСА):
единственный фокус проекта — AI research (arXiv, GitHub,
HackerNews/Lobsters, OpenAlex как enrichment). Крипто (CoinGecko,
Cointelegraph) закрывается.

  — incoming_messages: снесён весь backlog, включая крипто-
    контент (TRUNCATE 19.09, без дампа)
  — CoinGecko (n8n-путь): остановлен, workflow active=false
  — Cointelegraph (forwarder-путь): ПРОДОЛЖАЕТ РАБОТАТЬ — не
    входил в периметр Шага 1, переносится на Шаг 3 плана
    перестройки вместе с остальной переработкой forwarder.py
  — tg-001: расхождение документ/факт устранено

Не закрыто до Шага 3: полная остановка crypto-сбора (forwarder.py),
сам forwarder.py и .env/docker-compose переменные, специфичные
для Telegram-каналов.

## ИЗВЕСТНЫЕ ПРОБЛЕМЫ (актуализировано 2026-09-22)

⚠️ arxiv_cs_lg — ПОЛНОСТЬЮ НЕФУНКЦИОНАЛЕН (0 строк любого статуса),
   SQL-баг конкатенации в "PG Insert" — ПРИОРИТЕТ 1 для следующей
   технической сессии
⚠️ n8n hours-Schedule Trigger не самовосстанавливается после cold
   start процесса без явного рестарта контейнера — компенсируется
   watchdog-циклом автоматически, не блокер, но может создавать
   окна тишины
⚠️ Telegram forwarder — 5 из 7 каналов не работают, не трогать
   до Шага 3 плана перестройки
⚠️ HackerNews — мёртв, причина известна (hnrss.org поиск сломан),
   решение не принято
⚠️ I6 failures — схема INSERT не унифицирована; сборщики
   (arxiv/hn/github) вообще не пишут в failures напрямую
⚠️ /health отдаёт "telegram":"active" независимо от реального
   статуса потока — вводит в заблуждение, пока поток намеренно
   остановлен
⚠️ Возможный мёртвый slug OpenRouterAgent (gpt-oss-20b вместо
   nemotron-3) — не подтверждено/не исправлено в этой сессии
⚠️ maia-lab-sentinel/nc_glances/portainer — назначение неизвестно
⚠️ LibreChat — нестабилен, вновь в Restarting (22.09), не
   расследовано
⚠️ watchdog — Telegram-уведомления не настроены, алерты о падении
   контейнеров не видны Архитектору нигде кроме лога
⚠️ n8n restart policy = "no" на основных MAIA-контейнерах, НО
   watchdog.sh компенсирует автоматическим восстановлением за
   ~5-10 минут (см. раздел ИНФРАСТРУКТУРА — исправлено 22.09)
⚠️ n8n SQLite содержит ~4350 дублирующих execution-записей —
   не критично, требует очистки
⚠️ cointelegraph продолжает писать в incoming_messages до Шага 3
   плана перестройки — известный, принятый разрыв
⚠️ sequence incoming_messages_id_seq (~311к) — расхождение с
   ожидаемым, не расследовано, не блокирует
⚠️ Недокументированные узлы внутри Analyzer ("Arxiv Fetch"/
   "HN RSS Fetch" варианты — mcp_protocol/llm_agents/autonomous) —
   не описаны, не расследованы
⚠️ DDA Шаг 5/6 — открыты

ЗАКРЫТО:
  ✅ /challenge, entropy_protocol_loop, I7 хардкод, мёртвый код,
     4 фриланс-таблицы, python_jobs, docker-compose.yml
     невалидность
  ✅ Скоринг восстановлен (09.09), баги content/source исправлены
     (14.09), очередь DESC (12.09)
  ✅ Оплата OpenRouter подтверждена (11.09/14.09)
  ✅ tg-001 — расхождение документ/факт устранено, нода реально
     удалена (19.09)
  ✅ N8N_API_KEY ротирован (12.09), .gitignore создан (12.09)
  ✅ notify_news_loop() — подтверждён устойчивым end-to-end (19.09),
     затем осознанно остановлен (22.09, фокус на entity-слое)
  ✅ Вектор "Обнуление" — incoming_messages снесён, CoinGecko
     остановлен (19.09)
  ✅ S2 → OpenAlex — замена enrichment-источника, без блокеров
     доступа (21.09)
  ✅ Entity-слой — первая реализация: taблица paper_entities
     создана, метод matching подтверждён живыми данными (21-22.09)
  ✅ watchdog-механизм автозапуска — задокументирован верно,
     расхождение с прежней формулировкой RUNTIME.md устранено
     (22.09)
  ✅ n8n schedule-trigger баг — диагностирован полностью,
     воспроизведён дважды, причина установлена (22.09)
