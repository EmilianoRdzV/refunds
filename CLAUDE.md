# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Contexto

Proyecto de práctica para un sprint de 7 días (25-31 oct 2026) de preparación a una entrevista de **soporte L2 Java**. Es un servicio de pagos, reembolsos y facturación con arquitectura hexagonal ligera. Leer `SPRINT.md` (plan diario, alcance, reglas) y `ARQUITECTURA.md` (dominio completo, endpoints, migraciones, casos de negocio) antes de construir algo nuevo. La documentación está en español; responder en español.

## Reglas de trabajo (vienen de SPRINT.md, son obligatorias)

- **Claude escribe solo 2 de cada 10 piezas** como plantilla; el usuario escribe el resto y Claude lo revisa. No implementar de más aunque sea rápido. El Día 1, Claude hace `Money` (record) y `RefundStatus` (enum con transiciones).
- Cada clase lleva un comentario de una línea `// CONCEPTO: ...` con el concepto que practica.
- **Sin Lombok, sin Spring Security, sin DevTools.** Constructores, `equals` y `hashCode` se escriben a mano.
- **Spring Boot 4.1.2 / Java 21.** Si algo no compila, revisar el error real en lugar de copiar patrones de Spring Boot 3.x (p. ej. el starter web es `spring-boot-starter-webmvc`, y hay starters `*-test` separados).
- **Alcance de esta semana: solo P1** (ver tabla de prioridades en `ARQUITECTURA.md`). Fuera: facturación/timbrado, outbox, eventos de dominio, ArchUnit, circuit breaker, OpenAPI, Testcontainers.
- **Defectos intencionales, no "arreglarlos":** el seed de Flyway trae datos sucios plantados (duplicados, reembolsos > pago, estados inconsistentes) y la migración de índices omite 2 índices a propósito para el drill de `EXPLAIN`. Las fallas del Día 5 (NPE, pool Hikari agotado, consulta lenta, disco lleno, OOM, config mal puesta) **las planta el usuario, nunca Claude**.
- **Git lo hace el usuario a mano, para practicar.** Claude no ejecuta `git switch -c`, `commit`, `push` ni `tag`, ni crea PRs. Al empezar y al cerrar cada etapa, Claude le recuerda qué rama toca, qué commits sugiere y qué debe llevar el PR, según la sección "Flujo de Git" y las líneas **Git (tú)** de cada día en `SPRINT.md`. Ramas: `main` (liberado), `development` (integración), y `feature/*`, `fix/*`, `docs/*`, `ci/*` creadas desde `development`. Commits con Conventional Commits.

## Comandos

```bash
./mvnw spring-boot:run                                   # levantar la API (puerto 8080)
./mvnw package                                           # JAR en target/
./mvnw test                                              # todas las pruebas
./mvnw test -Dtest=NombreDeClaseTest                     # una clase
./mvnw test -Dtest=NombreDeClaseTest#nombreDelMetodo     # un método
curl -s localhost:8080/actuator/health | jq              # verificación básica
```

Requiere PostgreSQL con base `refunds`. La conexión se toma de las variables de entorno `DB_URL`, `DB_USER` y `DB_PASSWORD` (la contraseña va en la Run Configuration de IntelliJ, nunca en Git). `application.properties` debe usar `ddl-auto=none` (el esquema lo controla Flyway), `open-in-view=false` y `server.shutdown=graceful`. Como el proyecto incluye JPA y Flyway, la app (y `RefundsApplicationTests.contextLoads`) no arranca sin una base accesible. Por eso la conexión se configura desde el Día 1, aunque las tablas y el adaptador JPA lleguen hasta el Día 3. Para correr `./mvnw test` desde la terminal, hay que exportar esas variables antes.

## Arquitectura

Paquete base: `com.erv.refunds`. Hexagonal en tres capas:

- `domain` (`model`, `policy`, `exception`): **no importa Spring ni JPA**. Value objects como records inmutables con validación en el constructor compacto (`Money`, IDs con UUID, `IdempotencyKey`, etc.). Los enums de estado exponen `canTransitionTo(next)` y una transición inválida lanza `InvalidStateTransitionException`. `Payment` es la raíz del agregado y mantiene `refundedTotal`, que nunca excede lo pagado. `PaymentMethod` es abstracta con subclases por tipo; `RefundPolicy`/`FeePolicy` son estrategias polimórficas. Todas las excepciones heredan de `DomainException`.
- `application` (`port.in`, `port.out`, `service`): un servicio por caso de uso, con `@Transactional` **solo aquí**. Los DTOs de casos de uso son records `Command`/`Result`, distintos de los DTOs web. Los puertos de salida (repositorios, `IdempotencyStorePort`, `PaymentGatewayPort` simulado, `ClockPort` para tests deterministas) son interfaces.
- `infrastructure`: `adapter.in.web` (controllers bajo `/api/v1`, DTOs, `GlobalExceptionHandler` que devuelve `ProblemDetail` RFC 7807 con `code` estable como `REFUND_EXCEEDS_PAYMENT`, `CorrelationIdFilter` con `X-Correlation-Id` en MDC), `adapter.in.scheduler` (`RefundRetryJob` con `@Scheduled`), `adapter.out.persistence` (`memory` primero y `jpa` después, con entidades JPA **separadas** de las del dominio y mappers).

El adaptador en memoria (Día 2) se reemplaza por JPA (Día 3) **sin tocar `domain`**; esa sustitución es lo que demuestra la arquitectura.

Puntos de robustez que el diseño debe respetar:
- `POST /api/v1/refunds` exige el header `Idempotency-Key`: la misma clave devuelve la misma respuesta sin crear otro registro.
- `@Version` (locking optimista) en `Payment` y `Refund`: dos reembolsos concurrentes sobre un pago → uno gana y el otro recibe 409.
- Reintentos con backoff exponencial (`RetryPolicy` + `RetryQueue` como `PriorityQueue` por `nextAttemptAt`); un timeout del gateway lleva a `PENDING_RETRY` y después de 5 intentos a `FAILED_PERMANENT`.
- Ventana de reembolso de 30 días; moneda distinta → `CurrencyMismatchException`.

La lista completa de casos de negocio que la API debe resolver (sección 6 de `ARQUITECTURA.md`) sirve como guía de pruebas.

Migraciones Flyway en `src/main/resources/db/migration` (`V1__core.sql`, `V2__invoices.sql`, `V3__audit_idempotency_outbox.sql`, `V4__indexes.sql`, `V5__seed.sql`); de esta semana solo aplican las tablas de P1.
