# DeepEval System Architecture Documentation

## 1. Назначение и основные сценарии использования системы

### 1.1 Назначение
DeepEval — это фреймворк для оценки и тестирования больших языковых моделей (LLM) и систем на их основе. Аналогичен Pytest, но специализирован для унит-тестирования выходов LLM. Система позволяет разработчикам легко определять оптимальные модели, промпты и архитектуру для улучшения RAG-пайплайнов, агентных рабочих процессов, предотвращения дрейфа промптов и перехода на самохостинг моделей с уверенностью.

### 1.2 Основные сценарии использования

1. **End-to-end оценка LLM-приложений**
   - Тестирование RAG-пайплайнов (Retrieval-Augmented Generation)
   - Оценка чат-ботов и диалоговых систем
   - Тестирование AI-агентов с инструментами
   - Интеграция с LangChain и LlamaIndex

2. **Компонент-уровневая оценка**
   - Трассировка отдельных компонентов (LLM-вызовы, retrievers, tool calls, агенты)
   - Прикладное применение метрик на уровне компонентов
   - Ненарушащая трассировка без рефакторинга кода

3. **Оценка датасетов в пакетном режиме**
   - Создание и аннотация датасетов для оценки
   - Бесшовная интеграция с CI/CD окружениями
   - Сравнение итераций моделей и промптов

4. **Red Teaming и безопасность**
   - Выявление 40+ уязвимостей безопасности
   - Стратегии усиления атак (prompt injections и др.)
   - Проверка на токсичность, bias, SQL injection

5. **Бенчмаркинг моделей**
   - Поддержка популярных бенчмарков: MMLU, HellaSwag, DROP, BIG-Bench Hard, TruthfulQA, HumanEval, GSM8K
   - Оценка любых LLM с минимальным количеством кода

6. **Интеграция с платформой Confident AI**
   - Хранение и сравнение результатов в облаке
   - Генерация и шеринг отчётов
   - Отладка результатов через LLM-трейсы

---

## 2. Основные сервисы системы и карта взаимодействия между ними

### 2.1 Архитектурная диаграмма

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DeepEval System Architecture                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐                 │
│  │   User       │────▶│  CLI/Pytest  │────▶│  evaluate.py │                 │
│  │   Input      │     │   Interface  │     │   (Entry)    │                 │
│  └──────────────┘     └──────────────┘     └──────────────┘                 │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      Core Evaluation Engine                           │  │
│  ├───────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐                │  │
│  │  │ execute.py   │  │ evaluate.py   │  │  dataset.py  │                │  │
│  │  │ (Execution)  │  │ (Orchestration│  │ (Test Cases) │                │  │
│  │  └──────────────┘  └───────────────┘  └──────────────┘                │  │
│  │                                                                       │  │
│  │  ┌──────────────┐  ┌───────────────┐  ┌──────────────┐                │  │
│  │  │ metrics/     │  │ models/       │  │ tracing/     │                │  │
│  │  │ (30+ Metrics)│  │ (LLM Wrappers)│  │ (Component)  │                │  │
│  │  └──────────────┘  └───────────────┘  └──────────────┘                │  │
│  │                                                                       │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐                │  │
│  │  │ test_case/   │  │ test_run/    │  │ cache/        │                │  │
│  │  │ (Test Data)  │  │ (Tracking)   │  │ (Optimization)│                │  │
│  │  └──────────────┘  └──────────────┘  └───────────────┘                │  │
│  │                                                                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────┐     ┌─────────────────┐     ┌──────────────┐              │
│  │  Confident   │◀────│  TraceApi       │◀────│  TraceManager│              │
│  │   AI Platform│     │  (Serialization)│     │  (Collection)│              │
│  └──────────────┘     └─────────────────┘     └──────────────┘              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Основные сервисы и их описание

#### 2.2.1 `evaluate/` — Ядро системы оценки

| Файл | Описание |
|------|----------|
| `evaluate.py` | Главный входной пункт. Функции `assert_test()` (синхронный тест) и `evaluate()` (пакетная оценка). Координирует весь процесс оценки. |
| `execute.py` | Реализация логики выполнения тестов. Содержит синхронные и асинхронные исполнители для LLM, MLLM и conversational тест-кейсов. Поддержка компонент-уровневой оценки через трейсинг. |
| `types.py` | Типизация результатов оценки (`TestResult`, `EvaluationResult`). |
| `utils.py` | Утилиты для валидации, агрегации метрик и вывода результатов. |
| `configs.py` | Конфигурационные классы: `AsyncConfig`, `DisplayConfig`, `CacheConfig`, `ErrorConfig`. |

#### 2.2.2 `metrics/` — Система метрик (30+ типов)

| Категория | Метрики |
|-----------|---------|
| **Benchmarks** | MMLU, GSM8K, HellaSwag, HumanEval, TruthfulQA |
| **RAG** | AnswerRelevancy, Faithfulness, ContextualRecall/Precision/Relevancy |
| **Content Quality** | GEval, Hallucination, Summarization |
| **Safety** | Toxicity, Bias, PIILeakage, NonAdvice, Misuse, RoleViolation |
| **Agentic** | TaskCompletion, ToolCorrectness |
| **Conversational** | TurnRelevancy, ConversationCompleteness, KnowledgeRetention, RoleAdherence |
| **MCP** | MCPTaskCompletion, MCPUse, MultiTurnMCPUse |
| **Multimodal** | TextToImageMetric, MultimodalGEval и др. |

Базовые классы: `BaseMetric`, `BaseConversationalMetric`, `BaseMultimodalMetric`, `BaseArenaMetric`.

#### 2.2.3 `models/` — Обёртки LLM-моделей

| Модуль | Описание |
|--------|----------|
| `llms/` | GPTModel, AzureOpenAIModel, OllamaModel, AnthropicModel, GeminiModel, AmazonBedrockModel, LiteLLMModel, KimiModel, DeepSeekModel, LocalModel |
| `embedding_models/` | OpenAIEmbeddingModel, AzureOpenAIEmbeddingModel, OllamaEmbeddingModel, LocalEmbeddingModel |
| `mlllms/` | MultimodalOpenAIModel, MultimodalOllamaModel, MultimodalGeminiModel |

Базовые классы: `DeepEvalBaseLLM`, `DeepEvalBaseMLLM`, `DeepEvalBaseEmbeddingModel`.

#### 2.2.4 `tracing/` — Система трассировки компонентов

| Файл | Описание |
|------|----------|
| `tracing.py` | `TraceManager` — сбор и постинг трейсов в Confident AI. Поддержка выборки (sampling). |
| `types.py` | Типы трейсов: `Trace`, `BaseSpan`, `LlmSpan`, `AgentSpan`, `RetrieverSpan`, `ToolSpan`. |
| `patchers.py` | Патчинг внешних библиотек (OpenAI, LangChain, LlamaIndex) для автоматической трассировки. |
| `api.py` | Serialisation трейсов для отправки в Confident AI (`TraceApi`, `BaseApiSpan`). |
| `context.py` | Thread-local контекст для получения активного трейса/спана (`current_trace_context`, `current_span_context`). |

#### 2.2.5 `dataset/` — Управление тестовыми данными

| Файл | Описание |
|------|----------|
| `dataset.py` | `EvaluationDataset` — контейнер для `Golden`/`ConversationalGolden`. Поддержка CSV, JSON, JSONL. |
| `golden.py` | `Golden` — тестовый пример с входными данными, ожидаемыми результатами, контекстом. |
| `api.py` | Интеграция с Confident AI API для загрузки/удаления датасетов. |

#### 2.2.6 `test_case/` — Структура тест-кейсов

| Файл | Описание |
|------|----------|
| `test_case.py` | `LLMTestCase`, `ConversationalTestCase`, `MLLMTestCase`. Поля: input, actual_output, expected_output, retrieval_context, tools_called. |

#### 2.2.7 `test_run/` — Управление сессией тестирования

| Файл | Описание |
|------|----------|
| `test_run.py` | `TestRunManager`, `TestRun`, `MetricData`. Отслеживание хода тестирования. |
| `cache.py` | Кэширование результатов метрик (`Cache`, `CachedTestCase`, `CachedMetricData`). |
| `hyperparameters.py` | Обработка гиперпараметров оценки. |

#### 2.2.8 `confident/` — Интеграция с платформой

| Файл | Описание |
|------|----------|
| `api.py` | Confident AI API client. Отправка трейсов и результатов. |

---

## 3. Сквозной пример взаимодействия сервисов

### 3.1 Сценарий: Тестирование LLM-модели в выделенном инференсе через OpenAI API

#### 3.1.1 Описание сценария
Пользователь создаёт тест-кейс для оценки LLM-приложения, использующего OpenAI API. Система выполняет:
1. Создание тест-кейса с входными данными
2. Вызов LLM через OpenAI API
3. Применение метрик для оценки выхода
4. Сбор и отправка трейсов в Confident AI (опционально)
5. Агрегация результатов

#### 3.1.2 Диаграмма состояний

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    State Diagram: LLM Evaluation Scenario                       │
└─────────────────────────────────────────────────────────────────────────────────┘

                    ┌───────────────┐
                    │   START       │
                    │  (User Input) │
                    └───────┬───────┘
                            │
                            ▼
                    ┌─────────────────────┐
                    │  LLMTestCase        │
                    │  Creation           │
                    │  - input            │
                    │  - expected_output  │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Test Execution     │
                    │  (assert_test())    │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Metric Creation    │
                    │  GEval/Other        │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Measure() Call     │
                    │  (Sync/Async)       │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼────────────────────────────────────┐
                    │  Model Inference Layer                        │
                    ├───────────────────────────────────────────────┤
                    │  ┌───────────────┐    ┌───────────────────┐   │
                    │  │ GPTModel      │───▶│ OpenAI Client     │   │
                    │  │ (LLM Wrapper) │    │ (API Call)        │   │
                    │  └───────────────┘    └───────────────────┘   │
                    │                                               │
                    │  Return: actual_output                        │
                    └───────────────────────────────────────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Metric Evaluation  │
                    │  GEval.measure()    │
                    │  - Compare output   │
                    │  - Compute score    │
                    │  - Generate reason  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Result Storage     │
                    │  MetricData         │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Trace Collection    │
                    │  (if tracing enabled)│
                    │  TraceManager.post() │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼──────────┐
                    │  Assertion Check    │
                    │  score >= threshold │
                    └──────────┬──────────┘
                               │
                ┌──────────────┴──────────────┐
                │                             │
                ▼                             ▼
       ┌──────────────┐              ┌──────────────┐
       │   PASS       │              │   FAIL       │
       │   (success)  │              │  (error)     │
       └──────┬───────┘              └──────┬───────┘
              │                             │
              ▼                             ▼
       ┌──────────────┐              ┌──────────────┐
       │ Report       │              │ Raise        │
       │ Generate     │              │ Exception    │
       └──────────────┘              └──────────────┘
```

#### 3.1.3 Последовательность вызовов (Sequence Diagram)

```
┌─────┐    ┌──────────┐       ┌──────────┐    ┌───────────┐    ┌─────────┐
│ User│    │  assert_ │       │  execute │    │ metric    │    │  model  │
│     │    │   test() │       │   ()     │    │ .measure  │    │  client │
└─────┘    └─────┬────┘       └─────┬────┘    └─────┬─────┘    └────┬────┘
                 │                  │               │               │
                 │ test_case+metrics│               │               │
                 │─────────────────>│               │               │
                 │                  │               │               │
                 │                  │ test_case     │               │
                 │                  │──────────────>│               │
                 │                  │               │               │
                 │                  │               │ output        │
                 │                  │               │──────────────>│
                 │                  │               │               │
                 │                  │               │ actual_output │
                 │                  │               │<──────────────│
                 │                  │               │               │
                 │                  │ metric.measure()              │
                 │                  │────────────────────────────────
                 │                  │               │               │
                 │                  │               │ score         │
                 │                  │               │<──────────────│
                 │                  │               │               │
                 │                  │ result        │               │
                 │                  │<──────────────│               │
                 │                  │               │               │
                 │ test_result      │               │               │
                 │<─────────────────│               │               │
                 │                  │               │               │
```

#### 3.1.4 Пример кода

```python
from deepeval import assert_test
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCase

# Создание метрики
correctness_metric = GEval(
    name="Correctness",
    criteria="Determine if the output is correct based on expected output.",
    evaluation_params=[
        LLMTestCaseParams.ACTUAL_OUTPUT,
        LLMTestCaseParams.EXPECTED_OUTPUT
    ],
    threshold=0.5
)

# Создание тест-кейса
test_case = LLMTestCase(
    input="What is the weather like today?",
    actual_output="It's sunny with a high of 25°C.",
    expected_output="The weather is sunny with temperatures around 25 degrees."
)

# Выполнение теста
assert_test(test_case, [correctness_metric])
```

---

## 4. Приоритезированный список проблем

### 4.1 Критерии приоритизации

| Критерий | Вес | Описание |
|----------|-----|----------|
| **Критичность** | 40% | Влияние на работу системы, безопасность, надёжность |
| **Трудоёмкость устранения** | 30% | Время и ресурсы, необходимые для исправления |
| **Частота возникновения** | 30% | Насколько часто проблема встречается в реальных сценариях |

### 4.2 Список проблем

#### CRITICAL (Высокая критичность, низкая трудоёмкость)

| # | Проблема | Критичность | Трудоёмкость | Частота | Описание | Решение |
|---|----------|-------------|--------------|---------|----------|---------|
| 1 | **Отсутствие валидации API ключей** | 🔴 Critical | 🟢 Low | 🔴 High | Если `confident_api_key` не указан, система продолжает работу, но трейсы теряются | Добавить валидацию на уровне `TraceManager.__init__()` и `configure()` |
| 2 | **Утечка контекста в трейсинге** | 🔴 Critical | 🟢 Low | 🟡 Medium | Thread-local контекст может пересекаться между асинхронными задачами | Использовать `asyncio.Task`-специфичный контекст |
| 3 | **Отсутствие retry для API вызовов** | 🔴 Critical | 🟢 Low | 🟡 Medium | Ошибки сети приводят к потере данных | Добавить retry mechanism в `TraceManager.post_trace()` |
| 4 | **Дублирование cache write** | 🟠 High | 🟢 Low | 🔴 High | Два вызова `cache_test_case()` могут приводить к дублированию | Объединить вызовы в один |
| 5 | **Отсутствие таймаута для LLM вызовов** | 🔴 Critical | 🟢 Low | 🔴 High | Блокирующий вызов к LLM может "зависнуть" систему | Добавить timeout в `GPTModel.invoke()` |

#### HIGH (Средняя критичность, низкая/средняя трудоёмкость)

| # | Проблема | Критичность | Трудоёмкость | Частота | Описание | Решение |
|---|----------|-------------|--------------|---------|----------|---------|
| 6 | **Отсутствие обработки rate limits** | 🟠 High | 🟢 Low | 🔴 High | OpenAI API rate limits могут привести к 429 ошибкам | Добавить exponential backoff |
| 7 | **Неверное использование skip_on_missing_params** | 🟠 High | 🟡 Medium | 🟡 Medium | Параметр может быть игнорирован в некоторых ветках | Центральный контроль в `execute_test_cases()` |
| 8 | **Отсутствие логирования ошибок метрик** | 🟠 High | 🟡 Medium | 🟡 Medium | Ошибки метрик теряются без логов | Добавить structured logging |
| 9 | **Memory leak в TraceManager** | 🟠 High | 🟡 Medium | 🟡 Medium | Traces накапливаются без очистки | Добавить cleanup на exit |
| 10 | **Отсутствие валидации input параметров** | 🟠 High | 🟡 Medium | 🟡 Medium | `None` input может привести к ошибкам в метриках | Добавить валидацию в `BaseMetric.measure()` |

#### MEDIUM (Низкая критичность, высокая трудоёмкость)

| # | Проблема | Критичность | Трудоёмкость | Частота | Описание | Решение |
|---|----------|-------------|--------------|---------|----------|---------|
| 11 | **Отсутствие поддержки Retryable Span** | 🟡 Medium | 🟠 Medium | 🟡 Medium | При ошибке span не может быть переиспользован | Добавить retryable span pattern |
| 12 | **Сложность кэширования conversational метрик** | 🟡 Medium | 🟠 Medium | 🟢 Low | Conversational метрики пока не кэшируются | Реализовать кэш для multi-turn |
| 13 | **Отсутствие интеграции с Redis** | 🟡 Medium | 🟠 Medium | 🟢 Low | Disk cache медленный для больших датасетов | Добавить Redis cache backend |
| 14 | **Отсутствие parallel execution для component evals** | 🟡 Medium | 🟠 Medium | 🟢 Low | Agentic evals выполняются последовательно | Добавить async execution |
| 15 | **Сложность отладки асинхронных трейсов** | 🟡 Medium | 🟠 Medium | 🟡 Medium | Trace ordering может быть нарушен | Добавить trace correlation IDs |

#### LOW (Низкая критичность, высокая трудоёмкость)

| # | Проблема | Критичность | Трудоёмкость | Частота | Описание | Решение |
|---|----------|-------------|--------------|---------|----------|---------|
| 16 | **Отсутствие визуализации метрик** | 🟢 Low | 🟠 Medium | 🟢 Low | Результаты выводятся текстом | Добавить chart generation |
| 17 | **Отсутствие интеграции с Slack/Email** | 🟢 Low | 🟠 Medium | 🟢 Low | Нет уведомлений о проваленных тестах | Добавить notifier |
| 18 | **Отсутствие support для custom metrics inheritance** | 🟢 Low | 🟠 High | 🟢 Low | Пользователи не могут легко расширять метрики | Добавить base template |
| 19 | **Отсутствие async support в CLI** | 🟢 Low | 🟠 High | 🟢 Low | CLI только синхронный | Добавить async CLI |
| 20 | **Отсутствие documentation для advanced usage** | 🟢 Low | 🟠 High | 🟢 Low | Сложные сценарии не документированы | Добавить advanced docs |

---

## 5. План устранения проблем

### 5.1 Таблица плана

| Приоритет | Проблема | Срок | Ответственный | Статус |
|-----------|----------|------|---------------|--------|
| **P0** | Отсутствие валидации API ключей | Week 1 | Core Team | 📝 In Progress |
| **P0** | Отсутствие таймаута для LLM вызовов | Week 1 | Core Team | 📝 In Progress |
| **P0** | Отсутствие retry для API вызовов | Week 1-2 | Core Team | ⏳ Planned |
| **P1** | Утечка контекста в трейсинге | Week 2 | Core Team | ⏳ Planned |
| **P1** | Дублирование cache write | Week 2 | Core Team | ⏳ Planned |
| **P1** | Отсутствие обработки rate limits | Week 2-3 | Core Team | ⏳ Planned |
| **P2** | Неверное использование skip_on_missing_params | Week 3 | Core Team | ⏳ Planned |
| **P2** | Отсутствие логирования ошибок метрик | Week 3 | Core Team | ⏳ Planned |
| **P2** | Memory leak в TraceManager | Week 4 | Core Team | ⏳ Planned |
| **P3** | Отсутствие поддержки Retryable Span | Week 5 | Core Team | ⏳ Planned |
| **P3** | Сложность кэширования conversational метрик | Week 5-6 | Core Team | ⏳ Planned |

### 5.2 Детальный план для P0-P1 проблем

#### P0: Отсутствие валидации API ключей
**Решение:**
```python
# В TraceManager.__init__()
def __init__(self):
    # ... existing code ...
    if self.confident_api_key is None and is_confident():
        raise ValueError("Confident AI API key is required. Set CONFIDENT_API_KEY env var.")
```

**Срок:** Week 1  
**Трудозатраты:** 4 часа

#### P0: Отсутствие таймаута для LLM вызовов
**Решение:**
```python
# В GPTModel.invoke()
def invoke(self, messages: List[Message]) -> str:
    timeout = int(os.getenv("DEEP_EVAL_LLM_TIMEOUT", "30"))
    try:
        response = client.chat.completions.create(
            model=self.model,
            messages=messages,
            timeout=timeout,
        )
        return response.choices[0].message.content
    except TimeoutError:
        raise TimeoutError(f"LLM call timed out after {timeout} seconds")
```

**Срок:** Week 1  
**Трудозатраты:** 2 часа

#### P0: Отсутствие retry для API вызовов
**Решение:**
```python
# В TraceManager.post_trace()
import asyncio
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=4, max=10))
def post_trace(self, trace: Trace) -> Optional[str]:
    # ... existing code ...
```

**Срок:** Week 1-2  
**Трудозатраты:** 6 часов

#### P1: Утечка контекста в трейсинге
**Решение:**
```python
# В tracing/context.py
from contextvars import ContextVar

current_trace_context: ContextVar[Trace] = ContextVar("current_trace", default=None)

async def get_trace() -> Trace:
    token = current_trace_context.get()
    if token is None:
        raise RuntimeError("No active trace context")
    return token
```

**Срок:** Week 2  
**Трудозатраты:** 4 часа

#### P1: Дублирование cache write
**Решение:**
```python
# В execute.py
# Объединить два вызова cache_test_case() в один
global_test_run_cache_manager.cache_test_case(
    test_case,
    new_cached_test_case,
    test_run.hyperparameters,
    to_temp=False,  # Один вызов вместо двух
)
```

**Срок:** Week 2  
**Трудозатраты:** 2 часа

#### P1: Отсутствие обработки rate limits
**Решение:**
```python
# В GPTModel.invoke()
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(5), wait=wait_exponential(multiplier=1, min=8, max=32))
def invoke(self, messages: List[Message]) -> str:
    try:
        response = client.chat.completions.create(
            model=self.model,
            messages=messages,
        )
        return response.choices[0].message.content
    except RateLimitError as e:
        raise RateLimitError(f"Rate limit exceeded: {e}")
```

**Срок:** Week 2-3  
**Трудозатраты:** 8 часов

---

## 6. Заключение

DeepEval представляет собой модульную, масштабируемую систему для оценки LLM-приложений. Архитектура основана на:

1. **Слоёном подходе**: тест-кейсы, метрики, модели, трейсинг
2. **Асинхронной обработке**: поддержка параллельного выполнения
3. **Кэшированием**: оптимизация дорогостоящих вычислений
4. **Интеграцией**: с CI/CD, Confident AI, внешними LLM провайдерами

Основные риски системы связаны с:
- Надёжностью API вызовов (требуется retry, timeout, rate limit handling)
- Правильным управлением контекстом в асинхронной среде
- Утечкой памяти при длительных сессиях тестирования

Рекомендуется начать с устранения P0-P1 проблем для повышения надёжности системы.