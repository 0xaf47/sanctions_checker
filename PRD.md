# PRD: Sanctions Monitoring Platform (v2, rebuild from scratch)

## 1) Обзор приложения и цели

### 1.1 Контекст
Текущая версия проекта уже подтверждает жизнеспособность идеи:
- есть ETL-пайплайн, который загружает XML (UN consolidated), нормализует данные и импортирует их в API;
- есть полнотекстовый и fuzzy-поиск по санкционным сущностям;
- есть контейнеризация, базовые миграции и API-ключ для импорта.

Новая версия должна сохранить сильные стороны, но стать безопаснее, проще в сопровождении одним разработчиком и готовой к поэтапному масштабированию с помощью ИИ-агентов.

### 1.2 Проблема
Нужно быстро и регулярно проверять физлиц/юрлиц по разнородным санкционным спискам (XML/CSV/HTML/API), при этом:
- не ломаться при изменениях форматов источников;
- давать релевантный поиск (алиасы, транслитерации, опечатки);
- обеспечивать безопасность и операционную надежность (мониторинг, алерты, восстановление).

### 1.3 Цели (MVP + evolution)
1. Собрать стабильный и безопасный v2 с ядром из 1-2 источников (UN + один приоритетный источник).
2. Обеспечить операционную прозрачность: health checks, ETL-история, Telegram-оповещения.
3. Упростить расширение: новые парсеры, новая логика матчинга, переход к микросервисам без полного переписывания.
4. Создать архитектуру, удобную для solo-разработки с ИИ:
   - четкие контракты между модулями,
   - строгие схемы данных,
   - автотесты + миграции,
   - минимизация «скрытой магии».

### 1.4 Non-goals (для первой версии)
- Не строить enterprise-инфраструктуру «как у банка» на старте.
- Не реализовывать сложный ML matching до стабилизации rule-based поиска.
- Не подключать десятки источников сразу.

---

## 2) Целевая аудитория

### 2.1 Primary users
- Малый/средний бизнес, комплаенс-специалисты, фин/крипто-операторы, которым нужна быстрая предварительная санкционная проверка.
- Сам разработчик (owner/operator), которому нужен удобный мониторинг работоспособности.

### 2.2 Secondary users
- Внутренние интеграторы (боты, CRM, внутренние сервисы), использующие API поиска.

### 2.3 User jobs-to-be-done
- Проверить имя/компанию с настраиваемым порогом точности.
- Понять, почему найден матч (какие поля совпали, какая программа санкций).
- Получать уведомления о падениях ETL/API и об успешных обновлениях данных.

---

## 3) Основные функции и функциональность

## Feature Group A — Ingestion & ETL

### A1. Source registry
Описание:
- Единый реестр источников: код, URL/API endpoint, формат, расписание, флаг активности, приоритет.

Technical considerations:
- Конфиг источников хранится в БД + YAML fallback для локальной отладки.
- Плагины-парсеры реализуют единый интерфейс `parse(raw_file) -> list[RawEntity]`.

Acceptance criteria:
- Можно добавить новый источник без изменения core ETL (только новый parser module + запись в source registry).
- Отключенный источник не запускается в расписании.

### A2. ETL orchestration
Описание:
- Планировщик задач ETL (cron/manual), ретраи, тайм-ауты, идемпотентность загрузки.

Technical considerations:
- На старте: APScheduler/Celery beat (простой режим).
- Идемпотентность через fingerprint версии файла/response (sha256).

Acceptance criteria:
- Повторный запуск с тем же snapshot не создает дубли.
- Ошибка одного источника не валит весь ETL batch.

### A3. Normalization pipeline
Описание:
- Приведение к унифицированной схеме сущности: names, aliases, identifiers, addresses, sanctions, metadata.

Technical considerations:
- Отдельный слой маппинга raw -> canonical model.
- Валидация схем через Pydantic models.

Acceptance criteria:
- Некорректные записи логируются в quarantine-таблицу/файл с причиной.
- Валидные записи проходят в upsert без падения процесса.

### A4. Import API (internal)
Описание:
- Внутренний endpoint для bulk import с подписью/ключом и rate-limits.

Acceptance criteria:
- При частичной ошибке возвращается детальный отчет (processed/created/updated/failed).
- Поддерживается batch size конфигурацией.

---

## Feature Group B — Search & Match

### B1. Fast exact + fuzzy search
Описание:
- Поиск по canonical_name, aliases, частичным совпадениям, транслитерации.

Technical considerations:
- PostgreSQL: `tsvector` + `pg_trgm` + индекс по aliases.
- Порог similarity настраивается per-request.

Acceptance criteria:
- Запросы до 20 результатов работают стабильно на датасете 100k+ сущностей.
- Для каждого результата возвращается score и explain-поля.

### B2. Explainability of match
Описание:
- API возвращает объяснение: какие поля совпали, какой алгоритм дал вес.

Acceptance criteria:
- Каждый результат содержит `match_score` (0..1) и `match_reasons`.
- Пользователь может отличить «точное совпадение» от «вероятного».

### B3. Multi-list filtering
Описание:
- Фильтрация по источникам/программам санкций/типу сущности.

Acceptance criteria:
- При пустом фильтре поиск по всем активным источникам.
- С фильтром возвращаются только релевантные записи.

---

## Feature Group C — Security & Compliance Baseline

### C1. HTTPS everywhere
Описание:
- Трафик к публичному API только по HTTPS.

Technical considerations:
- Reverse proxy (Caddy/Nginx + Let's Encrypt).
- HSTS и редирект HTTP->HTTPS.

Acceptance criteria:
- HTTP-запросы редиректятся на HTTPS.
- TLS-сертификаты обновляются автоматически.

### C2. DB isolation
Описание:
- БД недоступна извне.

Technical considerations:
- Убрать проброс `5432:5432` в production.
- Частная docker network/VPC.
- Отдельные роли DB: read/write/etl.

Acceptance criteria:
- Внешнее подключение к БД невозможно без bastion/VPN.
- Приложения используют least-privilege учетные записи.

### C3. Secret management
Описание:
- Безопасное хранение API keys, DB credentials, bot token.

Acceptance criteria:
- Нет секретов в git/образах.
- Есть ротация ключей (manual + documented runbook).

### C4. AuthN/AuthZ
Описание:
- Разделение прав: публичный search API, внутренний import API, admin endpoints.

Acceptance criteria:
- Import endpoint недоступен без валидного service token.
- Ограничения частоты запросов для публичного API.

---

## Feature Group D — Observability & Reliability

### D1. Structured logging
- JSON-логи с correlation_id и source_id.

Acceptance criteria:
- По log_id можно отследить весь путь: ingest -> normalize -> upsert -> search.

### D2. Metrics + health
- `/health/live`, `/health/ready`, `/metrics` (Prometheus format).

Acceptance criteria:
- Ready=false при недоступной БД/критичных зависимостях.

### D3. Telegram alerting bot
Описание:
- Алерты в Telegram при:
  - падении ETL,
  - недоступности API,
  - успешной загрузке новой версии источника,
  - аномально малом/большом числе обновлений.

Acceptance criteria:
- Настраиваемые каналы: `critical`, `info`.
- Дедупликация одинаковых ошибок в пределах окна (например, 30 минут).

---

## Feature Group E — Performance & Async

### E1. Async I/O
- Асинхронные HTTP-загрузки источников и async DB operations.

Acceptance criteria:
- ETL не блокируется медленным источником (параллелизм конфигурируемый).

### E2. Caching layer
- Redis для кэширования популярных поисковых запросов.

Acceptance criteria:
- P95 latency для повторных запросов ниже чем без кэша (метрика в дашборде).

### E3. Background jobs
- Отложенные задачи: recalculation indexes, warmup cache, data quality checks.

Acceptance criteria:
- Long-running операции не блокируют API response path.

---

## Feature Group F — ML-assisted matching (Phase 3+)

### F1. Candidate generation + reranking
- Этап 1: быстрый rule-based candidate retrieval.
- Этап 2: ML reranker (например, gradient boosting / sentence embeddings).

Acceptance criteria:
- ML можно отключить флагом без деградации базовой функциональности.
- Есть offline-evaluation pipeline (precision@k, recall@k).

---

## 4) Рекомендации по техническому стеку

## 4.1 Базовая рекомендация (для solo + AI)
- Backend API: FastAPI
- DB: PostgreSQL + pg_trgm + full-text
- Cache/queue: Redis
- ETL workers: Python + Celery/RQ (начать с RQ/Celery-light)
- Migrations: Alembic
- Infra: Docker Compose (dev), далее optional Kubernetes/nomad не раньше Phase 3
- Reverse proxy/TLS: Caddy (проще для авто-HTTPS)
- Monitoring: Prometheus + Grafana + Alertmanager + Telegram bridge

Почему это оптимально:
- Минимум контекста и когнитивной нагрузки для 1 разработчика.
- Большая совместимость с AI-агентами и примерами.
- Легкая эволюция к сервисному разделению.

## 4.2 Альтернативы (кратко)
- Go для ETL/API: выше производительность, но выше стоимость переписывания и поддержки в одиночку.
- Elasticsearch/OpenSearch: мощный поиск, но усложнение ops. Рекомендуется после достижения предела PostgreSQL.

Решение:
- Стартовать на PostgreSQL-first архитектуре.

---

## 5) Концептуальная модель данных

## 5.1 Core entities

### `source`
- `id: int`
- `code: string (unique)`
- `name: string`
- `type: enum(xml,csv,api,html,json)`
- `endpoint: string`
- `schedule_cron: string|null`
- `is_active: bool`
- `created_at: datetime`
- `updated_at: datetime`

### `entity`
- `id: bigint`
- `entity_type: enum(person,organization,vessel,aircraft,other)`
- `canonical_name: text`
- `normalized_name: text` (for matching)
- `primary_source_id: int`
- `is_active: bool`
- `created_at: datetime`
- `updated_at: datetime`

### `entity_alias`
- `id: bigint`
- `entity_id: bigint (FK)`
- `alias: text`
- `normalized_alias: text`
- `quality: enum(low,medium,high,unknown)`

### `entity_identifier`
- `id: bigint`
- `entity_id: bigint (FK)`
- `id_type: string` (passport, tax_id, registration, etc.)
- `id_value: string`
- `country: string|null`

### `entity_address`
- `id: bigint`
- `entity_id: bigint (FK)`
- `country: string|null`
- `region: string|null`
- `city: string|null`
- `address_line: text|null`
- `postal_code: string|null`

### `sanction_record`
- `id: bigint`
- `entity_id: bigint (FK)`
- `source_id: int (FK)`
- `program: string|null`
- `sanction_type: string|null`
- `listed_on: date|null`
- `last_updated_on: date|null`
- `legal_basis: text|null`
- `raw_payload: jsonb`
- `created_at: datetime`

### `entity_version`
- `id: bigint`
- `entity_id: bigint (FK)`
- `version_no: int`
- `diff_summary: jsonb`
- `snapshot_payload: jsonb`
- `created_at: datetime`

### `etl_run`
- `id: bigint`
- `source_id: int (FK)`
- `run_type: enum(manual,scheduled,retry)`
- `status: enum(success,partial_failed,failed)`
- `started_at: datetime`
- `finished_at: datetime|null`
- `download_hash: string|null`
- `processed_count: int`
- `created_count: int`
- `updated_count: int`
- `failed_count: int`
- `error_log: text|null`

### `api_key`
- `id: int`
- `key_hash: string` (не хранить plaintext)
- `owner: string`
- `scope: enum(search,import,admin)`
- `is_active: bool`
- `expires_at: datetime|null`

## 5.2 Key relationships
- `entity 1..n entity_alias`
- `entity 1..n entity_identifier`
- `entity 1..n entity_address`
- `entity 1..n sanction_record`
- `entity 1..n entity_version`
- `source 1..n sanction_record`
- `source 1..n etl_run`

---

## 6) Принципы UI/UX

Даже при API-first подходе нужен минимальный web UI (admin + search page):

1. Простая форма поиска
   - поле имени/компании,
   - слайдер/поле порога точности,
   - фильтры по источникам.

2. Прозрачность результата
   - отображать score,
   - показывать aliases, program, sanction type, источник, дату обновления.

3. UX для оператора
   - экран ETL-статусов: последние запуски, ошибки, длительность.
   - quick actions: rerun source, disable source.

4. Internationalization-ready
   - UTF-8, поддержка кириллицы/латиницы/иероглифов.

Acceptance criteria:
- Новичок понимает результат поиска без чтения документации.
- Оператор за <2 минут локализует причину сбоя ETL.

---

## 7) Соображения по безопасности

1. Сеть и доступ
- API за reverse proxy, HTTPS-only.
- DB не публикуется наружу.
- Разделение окружений: dev/stage/prod.

2. Данные и секреты
- Секреты через env/secret store.
- API keys хранить в hashed виде (bcrypt/argon2).
- Регулярная ротация ключей.

3. App security baseline
- Input validation на всех endpoint.
- Ограничение размера payload и timeout.
- Rate limiting + basic WAF rules на прокси.

4. Auditability
- Логировать admin/import actions.
- Хранить историю изменений сущностей и источников.

5. Supply chain
- Пиновать версии зависимостей.
- SAST/Dependency scanning в CI (например, pip-audit + bandit).

---

## 8) Этапы разработки / Milestones

## Milestone 0 — Discovery & foundation (1 неделя)
- Формализация доменной модели.
- Репозиторий, code style, CI skeleton.
- Подготовка ADR документов (2-3 ключевых решения).

Definition of done:
- Архитектурная схема v2 утверждена.
- Есть backlog с приоритетами Must/Should/Could.

## Milestone 1 — Secure MVP core (2-3 недели)
- FastAPI + PostgreSQL + Alembic.
- Source registry + один parser (UN) + ETL run history.
- Search API с scoring + explain.
- HTTPS + закрытая DB + API key scopes.

DoD:
- E2E flow: download -> normalize -> import -> search.
- Telegram alerts для success/failure ETL.
- Smoke/integration tests green.

## Milestone 2 — Reliability & performance (2 недели)
- Async orchestration, retries, dead-letter handling.
- Redis cache для top queries.
- Наблюдаемость: metrics/dashboard/alerts.

DoD:
- SLA draft: API availability >= 99% (non-financial SLA, internal target).
- P95 latency задокументирован и улучшается с cache.

## Milestone 3 — Multi-source expansion (3-4 недели)
- Добавление 3-5 источников (по приоритету ценности).
- Улучшение нормализации/дедупликации.
- Фильтрация по источникам в API/UI.

DoD:
- Новые источники подключаются через plugin-паттерн.
- Покрытие интеграционными тестами на каждый источник.

## Milestone 4 — ML-ready architecture (опционально)
- Dataset для offline-оценки матчинга.
- Прототип reranker модели.
- Feature flag для безопасного rollout.

---

## 9) Потенциальные проблемы и способы их решения

1. Дрейф форматов источников
- Решение: контрактные тесты парсеров + alert при росте parse failures.

2. Рост дублей и ложных матчей
- Решение: разделить candidate retrieval и final scoring; добавить feedback loop.

3. Перегрузка API при массовом поиске
- Решение: rate limit + cache + пагинация + async logging.

4. Сложность поддержки в одиночку
- Решение: строгая модульность + авто-документация + генерация boilerplate через AI templates.

5. Риски безопасности
- Решение: security checklist в CI/CD и ежемесячный hardening review.

---

## 10) Потенциальные затраты

## 10.1 Базовые
- VPS/Cloud instance (API + ETL + DB): ~$20-100/мес в зависимости от нагрузки.
- Объектное хранилище для snapshots (опционально): ~$5-20/мес.
- Мониторинг (self-hosted): минимально, но требует времени.

## 10.2 Инструменты
- LLM подписки/агенты: ~$20-200/мес.
- Домен + TLS обычно low-cost/встроено.

## 10.3 Ростовые
- Redis managed, backup storage, вторичный инстанс БД.
- ML-инфраструктура (если включать embeddings/reranking).

---

## 11) Возможности будущего расширения

1. Микросервисная декомпозиция
- `ingestion-service`
- `normalization-service`
- `search-service`
- `notification-service`

2. Data product features
- API отчеты по изменениям санкционных профилей.
- Watchlists пользователей и webhook-уведомления.

3. ML
- Entity resolution model.
- Semantic search по notes/legal basis.

4. Enterprise-ready
- SSO/OIDC, tenant isolation, audit exports.

---

## 12) Техническая декомпозиция (sprint-ready backlog)

## Sprint A — Platform bootstrap
- [ ] Инициализировать новый репозиторий/структуру модулей.
- [ ] Добавить Alembic и базовые миграции.
- [ ] Поднять локальный compose без внешнего доступа к DB.

## Sprint B — Core domain + ETL v1
- [ ] Реализовать domain models + repositories.
- [ ] Реализовать parser interface + UN parser.
- [ ] Реализовать ETL run tracking.

## Sprint C — Search API v1
- [ ] Реализовать search endpoint с filters + score + explain.
- [ ] Добавить индексы, тюнинг SQL.

## Sprint D — Security hardening
- [ ] HTTPS reverse proxy.
- [ ] Key hashing + key scopes.
- [ ] Rate limiting + payload limits.

## Sprint E — Ops & monitoring
- [ ] Prometheus/Grafana dashboard.
- [ ] Telegram bot alerts.
- [ ] Runbooks: incident response, key rotation, backup/restore.

## Sprint F — Scale & quality
- [ ] Redis cache.
- [ ] Асинхронные очереди.
- [ ] Test suite growth (unit + integration + contract tests).

---

## 13) Диаграммы (текстовые)

## 13.1 High-level architecture
```text
[Sources: XML/CSV/API]
          |
          v
  [Ingestion Workers] ---> [Raw Snapshot Storage]
          |
          v
 [Normalization Pipeline] ---> [Quarantine Store]
          |
          v
     [Import Service/API]
          |
          v
   [PostgreSQL + Indexes] <--> [Redis Cache]
          |
          v
      [Search API/UI]
          |
          v
   [Users / Integrations]

[Observability Stack] monitors all blocks
[Telegram Bot] receives alerts/events
```

## 13.2 Data flow (ETL)
```text
download -> validate -> parse -> normalize -> deduplicate -> upsert -> versioning -> metrics/alerts
```

---

## 14) Риски текущей версии, которые закрываем в v2

- Секреты и ключи в открытом виде.
- Потенциально открытая БД наружу.
- Недостаточная изоляция import API.
- Частично дублирующаяся логика и смешение уровней (router/crud/SQL).
- Недостаточная формализация ошибок и retry-политик.
- Ограниченная наблюдаемость (нет единой метрик-модели).

---

## 15) Открытые вопросы (нужно подтвердить перед финальной реализацией)

1. Какие 3-5 источников после UN являются приоритетными в первом релизе?
2. Нужен ли публичный UI для клиентов или только API + внутренняя админка?
3. Какие целевые SLA/SLO по API и ETL считаем обязательными?
4. Какая стратегия деплоя предпочтительна: один VPS или managed cloud сервисы?
5. Нужны ли юридически значимые отчеты (PDF/экспорт) уже в MVP?
6. На каком языке должен быть интерфейс: RU only или сразу RU/EN?
7. Планируется ли мультиарендность (разные клиенты в одном инстансе)?
8. В какой момент готовы включать ML-reranking: после какого объема данных/ошибок матчинга?

> Важно: после ваших ответов на эти вопросы PRD можно быстро уточнить до финальной версии implementation-ready с оценкой сроков по часам.
