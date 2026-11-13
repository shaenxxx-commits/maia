# MAIA — RUNTIME
Обновлён: 2026-11-13
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
✅ Копия бэкапов на другом диске выполнена 11.11: архив
/mnt/shared/maia_backups/maia_backups_2026-11-11.tar.gz (1378802807
байт, sha256 7e0a3642ddf8d48ba24b4ea34c737a73665fd00db835a8fa42f3d1f7358fc6fb),
не зашифрован, ручной, расписания нет, повтор — по блоку АДМ.
Тома Docker/Postgres/n8n по-прежнему только на HDD.
 
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

⚠️ Пометка 2026-11-08: запись историческая (2026-10-02). С 2026-10-26 21:21:54 +02:00
primary — nvidia/nemotron-3-ultra-550b-a55b:free (см. ниже «АРХИТЕКТУРА СКОРИНГА»,
подсекция «Простой 20-26.10 и смена модели»). super-120b — в списке мёртвых ниже.

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

Срез на новой модели до 30.10 (источник: передача КОНЦЕПТ
2026-11-04): оценено 385, успешных 373 (relevant 195,
irrelevant 178), score_failed 12; доля score ≥ 8 — 16,4%,
≥ 9 — 0,8%; ≥ 8 у cs.AI 55 из 258, у cs.LG 6 из 115. Причина
перекоса cs.AI/cs.LG не установлена.

Analyzer (легаси-имя "MAIA DeepSeek Analyzer v3.0" — реально
вызывает OpenRouter напрямую, пишет в PostgreSQL) — 10 узлов
(7 после вырезания фантомов 2026-10-01, +3 журнальных узла I6
применены 2026-10-28; versionId e3f8dd52-5b3b-4008-bb05-61da5b0b8928,
active True). 5 fetch-веток и IF relevant — структурные фантомы
эпохи tg-001, вырезаны 2026-10-01.
 
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

### I6 в Analyzer — применён 2026-10-28
Активный PUT 28.10: workflow wLhFlkvOR4skwPQo, 10 узлов (7 после
вырезания фантомов 2026-10-01, +3 журнальных), versionId
e3f8dd52-5b3b-4008-bb05-61da5b0b8928, active True.
Контроль по executions: пары «запуск журнального узла → строка
failures» — 27745→id 16, 27768→id 17. Вердикт: РАБОТАЕТ.
Ограничение: ON CONFLICT по action_hash (provider+код+текст+час)
схлопывает повторы в одном часе; failures показывает события, а не
число потерянных статей.
Ветка Format node error (ошибка узла OpenRouter) в бою не
срабатывала.

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
  hackernews        — workflow активен, но падает с ошибкой (последний
                       запуск 12.11 14:00 — error; см. MAIA Map).
                       Ошибки в failures не попадают (I6 для сборщиков
                       не выполнен). По прежним данным — МЁРТВ с
                       2026-06-12, hnrss.org/newest?q= даёт 0 items;
                       сейчас workflow активен, но падает с ошибкой.
                       Решение не принято.
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

### Workflow «MAIA Pulse» (id feO6o5leFOJREydC)
Создан 03.11 скриптом ~/pulse_workflow_create.py v6
(sha256 1ab5e0087582a0cc14d62ed3b20285ed2c5c6f73297d2670065919f69ac6bd58,
363 строки, 20396 байт). Заменён на v8 09.11 (PUT v3, выполнил
Архитектор, exit 0; versionId def4d40c-c5e6-4120-a6ae-b1e64f12a52c →
93dc2c77-aca5-4468-bb87-6c4331bf7d70). 15 узлов, 11 connections.
Ключевое в v8: узел «IF insert ok» между pulse_insert и
telegram_send (условие $json.success equals true; ветка false идёт
в failures pulse без отправки), parse_mode HTML + экранирование
&,<,>, appendAttribution false, onError continueRegularOutput у
pulse_insert, пробел между закрывающими скобками в вложенных
выражениях (} }), чтобы splitExpression не резал по первой паре }}.
Ручной запуск 28177: success, сводка за 2026-11-06 доставлена в
Telegram, в pulse_log одна строка sent, новых failures нет.
Расписание (минуты, интервал 30) включено Архитектором осознанно;
по его слову он перед этим выключил Active и сделал ещё один
прогон, состояние после этого командой не проверено ([СО СЛОВ]).
Экспорт v6: ~/maia_project/backups/wf_maia_pulse_20261103.json
(16761 байт, права 600, sha256 afa286931595c42251a84d5152ed6adeb505d3807a2e85beead42b4389164351).
Применены: ~/pulse_workflow_create_v8.py (sha256 083cf9a7e49b1360ebb397994d39e3d60fb473fdaed6f58614b639deb4d0b2e9, 407 строк / 23284 байт) и ~/pulse_workflow_update.py v3 (sha256 64847e98cf723012fe3cc317801ac673b98ed3c375e140a0efc9747862913014, 201 строка / 8776 байт). Прежние v7 и PUT v2 заменены, к применению не предназначены (путь ~/pulse_workflow_update.py перезаписан v3). В docstring PUT v3 остались упоминания «v7» (строки 9, 10, 119), только комментарии, поведение не меняют. Резервная копия v6 перед PUT: ~/maia_project/backups/wf_maia_pulse_before_update_20261109T225015.json (16819 байт, права 600).
Причина прошлых ошибок: executions 28099 и 28101 — Bad Request
«can't parse entities» (Markdown по умолчанию, 7 подчёркиваний в
тексте); 28093 — сетевой обрыв соединения, причина обрыва
НЕИЗВЕСТНО. Лечение в v8: parse_mode HTML + экранирование.

### Workflow «MAIA Digest» и таблица digest_log
Создано 10.11. Таблица digest_log: 8 колонок (id serial PK, d date,
article_id integer, kind text default 'top', status text default
'pending', payload jsonb, sent_at timestamptz, created_at timestamptz),
UNIQUE (article_id), CHECK kind IN (top, probe), CHECK status IN
(pending, sent), индексы по d и status. Скрипт создания:
~/digest_log_create_v2.sql (sha256 9a3b20d2cf20b5d4aa3ed7f02aeb639eb51f95adda42fdc27921e0c0671323d9,
47 строк / 3095 байт). Только добавление строк (I2), UPDATE только
pending→sent и перезапись pending при повторной вставке.

Workflow «MAIA Digest», id AFNTI7dZuu2RNF6b, versionId
60a2671d-8980-4559-9099-077dc7021e0b, 22 узла, 15 connections.
Создан 10.11 скриптом ~/digest_workflow_create_v2.py (sha256
9396c0bb27af402474eb9cd456f09f066549d7f3c554b3d121afff063dcc2e3d,
407 строк / 22815 байт), POST через Python (L7), без активации.
Credentials: Postgres Vf0XETwZzK1j2um6, Telegram VfyxOKPqufFLW88j.

Схема: триггер каждые 30 минут; выпуск раз в сутки при первом
запуске после 09:00 Europe/Kyiv, если за сегодня нет sent-строк;
пул — arxiv_cs_ai и arxiv_cs_lg (оценки новой модели с 26.10, без
meta.groq.rescored, статусы relevant/irrelevant/processed, Announce
Type не replace и не replace-cross, received_at не старше 7 суток,
статьи нет в digest_log как sent); топ-15 по score без порога; при
равных score cs.AI и cs.LG чередуются, внутри источника свежие
вперёд; по понедельникам — проба из 3–5 случайных отсечённых
статей (kind probe). Без LLM. Сообщение: parse_mode HTML,
экранирование &, <, >, до 4096 символов. Названия статей — на
английском (решение Архитектора 10.11, известное ограничение, не
задача).

Защита от повторов и блэкаутов: статьи при сбое отправки остаются
pending и возвращаются в пул; update sent отмечает только id этого
выпуска; узел «IF insert ok» перед Telegram; ошибка запроса пула
пишется в failures (provider digest_pool), пустой пул не пишется.
Обрыв питания между отправкой и отметкой sent даёт один повтор
сводки («хотя бы один раз»). Окно пула 7 суток: при простое дольше
недели статьи выпадают. Граница суток: худший случай — пропуск дня,
не дубль.

Ручной запуск 28209 (10.11, 13:23 EET): success, все узлы по
true-веткам, Telegram ok=true, длина текста 2251, в digest_log 15
строк sent (d 2026-11-10, kind top), pending 0; все 15 статей в
incoming_messages со score и Announce Type не replace/replace-cross;
новых failures нет; Digest остался active=False; Analyzer и MAIA
Pulse не тронуты.

L3-бэкап: ~/maia_project/backups/2026-11-10_pre_digest/ (дамп
maia_memory 13552248 байт, права 600; экспорт 9 workflow;
SHA256SUMS; секретов 0; каталог 700).

Данные для графика: на 10.11 пул — 29 статей; до этого сводка за
день на новой модели собиралась из 145–201 оценённых статей.
failures max id = 29 на 10.11 по отчёту АДМ MS 204; содержимое
id 27–29 в отчёты не вошло — НЕИЗВЕСТНО.

Расписание: [СО СЛОВ Архитектора] Active включён («Active включен,
передавай контрольную точку АРХИВАТОРУ», 10.11.2026); командой
состояние active не проверено. Первый выпуск по расписанию ожидается
11.11 после 09:00; сегодняшний выпуск уже в журнале.

### Credential n8n — только id
«MAIA Notifier Bot» — id VfyxOKPqufFLW88j, тип telegramApi.
«Postgres account» — id Vf0XETwZzK1j2um6, тип postgres. Регистр
букв важен. Значения секретов не вносятся (I7).

### Живая схема MAIA Map (служба)
Интерактивная 3D-схема работы системы. Каталог ~/maia-map
(server.py 533 строки, sha256 175d2347d16a1b0b57f2f29d0f5b4652379388b9f9f35eceb1706d906021b34c;
index.html; app.js; vendor/3d-force-graph.min.js — библиотека
3d-force-graph 1.80.2, лицензия MIT, локальный файл без обращения к
интернету). Адрес http://127.0.0.1:3002, слушает только 127.0.0.1,
только GET, проверка заголовка Host, заголовок
Content-Security-Policy, пароля нет (решение Архитектора по I14).

Данные собирает сама и отдаёт браузеру только метаданные:
контейнеры (docker, нагрузка из Glances), узлы и связи workflow
(n8n API, только GET; ключ читается из ~/maia_project/.env, в
браузер не уходит), последние запуски, состояние ядра (/health).
Опрос раз в 10 секунд. На схеме 82 узла и 176 связей на 13.11:
6 контейнеров, 70 узлов workflow, 6 внешних сервисов; неактивные
workflow скрыты.

Анимация трёх видов (подписана на странице): нагрузка контейнеров
(реальная), проверка состояния каждые 10 с (не трафик), запуски
n8n (по данным запусков). Режимы «Демо» и «Повтор 60 минут».

Автозапуск: systemd-юнит /etc/systemd/system/maia-map.service
(sha256 409cf8ad667e56b4bba0283cda27b3553db672714db7e4cb18f3d50dd434e3c4),
User=shaen, Restart=on-failure, RestartSec=5, вывод в
~/maia-map/map.log. Команды: sudo systemctl start|stop|restart
maia-map.service; откат: disable --now, удалить юнит,
daemon-reload. Проверки АДМ 13.11: защита отдачи файлов (выход из
каталога vendor невозможен, 14 запросов), восстановление после
kill -9 за 5 с. Проверка после перезагрузки — за Архитектором.

Прототип страницы через n8n (workflow «MAIA Map», id
R1AoF8dqm1TNFpMd) Архитектор отверг как неподходящий: он
неактивный, остаётся как есть (не включать, не менять, не
удалять). Файл ~/maia_map_preview.html — устаревший макет.

Схема показала: workflow HackerNews активен, последний запуск
(12.11 14:00) завершился ошибкой. См. уточнение в разделе N8N —
СТАТУС ИСТОЧНИКОВ (выше) и в ИЗВЕСТНЫХ ПРОБЛЕМАХ (ниже).

Решение Архитектора: схема строится отдельной службой, а не
React-приложением по адресу /topology/data в nc_core.py (L14, I14).

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
  failures            — 23 строки (id 1–26; id 14, 18, 24 в
                       текущем выводе отсутствуют, причина не
                       расследована). Исторические (по выводу SELECT
                       от 08.11): id 1 — human_reject, 23.02;
                       id 2 — test error from hooks, 06.05,
                       провайдер не указан; id 3 — _rescore_suggest
                       (uuid = integer), 06.05, провайдер не указан,
                       по AUDIT_2026-07 — ошибка /challenge; id 4 —
                       test (РЧ1), 04.09; id 5 — forwarder, 27.09.
                       id 6 — тест (ENOTFOUND api.openalex.invalid);
                       id 7–13 — openalex_enrich и forwarder
                       09.10–21.10. id 15 — analyzer_openrouter,
                       ENOTFOUND openrouter.invalid (тестовая по
                       домену). id 16, 17 — 29.10, analyzer_parse,
                       «Service temporarily…» — журнальные I6 (см.
                       «АРХИТЕКТУРА СКОРИНГА», «I6 в Analyzer»).
                       id 19–25 — 01.11: analyzer_parse (19, 23, 25),
                       analyzer_openrouter (22, EAI_AGAIN), forwarder
                       (20), openalex_enrich (21). id 26 — 09.11
                       20:35:15, analyzer_parse, «Body error: A Timeout
                       Occurred» (до PUT v3). Схема по факту: id serial,
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

### pulse_log и pulse_config (созданы 01.11.2026)
Созданы 01.11 скриптом ~/pulse_apply.py v8
(sha256 3c8065966daec780b36836f790760326798c603a59e207cb25ff43b5ed8eb842,
395 строк, 19918 байт), применён Архитектором. L3-бэкап:
backups/2026-11-01_pre_pulse/ (дамп maia_memory.dump 11962097 байт,
sha256 ebbef018ccbb226e2a7b5143ad0bd73c5faadb4a9f907730bbc5bc432164bbc1;
9 workflow в workflows/; SHA256SUMS; секретов 0).
pulse_log: 6 колонок, 3 ограничения; статус pending|sent; UNIQUE(d).
На момент запуска 28177 (09.11) — 1 строка sent (d=2026-11-06);
дальнейшее не измерено.
pulse_config: 4 колонки; telegram_chat_id — длина 10 (значение в
документы не вносится, I7).

digest_log: 8 колонок, 2 CHECK, UNIQUE (article_id), индексы по d и
status; на момент запуска 28209 (10.11) — 15 строк sent
(d 2026-11-10, kind top), pending 0; дальнейшее не измерено.

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
 
ХОСТ (инвентаризация 13.10; хост-ветка частично закрыта, см. DECISIONS.md):
⚠️ Тома Docker/Postgres и бэкапы — на одном старом HDD sdb (58510 ч,
   5121 включений, CRC 1332, 13 ATA-ошибок давних, data_checksums=off);
   копия бэкапов на Kingston — 11.11 (см. «Права на бэкапы»)
✅ Е. Копия бэкапов на другой диск — выполнена 11.11
⚠️ GDM suspend на экране входа (targets не masked) — отложено
⚠️ Kingston /mnt/shared смонтирован ro (fstab; прежний флаг
   «грязный» снят 11.11; на время копии диск включался на запись и
   возвращён в ro). Свободно 203G из 448G. Диск игровой, нужен из
   Windows.
⚠️ Б (GDM suspend), автозапуск при подаче питания, ИБП, перенос
   томов на SSD — отложены решением Архитектора; настройки GDM не
   менялись
⚠️ BIOS Restore AC Power Loss не измерен
⚠️ ИБП нет или не виден
⚠️ Темп enrichment 25-43 запуска/сутки из 96 — следствие простоев хоста
 
СИСТЕМА:
⚠️ Приток в очередь (700-2900 строк/сутки) превышает бюджет Analyzer
   (480/сутки) — решение принято, archived_unscored не реализован
⚠️ Telegram forwarder — 5 из 7 каналов не работают, не трогать до
   Шага 3 плана перестройки
⚠️ HackerNews — workflow активен и падает с ошибкой (12.11 14:00);
   ошибки в failures не попадают (I6 для сборщиков не выполнен);
   решение не принято
⚠️ I6 failures — Analyzer и сборщики не пишут в failures напрямую;
   простой Analyzer 20-26.10 это показал: ошибки не попали
⚠️ Нода «OpenRouter API»: не заданы timeout, retryOnFail, maxTries,
   onError (дефолты n8n); 29.10–01.11 пять записей analyzer_parse:
   id 16, 17 — «Upstream error from Nvidia: Service temporarily
   overloaded» (29.10); id 19 — «Internal server error»;
   id 23, 25 — «Service temporarily overloaded» (01.11);
   id 26 (09.11) — «Body error: A Timeout Occurred»
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
⚠️ Ротация пароля БД не проводилась (AUDIT_2026-09, п. 9.4) —
   решение за Архитектором
⚠️ Сбор arXiv 29.10 проверен частично: один запуск, все 407 id
   уже в БД, потерь не обнаружено; проверка попадания статей,
   объявленных 29.10, не выполнена
⚠️ Дайджест — сообщение на английском (названия arXiv не
   переводятся; решение Архитектора 10.11, не задача)
⚠️ Окно пула дайджеста — 7 суток: при простое дольше недели
   статьи выпадают (не задача)
⚠️ Обрыв между отправкой и отметкой sent в дайджесте допускает
   один повтор сводки («хотя бы один раз»)
 
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
