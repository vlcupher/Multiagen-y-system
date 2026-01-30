# Multiagency-system
Проект реализован в рамках мега школы итмо по треку мультиагентные системы
Репозиторий содержит Jupyter Notebook с прототипом мультиагентной системы для проведения технического интервью. Диалог оркестрируется графом состояний; система сохраняет контекст, адаптирует сложность вопросов и формирует итоговый структурированный фидбэк.

## Быстрый запуск
### Требования
- Python 3.10+
- Jupyter Notebook / JupyterLab

### Установка зависимостей
```bash
pip install langchain langchain-openai langchain-anthropic langchain-mistralai langchain-google-genai langgraph python-dotenv
```

### Конфигурация LLM
Провайдер выбирается через переменную окружения `LLM_PROVIDER` (по умолчанию: `deepseek`).

Поддерживаемые варианты в `get_llm()`:
- `anthropic`, `gigachat`, `mistral`, `gemini`, `deepseek` (и fallback на `gpt-4o-mini`)

Ожидаемые переменные окружения (по провайдеру):
- `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `MISTRAL_API_KEY`, `GOOGLE_API_KEY`, `DEEPSEEK_API_KEY`

Примечание: в ветке `deepseek` ключ API задан в коде при создании клиента; для корректного использования рекомендуется заменить на чтение из переменной окружения.

## Архитектура (фактическая реализация)
Оркестрация реализована через `StateGraph(InterviewState)` и скомпилированный граф `interview_graph`.

### Узлы графа
- `router` — проверка условий остановки и маршрутизация
- `off_topic_check` — детекция off-topic и обновление `off_topic_count`
- `redirect_save` — сообщение возврата в русло и логирование хода при `off_topic_count >= 2`
- `observer` — анализ ответа, выбор темы/сложности, инструкция интервьюеру (внутренний слой)
- `evaluator` — скрытая оценка качества ответа (`score` в диапазоне 0..1)
- `interviewer` — генерация видимого вопроса/реплики
- `save_turn` — фиксация хода в `turns`
- `log_stop_turn` — технический ход завершения (по стоп-фразе)
- `feedback` — генерация итогового отчёта и запись в `final_feedback`

### Состояние (InterviewState)
Ключевые поля:
- `messages` — история сообщений
- `position`, `grade`, `experience`, `candidate_name`
- `current_difficulty` (`easy|medium|hard`), `topics_covered`
- `internal_thoughts` — внутренняя «рефлексия» агентов (используется для логов)
- `turns`, `performance_history`, `off_topic_count`, `final_feedback`

## Управление сессией
### Стоп-фразы
Остановка распознаётся по вхождению одной из строк:
`стоп интервью`, `завершить интервью`, `закончить`, `стоп`, `конец интервью`, `finish`.

### Точки входа
- `start_interview()` — инициализация состояния и первое сообщение
- `step_interview(state, user_message)` — один шаг диалога через `interview_graph.invoke()`
- `run_interview_loop(max_turns=15)` — интерактивный цикл на `input()`
- `save_interview_log(state, filepath="interview_log.json", strict_format=False)` — сохранение лога

## Логи
Лог сохраняется в JSON (по умолчанию `interview_log.json`):

```json
{
  "participant_name": "...",
  "turns": [],
  "final_feedback": "..."
}
```

Ход интервью в `turns` (типовой набор полей):
- `turn_id`
- `agent_visible_message` — предыдущий вопрос/реплика интервьюера, на который отвечал кандидат
- `user_message` — ответ кандидата
- `internal_thoughts` — внутренние сообщения (Observer/Evaluator/служебные фрагменты)
- `performance_metrics` — например `{"score": 0.0..1.0}`

При `strict_format=True` в `turns` сохраняются только: `turn_id`, `agent_visible_message`, `user_message`, `internal_thoughts`.
