# MAIA — RUNTIME
Обновлён: 2026-08-02
Обновляет: КОНЦЕПТ после аудита / УЧ после значимой сессии

⚠️ Полная картина расхождений документ/реальность → AUDIT_2026-07.md
Этот документ отражает состояние ПОСЛЕ аудита И после исполнения
Шага 4 плана восстановления (гигиена — завершена 2026-08-02).
Открыт Шаг 2 (основной контур) — см. ROADMAP.md.

---

## ИНФРАСТРУКТУРА

Host:       Ubuntu 24.04 LTS · hostname: nova · SSD 447GB
CPU:        12 ядер
RAM:        15GB total · ~10GB доступно
Dual boot:  Windows (игры) — не перезагружать пока MAIA работает
            ⚠️ При переключении на Windows или выключении ПК
            Docker останавливается резко → watchdog шторм алертов
            cold start: docker start maia-db maia-core-v2 maia-forwarder n8n
Docker:     29.4.2 · автостарт включён
Claude Code: недоступен — Free план (см. DECISIONS.md: ИНСТРУМЕНТЫ)

## КОНТЕЙНЕРЫ

12 контейнеров, все Up. Почти все стартовали в одно окно
(06:19-06:20) — похоже на общий рестарт хоста, причина не
устанавливалась.

maia-db        — PostgreSQL · порт 5432
maia-core-v2   — nc_core.py · порт 8080
maia-forwarder — forwarder.py · Telegram listener
n8n            — слой сбора · порт 5678

librechat        — интерфейс моделей · порт 3080
vectordb         — LibreChat зависимость
rag_api          — LibreChat зависимость
chat-mongodb     — LibreChat зависимость
chat-meilisearch — поиск · периодически рестартует (не критично)

Недокументированные (статус открыт, требуют решения Архитектора):
  maia-lab-sentinel — подтверждена чистой заглушкой (tail -f
                       /dev/null, Mounts: [], без healthcheck).
  nc_glances        — API реально живой (GET /api/4/cpu отвечает).
                       Назначение не установлено.
  portainer         — UI управления Docker. Использование не
                       подтвердить/не опровергнуть по логам.

maia-grafana — удалена (см. DECISIONS.md, 2026-06-14).

Восстановление MAIA:
  docker start maia-db maia-core-v2 maia-forwarder n8n

Восстановление LibreChat:
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

Primary:  openai/gpt-oss-20b:free
Fallback: nvidia/nemotron-3-super-120b-a12b:free
Fallback: nvidia/nemotron-nano-9b-v2:free

Дневной лимит: 1000 запросов/день (с 2026-08-02, после разовой
покупки $10 кредитов — см. DECISIONS.md, раздел LLM).
⚠️ Действие нового лимита на практике не подтверждено — первая
задача Шага 2 плана восстановления.

Мёртвые (не использовать) — см. DECISIONS.md для полного списка.

## АРХИТЕКТУРА СКОРИНГА

Мега-workflow bCa2ecXHEEhIse1c БОЛЬШЕ НЕ СУЩЕСТВУЕТ. Реальная
структура — 6 отдельных workflow: MAIA DeepSeek Analyzer v3.0
(центральный, вызывает OpenRouter НАПРЯМУЮ, сам пишет результат
в PostgreSQL) + 5 сборщиков по источникам.

nc_core.py используется только как источник промпта
(GET /scoring_prompt). /process существует в коде, но не
используется этой схемой — см. DECISIONS.md: I13.

Скоринг реально пишется в поле status (relevant/irrelevant/pending),
не в meta->>'score'.

## N8N — СТАТУС ИСТОЧНИКОВ

Живые (на 2026-07-19):
  CoinGecko — HTTP Request free API · каждые 15 мин

⚠️ ПЕРЕСМОТРЕНО (2026-08-02): GitHub Trending НЕ живой — простой
7+ дней (с 2026-07-24 08:00 UTC). Причина не диагностирована.

Ранее протухшие (требуют проверки актуального статуса):
  HackerNews  — молчал 36 дней (на 2026-07-19)
  ArXiv cs.AI — молчал 5 дней
  ArXiv cs.LG — молчал 24 дня

Дублирующие коллекторы — РЕШЕНИЕ ОТМЕНЕНО (2026-08-02, см.
DECISIONS.md): реального дублирования нет, охват важнее
гипотетической экономии LLM-квоты.

Очередь скоринга ("PG Read pending", Cron ~120 сек, LIMIT 5) берёт
строго по приоритету источника: arxiv/hackernews → github_trending →
остальное. CoinGecko практически не доходит до обработки.

## ДРУГИЕ ИСТОЧНИКИ

Telegram каналы — forwarder.py. Аккаунт физически не состоит ни
в одном из 7 целевых каналов. Решение: подписать — часть Шага 2,
не исполнено. Побочный баг: last_id в buffer_reader() не
персистится.

## ЭНДПОИНТЫ (maia-core-v2)

/health, /upload, /files, /resonance, /scoring_prompt, /dashboard,
/mcp, /mcp/v1

⚠️ /process — существует, не вызывается n8n (I13).
⚠️ /challenge — ВЫРЕЗАН ЦЕЛИКОМ (2026-08-02, Шаг 1).
/suggest — НЕ СУЩЕСТВУЕТ.
/mcp/v1 — полноценный рабочий JSON-RPC 2.0 MCP сервер.

## БАЗА ДАННЫХ

Host: maia-db:5432 · DB: maia_memory · User: maia_user

Реальная схема — 7 таблиц (было 11 до Шага 4):
  incoming_messages — 14320+ строк. Колонка времени: received_at.
  scoring_prompts    — v1.0 активен, 1 строка
  resource_ledger    — 9 строк
  architect_decision_log — 6 строк
  failures            — 3 строки (не пишется корректно, Шаг 2)
  agents               — 3 строки
  event_chain           — 0 строк

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

Найдены аудитом ещё два канала — подлежат закрытию:
  /mcp/v1 maia_query — порог ≥6
  dashboard — дублирует данные

tg-001 — остаётся отключённой. Credential физически не существует.

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

## КОД — nc_core.py

Объём: ~1064 строки (проверено 2026-08-02).
⚠️ Ранее указывалось "1300+ строк" — расхождение не расследовано.

Устранено (Шаг 4, 2026-08-02):
  — entropy_protocol_loop — вырезан
  — GeminiAgent — незакрытый вызов заменён на app.groq_agent
  — DeepSeekAgent — удалён
  — I7 — токены/пароль вынесены
  — patch_analyzer.py — переименован в .deprecated

## ИЗВЕСТНЫЕ ПРОБЛЕМЫ (на 2026-08-02, после Шага 4)

⚠️ HackerNews/ArXiv — требуют проверки актуального статуса
⚠️ GitHub Trending — 7+ дней простоя, не диагностировано
⚠️ Очередь скоринга голодает CoinGecko
⚠️ Telegram forwarder — не подписан на каналы (Шаг 2)
⚠️ I6 failures — схема INSERT не унифицирована (Шаг 2)
⚠️ OpenRouter лимит 1000/день — не подтверждён на практике (Шаг 2)
⚠️ maia-lab-sentinel/nc_glances/portainer — назначение неизвестно
⚠️ LibreChat CREDS_KEY/CREDS_IV — дефолтные значения
⚠️ DDA Шаг 5/6 — открыты

ЗАКРЫТО:
  ✅ /challenge, entropy_protocol_loop, I7 хардкод, мёртвый код,
     4 фриланс-таблицы, python_jobs
