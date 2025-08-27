### **Conductor: A Declarative Concurrency Orchestration Framework**

**One-Liner:** Conductor separates *what* your code does (business logic) from *how* it runs (concurrency patterns), transforming complex async/parallel logic from any language into a simple, verifiable manifest file.

---

### **Core Architecture: The "Sidecar & Compiler" Model**

Conductor использует гибридную модель, сочетающую надежность **Infrastructure as Code** (как Terraform) с гибкостью **Service Mesh** (как Istio). Это позволяет определять сложную логику выполнения декларативно, а затем компилировать её в нативный, высокопроизводительный код для любой целевой платформы.

```mermaid
graph TD
    subgraph "Design-Time (Developer's Machine)"
        A[Manifest File (concurrency.yaml)] --> B{Conductor Compiler & AI Linter}
        C[Business Logic (your functions in Python, Go, JS...)] --> B
    end
    
    subgraph "Compile-Time"
        B --> D[Generated Glue Code (Orchestrator)]
        B --> E[Instrumentation & Tracing Hooks]
    end
    
    subgraph "Run-Time (Server / Cloud)"
        F[Application Entrypoint] --> D
        D -- "Invokes (via Mapping Engine)" --> C
        D -- "Emits Events" --> G[Conductor Sidecar / Broker]
        G -- "Manages State & Policy" --> D
        G -- "Reports Telemetry" --> H(Observability Stack)
    end
    
    H[Prometheus, Grafana, Jaeger]
```

### **The Three Key Components**

#### **1. The Manifest (`concurrency.yaml`) - The Single Source of Truth**

Это сердце системы — декларативный контракт, описывающий поток управления в виде **Directed Acyclic Graph (DAG) задач**. Он отделяет стратегию выполнения от реализации.

*   **Язык:** YAML или HCL, привычные для инженеров.
*   **Ключевые секции:**
    *   `tasks`: Атомарные единицы работы. Каждая задача маппится на реальную функцию в коде через `ref`.
    *   `workflows`: DAG, описывающий логику выполнения: последовательную, параллельную или условную.
    *   `policies`: Переиспользуемые правила отказоустойчивости (retries, timeouts, circuit breakers).
    *   `resources`: Внешние зависимости (БД, API), позволяющие управлять пулами соединений и другими ресурсами.

**Пример `concurrency.yaml`:**
```yaml
version: "1.0"

# 1. Define atomic tasks and map them to your code's functions/methods/services.
tasks:
  - id: fetch_user
    ref: "user_service.get_user_by_id" # Mapping to a function
    policy: retry_default
  
  - id: check_permissions
    ref: "auth_service.check_permissions" # Mapping to a gRPC endpoint
    policy: fast_fail

  - id: process_payment
    ref: "payment_provider.charge"
    policy: retry_idempotent

  - id: send_receipt
    ref: "notification_service.send_email"
    policy: fire_and_forget

# 2. Define reusable, high-level policies.
policies:
  - id: retry_default
    retries: 3
    strategy: exponential_backoff
    timeout_ms: 500
  - id: fast_fail
    retries: 0
    timeout_ms: 100
  - id: fire_and_forget
    retries: 2
    critical: false # Workflow continues even if this task fails after retries.

# 3. Compose tasks into a logical workflow.
workflows:
  - id: process_order_workflow
    entrypoint: true
    
    # Simple sequence of critical steps
    - task: fetch_user
    - task: check_permissions

    # A block of non-critical tasks that can run in parallel
    - parallel:
        - task: process_payment
        - task: send_receipt
```

#### **2. The Conductor Compiler - The "Smart" Code Generator**

Это `build-time` инструмент, который выступает в роли "умного компилятора" для вашего манифеста, превращая его в исполняемый код.

*   **Что он делает:**
    1.  **Парсинг и Верификация:** Читает `concurrency.yaml`, строит граф зависимостей и проверяет его на синтаксис, семантику и потенциальные дедлоки.
    2.  **Интроспекция Кода (Mapping Engine):** Сканирует кодовую базу (через AST-парсеры, аннотации или gRPC reflection) для резолва ссылок (`ref`). Он гибко маппит задачи на локальные функции, методы, контейнерные сервисы или внешние API.
    3.  **Генерация Оркестратора:** Создает нативный `orchestrator` для целевого языка (`.ts`, `.go`, `.py`). Этот сгенерированный файл содержит всю "грязную" логику: `async/await`, горутины, каналы, циклы `for` для ретраев, `setTimeout` для таймаутов.
    4.  **Внедрение Инструментации:** Автоматически вставляет в сгенерированный код вызовы к `Sidecar` для отслеживания состояния, отправки метрик и логов.

**Как это выглядит для разработчика:**
Он пишет простые, чистые, **"бесцветные"** (синхронные или асинхронные — не важно) функции, сфокусированные на бизнес-логике.

```python
# user_service.py (чистая бизнес-логика)
def get_user_by_id(user_id: str) -> User:
    # ... just the logic, no async, no retries, no timeouts.
    return db.query(...)
```
А `Compiler` генерирует весь необходимый "клей" для управления потоком выполнения.

#### **3. The Conductor Sidecar / Broker - The Runtime Brain**

Это легковесный процесс (сайдкар) или библиотека, которая работает рядом с вашим приложением и отвечает за состояние, выполнение политик и наблюдаемость в `run-time`.

*   **Почему "Sidecar / Broker"?** Этот паттерн отделяет бизнес-логику от инфраструктурной, обеспечивая универсальность.
*   **Что он делает:**
    1.  **State Management:** Отслеживает состояние каждого запущенного `workflow` (какая задача выполняется, сколько попыток осталось). Это позволяет перезапускать упавшие воркфлоу с места сбоя.
    2.  **Policy Enforcement:** Оркестратор может запрашивать у сайдкара динамические решения. Например: "Circuit breaker для `payment_provider` открыт? Если да, то не выполняем `process_payment`, а сразу переходим к fallback-логике".
    3.  **Observability Hub:** Собирает метрики, логи и трейсы со всех воркфлоу и отправляет их в стандартные стеки (Prometheus, Grafana, Jaeger). Вы получаете **из коробки** дашборд, где видите, какие задачи чаще всего падают, сколько времени занимают и т.д.

### **Как AI-Агент Усиливает Эту Систему**

ИИ в Conductor — это не магия, а **инженер-напарник**, который работает на двух уровнях:

1.  **Manifest Linter & Optimizer (Design-Time):**
    *   AI-агент — это "умный линтер" для вашего `concurrency.yaml`. Он читает манифест и дает рекомендации уровня Principal Engineer:
        *   "**Предупреждение о риске:** В вашем параллельном блоке задача `process_payment` критически важна, а `send_receipt` — нет. Ошибка в отправке письма может завалить всю транзакцию. Рекомендую вынести `send_receipt` в отдельный `fire-and-forget` воркфлоу".
        *   "**Оптимизация производительности:** Задачи `fetch_user` и `fetch_product_info` не зависят друг от друга. Их можно выполнять параллельно, чтобы сократить общую задержку на ~200мс".

2.  **Natural Language to Manifest:**
    *   Разработчик пишет в комментарии или в чате: `// Conductor: run fetch_user and check_permissions sequentially, then in parallel process payment and send a receipt`.
    *   AI-агент **генерирует** соответствующий `workflow` в `concurrency.yaml`, экономя время и снижая вероятность ошибок.

### **Питч (в 30 секунд)**

> "Мы создали **Conductor**. Это фреймворк, который выносит всю логику асинхронности и параллелизма из кода в **декларативный YAML-манифест**. Вы пишете простую бизнес-логику на любом языке, а наш компилятор генерирует весь необходимый `async/await` или `goroutine` код. Сайдкар-агент дает вам state management и observability из коробки. **По сути, это Terraform для конкурентности**".