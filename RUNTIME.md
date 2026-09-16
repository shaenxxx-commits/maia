# MAIA — RUNTIME
Обновлён: 2026-09-14
Обновляет: КОНЦЕПТ после аудита / УЧ после значимой сессии

⚠️ Полная картина расхождений документ/реальность → AUDIT_2026-07.md
Этот документ отражает состояние после цикла восстановления
скоринга (09.09-14.09.2026) и четырёх независимых read-only
диагностик 14.09.2026.

---

## ИНФРАСТРУКТУРА

Host:       Ubuntu 24.04 LTS · hostname: nova · SSD 447GB
CPU:        12 ядер
RAM:        15GB total · ~9.5GB доступно
Dual boot:  Windows (игры) — не перезагружать пока MAIA работает
            ⚠️ При переключении на Windows или выключении ПК
            Docker останавливается резко → watchdog шторм алертов
            cold start: docker start maia-db maia-core-v2 maia-forwarder n8n
Docker:     29.4.2
Claude Code: недоступен — Free план (см. DECISIONS.md: ИНСТРУМЕНТЫ)

⚠️ ИСПРАВЛЕНО (14.09.2026): "автостарт включён" — НЕ подтверждается
для основных MAIA-контейнеров. Restart policy = "no" для n8n,
maia-db, maia-core-v2, maia-forwarder (подтверждено технически).
Только maia-lab-sentinel имеет unless-stopped. После ребута хоста
контейнеры не поднимаются автоматически — требуется ручной
cold start (команда выше). systemd-unit maia-watchdog.service
существует, но disabled — не обеспечивает автозапуск.

## КОНТЕЙНЕРЫ

12 контейнеров.

maia-db        — PostgreSQL · порт 5432
maia-core-v2   — nc_core.py · порт 8080
maia-forwarder — forwarder.py · Telegram listener
n8n            — слой сбора · порт 5678

LibreChat        — ⚠️ ОСТАНОВЛЕН вручную (14.09.2026). Постоянный
                    crash-loop: DNS NXDOMAIN на
                    _mongodb._tcp.cluster0.dhsvvox.mongodb.net
                    (MongoDB Atlas). Реальное имя контейнера —
                    "LibreChat" (с заглавной буквы, не "librechat" —
                    исправлена опечатка в этом документе).
                    Восстановление: docker compose up -d LibreChat,
                    после отдельного решения по MongoDB Atlas DNS.
vectordb         — LibreChat зависимость, не остановлен
rag_api          — LibreChat зависимость, не остановлен.
                    Технически бесполезен без LibreChat — решение
                    об остановке не принято, оставлен как есть.
chat-mongodb     — LibreChat зависимость, не остановлен, та же
                    оговорка
chat-meilisearch — LibreChat зависимость, не остановлен, та же
                    оговорка

Недокументированные (статус открыт, требуют решения Архитектора):
  maia-lab-sentinel — подтверждена чистой заглушкой (tail -f
                       /dev/null, Mounts: [], без healthcheck).
  nc_glances        — API реально живой (GET /api/4/cpu отвечает).
                       Назначение не установлено.
  portainer         — UI управления Docker. Использование не
                       подтвердить/не опровергнуть по логам.

maia-grafana — удалена (см. DECISIONS.md, 2026-06-14).

Восстановление MAIA (после ребута хоста — требуется вручную,
автостарт не подтверждён):
  docker start maia-db maia-core-v2 maia-forwarder n8n

Восстановление LibreChat (при возврате к этому вектору):
  cd /home/shaen/librechat && docker compose up -d

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

⚠️ ИСПРАВЛЕНО (09.09.2026): предыдущий primary
openai/gpt-oss-20b:free депрекирован OpenRouter (HTTP 404, не
квота) — скоринг был полностью остановлен минимум с 04.09 до
09.09.2026. Заменён на текущий slug.

Оплата $10 — ПОДТВЕРЖДЕНА фактически (переход тира 11.09
~20:25 Kyiv, финально подтверждено данными аккаунта 14.09:
total_credits: 10, total_usage: 0.19 — баланс расходуется
экономно). Точный потолок нового тира не установлен (слепые зоны
из-за выключений хоста), минимум ~200+ успешных запросов/день
наблюдалось.

⚠️ N8N_API_KEY не используется nc_core.py / MAIA Core вообще —
подтверждено отдельной read-only диагностикой (14.09). Ключ
релевантен только для n8n Public API напрямую (см. раздел N8N
ниже).

Мёртвые (не использовать): gpt-oss-120b:free, gpt-oss-20b:free
(депрекирован 09.09), gemma-4-26b-a4b:free (429),
gemma-4-31b-it:free (429), gemini-2.5-flash (кредиты исчерпаны),
llama-3.2-3b:free (429), mistral-7b:free (404)

## АРХИТЕКТУРА СКОРИНГА

Мега-workflow bCa2ecXHEEhIse1c БОЛЬШЕ НЕ СУЩЕСТВУЕТ. Реальная
структура — 6 отдельных workflow: MAIA DeepSeek Analyzer v3.0
(центральный, вызывает OpenRouter НАПРЯМУЮ, сам пишет результат
в PostgreSQL) + 5 сборщиков по источникам.

nc_core.py используется только как источник промпта
(GET /scoring_prompt). /process существует в коде, но не
используется этой схемой — см. DECISIONS.md: I13.

✅ ИСПРАВЛЕНО (14.09.2026): баг "пустой content" — нода
"OpenRouter API" ссылалась на $json.id/source/raw_text/source_url
после ноды "Get Scoring Prompt" (HTTP Request, полностью заменяет
json item) — поля были undefined. Исправлено на явные кросс-ссылки
$node["Prepare"].json.*. Баг существовал минимум с 12.07.2026
(~2 месяца) — ВЕСЬ backlog status=relevant/irrelevant за период
~12.07-14.09 не является достоверной оценкой контента (см.
ROADMAP.md, открытый пункт про пересчёт).

✅ ИСПРАВЛЕНО (14.09.2026): баг "пустой source/source_url" — SQL
в "PG Read pending" не выбирал source вообще, source_url ссылался
на несуществующую колонку через meta->>'source_url'. Исправлено
тем же циклом.

Скоринг реально пишется в поле status. Реальные значения:
pending_review (не pending, как было указано ранее), irrelevant,
relevant, processed (8+ строк, ранее не упомянут в документах).

## N8N — СТАТУС ИСТОЧНИКОВ

Очередь "PG Read pending" — ✅ ИСПРАВЛЕНО (12.09.2026): сортировка
изменена с id ASC на received_at DESC. Свежие данные скорятся
первыми. Приоритет источников (CASE arxiv/hackernews →
github_trending → остальное) не тронут, работает поверх новой
сортировки.
Компромисс (принят сознательно Архитектором): старый backlog
arxiv/hackernews (~4507 записей, до 01.09) не обрабатывается,
пока сохраняется приток свежих данных.

Живые источники (подтверждено 14.09):
  coingecko        — 10467+ записей, активен
  cointelegraph     — 29+ записей, активен (через Telegram forwarder)
  arxiv_cs_ai       — активен
  arxiv_cs_lg       — активен
  github_trending   — активен

⚠️ ИСПРАВЛЕНО (14.09.2026): hackernews — МЁРТВ с 2026-06-12,
причина УСТАНОВЛЕНА (независимо, 3 диагностики): workflow
использует hnrss.org/newest?q=... (поисковые запросы) — этот
эндпоинт возвращает 0 items с середины июня. Прямой RSS без
поиска (hnrss.org/newest?points=50) работает. Внешняя проблема
источника, MAIA не виновата. Решение (сменить URL/отключить) —
не принято, отложено Архитектором.

N8N_API_KEY — РАБОЧИЙ ключ (прежняя формулировка "мёртвая
заглушка" была неверна). Используется для авторизации n8n Public
API напрямую (подтверждено PUT/GET, 12.09/14.09). НЕ используется
nc_core.py/MAIA Core. Ротирован 12.09.2026, новое значение только
в .env, не проходило через чат.

n8n Public API на этом инстансе: PATCH не поддерживается (405),
рабочий метод — PUT.

## ДРУГИЕ ИСТОЧНИКИ

⚠️ ИСПРАВЛЕНО (14.09.2026): Telegram forwarder — прежняя запись
"6 из 7 подписаны и работают" (21.08/23.08) НЕ ПОДТВЕРЖДАЕТСЯ
актуальной проверкой. Реально работают 2 из 7:
  cointelegraph — работает
  coingecko     — работает
  AnthropicAI   — НЕ работает. Причина установлена: канал сменил
                  идентичность на "Claude" (username anthropicai),
                  entity под старым именем отсутствует в Telethon-
                  сессии.
  LangChainAI, huggingface, whale_alert, coindesk — НЕ работают.
    Entities присутствуют в SQLite-сессии форвардера, но updates
    не приходят. Причина НЕ установлена. Ошибок доступа
    (ChannelInvalid/Forbidden/FLOOD_WAIT) в логах нет.
Решение Архитектора: не трогать до стабилизации остальной системы.
Побочный баг: last_id в buffer_reader() не персистится (не
устранено, не блокирует работу).

## ЭНДПОИНТЫ (maia-core-v2)

/health, /upload, /files, /resonance, /scoring_prompt, /dashboard,
/mcp, /mcp/v1

⚠️ /process — существует, не вызывается n8n (I13).
⚠️ /challenge — ВЫРЕЗАН ЦЕЛИКОМ (2026-08-02, Шаг 1).
/suggest — НЕ СУЩЕСТВУЕТ.
/mcp/v1 — полноценный рабочий JSON-RPC 2.0 MCP сервер.

## БАЗА ДАННЫХ

Host: maia-db:5432 · DB: maia_memory · User: maia_user

Реальная схема — 7 таблиц:
  incoming_messages — 30000+ строк. Колонка времени: received_at.
  scoring_prompts    — v1.0 активен, 1 строка
  resource_ledger    — 9 строк
  architect_decision_log — 6 строк
  failures            — 4 строки (не пишется корректно, ROADMAP 2.3)
  agents               — 3 строки
  event_chain           — 0 строк

Актуальный снимок incoming_messages (14.09.2026, 15:50 EEST):
  pending_review — 16346
  irrelevant — 9868
  relevant — 1546
  processed — 56
  За 14.09 (с 00:00 UTC): 17 новых relevant

⚠️ УДАЛЕНЫ (2026-08-02): chat_history, context_memory, orders,
suggest_feedback — дамп сохранён (dump_freelance_tables_20260802.sql).

Заявлены ранее, НЕ СУЩЕСТВУЮТ: resonance_trust, virtual_credits.

FK — ни одного. UNIQUE — agents(agent_id), failures(action_hash),
incoming_messages(md5(norm_text)).

Дедупликация РАБОТАЕТ, за исключением инцидента 2026-04-25
13:58–15:53 (причина не установлена).

python_jobs — подтверждено НЕ собирается с 2026-05-15, архив,
717 строк не тронуты (I1/I2).

## УВЕДОМЛЕНИЯ

Единственный официальный канал: Telegram @maia_notifier_bot —
notify_news_loop() · порог score ≥ 8.

✅ ПОДТВЕРЖДЁН РАБОЧИМ end-to-end (14.09.2026): 5 сообщений
score=8.0 доставлены, подтверждены Архитектором визуально
(12:12-12:29, темы cs.AI: Ecdysis runtime harnesses, RetroThinker
speech LLM, рекурсивное самоулучшение ИИ-агентов и др.). Первое
реальное подтверждение канала за всё время проекта.

✅ tg-001 ("Telegram Architect") — УДАЛЕНА окончательно из
workflow (14.09.2026), не просто disabled. Credential не
существовал (404, подтверждено трижды — 28.07, 05.09, 14.09).
chatId ноды (1692134296) идентифицирован как служебный
DevOps-алерт-канал ("Nova Cortex Control"), не сигнальный.
Ложная тревога 14.09 о реальном срабатывании ноды — опровергнута:
disabled-нода в n8n работает как pass-through с executionTime≈0,
создаёт ложное впечатление выполнения при поверхностной проверке
execution log (см. DECISIONS.md, раздел МЕТОДОЛОГИЯ).

## SCORING

Prompt v1.0 активен в scoring_prompts.
Резерв: /home/shaen/maia_project/scoring_prompt_v1.0.txt

## WATCHDOG

Активен:   watchdog.sh · systemd · каждые 5 мин
Отключён:  watchdog.py.disabled
Cooldown:  CONTAINER_ALERT_COOLDOWN=900
Grace:     180с после старта
STATE_DIR: /home/shaen/maia_project/.watchdog_state

## LIBRECHAT

Версия: v0.8.6-rc1 · Адрес: http://192.168.1.104:3080
MongoDB: Atlas M0 Free · Stockholm
⚠️ ОСТАНОВЛЕН (14.09.2026) — см. раздел КОНТЕЙНЕРЫ выше.

## GIT / .GITIGNORE

✅ .gitignore СОЗДАН (12.09.2026) — отсутствовал с начала проекта.
Покрывает: .env и .env.backup*, *.key, *.session, secrets/*,
**/credentials*, backup_*.json/py, wf_analyzer_*.json,
.watchdog_state/, dump_*.sql, *.deprecated_*, *.log,
__pycache__/, *.pyc.

## КОД — nc_core.py

Объём: ~1064 строки (проверено 2026-08-02).
⚠️ Ранее указывалось "1300+ строк" — расхождение не расследовано.

Устранено (Шаг 4, 2026-08-02):
  — entropy_protocol_loop — вырезан
  — GeminiAgent — незакрытый вызов заменён на app.groq_agent
  — DeepSeekAgent — удалён
  — I7 — токены/пароль вынесены
  — patch_analyzer.py — переименован в .deprecated

## ИЗВЕСТНЫЕ ПРОБЛЕМЫ (актуализировано 2026-09-14)

⚠️ Telegram forwarder — 5 из 7 каналов не работают (детали в
   разделе ДРУГИЕ ИСТОЧНИКИ выше), не трогать до стабилизации
⚠️ HackerNews — мёртв, причина известна (hnrss.org поиск сломан),
   решение не принято
⚠️ I6 failures — схема INSERT не унифицирована (ROADMAP 2.3)
⚠️ Точный потолок нового тира OpenRouter не установлен
⚠️ maia-lab-sentinel/nc_glances/portainer — назначение неизвестно
⚠️ LibreChat CREDS_KEY/CREDS_IV — дефолтные значения (LibreChat
   сейчас остановлен, актуальность вопроса снижена)
⚠️ n8n restart policy = "no" на основных MAIA-контейнерах —
   автостарт после ребута хоста не работает, требуется ручной
   cold start
⚠️ Пересчёт backlog за период бага "пустой content" (~12.07-14.09)
   — не решено (полный/частичный/не пересчитывать), см. ROADMAP.md
⚠️ n8n SQLite содержит ~4350 дублирующих execution-записей —
   не критично, требует очистки
⚠️ DDA Шаг 5/6 — открыты

ЗАКРЫТО:
  ✅ /challenge, entropy_protocol_loop, I7 хардкод, мёртвый код,
     4 фриланс-таблицы, python_jobs, docker-compose.yml
     невалидность
  ✅ Скоринг восстановлен (09.09) — slug заменён
  ✅ Оплата OpenRouter подтверждена (11.09/14.09)
  ✅ Очередь — сортировка DESC (12.09)
  ✅ Баг пустой content — исправлен (14.09)
  ✅ Баг пустой source/source_url — исправлен (14.09)
  ✅ tg-001 — удалена окончательно (14.09)
  ✅ N8N_API_KEY — ротирован (12.09), подтверждён рабочим для n8n
     API, не используется nc_core.py
  ✅ .gitignore — создан (12.09)
  ✅ notify_news_loop() — подтверждён рабочим end-to-end (14.09)
  ✅ LibreChat crash-loop — остановлен вручную (14.09)
