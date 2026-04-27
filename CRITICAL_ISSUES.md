# CRITICAL ISSUES IN DEEP-EVAL

## Введение

Этот документ содержит анализ критических проблем обнаруженных в основных сервисах DeepEval, их описание и рекомендации по устранению.

---

## 1. Сервис `evaluate` (Оценочный оркестратор)

### 1.1 Проблема: Отсутствие robust input validation
**Описание:**
Функция `evaluate()` принимает данные без глубокой валидации на входе, что может привести к ошибкам исполнения при подаче некорректных типов данных или пустых значений критических полей.

**Последствия:**
- `TypeError` при обработке тестовых случаев
- Непредсказуемое поведение в batch-режимe
- Сложность отладки при асинхронном запуске

**Рекомендации по устранению:**
1. Добавить отдельный класс `InputValidator` в `deepeval/evaluate/utils.py`
2. Реализовать валидацию для всех полей `evaluate()`:
   - Проверка non-empty списков тестовых случаев
   - Валидация metrices списка
   - Проверка корректности async_config параметров
3. Использовать `Optional` типы с явными сообщениями об ошибках
4. Добавить unité-тесты для edge-кейсов валидации

```python
# Пример реализации
def validate_inputs(test_cases, metrics, async_config, ...):
    if not test_cases:
        raise ValueError("test_cases cannot be empty")
    for mc in test_cases:
        if not mc.input:
            raise ValueError("Each test_case must have non-empty 'input'")
    if not metrics:
        raise ValueError("metrics list cannot be empty")
    # ... продолжить валидацию
```

**Приоритет:** Высокий

---

### 1.2 Проблема: Не достаточная обработка ошибок в `aggregate_results()`
**Описание:**
При агрегации результатов оценки используется bare try-except, который скрывает информацию об ошибках метрик от пользователя.

**Последствия:**
- Потеря контекста сбоя метрики
- Невозможность корректного анализа проваленных тестов
- Нечеткие сообщения об ошибках для клиента

**Рекомендации по устранению:**
1. Заменить bare except на дефайлэбили exceptions (ValueError, RuntimeError)
2. Логировать все ошибки с уровнем ERROR в logging
3. Возвращать структурированный `ErrorResult` с полным контекстом
4. Добавить `fail_fast` опцию для остановки при первых критических сбоях

```python
# Пример реализации
try:
    result = metric.measure(test_case)
except (ValueError, RuntimeError) as e:
    logger.error(f"[Metric {metric.name}] Failed: {e}", exc_info=True)
    raise ValueError(f"Metric {metric.name} failed: {e}")
```

**Приоритет:** Высокий

---

### 1.3 Проблема: Ограниченная поддержка retry для fluke-метрик
**Описание:**
Асинхронная оценка не имеет механизма повторного запуска (retry) для потенциально нестабильных метрик (например, G-Eval, GEval).

**Последствия:**
- Фейл тестов из-за временных проблем AI
- Невоспроизводимость результатов
- Потеря ресурсов на асинхронные вызовы

**Рекомендации по устранению:**
1. Добавить `MaxRetryConfig` в `deepeval/evaluate/configs.py`
2. Реализовать RetryStrategy с экспоненциальным backoff
3. Добавить метаданные для метрик, которые поддерживают retry
4. Логировать каждый retry attempt с timestamp

**Приоритет:** Средний

---

## 2. Сервис `Metric` (Метрический двигатель)

### 2.1 Проблема:_race condition в shared cache
**Описание:**
Кэширование результатов `measure()` реализовано через стандартный `dict` без `ThreadLock`, что может привести к гонкам при параллельном запуске тестов.

**Последствия:**
- Потеря кэшированных данных
- Дублирование LLM запросов
- Потенциальные NBes с запутанными данными

**Рекомендации по устранению:**
1. Использовать `threading.Lock` или `asyncio.Lock` для защиты cache
2. Добавить `context.asyncio.Lock()` для асинхронных операций
3. Реализовать time-to-live (TTL) для кэшированных записей
4. Добавить инвалидацию cache по версиям тестовых случаев

```python
# Пример реализации
class CachedMetric(BaseMetric):
    def __init__(self, ...):
        self._cache = {}
        self._lock = asyncio.Lock()
    
    async def _get_from_cache(self, key):
        async with self._lock:
            return self._cache.get(key)
    
    async def _set_in_cache(self, key, value):
        async with self._lock:
            self._cache[key] = (value, time.time())
```

**Приоритет:** Высокий

---

### 2.2 Проблема: Отсутствие детального логирования метаданных в `score_breakdown`
**Описание:**
Метрики возвращают `score_breakdown` но часто оставляют его пустым или минимальным, что затрудняет понимание частных компонентов_final score.

**Последствия:**
- Отсутствие детализации при анализе проваленных тестов
- сложности при debugging гемеральных метрик
- невозможность partial credit системы

**Рекомендации по устранению:**
1. Добавить обязательное заполнение `score_breakdown` поля во всех метриках
2. Добавить примеры в `BaseMetric` абстрактном классе
3. Использовать JSON-serializable структуры для breakdown
4. Добавить helper метод `validate_score_breakdown()`

```python
# Пример реализации
score_breakdown = {
    "criteria_match": 0.85,
    "metric_part1": 0.70,
    "metric_part2": 0.95,
    "details": {
        "matched_keys": ["key1", "key2"],
        "missing_keys": ["key3"]
    }
}
```

**Приоритет:** Средний

---

### 2.3 Проблема: Не соответствуете `measure()` и `a_measure()` behavior
**Описание:**
В некоторых метриках асинхронная версия `a_measure()` не всегда корректно delegate к синхронной или имеет разную логику вычисления.

**Последствия:**
- Непредсказуемость при async mode
- Различия в результатах между sync/async запуском
- Сложности при migration legacy кода

**Реcommendации по устранению:**
1. Добавить `METRIC_MODE = "async"` константами в каждом классе метрики
2. Добавить утилитный метод `_ensure_async()` в `BaseMetric`
3. Провести аудит всех метрик на consistency
4. Добавить едини-тесты для композиции sync/async режимов

**Приоритет:** Средний

---

## 3. Сервис `TestCase` (Контейнер для входных данных)

### 3.1 Проблема: Отсутствие валидации обязательных полей в dataclass'ах
**Описание:**
Классы `LLMTestCase`, `ConversationalTestCase`, `MLLMTestCase` не имеют по умолчанию валидации обязательных полей через `dataclasses.field(default_factory)` или `validate` method.

**Последствия:**
- Пустые тестовые случаи проходят валидацию
- Ошибки рассчитания на этапе исполнения (`IndexError`, `AttributeError`)

**Рекомендации по устранению:**
1. Добавить `post_init()` method с проверкой не-пустых полей
2. Использовать `strict=True` в современных версиях Python
3. Добавить `validate()` метод с детальными сообщениями об ошибках

```python
@dataclass
class LLMTestCase:
    input: str
    # ...
    
    def __post_init__(self):
        if not self.input.strip():
            raise ValueError("'input' cannot be empty")
```

**Приоритет:** Высокий

---

### 3.2 Проблема: lacks immutability guarantee
**Описание:**
Test case объекты изменяемы после создания открытого `dataclass` без `frozenset` или `__slots__`, что нарушает принцип чтенияTestData.

**Последствия:**
- Неожиданное изменение данных во время тестирования
- Битые результаты при многократном руне одного тестового кейса
- Невозможность использовать TestCase в качестве ключа словаря

**Рекомендации по устранению:**
1. Добавить `__slots__` для memory efficiency и базовой защиты
2. Создать readonly корректуру через `property()` методы
3. Добавить `frozen_test_case()` factory для readonly преобразования
4. Рассмотреть использование `dataclasses.field(init=False, default=...)`

**Приоритет:** Средний

---

## 4. Сервис `Tracing` (Обсерваемость)

### 4.1 Проблема: `_dict` глобальная переменная без thread-local isolation
**Описание:**
`TraceManager._dict` использует глобальный `dict` без изоляции по потокам, что вызывает race conditions при параллельном tracing.

**Последствия:**
- Запутанные traces из разных потоков
- Потеря треков в многопоточном окружении
- Неправильные результаты OT-export

**Рекомендации по устранению:**
1. Использовать `threading.local()` для thread-local storage
2. Добавить `thread_id` метаданный к каждому trace
3. Добавить методы `get_traces_by_thread()`
4. Реализовать cleanup для thread exit

```python
class TraceManager(BaseManager):
    def __init__(self):
        self._storage = threading.local()
    
    def _get_thread_dict(self) -> Dict:
        if not hasattr(self._storage, 'current'):
            self._storage.current = {}
        return self._storage.current
```

**Приоритет:** Высокий

---

### 4.2 Проблема: Отсутствие cleanup для Observer context Manager
**Описание:**
Observer context manager не гарантирует полное cleanup сохраненных traces при аномалиях диска или прерывании.

**Последствия:**
- Потеря критических метаданных
- Неполные трейски результаты
- Утечки памяти от забытых context'ов

**Рекомендации по устранению:**
1. Добавить `tearDown()` в `Observer` с try-finally
2. Реализовать forced shutdown для cleanup
3. Добавить временную лимитируя для tests (таймаут)
4. Log warning при прерывании без полного cleanup

**Приоритет:** Средний

---

### 4.3 Проблема: `SpanType` enum не расширяемы для новых типов
**Описание:**
Обсный `SpanType` enum фиксированный (AGENT, LLM, RETRIEVER, TOOL, BASE), что требует маппинга для новых типов.

**Последствия:**
- Сложность расширения для новых компонентов
- Нарушая patterns при добавлении MCP spans
- необходимость доп.MApping логики

**Рекомендации по устранению:**
1. Использовать `StrEnum` или `Literal["*", "*"]` для extensible enum
2. Добавить `SpanType.DEFAULT` для fallback
3. Добавить `new_span_type("name")` factory method
4. Протестировать на предмет присоединения новых типов

**Приоритет:** Низкий

---

## 5. Сервис `Dataset` (Управление данными)

### 5.1 Проблема: `EvaluationDataset.save()` не работает с parallel writes
**Описание:**
Сохранение тестовых данных на диск не имеет блокировок при параллельной записи из разных потоков.

**Последствия:**
- Коррупция файла на диск
- Lost тестовых случаев
- Неполное сохранение данных

**Рекомендации по устранению:**
1. Добавить `threading.Lock` для `save()` операции
2. Использовать atomic move для файлов `temporary -> final`
3. Добавить rollback mechanism для неудачных save
4. Log corrupted file errors с деталями

**Приоритет:** Средний

---

### 5.2 Проблема: Отсутствие индекса/кэша для быстрого поиска test cases
**Описание:**
`EvaluationDataset` использует линейный поиск по списку test cases, что O(n) при масштабирование.

**Последствия:**
- Медленный поиск тестовых случаев
- Slow batch операции для больших датасетов
- prolonged GPU idle time

**Рекомендации по устранению:**
1. Добавить `dict[str, TestCase]` hashing index по `id`
2. Добавить частичный индекс по `name` и `tags`
3. Кэшировать результаты поиска
4. Добавить benchmark для поиска операций

**Приоритет:** Низкий

---

## 6. Сервис `Confident AI Integration` (Cloud)

### 6.1 Проблема: API Rate Limiting не обрабатывается автоматически
**Описание:**
Отправка данных на Confident AI не имеет автоматической обработки 429 rate limit errors.

**Последствия:**
- Crash batch-оценки при пике нагрузка
- Потеря результатов из-за временных ограничений
- гораздо fewer по сравнению с возможностями

**Рекомендации по устранению:**
1. Добавить `RateLimitStrategy` с exponential backoff
2. Автоматически обрабатывать HTTP 429 responses
3. Добавить alert при достижении rate limit threshold
4. Кэшировать недавно отправленные batch'ы

**Приоритет:** Высокий

---

### 6.2 Проблема: Timeout на сетевых вызовах не настроен
**Описание:**
HTTP запросы к Confident AI не имеют явного timeout, что приводит к зависанию в infinite loop при отклике.

**Последствия:**
- Hanging evaluation при плохом network
- Illegal resource usage
- Нет responses клиента

**Рекомендации по устранению:**
1. Добавить `ConnectionPool` для aiohttp с `timeout=60s`
2. Добавить retry на timeout с exponential backoff
3. Добавить circuit breaker pattern для неудачных запусков
4. Log каждый timeout с retry контекстом

**Приоритет:** Высокий

---

### 6.3 Проблема: Отсутствие end-to-end тестов для cloud integration
**Описание:**
Нет тестов для проверки реальной работы с Confident API, включая подписи и хеш-сальти.

**Последствия:**
- Regressions при изменении API
- Неверные API документы
- Уязвимости интеграции

**Рекомендации по устранению:**
1. Добавить Mock для Confident в тестовом окружение
2. Протестировать все Endpoint (DATASET, TRACES, EVALS)
3. Добавить unit-тесты для подписи (signature)
4. Добавить конвейерный тест (end-to-end) в CI/CD

**Приоритет:** Средний

---

## 7. Сервис `Models` (ML Engine)

### 7.1 Проблема: Token counting не точный для различных LLM
**Описание:**
`count_tokens()` метод использует general алгоритм, который неточен для специфичных моделей (GPT-4 vs GPT-3.5).

**Последствия:**
- Неточный cost tracking
- неправильный billing calculation
- Неверные performance сравнения

**Рекомендации по устранению:**
1. Добавить `model_specific_tokenizer` опции
2. Использовать `tiktoken` для GPT моделей
3. Поддержать `openai.apiBird` для точного подсчета
4. Добавить fallback для custom моделей

**Приоритет:** Средний

---

### 7.2 Проблема: Отсутствие rate limiting для inference calls
**Описание:**
Запросы к LLM API не имеют автоматического rate limiting, что может привести к срабатыванию API квот.

**Последствия:**
- 429 errors в batch evaluations
- Потеря ресурсов
- N/A пользователя

**Рекомендации по устранению:**
1. Добавить `TokenBucketRateLimiter` к `DeepEvalBaseLLM`
2. Настраиваемый rate limit через config
3. Пакетирование запросов перед отправкой
4. Retry на API timeout с backoff

**Приоритет:** Высокий

---

## 8. Сводка по приоритетам

| Приоритет | Проблема | Сервис |
|-----------|----------|--------|
| **Критический** | Input validation отсутствует | evaluate |
| **Критический** | Shared cache race condition | Metric |
| **Критический** | Thread-local isolation в tracing | Tracing |
| **Критический** | API Rate Limiting не обрабатываем | Confident AI |
| **Критический** | Timeout на сетевых вызовах не настроен | Confident AI |
| **Критический** | Rate limiting для inference | Models |
| Высокий | Error handling в aggregate | evaluate |
| Высокий | Input validation в TestCase | TestCase |
| Высокий | Cleanup Observer context | Tracing |
| Высокий | End-to-end test для cloud | Confident AI |
| Средний | Score breakdown logging | Metric |
| Средний | sync/async consistency | Metric |
| Средний | Immutability для TestCase | TestCase |
| Средний | Cleanup для Observer | Tracing |
| Средний | Token counting accuracy | Models |
| Низкий | SpanType extensible enum | Tracing |
| Низкий | Поиск в Dataset | Dataset |

---

## Рекомендации по внедрению

### Фаза 1 (Непосредственный)
1. Input validation для `evaluate()` + `TestCase`
2. Thread-safe cache для метрик
3. API Rate Limiting + Timeout для Confident AI

### Фаза 2 (1-2 недели)
1. Error handling в aggregate
2. Cleanup Observer
3. Rate limiting inference

### Фаза 3 (1-2 месяца)
1. End-to-end test coverage
2. Token counting accuracy
3. Performance optimization

### Мониторинг прогресса
- Добавить метрики `pending_fixes` и `completed_fixes` в `GIT_LOG.md`
- Проводить регулярный audit документации
- Обновлять `CRITICAL_ISSUES.md` по мере устранения проблем

---

## Дата обновления

2026-04-28

## От

Cline (AI Software Engineer)