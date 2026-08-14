# NOVA CORTEX / MAIA — CLAUDE CONFIGURATION PLAN v1.5
Дата: 2026-05-31

---

СЛОЙ 1 — ПРОЕКТ (claude.ai Projects)

Project Instructions (системный промпт):

    Ты работаешь внутри проекта NOVA CORTEX / MAIA.
    Главный документ: CLAUDE.md — читай его первым.

    ИЕРАРХИЯ ЧАТОВ:
      КОНЦЕПТ   — стратегический надзор, хранитель концепции
      УЧ N      — оперативное управление рабочими чатами
      РЧ N      — исполнение одной задачи

    ПРАВИЛА ДЛЯ ВСЕХ ЧАТОВ:
      — I2: никаких DELETE в PostgreSQL, только append
      — I6: все ошибки в таблицу failures
      — I7: личные данные нигде, токены только в .env
      — I13: не строить коллекторы — интегрировать готовое
      — Перед docker rm: экспорт workflows + бэкап volumes (L3)
      — PUT workflow в n8n: только через Python (L7)
      — CLAUDE.md — точка входа, единственный источник истины

    ДЛЯ РАБОЧИХ ЧАТОВ:
      — Получил задачу → выполнил → отчёт в УЧ
      — Нестандартное решение → сообщить в УЧ до действия
      — Начинать нумерацию с МС 1 независимо от других чатов

---

СЛОЙ 2 — CLAUDE CODE

CLAUDE.md — актуальная версия
Путь: /home/shaen/maia_project/docs/CLAUDE.md

settings.json
Путь: /home/shaen/maia_project/.claude/settings.json

    {
      "permissions": {
        "deny": [
          ".env",
          "*.key",
          "sessions/*.session",
          "secrets/*",
          "**/credentials*"
        ]
      },
      "model": "opusplan"
    }

Hooks
Путь: /home/shaen/maia_project/.claude/hooks/

pre-sql.sh — проверка I2:

    #!/bin/bash
    if echo "$1" | grep -qi "DELETE\|DROP TABLE\|TRUNCATE"; then
      echo "BLOCKED: I2 violation — DELETE/DROP запрещены в MAIA"
      exit 1
    fi

post-error.sh — логирование в failures:

    #!/bin/bash
    # ⚠️ Исправлено 2026-08-11: оригинал использовал несуществующую
    # схему (error_text, created_at) — тот же баг, что AUDIT_2026-07
    # нашёл в load_scoring_prompt(). Реальная схема failures:
    # id, action_hash (UNIQUE), provider, error, context (json),
    # timestamp (default now()). См. DECISIONS.md: "failures —
    # унифицировать схему INSERT" (в очереди Шага 2). Хеширование
    # action_hash ниже — временная заплатка, не финальное решение;
    # реальная логика должна определиться в рамках Шага 2.
    docker exec maia-db psql -U maia_user -d maia_memory \
      -c "INSERT INTO failures (action_hash, provider, error, context, timestamp) \
          VALUES (md5('$1'||NOW()::text), 'hook', '$1', '{}'::jsonb, NOW()) \
          ON CONFLICT (action_hash) DO NOTHING;"

pre-docker.sh — проверка L3:

    #!/bin/bash
    if echo "$1" | grep -qi "docker rm\|docker-compose down"; then
      echo "BLOCKED: L3 — экспортируй workflows и сделай бэкап volumes"
      echo "Подтверди: export + backup выполнены (yes/no)"
      read confirm
      if [ "$confirm" != "yes" ]; then exit 1; fi
    fi

pre-n8n.sh — проверка L7:

    #!/bin/bash
    if echo "$1" | grep -qi "PUT.*workflow\|curl.*n8n"; then
      echo "BLOCKED: L7 — PUT workflow только через Python, не curl/UI"
      exit 1
    fi

---

СЛОЙ 3 — МОДЕЛИ ПО ЗАДАЧАМ

    /model opusplan   — архитектурные сессии
    /model sonnet     — оперативные задачи (код, фиксы)
    /model haiku      — механические задачи (шаблоны, переименования)

Правило выбора:
    Архитектура / анализ / стек    → opusplan
    Код / миграция / дебаг         → sonnet
    Повторяющиеся операции         → haiku

---

СЛОЙ 4 — MCP СЕРВЕРЫ

Подключены:
    Google Drive    — хранить и синхронизировать документы проекта
    Gmail           — уведомления о критических событиях
    Google Calendar — планирование этапов

---

СЛОЙ 5 — ИЕРАРХИЯ ЧАТОВ

    КОНЦЕПТ (claude.ai)
      Модель:    Sonnet
      Задача:    стратегия · архитектура · документы · надзор за УЧ
      Документы: только КОНЦЕПТ создаёт и изменяет

      УЧ N (claude.ai)
        Модель:  Sonnet
        Задача:  координация РЧ · оперативные решения
        Обновляет RUNTIME.md после значимых сессий

        РЧ N (claude.ai или терминал на сервере)
          Модель:    sonnet / opusplan (если Claude Code)
          Задача:    одна конкретная задача
          Инструмент: терминал (основной) · Claude Code (опция)
          Нумерация: МС 1, 2, 3... независимая

---

СЛОЙ 6 — ДОКУМЕНТАЦИЯ ПРОЕКТА

Структура:
    CLAUDE.md     — точка входа · читать первым
    RUNTIME.md    — живое состояние системы
    INVARIANTS.md — что нельзя нарушать
    DECISIONS.md  — что решили и почему
    ROADMAP.md    — этапы и условия выхода
    VISION.md     — философия и дальние горизонты
    TRANSITION.md — план перехода к DDA (временный)

Хранение (обновлено 2026-08-11 — было "сервер + Projects",
теперь три синхронных зеркала, см. DECISIONS.md):
    Сервер /home/shaen/maia_project/docs/ — источник истины
    GitHub-репозиторий — синхронное зеркало
    claude.ai Projects — синхронное зеркало
    История: git

Архив:
    STACK v10.5   — не обновлять, хранить как исторический документ
    CHECKLIST v2.1 — заменён на ROADMAP.md + RUNTIME.md · удалить

---

WATCHOUTS

    ⚠️ opusplan: иногда не переключается на Sonnet автоматически.
      Следить за расходом токенов. Если не отключается → /model sonnet.

    ⚠️ settings.json deny не защищает если файл явно передан в контекст.
      Не передавать .env руками.

    ⚠️ Hooks — экспериментальная функция Claude Code.
      Тестировать на безопасных командах перед боевым использованием.

    ⚠️ OpenRouter: залито $10, лимит 1000/день (обновлено — было
      "работаем на бесплатных моделях", см. DECISIONS.md раздел LLM).

    ⚠️ Watchdog: ложные алерты на maia-db — тюнинг отложен.

    ⚠️ Claude Code: недоступен на Free плане (обновлено — было
      "опциональный инструмент", см. DECISIONS.md раздел ИНСТРУМЕНТЫ).
      Основной workflow — терминал напрямую.

    ⚠️ Контекст-репортинг УЧ/РЧ ненадёжен — модель завышает цифру.
      Финальное решение о смене чата принимает Архитектор
      по реальному заполнению в интерфейсе.

---

MAIA Claude Configuration Plan v1.5
Обновляется при смене инфраструктуры или структуры документации.
