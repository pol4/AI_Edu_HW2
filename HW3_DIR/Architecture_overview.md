# DeepEval: Система тестирования и оценки LLM моделей

## 1. Назначение и основные сценарии использования системы

### Назначение
DeepEval — это фреймворк для автоматизированного тестирования и оценки больших языковых моделей (LLM) и мультимодальных моделей (MLLM). Система предоставляет инструменты для валидации качества ответов, обнаружения галлюцинаций, оценки релевантности, анализа настроения и биасов в генерации текста.

### Основные сценарии использования

1. **Тестирование LLM-моделей**
   - Оценка ответов модели с заданными входными данными (prompts) и ожидаемыми результатами
   - Валидация на соответствие ожиданиям (концептуальную правильность, токсичность, PII)
   - Автоматическое создание тестовых случаев для CI/CD

2. **Оценка чат-бота и диалоговых систем**
   - Тестирование многопользовательских диалогов с несколькими ходами
   - Проверка сохранения контекста между запросами
   - Оценка релевантности ответов для conversation

3. **Сравнение моделей (Arena)**
   - Парное сравнение двух или более LLM для выбора лучшего
   - A/B тестирование различных версий моделей
   - Оценка предпочтений пользователя между вариантами ответов

4. **RAG-системы**
   - Оценка контекстуальной точности и полноты (RAGAS)
   - Проверка фактической точности ответов с опорой на предоставленный контекст
   - Анализ релевантности ответа для сущностей из запроса

5. **Коллекция бенчмарков**
   - Запуск моделей на стандартных бенчмарках (MMLU, GSM8K, HumanEval, BBH)
   - Сравнение производительности с другими моделями
   - Валидация результатов на больших наборах данных

6. **Интеграция с ML-инструментами**
   - CrewAI, LangChain, LlamaIndex, Pydantic AI
   - Naive нечеткие токены для интеграции в существующие пайплайны
   - Обработка загрузок с Hugging Face

7. **Интуитивная оценка метрики**
   - Автоматическая рекомендация метрик в зависимости от типа задачи
   - Анализ тестовых случаев для выявления оптимальных метрик оценки
   - Построчная статистика прохождения тестов

8. **Веб-интерфейс для загрузки результатов**
   - Локальный сервер для просмотра результатов
   - Интерактивная визуализация метрик
   - Экспорт результатов в различные форматы

---

## 2. Основные сервисы системы и карта взаимодействия

### Архитектурная диаграмма системы

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DeepEval System Architecture                      │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│ User/CLI    │──────▶│ CLI Service │◀──────│ Config      │
│             │       │ (main.py)   │       │ Handler     │
└─────────────┘       └──────┬──────┘       └─────────────┘
                             │
┌─────────────┐       ┌──────┴─────────────┐       ┌─────────────┐
│   Dataset   │◀─────▶│ Test Cases Service │◀─────▶│ Model       │
│ (dataset.py)│       │                    │       │ Adapter     │
└─────────────┘       └──────┬─────────────┘       └─────────────┘
                             │
            ┌────────────────┼────────────────┐
            │                │                │
┌───────────▼────────┐ ┌─────▼────────┐ ┌────▼─────────┐
│  Evaluation Engine │ │ Metrics      │ │ Model        │
│ (evaluate.py)      │ │ Service      │ │ Evaluation   │
│                    │ │              │ │ Service      │
└────────────────────┘ └──────┬───────┘ └──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │ Results Storage   │
                    │ (files/SQL/Cloud) │
                    └───────────────────┘

┌───────────────────────────────────────────────────────────────────────────┐
│ Integration Modules (Plugins)                                             │
├─────────────────┬─────────────────┬─────────────────┬─────────────────────┤
│ CrewAI          │ LangChain       │ LlamaIndex      │ Pydantic AI         │
│ Integration     │ Integration     │ Integration     │ Integration         │
└─────────────────┴─────────────────┴─────────────────┴─────────────────────┘
```

### Детальное описание сервисов

#### 2.1 Dataset Service (`deepeval/dataset/`)
- **Компоненты**:
  - `EvaluationDataset` — центральный менеджер тестовых данных
  - `Golden` / `ConversationalGolden` — ожидаемые ответы
  - `APIDataset`, `APIQueueDataset` — взаимодействие с облачным хранилищем
- **Основные функции**:
  - Импорт тестовых случаев из CSV/JSON файлов
  - Добавление/удаление тест-кейсов и golden ответов
  - Сохранение и загрузка из облака (функции push/pull/queue)
  - Генерация golden из документов

#### 2.2 Test Case Service (`deepeval/test_case/`)
- **Компоненты**:
  - `LLMTestCase` — однопользовательский (prompt → response)
  - `ConversationalTestCase` — диалоговые сценарии
  - `ArenaTestCase` — тест для сравнения двух моделей
  - `MLLMTestCase` — тест для мультимодальных моделей
  - `MCPToolCall` / `MCPromptCall` / `MCPServer` — поддержка MCP
- **Основные функции**:
  - Определение параметров тестов (question, answer, expected_answer, etc.)
  - Проверка валидности параметров перед запуском
  - Создание парных треней для arena

#### 2.3 Evaluation Engine (`deepeval/evaluate/`)
- **Компоненты**:
  - `evaluate()` — основной входной метод
  - `execute_test_cases()` — выполнение тестовых случаев
  - `compare()` — сравнение моделей
  - `AsyncConfig`, `DisplayConfig`, `CacheConfig`, `ErrorConfig` — конфигурации
- **Основные функции**:
  - Оркестрация выполнения метрик для каждого тестового случая
  - Асинхронное выполнение с ограничением количества обфрат
  - Обработка ошибок и генерация отчетов
  - Сравнение результатов

#### 2.4 Metrics Service (`deepeval/metrics/`)
- **Компоненты**:
  - `BaseMetric` / `BaseConversationalMetric` / `BaseMultimodalMetric` / `BaseArenaMetric`
  - Специализированные метрики:
    - `HallucinationScore` / `SummaCModel`
    - `Conversational` (контекстуальная точность)
    - `Answer Relevancy` (RAGAS)
    - `Contextual Precision/Recall/Entities`
    - `Toxicity` (Detoxify)
    - `PII Leakage` (обнаружение приватных данных)
    - `Bias` / `UnbiasModel`
    - `JSON Correctness`
- **Основные функции**:
  - Вычисление scores для ответа на соответствие п самолёт
  - Support как синхронных, так и парных методов measurement
  - Агрегация результатов и вычисление средних / медиан

#### 2.5 Model Adapter (`deepeval/models/`)
- **Компоненты**:
  - `DeepEvalBaseLLM` — абстракция LLM
  - `DeepEvalBaseEmbeddingModel` — абстракция embedding модели
  - `DeepEvalBaseMLLM` — абстракция ML-модели
  - Adapters: OpenAI, Azure, HuggingFace, litellm
- **Основные функции**:
  - Абстракция вызова модели
  - Batch вход (беспредел бейджа для языковых запросов)
  - Долгоживущие алгоритмы для совместимости

#### 2.6 Configuration Service (`deepeval/cli/types.py`)
- **Компоненты**:
  - `Regions` — выбор географической зоны (для OpenAI/Azure)
  - Environment settings (model-specific)
  - Key management credentials
- **Основные функции**:
  - Управление API ключами (особенно распределяемые по регионам)
  - Выбор провайдеров (OpenAI, Azure, Ollama, local модели)
  - Конфигурация на основе окружения

#### 2.7 Benchmark Service (`deepeval/benchmarks/`)
- **Компоненты**:
  - `DeepEvalBaseBenchmark` — база класса для бенчмарков
  - ID бенчмарков (MMLU, BBH, HumanEval, GSM8K и др.)
- **Основные функции**:
  - Загрузка стандартных наборов данных для валидации
  - Автоматическое определение подходящих задач/metrik
  - Сравнение с эталонными результатами

#### 2.8 CLI Gateway (`deepeval/cli/`)
- **Компоненты**:
  - `main.py` — сбор всех команд
  - `server.py` — локальный сервер для результатов
  - `test.py` — запуск тестовых файлов
- **Основные функции**:
  - Выполнение входных команд (login, logout, view, recommend)
  - Запуск локальной серверной оценки
  - Интерактивный вывод (progress, результат)

### Карта взаимодействия

```
                              ┌─────────────────┐
                              │   User Input    │
                              │(CLI/API/Script) │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │ CLI Service     │
                              │  - login        │
                              │  - logout       │
                              │  - view         │
                              │  - test         │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │  ConfigManager  │
                              │  - keys         │
                              │  - regions      │
                              │  - providers    │
                              └────────┬────────┘
                                       │
                   ┌───────────────────┼───────────────────┐
                   │                   │                   │
         ┌─────────▼──────────┐ ┌──────▼──────────┐ ┌──────▼──────────┐
         │ DatasetService     │ │ TestCasesMgr    │ │  ModelAdapter   │
         │  - load()          │ │  - validate()   │ │  - generate()   │
         │  - push()          │ │  - build()      │ │  - embed()      │
         │  - pull()          │ │  - create()     │ │  - call()       │
         └─────────┬──────────┘ └──────┬──────────┘ └────────┬────────┘
                   │                   │                     │
                   │                   │                     │
         ┌─────────▼──────────┐ ┌──────▼──────────┐          ▼
         │  EvaluationEngine  │ │  MetricsEngine  │ ┌────────▼────────┐
         │  - evaluate()      │ │  - measure()    │ │  ResultStorage  │
         │  - compare()       │ │  - aggregate()  │ │  - files        │
         │  - execute()       │ │  - report()     │ │  - cloud        │
         └────────────────────┘ └─────────────────┘ └─────────────────┘
```

---

## 3. Сквозной пример взаимодействия сервисов системы

### Сценарий: Тестирование LLM-модели в выделенном инференсе через OpenAI API по сети

#### Описание сценария
1. Пользователь запускает оценку с помощью CLI для сравнения OpenAI модели s (доступной по HTTPS через OpenAI API)
2. Используется ограниченное количество параллельных запросов (async с semaphore)
3. Каждый тест-кейс проходит через цепочку сервисов для получения метрик
4. Результаты сохраняются локально и отправляются в облако

#### Состояния системы

```
┌──────────────────────────────────────────────────────────────────────────────┐
│              State Diagram: LLM Model Evaluation Process                     │
└──────────────────────────────────────────────────────────────────────────────┘

                ┌───────────────────────────────────────┐
                │            Idle                       │
                │  (Ожидание запуска оценки)            │
                └────────────────┬──────────────────────┘
                                 │ user initiates
                                 │  deepeval.evaluate(
                                 │    dataset, test_cases, metrics, model
                                 │  )
                    ┌────────────▼───────────────┐
                    │        Initializing        │
                    │  - Parsing config files    │
                    │  - Loading API keys        │
                    │  - Initializing model      │
                    └────────────┬───────────────┘
                                 │  model loaded
                                 │  dataset parsed
                   ┌─────────────▼─────────────────────┐
                   │           Loading                 │
                   │  - Cache miss check               │
                   │  - Load from disk or config       │
                   │  - Pre-process test cases         │
                   └─────────────┬─────────────────────┘
                                 │  dataset ready
                                 │  metrics initialized
                   ┌─────────────▼─────────────────────┐
                   │         Configured                │
                   │  - AsyncConfig applied            │
                   │  - DisplayConfig set              │
                   │  - ErrorConfig parameters         │
                   └─────────────┬─────────────────────┘
                                 │ validate params
                   ┌─────────────▼─────────────────────┐
                   │         Validation                │
                   │  - Input validation               │
                   │  - Parameter checking             │
                   └─────────────┬─────────────────────┘
                                 │ all valid
                   ┌─────────────▼─────────────────────┐
                   │      Execution Queue Building     │
                   │  - Test cases indexed             │
                   │  - Metrics assigned               │
                   │  - Semaphores created             │
                   └─────────────┬─────────────────────┘
                                 │ generate test
                    ┌────────────▼─────────────────────┐
                    │     Test Generation              │
                    │  - LLMTestCase creation          │
                    │  - Question, answer,             │
                    │    expected_answer, feedback     │
                    └────────────┬─────────────────────┘
                                 │ generate call
                   ┌─────────────▼─────────────────────┐
                   │    OpenAI API Call                │
                   │  - Request body: {prompt}         │
                   │  - Response: {answer}             │
                   └─────────────┬─────────────────────┘
                                 │ validate model call
                   ┌─────────────▼─────────────────────┐
                   │       Metric Execution            │
                   │  - measure() for each metric      │
                   │  - Hallucination, Toxicity,       │
                   │    Relevancy, etc.                │
                   └─────────────┬─────────────────────┘
                                 │ compute scores
                   ┌─────────────▼─────────────────────┐
                   │     Result Storage                │
                   │  - TestResult created             │
                   │  - Metrics aggregated             │
                   │  - Pass/Fail determination        │
                   └─────────────┬─────────────────────┘
                                 │ rate limit
                   ┌─────────────▼─────────────────────┐
                   │         Rate Limiting             │
                   │  - Semaphore enforces limits      │
                   │  - Batch if possible              │
                   └────────────┬──────────────────────┘
                                 │ rate limit exhausted
                    ┌────────────▼─────────────────────┐
                    │      Waiting for next batch      │
                    │  - Retry on failure              │
                    │  - Continue if non-blocking      │
                    └────────────┬─────────────────────┘
                                 │ all done
                   ┌─────────────▼─────────────────────┐
                   │       Completion                  │
                   │  - Aggregate results              │
                   │  - Generate report                │
                   │  - Error handling if needed       │
                   │  - Upload to cloud (optional)     │
                   └─────────────┬─────────────────────┘
                                 │ cleanup
                   ┌─────────────▼─────────────────────┐
                   │          Done                     │
                   │  - Results available              │
                   │  - Can view via CLI/web interface │
                   └───────────────────────────────────┘
```

#### Последовательность вызовов

```
User
  │
  │  deepeval.evaluate(
  │    dataset = EvaluationDataset(),
  │    test_cases = [LLMTestCase(...)],
  │    metrics = [HallucinationScore, RelevancyScore],
  │    model = OpenAIOnAPI()
  │  )
  │
  ▼
CLI Service (main.py)
  │
  │  parse_args() → init args
  │  login() → set API key
  │  view() → show result form
  │
  ▼
Config Handler (key_handler.py)
  │
  │  read_key_file() → get OpenAI key
  │  set_openai_env() → configure region/model
  │  Regions.OPENAI → configure call
  │
  ▼
Dataset Service (dataset.py)
  │
  │  dataset = EvaluationDataset()
  │  dataset.add_test_case(LLMTestCase(...))
  │  dataset.add_golden(Golden(expected_answer="..."))
  │
  ▼
Model Adapter (base_model.py)
  │
  │  model = OpenAIOnAPI("gpt-4")
  │  model.load_model() → init API client
  │  model.call(prompt) → send request to https://api.openai.com/v1/chat/completions
  │
  ▼
Evaluation Engine (evaluate.py)
  │
  │  async for test_result in a_evaluate_test_cases():
  │      - Create test case
  │      - Call model.get() → generate answer
  │      - For each metric:
  │          metric.measure(test_case, model_call)
  │          - HallucinationScore.compute(original, generated)
  │          - RelevancyScore.answer_relevancy(question, answer)
  │      - Create TestResult with all metrics
  │      - Store in results
  │      - yield test_result
  │
  ▼
Metrics Service (metrics/indicator.py)
  │
  │  measure_metric_task() → async medida
  │  measure_metrics_with_indicator() → show progress
  │  safe_a_measure() → handle exceptions
  │
  ▼
Result Storage
  │
  │  - Write to file (CSV/JSON)
  │  - Store in cloud dataset if configured
  │  - Enable cloud push()
  │
  ▼
Web Server (server.py)
  │
  │  Interactive display of results
  │  - Progress bars
  │  - Charts for metrics
  │  - Export options
  │
  ▼
User receives final report
```

---

## 4. Анализ уязвимостей, узких мест и точек отказа

### Классификация проблем по критериям

| Критичность | Описания | Трудоемкость | Сообщается |
|-------------|----------|--------------|------------|
| Критическая | Нарушает работу системы в целом | Высокая | High |
| Высокая | Серьезное влияние на качество | Средняя | Medium |
| Средняя | Локальные проблемы, которые можно обойти | Низкая | Low |

### Анализ уязвимостей и проблем

#### 4.1 Узкие места производительности

| ID | Пропуск | Описание | Критичность |
|----|---------|---------|-------------|
| PM-01 | Асинхронные вызовы через `a_execute_()` | Использование `asyncio.semaphore` увеличивает время ожидания при задержке от сервиса | Средняя |
| PM-02 | `measure_metric_task()` | Синхронный вызов длинного модели в `metrics.measure()` блокирует поток | Средняя |
| PM-03 | Парсинг `LLMTestCase` | Создание `LLMTestCase` парсного JSON каждое выполнение теста, создавая избыточные объекты | Высокая |
| PM-04 | Педата сток I/O | `predict_dataset()` с локальных файлов создает сверхрассеянные вызовы | Низкая |

#### 4.2 Точки отказа

| ID | Точка отказа | Влияние | Критичность |
|----|--------------|---------|-------------|
| FO-01 | Сгорание сессия (key_handler.py) | Потеря ключа доступа, полная остановка системы | Критическая |
| FO-02 | `OpenAIOnAPI` API | Потеря связи с API = остановка всех LLM-тестов | Критическая |
| FO-03 | Локальное хранилище `dataset.py` | Корумпированные данные в CSV/JSON | Высокая |
| FO-04 | `evaluation_engine` daemon | Дальше обработки *exit* после автостопов | Средняя |
| FO-05 | `metrics.measure` без `try-except` | Исключение в метрике прерывает весь воркфлоу | Средняя |

#### 4.3 Безопасность и уязвимости

| ID | Уязвимость | Описание | Влияние | Критичность |
|----|------------|---------|---------|-------------|
| SEC-01 | `key_handler.py` в открытом виде | API keys хранятся в плейн-тексте, могут быть считаны/украдены пятнышко | Высокая |
| SEC-02 | `telemetry.py` собирает данные пользователя | Анонимные данные *от* пользователя могут раскрыть информацию | Средняя |
| SEC-03 | Без аутентификации в `server.py` | Локальный сервер уязвим к XSS/CSRF | Высокая |
| SEC-04 | Проверка входных параметров | Некоторые `test_case.validate()` пропускают ошибки | Средняя |

#### 4.4 Проблемы качества результатов

| ID | Проблема | Описание | Влияние |
|----|----------|---------|---------|
| QUAL-01 | Зависимость от одной метрики | Некоторые метрики (например, Hallucination) могут давать ложные исходные проверки | Средняя |
| QUAL-02 | Отсутствие верификации результатов | Не все метрики генерируют результаты валидации | Низкая |
| QUAL-03 | Работа с контекстом в разговорах | Контекст диалога может быть потеряны/subтотизирован | Высокая |
| QUAL-04 | Не учитываются multiple-lingual-а | Не все метрики поддерживают мультилингвистику | Средняя |
| QUAL-05 | Отсутствие cache для повторных вызовов | Повторные вызовы модели тратят время и API-specific

---

## 5. План устранения критичных проблем

### Приоритетная таблица устранения

| ID | Проблема | Критичность | Приоритет | Устранение | Ответственность | Срок |
|----|----------|-------------|-----------|------------|-----------------|------|
| SEC-01 | Storage API key in plaintext | Критическая | P0 | - Добавить шифрование ключей (`encryption`)  <br> - Использовать `os.Getenv` вместо plaintext <br> - Добавить CLI опцию для создания secure key store | Security lead | SPRINT 1 |
| PM-01 | Async overhead with semaphores | Средняя | P1 | - Метод `execute_with_semaphore` заменить на custom thread pool <br> - Оптимизировать wait logic | Performance | SPRINT 1 |
| FO-01 | Key handler as single point | Критическая | P0 | - Добавить Failover to multiple keys <br> - Использовать env для Fallback <br> - Добавить cache для повторных запросов | Security | SPRINT 1 |
| PM-02 | Blocking in measure() | Средняя | P1 | - Использовать `asyncio.to_thread()` для тяжёлых метрик <br> - Добавить background thread pool | Performance | SPRINT 2 |
| QUAL-03 | Lost context in conversations | Высокая | P1 | - Улучшить контекст-перенос в диалогах <br> - Добавить summarization для длинных диалогов | Quality | SPRINT 2 |
| SEC-03 | `server.py` без аутентификации | Критическая | P0 | - Добавить JWT/Bearer токены <br> - Добавить CORS конфигурацию <br> - Использовать аутентификацию по IP | Security | SPRINT 1 |
| PM-03 | JSON serialization каждый run | Высокая | P1 | - Добавить кэш для `Object` <br> - Использовать сериализацию только один раз | Performance | SPRINT 2 |
| PM-04 | Predict I/O overhead | Средняя | P1 | - Добавить буферизацию данных <br> - Параллельная обработка I/O | Performance | SPRINT 2 |
| QUAL-01 | Dependency to single metric | Средняя | P1 | - Добавить многомерную оценку <br> - Добавить вес метрик для разных задач | Quality | SPRINT 2 |
| FO-05 | Exception in measure() <br> run | Высокая | P1 | - Обернуть все `measure()` в `try-except` <br> - Добавить `continue` для non-critical | Reliability | SPRINT 1 |
| SEC-02 | Telemetry leaks info | Низкая | P2 | - Уменьшить данные собираемых <br> - Добавить опцию `disable_telemetry` | Privacy | SPRINT 2 |
| FO-02 | OpenAIOnAPI failure | Критическая | P0 | - Добавить fallback models (Ollama, Grit) <br> - Добавить failover логику | Reliability | SPRINT 1 |
| SEC-04 | Parameter validation gaps | Средняя | P2 | - Расширить `validate_evaluate_inputs()` <br> - Добавить unit-тесты для всех путей | Reliability | SPRINT 2 |
| PM-05 | No cache for repeated calls | Средняя | P2 | - Добавить in-memory cache <br> - Добавить LRU policy | Performance | SPRINT 2 |

### Приоритет устранения

#### SPRINT 1 (Critical Priority)
1. **SEC-01** - Шифрование API ключей
2. **FO-01** - Failover ключей в `key_handler.py`
3. **SEC-03** - Аутентификация на сервере
4. **FO-02** - Fallback модели
5. **FO-05** - Обработка исключений в метриках

#### SPRINT 2 (High Priority)
1. **PM-01** - Оптимизация асинхронности
2. **PM-02** - Non-blocking metrics
3. **PM-03** - JSON serialization кэширование
4. **QUAL-03** - Контекст в диалогах
5. **SEC-04** - Validation gaps
6. **PM-05** - In-memory кэш

#### SPRINT 3 (Low Priority)
1. **PM-04** - I/O буферизация
2. **QUAL-01** - Multi-dimensional evaluation
3. **QUAL-02** - Result верификация
4. **SEC-02** - Telemetry data limits

### Метрики успеха устранения

| ID | Метрика успеха |
|----|----------------|
| SEC-01 | 0 доступных секрета в open реестре |
| FO-01 | 100% кэширование ключей |
| SEC-03 | JWT токены на всех endpoints |
| FO-02 | 0 простоев > 5 мин на нет |
| FO-05 | 100% исключений обработаны с логированием |
| PM-01 | < 200ms latency для 100 запросов |
| PM-02 | < 50% CPU idle time |
| PM-03 | < 10ms serialization только для образа |
| QUAL-03 | > 95% согласованных результатов в разговорах |
| PM-05 | > 50% cache hit rate |

---

## Заключение

DeepEval представляет собой сложную систему для оценки LLM-моделей с модульной архитектурой. Основные сервисы работают в тандеме для обеспечения полного цикла: от создания тестовых случаев до анализа результатов. Identiфицированные уязвимости требуют системного подхода к исправлениям — начиная от безопасности (API keys, authentication), заканчивая производительностью (async/parallelism, caching) и качеством (метрики, контекст). Приоритетная таблица устранения направлена на наиболее критичные проблемы, влияющие на надежность системы.
