# **VERITAS: Principles for Intentional, Verifiable Systems**

## **Verifiability · Evidence-First · Rationale Preservation · Intent Primacy · Teleological Design · Adaptive Complexity · Separation of Concerns**

---

## Манифест

> «Код — не источник правды. Код — компиляция намерений в исполняемую форму. Истинный источник правды — формализованное намерение, верифицируемое доказательство и сохранённое обоснование.»

VERITAS — это набор принципов для построения систем, где намерение отделено от реализации, каждое утверждение имеет доказательство, каждое решение имеет обоснование, и сложность управляема и явна.

---

## Часть I: Философское основание

### Проблема: Эпистемологический Долг

Современные системы накапливают **эпистемологический долг** — разрыв между тем, что система делает, и тем, что мы о ней знаем.

Этот долг проявляется в нескольких формах. Первая — **утраченное обоснование**: код существует, но никто не знает, почему он именно такой. Кто-то когда-то принял решение, но причина потеряна. Вторая форма — **дрейф намерений**: изначальная цель была одна, код эволюционировал, и теперь они не соответствуют друг другу. Третья — **непроверяемые утверждения**: «эта функция быстрая», «этот алгоритм корректен», «этот сервис надёжен» — но доказательств нет. Четвёртая форма — **скрытая сложность**: система кажется простой, но внутри — хаос зависимостей и неявных предположений.

VERITAS — это система принципов для предотвращения эпистемологического долга.

### Центральный тезис

Каждый артефакт системы (код, архитектура, решение, требование) должен отвечать на четыре вопроса:

- **ЧТО?** — Что этот артефакт должен делать? (Intent Primacy)
- **ПОЧЕМУ?** — Почему он существует и почему именно такой? (Rationale Preservation, Evidence-First)
- **КАК ПРОВЕРИТЬ?** — Как доказать, что он делает то, что должен? (Verifiability)
- **КУДА ВЕДЁТ?** — Как он связан с конечной целью системы? (Teleological Design)

---

## Часть II: The Seven Principles

### V — Verifiability (Верифицируемость)

> «Если утверждение нельзя проверить, оно не является знанием.»

#### Суть

Каждый артефакт системы должен быть автоматически верифицируем на соответствие заявленным свойствам. Утверждение без проверки — это мнение, не факт.

#### Проблема, которую решает

Системы полны утверждений, которые никто не проверяет. «Этот код быстрый», «этот сервис надёжный», «этот алгоритм корректный» — но доказательств нет. Эти утверждения накапливаются и становятся техническим долгом.

#### Практики

**Формализация свойств.** Каждое критичное свойство должно быть выражено в формате, допускающем автоматическую проверку.

```typescript
/**
 * @property complexity: O(n log n)
 * @property max_latency: 100ms for n=10000
 * @property invariant: output.length === input.length
 * @verified_by: test/sort.property.test.ts
 */
function sort<T>(items: T[], compareFn: (a: T, b: T) => number): T[] {
  // implementation
}
```

**Property-Based Testing.** Вместо тестирования примеров тестируем свойства для всех возможных входов.

```typescript
// Вместо: expect(sort([3,1,2])).toEqual([1,2,3])
// Используем:
fc.assert(fc.property(
  fc.array(fc.integer()),
  (arr) => {
    const sorted = sort(arr);
    // Свойство 1: Идемпотентность
    expect(sort(sorted)).toEqual(sorted);
    // Свойство 2: Сохранение элементов
    expect(sorted).toHaveLength(arr.length);
    // Свойство 3: Упорядоченность
    for (let i = 1; i < sorted.length; i++) {
      expect(sorted[i] >= sorted[i-1]).toBe(true);
    }
  }
));
```

**Constraint Verification.** Ограничения из требований автоматически преобразуются в проверки.

```yaml
constraint:
  id: "payment-latency-sla"
  description: "Payment processing must complete in < 2 seconds for p95"
  verification:
    type: "load_test"
    tool: "k6"
    script: "tests/load/payment.js"
    threshold: "http_req_duration{p(95)} < 2000"
    frequency: "on_deploy"
    blocking: true
```

**Observability Gates.** Деплой блокируется, если метрики не соответствуют заявленным свойствам.

```yaml
deployment:
  gates:
    - name: "latency_check"
      metric: "p95_latency"
      threshold: "< 200ms"
      window: "5m"
      action: "block_deploy"
    
    - name: "error_rate_check"
      metric: "error_rate"
      threshold: "< 0.1%"
      window: "5m"
      action: "rollback"
```

#### Инварианты принципа

1. Утверждение без автоматической проверки — это гипотеза, не факт.
2. Критичные свойства проверяются на каждый деплой.
3. Проверка должна быть повторяемой и детерминированной.
4. Результаты проверки сохраняются и доступны для аудита.

#### Метрики

- **Verification Coverage**: процент критичных свойств с автоматической проверкой. Цель — 100% для safety-critical, более 90% для бизнес-логики.
- **Mean Time to Verification Failure**: среднее время обнаружения нарушения свойства.

---

### E — Evidence-First (Доказательство прежде всего)

> «Утверждение без источника — это галлюцинация.»

#### Суть

Ни одно утверждение не может существовать в системе без явной привязки к источнику. Источник — это не комментарий «так надо», а конкретная ссылка: тикет, эксперимент, исследование, решение.

#### Проблема, которую решает

Код полон «магических» констант, неочевидных решений и предположений, источники которых потеряны. Когда разработчик уходит, знание уходит с ним.

#### Практики

**Provenance Tracking.** Каждый нетривиальный артефакт имеет метаданные о происхождении.

```typescript
/**
 * @source https://company.atlassian.net/browse/PERF-123
 * @rationale "A/B test showed 3 retries optimal for 99.9% success rate"
 * @date 2024-01-15
 * @author alice@company.com
 * @review_date 2024-07-15
 */
const MAX_RETRIES = 3;
```

**Sourceless Rejection.** База данных отвергает записи без указания источника.

```sql
CREATE TABLE claims (
  id UUID PRIMARY KEY,
  content TEXT NOT NULL,
  sources JSONB NOT NULL,
  validity FLOAT CHECK (validity BETWEEN 0 AND 1),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  
  -- Evidence-First: источник обязателен
  CONSTRAINT sources_not_empty CHECK (jsonb_array_length(sources) > 0)
);

-- Trigger для проверки качества источников
CREATE FUNCTION validate_sources() RETURNS TRIGGER AS $$
BEGIN
  IF NOT (NEW.sources @> '[{"url": ""}]'::jsonb) THEN
    RAISE EXCEPTION 'Each source must have a URL';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Evidence Ledger.** Центральный реестр всех утверждений с их доказательствами.

```yaml
evidence_ledger:
  - claim: "Redis reduces DB load by 70%"
    sources:
      - type: "experiment"
        url: "https://company.atlassian.net/browse/PERF-456"
        date: "2024-01-10"
        methodology: "A/B test, 1 week, 100K users"
        result: "DB queries reduced from 10K/s to 3K/s"
    validity: 0.9
    review_date: "2024-07-10"
  
  - claim: "User sessions expire after 30 days"
    sources:
      - type: "business_decision"
        url: "https://company.atlassian.net/browse/SEC-789"
        date: "2024-02-01"
        stakeholder: "Security Team"
        rationale: "Balance between UX and security risk"
    validity: 1.0
    review_date: "2025-02-01"
```

**Validity Calculation.** Достоверность утверждения вычисляется на основе качества источников.

```
validity = 1 - ∏(1 - weight_i × level_i × independence_i)

где:
- weight_i: вес источника (0-1)
- level_i: уровень доказательства (opinion=0.2, citation=0.5, replicated=0.8, formal_proof=1.0)
- independence_i: независимость от других источников (0-1)
```

#### Инварианты принципа

1. Артефакт без источника не может быть сохранён в систему.
2. Источник должен быть конкретным и доступным (URL, DOI, ticket ID).
3. Достоверность утверждения явно вычисляется и отображается.
4. Устаревшие источники помечаются и требуют пересмотра.

#### Метрики

- **Provenance Coverage**: процент кода с явным указанием источника. Цель — более 80% для критических компонентов.
- **Source Freshness**: средний возраст источников. Алерт, если более 50% источников старше 1 года.

---

### R — Rationale Preservation (Сохранение обоснования)

> «Код объясняет "как", но не объясняет "почему". Без "почему" код мёртв через 6 месяцев.»

#### Суть

Каждое нетривиальное решение должно иметь сохранённое обоснование, связанное с кодом. Обоснование включает: почему именно так, какие альтернативы рассмотрены, почему они отвергнуты.

#### Проблема, которую решает

Разработчики читают код и видят «что» и «как», но не понимают «почему». Это приводит к ошибочным рефакторингам, повторению старых ошибок и страху изменять «работающий» код.

#### Практики

**Inline Rationale.** Неочевидный код содержит обоснование.

```python
# @rationale: 100ms delay for read-after-write consistency
# @problem: External API returns stale data immediately after write
# @root_cause: Eventual consistency with 50-100ms replication lag
# @evidence: PERF-456 showed 99.9% consistency with 100ms, 95% with 50ms
# @alternatives_rejected:
#   - Polling: Adds complexity, unreliable
#   - Read-your-writes: Requires API changes, out of scope
# @valid_until: 2024-12-31 (remove after API upgrade to strong consistency)
await asyncio.sleep(0.1)
```

**Architecture Decision Records (ADR).** Архитектурные решения документируются в структурированном формате.

```markdown
# ADR-023: Use PostgreSQL for User Data

## Status
Accepted (2024-01-15)

## Context
We need a database for user profiles, authentication, and preferences.
Expected load: 10K reads/s, 500 writes/s. Data size: ~100GB.

## Decision
Use PostgreSQL 15 with read replicas.

## Rationale
- ACID guarantees required for authentication data
- Mature ecosystem, team expertise
- Cost-effective at our scale

## Alternatives Considered
- **MongoDB**: Rejected. Schema flexibility not needed, ACID harder to achieve.
- **CockroachDB**: Rejected. Over-engineered for current scale, 3x cost.
- **MySQL**: Viable but PostgreSQL has better JSON support for preferences.

## Consequences
- **Positive**: Strong consistency, mature tooling, team knows it
- **Negative**: Vertical scaling limits (mitigated by read replicas)
- **Risks**: Single region (accepted for MVP, revisit at Series B)

## Review Date
2024-07-15
```

**Decision Graph.** Решения связаны в граф, показывающий зависимости и следствия.

```yaml
decisions:
  - id: "ADR-023"
    title: "Use PostgreSQL"
    consequences:
      enables:
        - "ADR-025: Use Prisma ORM"
        - "ADR-028: Read replicas for analytics"
      constrains:
        - "ADR-030: Cannot use DynamoDB for sessions"
    
  - id: "ADR-025"
    title: "Use Prisma ORM"
    depends_on: ["ADR-023"]
    rationale: "Type-safe queries, migrations, team familiarity"
```

**Temporal Rationale.** Обоснования имеют срок годности.

```typescript
/**
 * @rationale "Workaround for third-party API bug"
 * @valid_until 2024-06-01
 * @revisit_trigger "When vendor releases v2.0"
 * @ticket VENDOR-123
 */
function applyWorkaround(data: Data): Data {
  // temporary hack
}
```

#### Инварианты принципа

1. Неочевидный код без `@rationale` не проходит code review.
2. Архитектурные решения без ADR не принимаются.
3. Обоснование включает рассмотренные альтернативы.
4. Временные решения имеют `@valid_until` и `@revisit_trigger`.

#### Метрики

- **Rationale Coverage**: процент неочевидного кода с обоснованием.
- **Decision Freshness**: процент ADR, пересмотренных в срок.

---

### I — Intent Primacy (Приоритет намерения)

> «Первичный артефакт — намерение. Код — производный.»

#### Суть

Разработка начинается с формализации намерения (что система должна делать), а не с написания кода (как она это делает). Код синтезируется из намерения и остаётся связанным с ним.

#### Проблема, которую решает

Традиционная разработка начинается с кода. Намерение существует только в голове разработчика или в Jira-тикете. Код и намерение дрейфуют друг от друга. Через 6 месяцев никто не знает, что код должен делать.

#### Практики

**Intent Specification Language.** Намерения формализуются в структурированном формате.

```yaml
intent:
  id: "user-authentication"
  goal: "Allow users to securely access their accounts"
  
  requirements:
    functional:
      - "User can register with email and password"
      - "User can login with email and password"
      - "User can reset password via email link"
      - "User can enable 2FA"
    
    non_functional:
      - "Login latency < 500ms for p95"
      - "Password hashed with bcrypt (cost 12)"
      - "Session expires after 30 days of inactivity"
  
  constraints:
    - "GDPR compliant (no password in logs)"
    - "Max 5 failed login attempts per hour"
    - "Password min 8 chars, 1 uppercase, 1 number"
  
  acceptance_criteria:
    - "User receives confirmation email within 1 minute"
    - "Invalid login shows generic error (security)"
    - "2FA codes valid for 30 seconds"
```

**Intent Graph.** Намерения связаны в граф (Goal → Requirements → Components → Code).

```
[Goal: User Authentication]
         ↓
[Requirement: User can login]
         ↓
[Component: AuthService]
         ↓
[Code: src/auth/login.ts]
```

**Bidirectional Traceability.** Можно двигаться в обе стороны:
- От намерения к коду: «Где реализовано требование X?»
- От кода к намерению: «Какое требование реализует этот код?»

```typescript
/**
 * @implements requirement:user-can-login
 * @satisfies constraint:max-5-failed-attempts
 */
async function login(email: string, password: string): Promise<Session> {
  // implementation
}
```

**Synthesis over Implementation.** Где возможно, код генерируется из намерения.

```yaml
# Intent
intent:
  id: "send-welcome-email"
  trigger: "user.registered"
  action: "email.send"
  template: "welcome"
  variables:
    user_name: "event.user.name"
    confirmation_link: "generateConfirmationLink(event.user.id)"

# Синтезированный код (генерируется автоматически)
# src/handlers/send-welcome-email.generated.ts
```

#### Инварианты принципа

1. Фича не начинается без Intent Specification.
2. Каждый файл кода связан с намерением (`@implements`).
3. Изменение намерения → пересинтез или ревью кода.
4. Orphan code (код без намерения) помечается для удаления.

#### Метрики

- **Intent Coverage**: процент кода, связанного с намерением.
- **Intent-Code Drift**: семантическое расхождение между намерением и реализацией.

---

### T — Teleological Design (Телеологическое проектирование)

> «Конец определяет начало. Цель диктует архитектуру.»

#### Суть

Проектирование идёт с конца к началу. Сначала определяются свойства успешного результата (EndAnchor), затем эти требования распространяются назад как ограничения на ранние этапы.

#### Проблема, которую решает

Традиционное проектирование идёт «вперёд»: делаем шаг, потом следующий, надеемся достичь цели. Проблемы обнаруживаются поздно, когда исправление дорого. Ранние решения принимаются без учёта их влияния на финальный результат.

#### Практики

**EndAnchor Definition.** Начинаем с определения свойств успешного результата.

```yaml
end_anchor:
  id: "quarterly-report"
  description: "Q4 2024 Financial Report"
  
  properties:
    format: "PDF"
    validity: "> 0.9"  # Все данные из verified sources
    completeness: "100%"
    latency: "< 5 seconds to generate"
    accessibility: "WCAG 2.1 AA"
  
  required_sections:
    - "Executive Summary"
    - "Revenue Analysis"
    - "Risk Assessment"
    - "Forward Guidance"
  
  verification:
    - type: "schema_validation"
    - type: "source_verification"
    - type: "accessibility_audit"
```

**BackConstraint Propagation.** Требования финала распространяются на ранние этапы.

```yaml
back_constraints:
  # От EndAnchor к этапу сбора данных
  - stage: "data_collection"
    derived_from: "end_anchor.validity > 0.9"
    constraints:
      - "All data sources must have validity > 0.8"
      - "Each claim must have ≥ 2 independent sources"
      - "Data freshness < 24 hours"
  
  # От EndAnchor к этапу обработки
  - stage: "data_processing"
    derived_from: "end_anchor.latency < 5s"
    constraints:
      - "Processing time budget: 3 seconds"
      - "Caching required for repeated queries"
  
  # От EndAnchor к этапу рендеринга
  - stage: "rendering"
    derived_from: "end_anchor.accessibility = WCAG 2.1 AA"
    constraints:
      - "All images have alt text"
      - "Color contrast ratio ≥ 4.5:1"
```

**Penultimate Sentinels.** Проверки за 1-2 шага до финала.

```yaml
penultimate_sentinels:
  - stage: "pre_render"
    checks:
      - "All required sections present"
      - "No unresolved placeholders"
      - "Total validity score ≥ 0.9"
    on_failure: "block_and_notify"
  
  - stage: "pre_publish"
    checks:
      - "PDF generated successfully"
      - "Accessibility audit passed"
      - "Stakeholder approval received"
    on_failure: "rollback_to_draft"
```

**Intent-Action Gap Measurement.** На каждом шаге измеряем отклонение от цели.

```
IAG = Σ(weight_i × deviation_i)

где:
- deviation_i = |actual_value - target_value| / target_value
- weight_i = критичность свойства

Если IAG > threshold: alert / block / rollback
```

#### Инварианты принципа

1. Проект не начинается без EndAnchor.
2. Каждый этап имеет BackConstraints, производные от EndAnchor.
3. IAG измеряется на каждом значимом шаге.
4. Нарушение BackConstraint блокирует переход к следующему этапу.

#### Метрики

- **EndAnchor Achievement Rate**: процент проектов, достигших EndAnchor с первой попытки.
- **BackConstraint Violation Rate**: частота нарушений ограничений на ранних этапах.

---

### A — Adaptive Complexity (Адаптивная сложность)

> «Простые случаи должны быть простыми. Сложные случаи должны быть возможными.»

#### Суть

Система должна адаптировать уровень сложности к задаче. Не должно быть «одного решения для всех случаев». Простые задачи решаются просто, сложные — сложнее, но явно.

#### Проблема, которую решает

Две крайности: **over-engineering** (простая задача решена сложно) и **under-engineering** (сложная задача решена слишком просто). Обе приводят к проблемам: первая — к излишней сложности, вторая — к хрупкости.

#### Практики

**Progressive Disclosure.** Сложность раскрывается по мере необходимости.

```yaml
# Уровень 1: Простой случай (80% use cases)
task:
  id: "send-email"
  action: "email.send"

# Уровень 2: Добавляем надёжность
task:
  id: "send-email"
  action: "email.send"
  policy: "retry_default"  # Пресет

# Уровень 3: Полная кастомизация
task:
  id: "send-email"
  action: "email.send"
  policy:
    retries:
      max_attempts: 5
      backoff:
        strategy: "exponential"
        base_ms: 100
        max_ms: 10000
        jitter: 0.1
    circuit_breaker:
      failure_threshold: 10
      timeout_seconds: 60
      half_open_requests: 3
    fallback:
      action: "queue.dead_letter"
      alert: "pagerduty.critical"
```

**Sensible Defaults.** 80% случаев работают «из коробки».

```typescript
// Defaults покрывают большинство случаев
const config = createConfig({
  // Только то, что специфично для проекта
  database: process.env.DATABASE_URL,
});

// Под капотом — разумные defaults
// retry: 3 attempts, exponential backoff
// timeout: 30 seconds
// logging: structured JSON
// metrics: Prometheus format
```

**Escape Hatches.** Всегда можно вернуться к императивному коду.

```typescript
// Декларативный подход работает для 95% случаев
@Workflow("process-order")
class ProcessOrderWorkflow {
  @Step() fetchOrder() {}
  @Step() validatePayment() {}
  @Step() fulfillOrder() {}
}

// Но для edge case можно использовать императив
@Step({ manual: true })
async customStep(context: WorkflowContext) {
  // Полный контроль
  if (specialCondition) {
    await customLogic();
  }
}
```

**Complexity Budget.** Явный лимит на сложность.

```yaml
complexity_budget:
  max_cyclomatic_complexity: 10
  max_dependencies_per_module: 5
  max_nesting_depth: 3
  max_file_length: 300
  
  alerts:
    - threshold: "budget * 0.8"
      action: "warn"
    - threshold: "budget * 1.0"
      action: "block_merge"
```

#### Инварианты принципа

1. Простой случай решается в одну строку (или близко к этому).
2. Каждый уровень сложности явно документирован.
3. Сложность добавляется только когда простое решение не работает.
4. Всегда есть escape hatch для edge cases.

#### Метрики

- **Complexity Distribution**: распределение сложности по модулям (должно быть long-tail: много простых, мало сложных).
- **Escape Hatch Usage**: частота использования императивного кода (должна быть низкой).

---

### S — Separation of Concerns (Разделение ответственности)

> ««Что делать», «как делать» и «почему делать» — три разных вопроса. Они не должны смешиваться.»

#### Суть

Каждый аспект системы (намерение, реализация, обоснование, конфигурация) должен быть изолирован в отдельный артефакт с чёткими границами.

#### Проблема, которую решает

Типичный код смешивает всё: бизнес-логику, инфраструктуру, конфигурацию, обоснование. Это делает код сложным для понимания, тестирования и изменения.

#### Практики

**Intent-Implementation-Rationale Triad.** Три артефакта для одной фичи.

```
feature/
├── intent.yaml           # ЧТО делать
├── implementation/       # КАК делать
│   ├── handler.ts
│   └── repository.ts
├── config.yaml           # Настройки
└── rationale.md          # ПОЧЕМУ так
```

**Policy as Code.** Инфраструктурные политики отделены от бизнес-логики.

```yaml
# policies/retry.yaml
policies:
  - id: "retry_default"
    retries: 3
    backoff: "exponential"
    
  - id: "retry_critical"
    retries: 5
    backoff: "exponential"
    circuit_breaker: true

# handlers/payment.ts — только бизнес-логика
@Policy("retry_critical")
async function processPayment(order: Order) {
  // Чистая бизнес-логика, никакой обработки ошибок
}
```

**Dependency Injection.** Инфраструктура вводится извне.

```typescript
// Бизнес-логика не знает об инфраструктуре
interface OrderRepository {
  findById(id: string): Promise<Order>;
  save(order: Order): Promise<void>;
}

interface PaymentGateway {
  charge(amount: Money, card: Card): Promise<PaymentResult>;
}

// Чистая бизнес-логика
class OrderService {
  constructor(
    private orders: OrderRepository,
    private payments: PaymentGateway,
  ) {}
  
  async processOrder(orderId: string): Promise<void> {
    const order = await this.orders.findById(orderId);
    await this.payments.charge(order.total, order.card);
    order.markAsPaid();
    await this.orders.save(order);
  }
}

// Инфраструктура собирается отдельно
const orderService = new OrderService(
  new PostgresOrderRepository(db),
  new StripePaymentGateway(stripeClient),
);
```

**Layered Architecture.** Чёткие слои с направленными зависимостями.

```
┌─────────────────────────────────────┐
│         Presentation Layer          │  ← HTTP, CLI, Events
├─────────────────────────────────────┤
│         Application Layer           │  ← Use Cases, Workflows
├─────────────────────────────────────┤
│           Domain Layer              │  ← Business Logic, Entities
├─────────────────────────────────────┤
│        Infrastructure Layer         │  ← DB, APIs, Files
└─────────────────────────────────────┘

Правило: Зависимости направлены только вниз.
Domain Layer не знает о Presentation или Infrastructure.
```

#### Инварианты принципа

1. Бизнес-логика не содержит инфраструктурного кода.
2. Конфигурация не хардкодится.
3. Обоснование не в комментариях, а в отдельном файле.
4. Слои имеют чёткие границы и интерфейсы.

#### Метрики

- **Layer Violation Rate**: количество нарушений направления зависимостей.
- **Configuration Hardcode Rate**: процент хардкоженных значений.

---

## Часть III: Patterns и Anti-Patterns

### Pattern 1: Provenance Chain

**Проблема:** «Откуда это взялось?»

**Решение:** Каждый артефакт несёт метаданные о происхождении.

```typescript
interface ProvenanceMetadata {
  // Evidence-First
  sources: {
    url: string;
    date: string;
    author: string;
  }[];
  validity: number;
  
  // Rationale Preservation
  rationale: string;
  alternatives_rejected: {
    option: string;
    reason: string;
  }[];
  
  // Teleological Design
  implements: RequirementID[];
  contributes_to: EndAnchorID;
  
  // Lifecycle
  created_at: Date;
  review_date: Date;
  valid_until?: Date;
}
```

---

### Pattern 2: Contract-First Design

**Проблема:** Код и намерение дрейфуют.

**Решение:** Контракт (намерение) первичен, реализация вторична.

```typescript
// 1. Определяем контракт
interface PaymentGateway {
  /**
   * @intent: Process payment for order
   * @requires: amount > 0 && validCard
   * @ensures: idempotent (safe to retry)
   * @errors: PaymentDeclined | NetworkError
   */
  charge(amount: Money, card: Card): Promise<PaymentResult>;
}

// 2. Генерируем тесты из контракта
const tests = generateContractTests(PaymentGateway);

// 3. Реализация должна удовлетворять контракту
class StripeGateway implements PaymentGateway {
  // Компилятор проверяет соответствие
}
```

---

### Pattern 3: Invariant Guards

**Проблема:** Бизнес-правила нарушаются в runtime.

**Решение:** Инварианты enforced на уровне типов и runtime.

```typescript
// Type-level invariant
type PositiveNumber = number & { __brand: 'positive' };

function assertPositive(n: number): asserts n is PositiveNumber {
  if (n <= 0) throw new InvariantViolation('Must be positive');
}

// Database-level invariant
CREATE TABLE accounts (
  balance DECIMAL(10,2) NOT NULL,
  CONSTRAINT balance_non_negative CHECK (balance >= 0)
);

// Application-level invariant
class Account {
  withdraw(amount: PositiveNumber) {
    const newBalance = this.balance - amount;
    assertPositive(newBalance); // Runtime guard
    this.balance = newBalance as PositiveNumber;
  }
}
```

---

### Pattern 4: Deficit-Driven Selection

**Проблема:** «Какой инструмент/паттерн выбрать?»

**Решение:** Выбор на основе явных дефицитов и constraints.

```yaml
current_deficit:
  id: "slow-api-response"
  metric: "p95 latency > 500ms"
  impact: "User drop-off increased 15%"
  root_cause: "Database queries not cached"

selection:
  candidates:
    - tool: "Redis"
      solves: "slow-api-response"
      price: ["Operational complexity", "Cache invalidation logic"]
      score: 85
      
    - tool: "Application-level caching"
      solves: "slow-api-response" 
      price: ["Memory usage", "Stale data risk"]
      score: 60
  
  selected: "Redis"
  rationale: "Score highest, team has experience, invalidation logic manageable"
```

---

### Pattern 5: Teleological Workflow

**Проблема:** Проект достигает цели случайно или не достигает вовсе.

**Решение:** EndAnchor определяется первым, BackConstraints управляют процессом.

```yaml
workflow:
  name: "Build Analytics Dashboard"
  
  end_anchor:
    properties:
      - "Dashboard loads in < 2 seconds"
      - "Data freshness < 5 minutes"
      - "All metrics have tooltips"
      - "Mobile responsive"
  
  stages:
    - name: "Data Pipeline"
      back_constraints:
        - "Query execution < 1 second (budget for 2s total)"
        - "Incremental updates every 5 minutes"
      verification: "load_test"
    
    - name: "API Layer"
      back_constraints:
        - "Response size < 100KB (for mobile)"
        - "Caching enabled"
      verification: "integration_test"
    
    - name: "Frontend"
      back_constraints:
        - "Bundle size < 200KB"
        - "First contentful paint < 1 second"
      verification: "lighthouse_audit"
  
  penultimate_sentinel:
    before: "deploy_to_production"
    checks:
      - "All back_constraints satisfied"
      - "End-to-end test passes"
      - "Stakeholder approval"
```

---

### Anti-Pattern 1: Mystery Code

**Нарушает:** Evidence-First, Rationale Preservation

```typescript
// ❌ Почему? Откуда? Когда пересмотреть?
await sleep(100);
if (user.score > 75) grantAccess();
```

```typescript
// ✅ С provenance и rationale
/**
 * @source PERF-123
 * @rationale "DB replication lag is 50-100ms, 100ms ensures consistency"
 * @valid_until 2024-12-31
 */
await sleep(100);

/**
 * @source PRD-456
 * @rationale "75th percentile of engagement = 'high-value user'"
 * @evidence "A/B test EXP-789: 75 maximizes retention vs fraud"
 */
if (user.score > 75) grantAccess();
```

---

### Anti-Pattern 2: Forward-Only Design

**Нарушает:** Teleological Design

```typescript
// ❌ Делаем шаг за шагом, надеемся достичь цели
async function buildReport() {
  const data = await fetchData();        // Какой бюджет времени?
  const processed = await process(data); // Какой формат нужен?
  const formatted = format(processed);   // Удовлетворит ли EndAnchor?
  return formatted;
}
```

```typescript
// ✅ EndAnchor первым, BackConstraints управляют
/**
 * @end_anchor: Report with validity > 0.9, latency < 5s
 * @back_constraint: fetch must complete in < 2s
 * @back_constraint: all data sources must have validity > 0.8
 */
async function buildReport(endAnchor: EndAnchor) {
  const data = await fetchData({ 
    timeout: 2000,  // BackConstraint
    minValidity: 0.8 // BackConstraint
  });
  
  // Проверяем IAG на каждом шаге
  assertIAG(data, endAnchor, { maxDrift: 0.1 });
  
  const processed = await process(data);
  assertIAG(processed, endAnchor, { maxDrift: 0.05 });
  
  return format(processed);
}
```

---

### Anti-Pattern 3: Mixed Concerns

**Нарушает:** Separation of Concerns

```typescript
// ❌ Всё смешано
async function processPayment(orderId: string) {
  const STRIPE_KEY = "sk_live_...";  // Конфигурация в коде
  
  for (let attempt = 0; attempt < 3; attempt++) {  // Политика в коде
    try {
      const order = await db.query("SELECT...");  // Инфраструктура
      if (order.amount > 1000) {  // Почему 1000?
        await notifyManager(order);  // Бизнес-логика
      }
      return await stripe.charge(order.amount);
    } catch (err) {
      await sleep(1000 * attempt);  // Retry logic
    }
  }
}
```

```typescript
// ✅ Разделено
// config.yaml
stripe_key: ${env.STRIPE_KEY}
high_value_threshold: 1000

// policies/retry.yaml  
payment_retry:
  max_attempts: 3
  backoff: exponential

// rationale.md
high_value_threshold = 1000 because...

// handlers/payment.ts — только бизнес-логика
@Policy("payment_retry")
async function processPayment(
  order: Order,
  config: Config,
  notifier: Notifier,
  payments: PaymentGateway
) {
  if (order.amount > config.highValueThreshold) {
    await notifier.notifyManager(order);
  }
  return payments.charge(order);
}
```

---

## Часть IV: Практическое Применение

### Code Review Checklist

```markdown
## VERITAS Review Checklist

### V — Verifiability
- [ ] Заявленные свойства имеют автоматические проверки?
- [ ] Property-based тесты для критичной логики?
- [ ] Observability (метрики, логи, трейсы) достаточна?

### E — Evidence-First
- [ ] Критичные константы имеют @source?
- [ ] Архитектурные решения ссылаются на ADR?
- [ ] Нет «магических» значений без объяснения?

### R — Rationale Preservation
- [ ] Неочевидный код имеет @rationale?
- [ ] Указаны альтернативы и причины отказа?
- [ ] Временные решения имеют @valid_until?

### I — Intent Primacy
- [ ] Есть Intent Specification для фичи?
- [ ] Код связан с намерением (@implements)?
- [ ] Можно понять «что» без чтения «как»?

### T — Teleological Design
- [ ] Определён EndAnchor?
- [ ] BackConstraints применяются?
- [ ] Измеряется Intent-Action Gap?

### A — Adaptive Complexity
- [ ] Простые случаи решены просто?
- [ ] Сложность обоснована и явна?
- [ ] Есть escape hatch для edge cases?

### S — Separation of Concerns
- [ ] Бизнес-логика отделена от инфраструктуры?
- [ ] Конфигурация вынесена из кода?
- [ ] Политики отделены от логики?
```

---

### Architecture Review Checklist

```markdown
## VERITAS Architecture Review

### V — Verifiability
- [ ] SLO определены и измеряемы?
- [ ] Есть chaos engineering / stress tests?
- [ ] Критичные инварианты формализованы?

### E — Evidence-First
- [ ] Все ADR имеют источники?
- [ ] Выбор технологий обоснован (benchmark, PoC)?
- [ ] Есть Evidence Ledger?

### R — Rationale Preservation
- [ ] Каждый сервис имеет «Why this exists»?
- [ ] Trade-offs документированы?
- [ ] Есть review dates?

### I — Intent Primacy
- [ ] Архитектура выведена из бизнес-требований?
- [ ] Есть трассировка Feature → Component?
- [ ] Intent Graph существует?

### T — Teleological Design
- [ ] Определены системные EndAnchors?
- [ ] BackConstraints применяются к подсистемам?
- [ ] Компоненты aligned с целями?

### A — Adaptive Complexity
- [ ] Сложность адекватна задаче?
- [ ] Нет over-engineering?
- [ ] Есть путь к упрощению?

### S — Separation of Concerns
- [ ] Слои изолированы?
- [ ] Нет циклических зависимостей?
- [ ] Границы между сервисами чёткие?
```

---

### Adoption Roadmap

**Неделя 1-2: Awareness**

Команда читает принципы VERITAS. Находим 5-10 примеров нарушений в текущем коде. Проводим ретроспективу: какие проблемы из-за нарушения VERITAS.

**Неделя 3-4: Pilot**

Выбираем один новый модуль. Применяем VERITAS с нуля:
- Intent Specification до кода
- ADR для решений
- Property-based тесты
- Полное разделение concerns

Измеряем: легче ли? Меньше ли багов? Быстрее ли code review?

**Месяц 2: Incremental Adoption**

Добавляем VERITAS в Code Review checklist. Новый код пишется с VERITAS. Критичные старые модули рефакторим постепенно.

**Месяц 3+: Tooling**

Автоматические линтеры для VERITAS. Генераторы Intent → Code. Дашборды compliance.

---

## Часть V: VERITAS vs. Другие Принципы

| Принцип | Фокус | Отличие VERITAS |
|---------|-------|-----------------|
| **SOLID** | Дизайн классов в ООП | VERITAS — о системах в целом, включая знания и решения |
| **GRASP** | Распределение ответственности | VERITAS — о разделении намерения, реализации, обоснования |
| **DDD** | Моделирование бизнес-домена | VERITAS добавляет верифицируемость и provenance |
| **12-Factor** | Облачные приложения | VERITAS — о любых системах |
| **Clean Architecture** | Изоляция бизнес-логики | VERITAS включает это + телеологичность + доказательства |

VERITAS — это **мета-принципы**, применимые на всех уровнях: код, архитектура, процессы, знания.

---

## Заключение

VERITAS — это философия intentional, verifiable, and evolvable systems.

**Семь принципов:**

1. **Verifiability**: Если нельзя проверить — это не знание
2. **Evidence-First**: Утверждение без источника — галлюцинация
3. **Rationale Preservation**: Без «почему» код мёртв через 6 месяцев
4. **Intent Primacy**: Намерение первично, код — производный
5. **Teleological Design**: Конец определяет начало
6. **Adaptive Complexity**: Простое — просто, сложное — возможно
7. **Separation of Concerns**: Что, как и почему — разные вопросы

**Ключевая идея:**

> «Каждое утверждение — доказательство. Каждое решение — обоснование. Каждый артефакт — проверяем. Код — производный. Намерение — первично.»
