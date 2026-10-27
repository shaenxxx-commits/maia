# MAIA — RUNTIME
Обновлён: 2026-10-27
Обновляет: КОНЦЕПТ после аудита / УЧ после значимой сессии
 
⚠️ Полная картина расхождений документ/реальность за 2026-07 →
AUDIT_2026-07.md (историческая точка, не отражает состояние после
22.09). Полная картина находок и устранённых дефектов АУДИТА
ФАНТОМОВ → AUDIT_2026-09-ФАНТОМЫ.md. Этот документ отражает
состояние после цикла восстановления скоринга (09.09-14.09.2026),
вектора "Обнуление" (19.09.2026), замены S2→OpenAlex (21-22.09),
АУДИТА ФАНТОМОВ 2026-09 (24.09-02.10), запуска постоянного
enrichment (07-12.10) и инвентаризации хоста (13.10).
 
---
 
## ИНФРАСТРУКТУРА
 
Host:       Ubuntu 24.04.4 LTS · hostname: nova · ядро 6.8.0-146
Dual boot:  Windows/Ubuntu — ПК он же рабочая машина Архитектора
            (собственная сборка), регулярно переключается между
            ОС и выключается; блэкауты. Рестарты и простои
            хоста — рутинная ситуация, не исключение. Круглосуточная
            работа невозможна, Архитектор обеспечивает большую часть
            суток и берёт на себя увеличение времени работы.
Docker:     29.5.3 (проверено 13.10; ранее было записано 29.4.2)
Claude Code: недоступен — Free план (см. DECISIONS.md: ИНСТРУМЕНТЫ)
 
### Железо (инвентаризация 13.10, РЧ26\16, по факту)
CPU:     Intel i5-11400F, 12 потоков, 2.6 ГГц (до 4.4), L3 12 MiB
RAM:     2x8 GB DDR4 2400, оба слота заняты, максимум платы 32 GB,
         ECC нет. Доступно ~9.5 GB при нагрузке MAIA.
Плата:   ASUS PRIME H510M-A, BIOS 1601 от 07.05.2022
GPU:     AMD Navi 10 (класс RX 5600 XT/5700). MAIA не использует.
Сеть:    eno1 1000 Mb/s, Wi-Fi не виден
БП:      не измерено
Диски:
  sdb  WDC WD10EZEX 931,5 GB HDD (ROTA=1) — загрузчик, /boot, LVM;
       корень 100G ext4 (занято 56G, свободно 37G), в VG свободно
       ~828 GB. ⚠️ ТОМА DOCKER (/var/lib/docker), Postgres, n8n И
       БЭКАПЫ (~1,3 GB) ЛЕЖАТ НА ЭТОМ ЖЕ HDD.
  sda  Kingston SA400 447 GB SSD, NTFS, /mnt/shared (276G занято)
  nvme0n1 Netac 232,9 GB, NTFS, Windows, не смонтирован
BootOrder 0003,0000,0001,0002,0004, timeout 1 с; какая запись —
Ubuntu, какая — Windows: не определено.
Обновления: 100 пакетов не применены, перезагрузка не требуется;
unattended-upgrades enabled/inactive.
 
### Питание и сон
Targets sleep/suspend/hibernate/hybrid-sleep: static (НЕ masked).
logind: переопределений нет.
GNOME пользователя: sleep-inactive-ac-type='nothing'.
⚠️ GDM (экран входа): sleep-inactive-ac-type='suspend', таймаут не
измерен. Формулировка "сон отключён" верна только для сессии
пользователя, не для экрана входа. Suspend (deep) зафиксирован
07.10 04:31, 08.10 04:28, 09.10 02:46 (дважды).
BIOS "Restore AC Power Loss": не измерено, нужен вход в BIOS
(Advanced → APM Configuration).
ИБП: upower видит только DisplayDevice, nut не установлен — ИБП
нет или не виден системе.

Suspend (S3): процессор остановлен, контейнеры, n8n и таймеры не
работают. Заставить MAIA работать в ожидании нельзя: это свойство
режима, а не настройка. Рабочий путь — не уходить в suspend вовсе
(пункт Б вектора "закрепление хоста", DECISIONS.md).
Основание: ответ прежнего КОНЦЕПТ Архитектору, приложение №1 к
handoff, п. 1.2; подтверждено Архитектором.
Дата записи: 2026-10-17
 
### История простоев (journalctl, глубина с 28.09, ~15 суток)
Журнал персистентный (564 MB). 35 загрузок; 24 из 34 завершились
штатно, 10 из 34 — без записи shutdown/reboot. Интервалы простоя
7-28 ч (напр. 03.10 14:31 → 04.10 18:15). Минимум три причины:
suspend на экране входа, переход в Windows, потеря питания/зависание;
доли разделить нельзя. Аптайм в % не вычислялся. Журнал глубже 28.09
не измерен.
Темп enrichment как следствие: 25-43 запуска/сутки вместо 96.
 
### Устойчивость к жёсткому выключению
Postgres 15.17: fsync=on, synchronous_commit=on, full_page_writes=on,
wal_sync_method=fdatasync, checkpoint_timeout=300,
⚠️ data_checksums=off. 4 recovery после unclean shutdown (10.10
21:51, 11.10 12:10, 12.10 13:54, 12.10 19:10 UTC), все завершены
(redo done); совпадений по PANIC / invalid checkpoint / could not
locate a valid checkpoint / invalid page / checksum — 0. Ошибок
ext4/NTFS/I/O за текущую загрузку нет.
Unsafe shutdowns по SMART: sdb 1610 (Power-Off_Retract), sda 1422,
nvme 830 — согласуются с частыми жёсткими остановами.
 
### SMART (13.10, smartmontools 7.4 — единственный пакет,
установленный в ходе инвентаризации)
Все три диска PASSED, короткие тесты не запускались.
sdb WD10EZEX (корень, Docker, Postgres, бэкапы): Reallocated 0,
Pending 0, Offline_Uncorrectable 0, Power_On_Hours 58510,
Power_Cycle_Count 5121, UDMA_CRC_Error_Count 1332 (VALUE 200,
WORST 001, THRESH 0), ATA Error Count 13 (последние 5 — на 17113 h,
~41397 ч назад, новых нет), температура 30. Self-tests: none logged.
⚠️ Внимание только sdb: динамика CRC и ATA-ошибок по одному снимку
не определима. Повторный снимок выполнен 18.10 (см. ниже).
sda Kingston: No Errors Logged, SSD_Life_Left 76, Power_On_Hours
34041, SATA_Phy_Error_Count 31.
nvme Netac: Critical Warning 0x00, Percentage Used 18%, Available
Spare 100%, Media errors 0, Power On Hours 12257. Self-test log:
"Invalid Field in Command", не измерено.
smartd: is-enabled=enabled при логе установки "not starting" — не
расследовалось.

### SMART sdb — снимок 18.10 (база 13.10)
PASSED. Reallocated, Pending, Offline_Uncorrectable — 0. UDMA_CRC
1332 и ATA Error Count 13 без изменений. Power_On_Hours 58570
(+60), Power_Cycle_Count 5139 (+18), Power-Off_Retract 1621 (+11),
температура 31. Short self-test «Completed without error» на 58570 ч.
 
### Профиль нагрузки MAIA (13.10)
6 активных контейнеров: <1% одного ядра, ~705 MiB RAM (n8n 385,
nc_glances 108, core 79, db 78, forwarder 54, sentinel 2). Хост: load
avg 0,57/0,69/0,56, idle 92-99%, iowait 0%; занято 3,9 из 15 GiB.
БД maia_memory 45 MB, том Postgres 104 MB, том n8n 3,15 GB. Образы
Docker 14,18 GB, тома 3,3 GB. Рост за период и пики не измерены.
Вывод: вопрос хоста — устойчивость, не мощность.
 
### Watchdog-автоподъём
Docker restart policy у 4 основных контейнеров (maia-db,
maia-core-v2, maia-forwarder, n8n) — "no". systemd-таймер
maia-watchdog.timer запускает maia-watchdog.service каждые 5 минут
независимо от того, что сам .service формально помечен "disabled"
(disabled = "не стартует сам при загрузке хоста"). При обнаружении
упавшего контейнера watchdog.sh выполняет
`docker compose up -d --no-deps <name>` автоматически.
⚠️ Время восстановления варьируется: случай 22.09 — ~43 секунды на
все 4 контейнера; 29.09-01.10 (4 останова хоста) — в среднем ~8
минут. Расхождение не расследовано. Планировать ~5-10 минут как
типичное время, без гарантии нижней границы.
  cold start (при необходимости ускорить):
  docker start maia-db maia-core-v2 maia-forwarder n8n
 
⚠️ Инцидент 27.09 (закрыт): порча .env (слитая без разделителя
команда nano с переменной POSTGRES_PASSWORD) обрывала watchdog.sh до
любых docker-вызовов. Причина и фикс → DECISIONS.md. Правило: после
любой ручной правки .env — обязательная проверка (`bash -n .env` или
безопасный `source`).
 
### Права на бэкапы (закрыто 2026-10-18)
Состояние: каталоги 700, файлы 600, владелец и группа shaen
(группа docker доступа не имеет). Два бывших root:root файла
(volumes_backup.tar.gz, n8n_data_20260925.tar.gz) теперь
shaen:shaen 600. Потребителей backups/ под другим пользователем
не найдено (systemd, crontab, watchdog, docker-compose,
монтирования контейнеров). Состояние до правки сохранено в
/home/shaen/backups_state_2026-10-18.txt (68 строк). Проверка
find по критерию (каталоги ≠ 700, файлы ≠ 600, владелец/группа
≠ shaen) — пуста.
История: на 13.10 — backups/ 775 shaen:docker, 19 файлов не 600
(migration.sql 755, прочие 664), 2 файла root:root 644.
⚠️ Копии бэкапов на другом диске не обнаружено (не измерено
отдельно) — отказ sdb потеряет данные и бэкапы одновременно.
 
## КОНТЕЙНЕРЫ
 
Docker показывает 12 контейнеров, 6 активных.
 
Активны:
maia-db        — PostgreSQL 15.17 · порт 5432
maia-core-v2   — nc_core.py · порт 8080, ⚠️ публикуется только на
                 127.0.0.1 (закрыт наружу 2026-10-01)
maia-forwarder — forwarder.py · Telegram listener
n8n            — слой сбора и enrichment · порт 5678 · v2.18.7
maia-lab-sentinel — заглушка (tail -f /dev/null), RestartPolicy
                 unless-stopped. РЕШЕНО архивировать вместе с
                 docker-compose.lab.yml (АУДИТ ФАНТОМОВ) — физически
                 не выполнено.
nc_glances     — веб-мониторинг хоста (порт 61208, 0.0.0.0), не
                 завязан на MAIA. Решение: оставить, порт можно
                 закрыть на localhost отдельным шагом (не выполнено).
 
Остановлены (RestartPolicy → no, 2026-10-01, данные в томах
сохранены, судьба томов — отдельное решение):
  LibreChat (НЕСТАБИЛЕН с 14.09 — DNS crash-loop на MongoDB Atlas,
    причина не устранена и не расследована дальше)
  vectordb, rag_api, chat-mongodb, chat-meilisearch (зависимости
    LibreChat)
 
portainer — контейнера нет; том portainer_data остался без
  контейнера (13.10), судьба тома не решена.
maia-grafana — удалена (см. DECISIONS.md, 2026-06-14).
 
## ПУТИ
 
project:    /home/shaen/maia_project/
sessions:   /home/shaen/maia/sessions/
nc_core:    /home/shaen/maia/sessions/nc_core.py
forwarder:  /home/shaen/maia/sessions/forwarder.py
docs:       /home/shaen/maia_project/docs/
backups:    /home/shaen/maia_project/backups/
 
✅ ПОДТВЕРЖДЕНО АУДИТОМ ФАНТОМОВ: контейнеры монтируют ТОЛЬКО
/home/shaen/maia/sessions/ (maia-core-v2 → /app, maia-forwarder →
/app/sessions, оба rw). /home/shaen/maia_project/ НЕ монтируется
никуда.
 
⚠️ Git: внешний /home/shaen/maia_project/.git (8 коммитов, БЕЗ
remote) существует отдельно от /home/shaen/maia_project/docs/.git
(6 коммитов, origin = github.com/shaenxxx-commits/maia, зеркалит
только docs/). docs/*.md во внешнем репозитории устарели; рекомендовано
убрать их из индекса внешнего репозитория. Секретов в docs-репо не
найдено.
 
## LLM (OpenRouter)
 
Primary:  nvidia/nemotron-3-ultra-550b-a55b:free (с 2026-10-26)
Fallback: nvidia/nemotron-nano-9b-v2:free

(26.10: в публичном каталоге OpenRouter эта модель не показана;
запрос к ней не делался; используется ли она, не установлено.
В ноде Analyzer одна модель, запасной нет. В nc_core.py Fallback
относится к OpenRouterAgent (/process заблокирован).)
 
free_model_daily_requests = 1000/сутки — подтверждённый факт из API.
 
✅ ИСПРАВЛЕНО (2026-10-02): default-slug класса OpenRouterAgent в
nc_core.py указывал на мёртвый "openai/gpt-oss-20b:free", fallback-
порядок ставил мёртвые модели первыми. Теперь default →
nvidia/nemotron-3-super-120b-a12b:free, живые модели первыми.
 
max_tokens поднят с 500 до 1500 (2026-10-02) — reasoning-модель
тратила лимит на рассуждение. Квота считается в запросах, не в
токенах.
 
Мёртвые (не использовать): nvidia/nemotron-3-super-120b-a12b:free
(404 от Nvidia с 20.10; каталог OpenRouter её показывает),
gpt-oss-120b:free, gpt-oss-20b:free, gemma-4-26b-a4b:free,
gemma-4-31b-it:free, gemini-2.5-flash, llama-3.2-3b:free,
mistral-7b:free. Также найден неидентифицированный остаток
Groq-эпохи ("llama-3.1-8b-instant") в коде.
 
N8N_API_KEY не используется nc_core.py — релевантен только для n8n
Public API напрямую. Ротирован дважды: 12.09 и 29.09 — актуально
значение от 29.09.
 
## АРХИТЕКТУРА СКОРИНГА
 
### Простой 20-26.10 и смена модели (26.10)
С 20.10 пошли ошибки OpenRouter: 20.10 — success 4, error 19,
оценок за сутки 78; 21.10 — 43 запуска, оценок 0; 22-24.10
запусков нет (вероятно, ПК выключен; конкретные сутки Архитектором не уточнены); 25.10 error 3,
26.10 error 7. Оценок по updated_at с 21.10 по 26.10 до смены
модели: 0. Ошибка: HTTP 404 на ноде "OpenRouter API", модель
nvidia/nemotron-3-super-120b-a12b:free, provider_name Nvidia,
текст "Function ... Not found for account ...". Воспроизведено
пробным запросом 26.10: super-120b дал 404, ultra-550b дал 200
(ключ, маршрут, аккаунт рабочие); модель при этом в публичном
каталоге OpenRouter (target alive: True). Причина у провайдера
не установлена. Workflow Analyzer не менялся с 2026-10-02
(updatedAt 2026-10-02T10:11:47Z) — правкой не вызван.

Проба замены (read-only, 20 статей, оценены 14-19.10, setseed 0.45,
запросы последовательно): ultra-550b дал валидный JSON 20 из 20,
finish_reason=length 0 из 20, средняя абс. разница с сохранённой
оценкой 1,100 (медиана 1,0), доля ≥ 8 была 55%, стала 45%,
время ответа медиана 19,0 с, максимум 31,5 с. Правило (задано
заранее): разница 1,0-2,0 = «сдвиг», годится с оговоркой,
граница по дате обязательна.

Смена: 2026-10-26 21:21:54 +02:00. Скрипт ~/model_swap.py (sha256
a5b23528dbef8bad72f0e10d9f9569ca5660f16e04e258822d666e21ec32bccf);
первая версия (sha256 39e97b90...13ce, сохранена как
~/model_swap_v1.py.bak) содержала дефект: замена не попадала в тело
PUT; найден на ревью КОНЦЕПТ до запуска, применялась исправленная
версия. Экспорт L3: backups/2026-10-26_pre_model_swap/
wf_analyzer_before_model_swap.json (700/600, литеральных секретов 0).
PUT status 200; versionId f7608825-2ad5-431f-ae6f-37dd2edf4342 →
49b0b812-fd5e-4409-b075-66ddc0acbf78; activeVersionId равен новому
versionId (F11); active True; узлов 7 до и после; OLD=0, NEW=1.
Изменена только строка model в jsonBody ноды "OpenRouter API".

Контроль (первые 5 запусков после смены, 21:30:35-22:30:35 +02:00):
success 5, error 0, score_failed 0, новых оценок 25, из них ≥ 8: 3;
в ответах model nvidia/nemotron-3-ultra-550b-a55b:free, provider
Nvidia. Тело запроса n8n сохраняет только для ошибочных запусков,
подтверждение модели — по ответу. Окно наблюдения ~1 ч 10 мин;
устойчивость за сутки и лимит free-модели не измерены.
Ошибки Analyzer в failures не попадают (I6) — простой не был
замечен сразу.

Срез скоринга на 26.10 (старая модель super-120b, n=6189 оценённых
cs.AI/cs.LG): score ≥ 6 60,4%, ≥ 8 36,7%, ≥ 9 0,3%; pending_review
14 524, из них старше 7 суток 11 695 (старейшая 21.09).

Граница по дате: оценки до 2026-10-26 21:21:54 +02:00 даны
super-120b, после — ultra-550b; старые не пересчитываются. Порог
дайджеста пересматривается после накопления данных на новой модели.

Срез 27.10 (хост поднят около 08:55, срез в 09:34 EET): после
смены модели оценено 40 статей (26.10: 30, из них score ≥ 8: 3;
27.10: 10, ≥ 8: 0), score_failed 0; доля ≥ 8 на n=40 равна 7,5%
(в пробе 45%, на старой модели по истории 36,7%); причина
разницы не установлена, окно узкое, судить при 300 и более
оценках.

Analyzer (легаси-имя "MAIA DeepSeek Analyzer v3.0" — реально
вызывает OpenRouter напрямую, пишет в PostgreSQL) — 7 нод (было 28).
5 fetch-веток и IF relevant — структурные фантомы эпохи tg-001,
вырезаны 2026-10-01.
 
⚠️ Переименование workflow признано безопасным (0 вхождений имени как
зависимости в проверенных местах), но SQLite n8n и
~/.config/systemd/user не проверены до конца. Не выполнено, решение
за Архитектором.
 
nc_core.py используется только как источник промпта
(GET /scoring_prompt). /process существует в коде, заблокирован (503).
 
Скоринг пишется в поле status. Значения: pending_review, irrelevant,
relevant, processed, и (с 2026-10-02) отдельный статус для строк, где
LLM-ответ не парсится после исчерпания попыток (не искажает статистику
релевантности).
 
### "1 из 5" — исправлено (2026-10-01/02)
Parse and Build SQL теперь обрабатывает все 5 items батча.
 
### Квота — перенастроена (2026-10-02)
Cron-узел "30sec" фактически срабатывал раз в минуту (потолок 7200
вызовов/сутки против лимита 1000). Интервал 900 с → 480 вызовов/сутки
(48% лимита). Узел не переименован намеренно.
 
### Старые pending_review — решение принято, не реализовано
Приток (700-2900/сутки) структурно превышает бюджет Analyzer
(480/сутки). Очередь DESC расставляет приоритет — старые записи
сознательно не оцениваются (I5). Техническая мера (статус
"archived_unscored" для pending_review старше N дней, явный UPDATE по
id) — не реализована.
 
## N8N — СТАТУС ИСТОЧНИКОВ
 
Приоритет источников в "PG Read pending" (CASE arxiv/hackernews →
github_trending → остальное) поверх сортировки received_at DESC.
 
⚠️ ИЗВЕСТНЫЙ ВОСПРОИЗВОДИМЫЙ БАГ (компенсируется watchdog): hours-
интервальные Schedule Trigger не переустанавливают таймер после cold
start процесса n8n без явного рестарта контейнера. minutes-интервал не
подвержен.
⚠️ РАСХОЖДЕНИЕ (09.10): фактический триггер cs.AI/cs.LG —
hoursInterval=2 (updatedAt cs.AI 29.09, cs.LG 25.09), а не 15 минут,
как ранее записано. GitHub-workflow не смотрелся. Не исправлено.
 
Источники (статус на 2026-10-13):
  arxiv_cs_ai       — активен, штатно. SQL-дефект конкатенации
                       исправлен 29.09, подтверждён 01.10.
  arxiv_cs_lg       — исправлен окончательно 25.09. Реальная причина —
                       LaTeX-конструкции ($\Omega$, $O(n^{-1})$) в
                       абстрактах при pgp.helpers.concat с values=[];
                       исправлено параметризованным запросом.
  github_trending   — активен.
  hackernews        — МЁРТВ с 2026-06-12. hnrss.org/newest?q=
                       (поисковый эндпоинт) даёт 0 items; прямой RSS
                       работает. Решение не принято.
  coingecko         — ОСТАНОВЛЕН (19.09), active=false, не удалён,
                       существует без функции намеренно.
  cointelegraph     — работает через forwarder.py (Telegram).
                       Переносится на Шаг 3 плана перестройки.
 
### Workflow «MAIA OpenAlex Enrich» (id 0LUeTcl1JKG0Y3i6)
Создан 2026-10-08 через Python (L7), активен, расписание включено
08.10 (ворота В3 подтверждены Архитектором лично).
Ноды: Schedule Trigger (minutes 15) → PG Candidates (LIMIT 100) →
Make Batches (по 25) → OpenAlex (httpRequest 4.2, таймаут 30 с,
onError continueRegularOutput, БЕЗ mailto) → Parse (привязка по doi)
→ PG Insert Entities / PG Insert Failures (json_to_recordset($1::json),
один JSON-параметр через queryReplacement, ON CONFLICT DO NOTHING).
Кандидаты: incoming_messages, arXiv cs.AI/cs.LG, Announce Type = new
(регулярка 'Announce Type: ([a-z-]+)' по raw_text), возраст 7-21 суток
по meta->>'pubDate', строки в paper_entities ещё нет.
⚠️ Сортировка: DESC (свежие вперёд) с 09.10, решение Архитектора
(вариант А); изначально было ASC.
count=0 — не ошибка, ничего не пишется, статья выпадает по возрасту.
Ошибки → failures (provider 'openalex_enrich').
Механика n8n 2.18.7 (postgres v2.5): результат каждого {{...}} в
queryReplacement проходит JSON.parse; валидный JSON уходит одним
параметром, иначе строка режется по запятым; выражение обязано
вернуть строку (JSON.stringify).
versionId на 09.10 22:30: 0478e0ec-12e6-4745-9158-42320c961471,
вероятно после правки DESC (не измерено). История версий через API
недоступна (404). При любой правке workflow — фиксировать versionId
до и после.
Темп: execution 132 за 08.10 00:04Z — 12.10 08:30Z, все success; по
суткам 29 / 43 / 33 / 25 (ожидалось 96). Упирается в простои хоста.
Не измерено: время обработки пакета в n8n, расход кредитов по
заголовкам из n8n, поведение лимита при исчерпании, пакеты >25 DOI.
 
N8N_API_KEY — рабочий (значение от 29.09). n8n Public API: PATCH не
поддерживается (405), рабочий метод — PUT; PUT на активном workflow
публикует сам. Credential: OpenRouter v2, GitHub v2, Postgres.
Variables: HTTP 403. В env контейнера n8n переменных MAIA_* нет.
⚠️ Вывод /credentials содержит личный email владельца (I7) — не
печатать.
 
## ДРУГИЕ ИСТОЧНИКИ
 
Telegram forwarder — реально работают 2 из 7 каналов:
  cointelegraph — работает (crypto, не остановлен)
  coingecko     — работает (Telegram-путь; n8n-путь остановлен)
  AnthropicAI, LangChainAI, huggingface, whale_alert, coindesk —
    НЕ работают, причина не установлена для большинства.
Решение Архитектора: не трогать до Шага 3 плана перестройки.
Побочный баг: last_id в buffer_reader() не персистится.
 
## ЭНДПОИНТЫ (maia-core-v2)
 
/health, /upload (защищён X-API-KEY), /resonance, /scoring_prompt,
/dashboard
 
🔴 ЗАБЛОКИРОВАНЫ (I14, 2026-10): /mcp, /mcp/v1, /process, /files —
все возвращают 503, тела функций не удалены.
 
АУДИТОМ ФАНТОМОВ (2026-10-01) установлено:
  — /mcp/v1 (maia_query) обходил остановку notify_news_loop(),
    рассылая уведомления score≥6.
  — /mcp и /mcp/v1 писали в agents без аутентификации.
  — /process не вызывался легитимным потребителем, тратил квоту.
  — /files отдавал список файлов без проверки.
Все четыре заблокированы до появления механизма аутентификации.
 
⚠️ /health отдаёт "telegram":"active" независимо от реального статуса
потока — известная проблема, не исправлена.
 
## БАЗА ДАННЫХ
 
Host: maia-db:5432 · DB: maia_memory (45 MB) · User: maia_user
Postgres 15.17.
 
8 таблиц, 1 схема, FK нет (архитектурное решение), 2 триггера (оба
подтверждены живыми: trg_fill_norm_text, trg_updated_at), 2 функции.
 
  incoming_messages — снесена 19.09 (TRUNCATE, исключение из I2), новый
                       поток с 21.09. 8950 строк на 02.10; актуальное
                       число не снималось. Текст хранится в raw_text
                       (НЕ в content). Структура для arxiv:
                       arxiv_id = substring(meta->>'url' from
                       'abs/(.+)$') (meta->>'url' вида
                       https://arxiv.org/abs/NNNN.NNNNN, без vN);
                       title MAIA = split_part(raw_text,' — arXiv:',1);
                       meta-ключи: category, groq, pubDate, url; даты
                       подачи статьи в данных нет, только pubDate
                       рассылки. Announce Type в потоке: new, cross,
                       replace, replace-cross (на 06.10: new ai 1315,
                       lg 1873); replace/replace-cross относятся к
                       старым статьям. Для matching только new.
                       Данные тома Postgres до TRUNCATE сохранены в
                       L3-бэкапе (backups/20260919_020953_pre_crypto_removal/
                       volumes_backup.tar.gz), восстановимость не
                       проверялась.
                       sequence incoming_messages_id_seq — не проблема
                       (следствие TRUNCATE без RESTART IDENTITY, дубли
                       по idx_unique_norm_text тратят nextval()).
  scoring_prompts    — v1.0 активен, 1 строка
  resource_ledger    — 10 строк. Ручной журнал, без кодового
                       потребителя.
  architect_decision_log — 6 строк. Тот же статус.
  failures            — 13 строк. Схема по факту: id serial,
                       action_hash text NULLABLE + UNIQUE, provider
                       text, error text, context JSON (НЕ jsonb),
                       timestamp without time zone default now().
                       id 6 — тестовая (ENOTFOUND api.openalex.invalid),
                       остаётся навсегда (I2). id 7 — 09.10 20:45,
                       openalex_enrich, 504 Gateway timeout, штатная
                       обработка. id 8-12 (11-21.10) — openalex_enrich:
                       EAI_AGAIN ×2, 504 ×2, 503 ×1 — штатная обработка
                       ответов OpenAlex. id 13 — 21.10 13:33:57,
                       provider forwarder, ошибка "buffer_reader loop:
                       Connection to Telegram failed 5 time(s)", context
                       {"function": "buffer_reader_loop"}. forwarder.py
                       пишет в failures (INSERT INTO failures, стр. 51);
                       mtime файла 2026-09-04 12:27:58.
                       ⚠️ I6 нарушен для сборщиков и Analyzer: их
                       ошибки в failures напрямую не попадают.
  agents               — 4 строки (3 тестовые + след audit-rch20).
                       Удалять нельзя (I2). pubkey всегда NULL.
  event_chain           — 0 строк. Таблица существует, код её не
                       использует (задел).
  paper_entities         — 5485 строк на 26.10 (было 2444 на 12.10; первая запись 07.10:
                       8 строк; далее workflow). Колонки: paper_id text
                       PK (короткий id OpenAlex, W...), arxiv_id text,
                       title text (OpenAlex), cited_by_count integer,
                       venue text, first_seen_at timestamptz NOT NULL
                       default now(), enriched_at timestamptz (now() в
                       момент INSERT), arxiv_id_source_hint text
                       (nullable, смысл не установлен, не используется,
                       NULL у всех строк, под свои признаки не
                       занимать). Индекс idx_paper_entities_arxiv_id
                       UNIQUE (arxiv_id) WHERE arxiv_id IS NOT NULL. FK,
                       CHECK, триггеров нет. Один замер на статью, без
                       перезаписи. Бэкапы: backups/20261007_pre_paper_
                       entities_insert/, backups/20261007_pre_openalex_
                       enrich/.
 
resonance_trust, virtual_credits — физически НЕ СУЩЕСТВУЮТ. STACK v10.5
(архив) называет их "реализованными".
 
FK — ни одного. UNIQUE — agents(agent_id), failures(action_hash),
incoming_messages(md5(norm_text)), paper_entities(arxiv_id) WHERE NOT
NULL.
 
⚠️ Журнал maia-db содержит тексты абстрактов arXiv из упавших запросов
(60 совпадений широкого шаблона). Возможно, упавшие INSERT продолжаются.
Не расследовалось.
 
## УВЕДОМЛЕНИЯ
 
notify_news_loop() — ОСТАНОВЛЕН (22.09.2026, обратимо, вызов
закомментирован в nc_core.py). Причина: фокус на entity-слое.
✅ Обход через /mcp/v1 устранён 01.10: Telegram не получает сообщений
ни одним путём.
До остановки — @maia_notifier_bot, порог score ≥ 8, подтверждён
end-to-end (14.09), до ~50 сообщений/день (19.09).
⚠️ Расхождение не расследовано: mtime nc_core.py — 19.09, документы
датируют комментирование 22.09.
Замена — ежедневный дайджест (Шаг 2 плана перестройки): условие
«до готовности entity-слоя» снято 27.10 (DECISIONS.md); запуск не
санкционирован, предпосылки в ROADMAP.md, Шаг 2.
tg-001 удалена окончательно (19.09).
 
## SCORING
 
Prompt v1.0 активен в scoring_prompts.
Резерв: /home/shaen/maia_project/scoring_prompt_v1.0.txt
 
## ENTITY-СЛОЙ (Шаг 3 плана перестройки)
 
Enrichment-источник: OpenAlex (заменил Semantic Scholar, 21.09).
Без ключа/формы/одобрения.
⚠️ ЛИМИТ: 1000 кредитов/сутки (x-ratelimit-limit 1000), 1 запрос =
1 кредит. Прежняя цифра "100 000/сутки" из документации замером НЕ
подтвердилась. Пакетный фильтр doi:a|b|c на до 25 DOI = 1 кредит;
порядок результатов не совпадает с порядком запроса, привязка только
по doi. Пакеты >25 и предел URL не измерены. Без mailto лимит не
изменился. Заголовки лимита из n8n не измерены.
mailto: в workflow НЕ передаётся (решение Архитектора). MAIA_EMAIL
(.env) в n8n не трогать.
⚠️ Аномалия: пакет из 25 DOI возвращает посторонние работы (не
arXiv), пакет из 5 — нет; найденные не теряются; workflow отбрасывает
посторонних привязкой по doi. Причина не измерена.
 
Метод matching (только для свежих id потока, Announce Type = new):
  GET /works?filter=doi:https://doi.org/10.48550/arxiv.{ID}
Строгий формат: полный DOI URL, arxiv строчными, без vN. Bare DOI и
/works/doi:{...} не работают. Title-поиск ненадёжен.
⚠️ На СТАРЫХ id (напр. 1706.03762, 1810.04805) DOI-метод даёт count=0:
препринт хранится как location внутри "склеенной" записи с чужим
doi/title/year. Для старых работает filter=locations.landing_page_url:
http://arxiv.org/abs/{ID} (https даёт 0); для свежих наоборот. Для
текущего потока ограничение не блокирует.
Расхождение title MAIA vs OpenAlex запись не блокирует (случай
2609.22107); пишется title OpenAlex.
 
Состояние на 12.10: paper_entities 2444; new старше 7 суток 3164,
обогащено 77,2%; остаток кандидатов в окне 710; старше 21 суток без
записи 10 (все pubDate 21.09), обогащено 37. Контрольная выборка 50 id
(pubDate 21-28.09): найдено 0 из 50 при двух замерах (09.10 и 12.10),
"найдено позже" = 0 (без контроля метода в замере Б). Причина
ненахождения не измерена. Доля привязки в тестах нестабильна (84%,
56%, 81%).
Приток new ~380/сутки. Статьи 29.09 (486 в остатке на 09.10)
достигают границы 21 суток ~20.10.
Повторный замер остатка и потерь — выполнен 26.10 (см. ниже).

Состояние на 26.10: paper_entities 5485 (+3041 с 12.10); new старше
7 суток 6194, обогащено 5412 (87,4%; было 77,2%); остаток в окне
7-21 суток 302 (было 710); старше 21 суток без записи 483 (прирост
473 с 12.10, из них 418 по pubDate 29.09-02.10; остальные 55 с
pubDate 22-28.09 вышли из окна уже после 12.10).
Контроль метода (DOI-запрос без mailto): 5 из 5 заведомо найденных;
из 25 выпавших найден 1 (2609.22554). Вместе с замером 12.10 (0 из
50): найдено позже 1 из 75. Причина ненахождения не измерена; вывод
осторожный — «не найдено при методе, работавшем до этого».
Аномалия пакета 25 DOI (посторонние работы в ответе, 13 из 14)
повторилась; на результат не влияет (привязка по doi); отдельный
замер с пакетами по 5 не срочен.
Темп (косвенно по enriched_at, n8n API не запрашивался): 6 суток
из 15 почти нулевые (17, 18, 20, 22, 23, 24.10); причина — ПК
Архитектора выключен (со слов).
Решение, предложение КОНЦЕПТ (подтверждения нет): журнал попыток
не строить; LIMIT и окно 21 сутки не менять.

### Полезность entity-слоя (измерено 26.10)

Один замер (paper_entities, n 5649): cited_by_count ненулевой у
3 статей (0,1%), медиана 0, p90 0, максимум 71; в возрастных
бакетах 11-14 и 15-21 суток все значения 0; venue не arXiv и не
NULL у 4 из 5649; связь с оценкой модели: n 3429, ненулевых
cited 0, корреляция не определена. Разбивка по возрасту содержит
дубли от join с incoming_messages, на доли не влияет.

Тест траектории: выборка 200 статей (enriched_at 07-12.10,
setseed 0.42), повторно найдено 199, рост cited_by_count у 0,
интервал между замерами 14-19 суток, 8 запросов к OpenAlex. По
правилам, заданным заранее: часть 1 «один замер почти пуст»
(доля ненулевых меньше 10%), часть 2 «при этом интервале
траектории нет» (рост меньше 10%).

Поля OpenAlex (25 статей, 1 запрос, найдено 25 из 25):
authorships 100%; хотя бы у одного автора institutions 76% (19
из 25); primary_topic, topics, concepts, open_access 100%;
referenced_works_count в ответе есть, значения 0 у всех;
language заполнен у 1 из 25 (4%); поля author.cited_by_count в
ответе нет.

Не измерено: месячная динамика cited_by_count (срок наблюдения
2-3 недели); причина ненахождения 1 статьи (2609.24250) при
повторном замере. Список выборки 200 удалён вместе с временными
файлами (по отчёту АДМ); повтор возможен по критерию выборки,
воспроизводимость порядка не гарантирована.
Решение: см. DECISIONS.md, запись proposed.
 
## WATCHDOG
 
Активен:   watchdog.sh · systemd-таймер (maia-watchdog.timer) · каждые
           5 мин · автоматически поднимает упавшие контейнеры
Отключён:  watchdog.py.disabled
Cooldown:  CONTAINER_ALERT_COOLDOWN=900
Grace:     180с после старта
STATE_DIR: /home/shaen/maia_project/.watchdog_state
⚠️ Telegram-уведомления watchdog не настроены — алерты нигде не видны
Архитектору, только в watchdog.log.
⚠️ Groq-логика внутри watchdog.sh (check_groq_failures) — живой код,
ни разу не срабатывал, фантом, кандидат на удаление.
⚠️ maia_project/watchdog.sh и watchdog_new.sh ("v2") существуют
параллельно рабочему sessions/watchdog.sh, нигде не подключены.
Решение не принято: внедрить v2 или архивировать.
 
## LIBRECHAT
 
Версия: v0.8.6-rc1 · MongoDB: Atlas M0 Free
🔴 ОСТАНОВЛЕНА (2026-10-01): вся группа (5 контейнеров),
RestartPolicy → no. Причина (MongoDB Atlas DNS ENOTFOUND) не
устранялась с 14.09. Данные в томах сохранены.
 
## GIT / .GITIGNORE
 
.gitignore создан 2026-05-31 (коммит af4739c), расширен 2026-09-12.
Покрывает: .env и .env.backup*, *.key, *.session, secrets/*,
**/credentials*, backup_*.json/py, wf_analyzer_*.json,
.watchdog_state/, dump_*.sql, *.deprecated_*, *.log, __pycache__/,
*.pyc.
⚠️ Секреты коммита af4739c (ab3fe96 и ранее) остаются физически в
.git/objects. Все найденные значения отозваны у провайдеров (ротация
29.09). Rewrite истории (BFG/filter-repo) — отдельная операция без
отката, не выполнялась.
 
## КОД — nc_core.py
 
Объём: 864 строки (после вырезания GroqAgent, 2026-10-02).
✅ Устранено АУДИТОМ ФАНТОМОВ: GroqAgent вырезан; slug и fallback-
порядок OpenRouterAgent исправлены; рассинхронизация ключей
meta->'groq'/'deepseek' устранена (nc_core.py:767,806 и
dashboard_patch.py).
 
⚠️ L14 (INVARIANTS.md): правка nc_core.py — только побайтово (open
rb/wb). Системный паттерн mojibake: 14 вхождений управляющего символа
C1 (0x98), двойное кодирование кириллической "И" на строках 66, 88,
105, 392, 632, 670. Решение "не трогать" принято явно (КОНЦЕПТ11);
полная чистка — отдельный вектор.
 
⚠️ Не вырезано: мёртвые константы GEMINI_API_KEY/DEEPSEEK_API_KEY,
/health-ветка (всегда "inactive"), ветка agent_type=='deepseek'.
Два бэкапа (backup_nc_core_20260801.py, nc_core_backup_scoring.py)
содержат DeepSeek-код — архивные. update_agent_metrics — осиротевшая
функция. Ссылки на утраченные версии: строка 270 "Resonance detector
v9.6", TODO на строке 665 про "stack v10.4 §4".
 
## DASHBOARD_PATCH.PY
 
Подключается через exec() на nc_core.py:704. ✅ Исправлено 2026-10-02:
meta.get('groq') or meta.get('deepseek') or {}.
⚠️ Не проверено: dashboard_patch.py берёт пароль БД из
POSTGRES_PASSWORD, nc_core.py — из DB_PASSWORD.
 
## КРИПТО-ВЕТКА (закрыта частично)
 
incoming_messages снесён (включая крипто-контент), CoinGecko (n8n-путь)
остановлен, Cointelegraph (forwarder-путь) работает — переносится на
Шаг 3 плана перестройки.
 
## ИЗВЕСТНЫЕ ПРОБЛЕМЫ (актуализировано 2026-10-26)
 
ХОСТ (инвентаризация 13.10; решения Б-Е открыты, см. DECISIONS.md):
⚠️ Тома Docker/Postgres и бэкапы — на одном старом HDD sdb (58510 ч,
   5121 включений, CRC 1332, 13 ATA-ошибок давних, data_checksums=off);
   копии бэкапов на другом диске нет
⚠️ GDM suspend на экране входа (targets не masked)
⚠️ Kingston /mnt/shared смонтирован ro (fuseblk, NTFS), запись
   отказана; причина не установлена; свободно 203G из 448G
⚠️ Защита данных (шаг Е: копия бэкапов и снимок на второй диск)
   на паузе (DECISIONS.md, 26.10); возобновление — при переносе
   томов на SSD, ухудшении SMART sdb, появлении потребителя
⚠️ BIOS Restore AC Power Loss не измерен
⚠️ ИБП нет или не виден
⚠️ Темп enrichment 25-43 запуска/сутки из 96 — следствие простоев хоста
 
СИСТЕМА:
⚠️ Приток в очередь (700-2900 строк/сутки) превышает бюджет Analyzer
   (480/сутки) — решение принято, archived_unscored не реализован
⚠️ Telegram forwarder — 5 из 7 каналов не работают, не трогать до
   Шага 3 плана перестройки
⚠️ HackerNews — мёртв, решение не принято
⚠️ I6 failures — Analyzer и сборщики не пишут в failures напрямую;
   простой Analyzer 20-26.10 это показал: ошибки не попали
⚠️ Нода «OpenRouter API»: не заданы timeout, retryOnFail, maxTries,
   onError (дефолты n8n)
⚠️ Новая модель ultra-550b: нет данных по устойчивости за сутки и
   по лимиту free-модели (окно наблюдения ~1 ч 10 мин)
⚠️ cs.AI/cs.LG: hoursInterval=2 (не 15 минут), баг hours-триггера не
   переисследован
⚠️ /health отдаёт "telegram":"active" независимо от статуса
⚠️ mojibake в nc_core.py — L14 без срока
⚠️ watchdog_new.sh ("v2") — внедрить или архивировать
⚠️ Groq-логика в watchdog.sh — фантом
⚠️ maia-lab-sentinel — решение "архивировать" не выполнено
⚠️ nc_glances — порт на localhost не закрыт
⚠️ LibreChat (5 контейнеров) остановлена, судьба томов не решена;
   том portainer_data без контейнера
⚠️ notify_news_loop/mtime nc_core.py расхождение — не расследовано
⚠️ Watchdog: время восстановления варьируется (43с vs ~8 мин)
⚠️ Telegram-уведомления watchdog не настроены
⚠️ Git-история: секреты физически в .git/objects (отозваны)
⚠️ Lobsters заявлен в фокусе, не реализован — статус открыт
⚠️ DeepSeek API-ключ — фантом, закрытие аккаунта за Архитектором
⚠️ Расхождение переменных пароля БД (POSTGRES_PASSWORD vs DB_PASSWORD)
⚠️ Переименование Analyzer — SQLite n8n и systemd user не проверены
⚠️ Журнал maia-db содержит абстракты из упавших запросов — не
   расследовано
⚠️ Две версии maia_memory_migration.sql / _utf8.sql — sha256 не сверен
⚠️ Файлы НЕОПРЕДЕЛЁННОГО статуса: Dockerfile.core/.forwarder,
   maia_project/CLAUDE.md, sessions/uploads/
⚠️ Не измерено: БП, BootOrder-запись Ubuntu/Windows, таймаут GDM,
   рост БД/томов и пики, NVMe self-test, статус smartd, расход
   кредитов OpenAlex из n8n
 
ENRICHMENT (см. раздел ENTITY-СЛОЙ): причина ненахождения части
статей в OpenAlex, причина нестабильной привязки, причина посторонних
работ в пакете 25 DOI — не измерены; замер около 20-21.10.
 
ЗАКРЫТО (2026-10, АУДИТ ФАНТОМОВ — полный список → DECISIONS.md,
AUDIT_2026-09-ФАНТОМЫ.md):
  ✅ Analyzer: простой 54ч · cs.AI/cs.LG SQL-дефекты · "1 из 5" · 43%
     искажённых оценок · /mcp/v1, /process, /files · порт 8080 ·
     GroqAgent · slug OpenRouterAgent · dashboard_patch.py · квота
     OpenRouter · PGPASSWORD вне .env · I7 (ротация) · 5 fetch-веток
     Analyzer · sequence · event_chain · путь кода
  ✅ sudo: подтверждён полный root, пароль вручную (13.10)
  ✅ Первая живая запись в paper_entities и постоянный enrichment
     запущены (07-08.10)
  ✅ Вопрос "журнал попыток" закрыт: не строить (12.10)
