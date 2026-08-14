# MAIA — документация

NOVA CORTEX / MAIA — Adaptive Signal Web.
Система квалификации и памяти сигналов из AI/крипто среды.

Единая точка входа для LLM-моделей с web-доступом.

## ТЕКУЩИЙ ПРИОРИТЕТ

Стабилизация ядра (см. ROADMAP.md, раздел "ПЛАН ВОССТАНОВЛЕНИЯ").

## Документы (читать в этом порядке)

| Файл | Тип | Описание | Ссылка |
|---|---|---|---|
| CLAUDE.md | Точка входа | Читать первым | [CLAUDE.md](https://github.com/shaenxxx-commits/maia/blob/main/CLAUDE.md) |
| RUNTIME.md | Состояние | Живое состояние системы | [RUNTIME.md](https://github.com/shaenxxx-commits/maia/blob/main/RUNTIME.md) |
| INVARIANTS.md | Правила | Нельзя нарушать | [INVARIANTS.md](https://github.com/shaenxxx-commits/maia/blob/main/INVARIANTS.md) |
| DECISIONS.md | История | Решения и обоснования | [DECISIONS.md](https://github.com/shaenxxx-commits/maia/blob/main/DECISIONS.md) |
| ROADMAP.md | План | Этапы и приоритеты | [ROADMAP.md](https://github.com/shaenxxx-commits/maia/blob/main/ROADMAP.md) |
| AUDIT_2026-07.md | Диагностика | Обязателен перед задачами с БД/n8n/nc_core.py/forwarder | [AUDIT_2026-07.md](https://github.com/shaenxxx-commits/maia/blob/main/AUDIT_2026-07.md) |
| TRANSITION.md | План перехода | Активен. Удалить только после завершения Шага 6 (DDA) | [TRANSITION.md](https://github.com/shaenxxx-commits/maia/blob/main/TRANSITION.md) |
| VISION.md | Философия | Не рабочий документ, дальние горизонты | [VISION.md](https://github.com/shaenxxx-commits/maia/blob/main/VISION.md) |

Raw-вариант (голый текст, без HTML GitHub) — по шаблону:
```
GitHub: https://github.com/shaenxxx-commits/maia/blob/main/<FILE>
Raw:    https://raw.githubusercontent.com/shaenxxx-commits/maia/main/<FILE>
```

## ARCHIVE/ (второстепенные, не нужны для текущей стабилизации)

STACK_v10.5.md · PROMPTS_v3.3.md · CONFIG_PLAN_v1.5.md

## Статус разделов

| Раздел | Статус |
|---|---|
| Основные документы (корень) | Активен |
| ARCHIVE/ | Второстепенно, не приоритет |

## Синхронизация

Сервер (`~/maia_project/docs/`) — источник истины. Этот
репозиторий и claude.ai Projects — синхронные зеркала: любая
правка вносится на сервер И сюда И в Projects в рамках одного
цикла, без разрыва между источниками. Не оставлять зеркала
устаревшими "до следующего раза" — именно это дважды создавало
расхождение в предыдущем репозитории (см. posoh-docs: VISION.md
был подменён контентом Sub-0, ROADMAP.md не содержал отметку
о завершённом Шаге 4).

Last synced from server: pending verification
(обновлять датой при каждом цикле синхронизации, не оставлять
"pending" после первого реального цикла)

## Смежные проекты

Проект POSOH/Sub-0 — отдельный, не входит в этот репозиторий.
См. https://github.com/shaenxxx-commits/posoh

## Временный статус

Структура будет пересмотрена после завершения Шага 5 плана
восстановления (ROADMAP.md). Пересмотр не означает автоматического
удаления TRANSITION.md — его статус определяется отдельным
условием (завершение Шага 6).
