# MAIA — документация

NOVA CORTEX / MAIA — Adaptive Signal Web.
Система квалификации и памяти сигналов из среды AI research
(arXiv, GitHub; OpenAlex как слой обогащения).

Единая точка входа для LLM-моделей с web-доступом.

## ТЕКУЩИЙ ПРИОРИТЕТ

Перестройка ведения документации (см. DECISIONS.md, раздел
"ПЕРЕСТРОЙКА ВЕДЕНИЯ ДОКУМЕНТАЦИИ (2026-10)"). По системе: замер
enrichment около 20-21.10 и вектор "закрепление хоста"
(см. ROADMAP.md и RUNTIME.md).

## Документы (читать в этом порядке)

| Файл | Тип | Описание | Ссылка |
|---|---|---|---|
| CLAUDE.md | Точка входа | Читать первым | [CLAUDE.md](https://github.com/shaenxxx-commits/maia/blob/main/CLAUDE.md) |
| RUNTIME.md | Состояние | Живое состояние системы | [RUNTIME.md](https://github.com/shaenxxx-commits/maia/blob/main/RUNTIME.md) |
| INVARIANTS.md | Правила | Нельзя нарушать | [INVARIANTS.md](https://github.com/shaenxxx-commits/maia/blob/main/INVARIANTS.md) |
| DECISIONS.md | История | Решения и обоснования, матрица прав на документы | [DECISIONS.md](https://github.com/shaenxxx-commits/maia/blob/main/DECISIONS.md) |
| ROADMAP.md | План | Этапы и приоритеты | [ROADMAP.md](https://github.com/shaenxxx-commits/maia/blob/main/ROADMAP.md) |
| PROMPTS_CHAT_LAUNCH.md | Роли | Промпты и порядок запуска чатов | [PROMPTS_CHAT_LAUNCH.md](https://github.com/shaenxxx-commits/maia/blob/main/PROMPTS_CHAT_LAUNCH.md) |
| AUDIT_2026-09-ФАНТОМЫ.md | Диагностика | Читать перед задачами с БД, n8n, nc_core.py, forwarder, credentials | [AUDIT_2026-09-ФАНТОМЫ.md](https://github.com/shaenxxx-commits/maia/blob/main/AUDIT_2026-09-%D0%A4%D0%90%D0%9D%D0%A2%D0%9E%D0%9C%D0%AB.md) |
| AUDIT_2026-07.md | Снимок | Исторический аудит 2026-07, не обновляется | [AUDIT_2026-07.md](https://github.com/shaenxxx-commits/maia/blob/main/AUDIT_2026-07.md) |
| CONFIG_PLAN_v1.5.md | Конфигурация | Конфигурация Claude-слоёв проекта | [CONFIG_PLAN_v1.5.md](https://github.com/shaenxxx-commits/maia/blob/main/CONFIG_PLAN_v1.5.md) |
| TRANSITION.md | План перехода | Активен. Удалить только после завершения Шага 6 (DDA) | [TRANSITION.md](https://github.com/shaenxxx-commits/maia/blob/main/TRANSITION.md) |
| VISION.md | Философия | Не рабочий документ, дальние горизонты | [VISION.md](https://github.com/shaenxxx-commits/maia/blob/main/VISION.md) |
| HANDOFF.md | Снимок | Передача между чатами КОНЦЕПТ, не обновляется | [HANDOFF.md](https://github.com/shaenxxx-commits/maia/blob/main/HANDOFF.md) |

Шаблон ссылок на отдельный файл:

    GitHub: https://github.com/shaenxxx-commits/maia/blob/main/<FILE>
    Raw:    https://raw.githubusercontent.com/shaenxxx-commits/maia/main/<FILE>

## ARCHIVE/ (второстепенные, не нужны для текущей работы)

STACK_v10.5.md · PROMPTS_v3.4.md · CONFIG_PLAN_v1.5.md (историческая
версия от 2026-05-31)

## Статус разделов

| Раздел | Статус |
|---|---|
| Основные документы (корень) | Активен |
| ARCHIVE/ | Второстепенно, не приоритет |

## Синхронизация

Сервер (`~/maia_project/docs/`, репозиторий docs/.git) — источник
истины. Этот репозиторий — зеркало: правка вносится на сервер и
отправляется сюда администратором после санкции Архитектора или
КОНЦЕПТ и после проверки на секреты (I7). Не оставлять зеркало
устаревшим "до следующего раза".

claude.ai Projects не является обязательным зеркалом
(решение Архитектора, DECISIONS.md, 2026-10-15); его использование
не определено.

Last synced from server: 2026-10-17
(обновлять датой при каждом цикле синхронизации)

## Смежные проекты

Проект POSOH/Sub-0 — отдельный, не входит в этот репозиторий.
См. https://github.com/shaenxxx-commits/posoh

## Временный статус

Идёт перестройка ведения документации; структура этого файла и
состав документов могут меняться. Статус TRANSITION.md определяется
отдельным условием (завершение Шага 6).
