# DDD Architecture for con7text

## 1. Цель домена

`con7text` предоставляет актуальную документацию по библиотекам через MCP/CLI/API, с фокусом на:
- точность ответа;
- управляемость интеграций (Gemini/Cursor/Claude и др.);
- безопасность и аудит.

## 2. Bounded Contexts

## 2.1 Documentation Retrieval
- Поиск library ID
- Запрос документации
- Ранжирование и валидация результата

## 2.2 Client Session Context
- Идентификация клиента/транспорта
- Профили доступа/ограничений
- Контекст запроса (IDE, version, auth state)

## 2.3 Observability & Audit
- Логирование событий
- Трассировка запроса end-to-end
- Redaction секретов

## 2.4 Governance & Contracts
- Контракты API (schema-first)
- Политики качества (tests, snapshots, CI gates)
- Эволюция контрактов и обратная совместимость

## 3. Core Domain Model

## 3.1 Aggregates

### DocumentationQuery (AR)
- `queryId`
- `libraryId`
- `queryText`
- `clientRef`
- `status` (`created|resolved|fetched|completed|failed`)

Инварианты:
- `queryText` обязателен;
- `libraryId` валиден по формату;
- переходы статусов только по state machine.

### ClientSession (AR)
- `sessionId`
- `clientIde`
- `transport`
- `authMode`

Инварианты:
- клиент должен быть определен до выполнения tool-call;
- чувствительные данные не попадают в события аудита.

### AuditTrail (AR)
- `traceId`
- `events[]`
- `riskFlags[]`

Инварианты:
- все внешние вызовы имеют audit event;
- payload логов проходит redaction policy.

## 3.2 Value Objects
- `LibraryId`
- `QueryText`
- `ClientIdentity`
- `TransportType`
- `RedactedPayload`
- `PolicyDecision`

## 3.3 Domain Events
- `QueryStarted`
- `LibraryResolved`
- `DocsFetched`
- `QueryFailed`
- `ClientDetected`
- `PolicyViolationDetected`
- `AuditRecorded`

## 4. Application Layer (Use Cases)

- `ResolveLibraryUseCase`
- `FetchDocsUseCase`
- `HandleClientRequestUseCase`
- `RecordAuditUseCase`
- `ValidateContractChangeUseCase`

## 5. Infrastructure Layer

- MCP/HTTP adapters
- Repository для аудита (file/db)
- Внешние API connectors
- Schema registry + CI checks

## 6. Repository structure (target)

```txt
con7text/packages/mcp/src/
  domain/
  application/
  infrastructure/
  interface/
```

## 7. Tactical migration plan

1. Выделить VO + Events из текущего кода без изменения поведения.
2. Вынести orchestration в use cases.
3. Ввести единый ACL для внешних клиентов и MCP.
4. Включить contract + snapshot quality gates в CI.
