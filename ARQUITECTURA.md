# Arquitectura ampliada: Payments, Refunds & Invoicing Service

Complementa a `SPRINT.md`. Dominio inspirado en lo que ya operas (reembolsos RAM, pagos referenciados, timbrado fiscal), para que cada falla sea una historia contable en la entrevista.

## Prioridades (para que quepa en 7 días)

| Nivel | Qué incluye | Cuándo |
|---|---|---|
| **P1 Núcleo** | Payment, Refund, Money, máquinas de estado, 6 endpoints, idempotencia, manejo global de errores, Postgres + Flyway | Días 1-3 |
| **P2 Robustez** | Invoice + timbrado simulado, reintentos con backoff, outbox, locking optimista, correlation ID, Actuator completo, ArchUnit | Días 3-5 |
| **P3 Extra** | Circuit breaker, OpenAPI, Testcontainers, métricas custom | Solo si sobra tiempo |

**Aviso honesto:** esto completo son unas 12-15 horas de construcción y el sprint da ~1 h/día para construir. Haz P1 completo, P2 en lo que alcance y P3 solo después de la entrevista. Si un día se atrasa, se recorta de abajo hacia arriba (P3, luego P2), nunca el drill.

---

## 1. Dominio (`domain`)

### Value objects (records inmutables)
| Clase | Concepto que practicas |
|---|---|
| `Money` (amount, currency) | Constructor compacto, validación, `add`/`subtract` que devuelven nuevo objeto |
| `PaymentId`, `RefundId`, `InvoiceId`, `CustomerId` | Records con UUID, `equals`/`hashCode` |
| `PaymentReference` | Código de pago referenciado (formato validado, expiración) |
| `IdempotencyKey` | Value object con validación de longitud y formato |
| `EmailAddress`, `TaxId` (RFC) | Validación en constructor, "parse, don't validate" |

### Enums con comportamiento
| Enum | Transiciones válidas |
|---|---|
| `PaymentStatus` | `PENDING → CONFIRMED / EXPIRED / CANCELLED`; `CONFIRMED → PARTIALLY_REFUNDED / REFUNDED` |
| `RefundStatus` | `REQUESTED → PROCESSING → COMPLETED`; `PROCESSING → PENDING_RETRY → PROCESSING`; `PENDING_RETRY → FAILED_PERMANENT`; `REQUESTED → CANCELLED` |
| `InvoiceStatus` | `DRAFT → STAMPING → STAMPED → CANCELLED`; `STAMPING → STAMP_FAILED` |
| `FailureReason` | `GATEWAY_TIMEOUT`, `INSUFFICIENT_FUNDS`, `INVALID_ACCOUNT`, `STAMP_REJECTED`... |
| `PaymentMethodType` | `CARD`, `REFERENCE`, `TRANSFER` |

Cada enum expone `canTransitionTo(next)` y las transiciones inválidas lanzan excepción de dominio.

### Entidades y agregados
| Clase | Rol | Reglas |
|---|---|---|
| `Customer` | Entidad | Email único, RFC opcional |
| `Payment` (raíz) | Agregado | Mantiene `refundedTotal`; nunca excede lo pagado |
| `PaymentMethod` (abstracta) → `CardPayment`, `ReferencePayment`, `TransferPayment` | Herencia/polimorfismo | Cada una valida sus propios datos |
| `Refund` | Entidad | Pertenece a un `Payment`; guarda intentos y último error |
| `Invoice` + `InvoiceLine` | Agregado | Total = suma de líneas; no se edita una vez timbrada |
| `AuditEntry` | Registro inmutable | Quién, qué, cuándo, antes/después |
| `RetryPolicy` | Estrategia | Backoff exponencial con tope y máximo de intentos |
| `RetryQueue` | Estructura de datos | `PriorityQueue` por `nextAttemptAt` |

### Políticas (interfaz + polimorfismo)
- `RefundPolicy` → `FullRefundPolicy`, `PartialRefundPolicy`, `ExpiredWindowPolicy` (ventana de 30 días).
- `FeePolicy` → `NoFee`, `FixedFee`, `PercentageFee`.

### Eventos de dominio (`sealed interface DomainEvent`)
`PaymentConfirmed`, `RefundRequested`, `RefundCompleted`, `RefundFailed`, `InvoiceStamped`. Practicas `sealed`, `record` y `switch` con patrones.

### Excepciones de dominio (jerarquía)
```
DomainException
├── InvalidMoneyException
├── InvalidStateTransitionException
├── RefundExceedsPaymentException
├── RefundWindowExpiredException
├── CurrencyMismatchException
├── PaymentNotRefundableException
└── DuplicateRequestException
```

---

## 2. Aplicación (`application`)

### Puertos de entrada (casos de uso)
`CreatePaymentUseCase`, `ConfirmPaymentUseCase`, `CancelPaymentUseCase`, `RequestRefundUseCase`, `RetryRefundUseCase`, `CancelRefundUseCase`, `GetRefundUseCase`, `ListRefundsUseCase`, `StampInvoiceUseCase`, `CancelInvoiceUseCase`, `ReprocessFailedRefundsUseCase`, `GetOpsStatsUseCase`.

### Puertos de salida
| Puerto | Adaptadores |
|---|---|
| `PaymentRepository`, `RefundRepository`, `InvoiceRepository`, `CustomerRepository` | memoria (Día 2) → JPA (Día 3) |
| `AuditLogPort` | JPA |
| `IdempotencyStorePort` | memoria → tabla `idempotency_keys` |
| `PaymentGatewayPort` | Simulado, con latencia y fallos configurables |
| `TaxStampingPort` (PAC) | Simulado, con rechazos y timeouts configurables |
| `EventPublisherPort` | Outbox en BD, publicado por un job |
| `ClockPort` | Reloj inyectable (hace los tests determinísticos) |

### Servicios
Un servicio por caso de uso, con `@Transactional` solo en la capa de aplicación (nunca en el dominio). Los DTOs de entrada y salida del dominio son `Command` y `Result` (records), distintos de los DTOs web.

---

## 3. Infraestructura (`infrastructure`)

### Controladores y endpoints (prefijo `/api/v1`)

**`PaymentController`**
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/payments` | Crea pago (soporta los 3 métodos) |
| GET | `/payments/{id}` | Detalle con total reembolsado |
| GET | `/payments?status=&customerId=&from=&to=&page=&size=` | Búsqueda paginada |
| POST | `/payments/{id}/confirm` | Simula confirmación del banco |
| POST | `/payments/{id}/cancel` | Cancela un pago pendiente |

**`RefundController`**
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/refunds` | Solicita reembolso; **header `Idempotency-Key` obligatorio** |
| GET | `/refunds/{id}` | Detalle con historial de intentos |
| GET | `/refunds?status=&paymentId=&page=&size=` | Búsqueda paginada |
| POST | `/refunds/{id}/retry` | Reintento manual |
| POST | `/refunds/{id}/cancel` | Cancela si aún no se procesó |

**`InvoiceController`**
| Método | Ruta | Descripción |
|---|---|---|
| POST | `/invoices` | Crea borrador desde un pago |
| POST | `/invoices/{id}/stamp` | Timbra (llama al PAC simulado) |
| GET | `/invoices/{id}` | Detalle y estado |
| POST | `/invoices/{id}/cancel` | Cancela una factura timbrada |

**`OpsController` (solo perfil `lab`)**
| Método | Ruta | Descripción |
|---|---|---|
| GET | `/ops/stats` | Conteos por estado, tasa de fallos, antigüedad del reembolso más viejo |
| POST | `/ops/reprocess` | Fuerza el reproceso de fallidos |
| GET | `/ops/failed-refunds` | Lista los `FAILED_PERMANENT` para revisión manual |

**`AuditController`**: `GET /audit?entityId=&entityType=`

**Actuator (L2 puro):** `/actuator/health` (con `liveness` y `readiness`), `/actuator/info`, `/actuator/metrics`, `/actuator/loggers` (cambiar nivel de log **en caliente**), `/actuator/threaddump`, `/actuator/heapdump`. Solo accesibles en el perfil `lab`.

### Adaptadores de entrada adicionales
- `adapter.in.scheduler`: `RefundRetryJob` (cada minuto toma de la cola lo vencido), `OutboxPublisherJob`, `ExpirePaymentsJob`.
- `adapter.in.web.error`: `GlobalExceptionHandler` con `@RestControllerAdvice` que devuelve **`ProblemDetail` (RFC 7807)** con `code` estable (`REFUND_EXCEEDS_PAYMENT`, etc.), mapeo claro de excepción a HTTP (400, 404, 409, 422, 503).
- `adapter.in.web.filter`: `CorrelationIdFilter` (lee o genera `X-Correlation-Id` y lo pone en MDC).

### Adaptadores de salida
`persistence.memory`, `persistence.jpa` (entidades JPA separadas de las del dominio, con mappers), `gateway.SimulatedPaymentGateway`, `stamping.SimulatedPacClient`.

---

## 4. Base de datos (Flyway, carpeta `db/migration`)

| Migración | Contenido |
|---|---|
| `V1__core.sql` | `customers`, `payments`, `refunds` con llaves foráneas y `CHECK` en montos |
| `V2__invoices.sql` | `invoices`, `invoice_lines` |
| `V3__audit_idempotency_outbox.sql` | `audit_log`, `idempotency_keys` (UNIQUE), `outbox_events` |
| `V4__indexes.sql` | Índices básicos. **Deja 2 índices sin crear a propósito** para el drill de `EXPLAIN` del Día 3 |
| `V5__seed.sql` o script aparte | ~10k pagos, ~2k reembolsos, con **datos sucios plantados** (duplicados, reembolsos > pago, estados inconsistentes) para detectarlos con SQL |

Agrega `@Version` (locking optimista) en `Payment` y `Refund`.

---

## 5. Robustez (lo que separa "una API" de "una API operable")

| Tema | Qué se implementa | Qué demuestra en la entrevista |
|---|---|---|
| Idempotencia | Mismo `Idempotency-Key` devuelve la misma respuesta, sin duplicar | Reembolsos duplicados en producción |
| Concurrencia | `@Version`; dos reembolsos simultáneos no exceden el pago | Condiciones de carrera |
| Reintentos | Backoff exponencial + máximo de intentos + `FAILED_PERMANENT` | Acción correctiva definitiva |
| Timeouts | En el cliente del gateway y del PAC | Hilos colgados, pool agotado |
| Observabilidad | Correlation ID en cada log, logs estructurados, `/actuator/*` | RCA con logs |
| Config externa | Variables de entorno, perfiles `dev` / `lab` | ConfigMap y Secret en k8s |
| Apagado limpio | `server.shutdown=graceful` | Despliegues sin cortar peticiones |
| Pruebas | Unitarias del dominio, `@WebMvcTest` de controllers, **ArchUnit** que falla si `domain` importa Spring | Hexagonal verificada, no solo dibujada |
| P3 | Circuit breaker (Resilience4j), springdoc OpenAPI, Testcontainers | Opcional |

---

## 6. Casos de negocio que debe resolver (también son tu banco de preguntas)

1. Reembolso parcial seguido de otro parcial que sumados exceden el pago → `REFUND_EXCEEDS_PAYMENT`.
2. Reembolso sobre pago aún `PENDING` o `EXPIRED` → rechazado.
3. Reembolso fuera de la ventana de 30 días → `RefundWindowExpiredException`.
4. Mismo `Idempotency-Key` enviado dos veces → misma respuesta, un solo registro.
5. Dos reembolsos concurrentes sobre el mismo pago → uno gana, el otro recibe 409.
6. Moneda distinta entre pago y reembolso → `CurrencyMismatchException`.
7. Gateway con timeout → `PENDING_RETRY`; tras 5 intentos → `FAILED_PERMANENT` y aparece en `/ops/failed-refunds`.
8. PAC rechaza el timbrado → `STAMP_FAILED`, con reintento manual.
9. Cancelar una factura ya cancelada → transición inválida.
10. Pago con referencia expirada → el job lo pasa a `EXPIRED`.
11. Reembolso que se completa pero el evento al outbox falla → debe quedar atómico (misma transacción).
12. Reinicio del servicio con reembolsos en `PROCESSING` → el job los retoma sin duplicar.

---

## 7. Mapeo a los días del sprint

| Día | Construcción ampliada |
|---|---|
| **1** | Value objects, enums con transiciones, `Payment`, `Refund`, herencia de métodos de pago, políticas, jerarquía de excepciones. Claude hace 2 (`Money`, `RefundStatus`), tú el resto |
| **2** | Puertos, servicios P1, adaptador en memoria, `PaymentController` y `RefundController`, `GlobalExceptionHandler`, idempotencia, `CorrelationIdFilter`, ArchUnit |
| **3** | Flyway + esquema, adaptador JPA con mappers, seed sucio, `@Version`, `RefundRetryJob`, índices a medias |
| **4** | Actuator completo, Logback a archivo, perfiles, `InvoiceController` + PAC simulado si hay tiempo, despliegue por SSH y Docker |
| **5** | `OpsController`, outbox, fallas plantadas por ti dentro de la API, k3s |
| **6-7** | Sin código nuevo importante: pipeline, rollback y repaso |

## 8. Estructura de paquetes final

```
com.emiliano.refunds
├── domain
│   ├── model (money, payment, refund, invoice, customer, audit)
│   ├── policy
│   ├── event
│   └── exception
├── application
│   ├── port.in
│   ├── port.out
│   ├── service
│   └── command / result
└── infrastructure
    ├── adapter.in.web (controller, dto, error, filter)
    ├── adapter.in.scheduler
    └── adapter.out
        ├── persistence (memory, jpa, mapper)
        ├── gateway
        ├── stamping
        └── events (outbox)
src/main/resources/db/migration (Flyway)
```
