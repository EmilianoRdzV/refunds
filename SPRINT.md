# Sprint 7 días: L2 Java Support Specialist (NTT DATA, Req 384813)

**Inicio:** domingo 25 oct 2026 · **Simulacro final:** sábado 31 oct 2026
**Versión:** 2 (recortada a lo que pide la vacante: soporte L2, no arquitectura)

## Qué pide la vacante (y dónde se practica)

| Requisito de la publicación | Peso de estudio | Dónde se practica |
|---|---|---|
| Java práctico y depuración | **25%** | Dominio P1, drills de conceptos, código con bugs, stack traces |
| Unix/Linux avanzado | **20%** | VM por SSH, `systemd`, logs, disco, procesos |
| SQL avanzado | **20%** | Postgres con seed sucio, ventanas, `EXPLAIN`, índices |
| Incidentes, RCA y acciones correctivas | **20%** | Fallas plantadas (Día 5), post-mortems, STAR |
| Scripting (Shell/Python/Bash) | **10%** | Scripts sobre los logs de tu propia API |
| CI/CD, inglés y colaboración global | **5%** + inglés transversal | Pipeline simple, respuestas STAR en inglés desde el Día 3 |

La publicación **no** menciona Kubernetes, AWS, Docker ni ITIL. Son tu fortaleza, pero no lo que evalúan. Por eso k3s es opcional y ITIL se reduce a lo esencial.

## Reparto de roles

- **Terminal de Claude (IntelliJ):** construye contigo. Escribe 2 piezas por tanda como plantilla; tú haces el resto y ella lo revisa.
- **Este chat:** drills, calificación 1-5, respuesta modelo, lista de puntos débiles, simulacros en inglés.

## Horario base

Lunes a sábado: 6:00-7:00 AWS, luego **7:00-9:00 este sprint** en tres bloques (A, construcción, B). Entrenamiento 17:00-18:30 intacto. Hoy domingo hay más horas.

---

## Requisitos

### En tu máquina
| Herramienta | Versión | Para qué |
|---|---|---|
| JDK | 21 LTS (Temurin) | Compilar y correr |
| IntelliJ IDEA | el que ya tienes | Desarrollo y depuración |
| Maven | `mvnw` del proyecto | Build |
| PostgreSQL + pgAdmin 4 | los que ya tienes (verificar con `psql --version`) | Base y consultas |
| Git + GitHub | cualquiera | Repo y Actions |
| Cliente SSH | OpenSSH de Windows | Entrar a la VM |
| curl + jq | cualquiera | Probar endpoints |
| Python 3 | 3.10+ | Scripts del Día 2 |
| Hipervisor | VirtualBox, Hyper-V o WSL2 con systemd | La VM |

### En la VM (Ubuntu Server 24.04 LTS)
- 2 vCPU, 4 GB RAM, 20 GB de disco (ajustar a la RAM de tu PC).
- `openjdk-21-jre-headless`, `openssh-server`, `curl`, `jq`, `htop`, `logrotate`, `cron`.
- Docker Engine (Día 4). **k3s + kubectl: solo opcional** (Día 5).
- Red: puente o NAT con reenvío del 22 y del 8080.

### Spring Initializr (valores exactos)
| Campo | Valor |
|---|---|
| Project / Language | Maven / Java |
| Spring Boot | **4.1.2** (sin SNAPSHOT ni RC) |
| Group / Artifact | `com.erv` / `refunds` |
| Package name | `com.erv.refunds` |
| Packaging / Configuration | Jar / Properties |
| Java | **21** |
| Dependencias | Spring Web, Validation, Spring Data JPA, PostgreSQL Driver, Spring Boot Actuator, Flyway Migration |

Sin Lombok (practicas constructores, `equals` y `hashCode` a mano), sin Security, sin DevTools. Si un import o dependencia no compila en la 4.x, se revisa el error en lugar de copiar de tutoriales de la 3.x.

`application.properties` base: usa `DB_URL`, `DB_USER` y `DB_PASSWORD` como variables de entorno (la contraseña se define en la Run Configuration de IntelliJ, nunca en Git), `ddl-auto=none`, `open-in-view=false` y `server.shutdown=graceful`.

---

## Alcance de construcción: solo P1

**Dentro:** `Money`, `Payment`, `Refund`, enums con transiciones de estado, jerarquía de métodos de pago, políticas de reembolso, excepciones de dominio, `RetryQueue`, `IdempotencyStore`, `PaymentController` y `RefundController`, `GlobalExceptionHandler` (ProblemDetail), `@Version`, Postgres + Flyway, job `@Scheduled` de reproceso.

**Fuera esta semana:** facturación y timbrado, outbox, eventos de dominio, ArchUnit, circuit breaker, OpenAPI, Testcontainers. (Siguen descritos en `ARQUITECTURA.md` como P2 y P3, para después de la entrevista.)

Hexagonal en versión ligera: `domain` no importa Spring ni JPA, con puertos como interfaces y adaptadores intercambiables (memoria → JPA). Convención de comentario en cada clase: `// CONCEPTO: ...` (una línea). Reparto: Claude hace 2 de cada 10, tú el resto.

```
com.erv.refunds
├── domain          (model, policy, exception)
├── application     (port.in, port.out, service)
└── infrastructure  (adapter.in.web, adapter.in.scheduler, adapter.out.persistence: memory, jpa)
```

## Plan de Postgres

| Fase | Dónde vive | Cuándo |
|---|---|---|
| 1 | Windows (tu instalación) | Días 1-3 |
| 2 | Contenedor Docker en la VM (`pg_dump -Fc` → `scp` → `pg_restore`, y **validar conteos y sumas**) | Día 4 |
| 3 | Pod en k3s con PVC | Día 5, solo si hay tiempo |

La API usa variables de entorno desde el Día 1, así que cambiar de fase no toca el código. El esquema vive en migraciones Flyway versionadas.

---

## Flujo de Git (lo haces tú desde la terminal, para practicar)

Claude **no** crea ramas, commits ni PRs: solo te recuerda en cada etapa qué toca y revisa lo que subes.

- `main`: solo lo que ya se "liberó" (fin de etapa). `development`: integración diaria.
- Una rama por etapa, creada desde `development`: `git switch development && git pull && git switch -c feature/<nombre>`.
- Commits pequeños con Conventional Commits: `feat(domain): ...`, `fix(web): ...`, `test(domain): ...`, `chore(config): ...`, `docs: ...`.
- PR de la rama hacia `development` con descripción de tres partes: **qué cambia**, **por qué**, **cómo probarlo** (`curl`, prueba, consulta). Merge y borrar la rama.
- Al cerrar una etapa entregable, PR de `development` → `main` y un tag (`git tag -a v0.x.0 -m "..."`, `git push --tags`).
- La contraseña de la base **nunca** entra a un commit; antes de cada `git add`, revisa con `git status` y `git diff --staged`.

---

## Día 1 · Domingo 25 oct: Cimientos + Linux

**Objetivo:** proyecto arrancando con `/actuator/health` en `UP`, dominio base modelado y diagnóstico de Linux hecho.

| Bloque | Qué |
|---|---|
| 11:30-12:15 | Verificar JDK 21, Postgres, pgAdmin; `CREATE DATABASE refunds`. Generar proyecto en Initializr, repo en GitHub con `main` y `development` (hecho). Configurar `application.properties` con `DB_URL`/`DB_USER`/`DB_PASSWORD` para que la app arranque contra la base vacía. |
| 12:15-13:30 | **Construcción:** Claude hace `Money` (record) y `RefundStatus` (enum con transiciones). Tú: `PaymentId`/`RefundId`, `PaymentStatus`, `Payment`, `Refund`, `PaymentMethod` → `CardPayment`/`ReferencePayment`, `RefundPolicy` → `FullRefund`/`PartialRefund`, excepciones de dominio. |
| 15:00-16:30 | **Drill:** diagnóstico Linux de 5 preguntas + escenarios (servidor lento, disco lleno, proceso colgado). |
| 17:00-18:30 | Entrenamiento. |

**Terminado cuando:** compila, `/actuator/health` devuelve `UP`, las piezas del dominio existen con su `CONCEPTO` y puedes explicar por qué `domain` no importa Spring.

**Si se alarga:** `RetryQueue` e `IdempotencyStore` pasan al Día 2.

**Git (tú):**
- `chore/config-db`: `application.properties` con variables de entorno → PR a `development`.
- `feature/dia1-dominio-base`: un commit por pieza (`feat(domain): Money como record inmutable`, etc.) → PR a `development` con la lista de conceptos practicados.

---

## Día 2 · Lunes 26 oct: Puertos, REST y scripting

| Bloque | Qué |
|---|---|
| 7:00-7:45 | **A · Drill Bash y Python:** scripts que parseen logs, cuenten errores por tipo, reprocesen fallidos y llamen a una API con manejo de errores (6 ejercicios con revisión). |
| 7:45-8:30 | **Construcción:** `RetryQueue`, `IdempotencyStore`, puertos `in` y `out`, servicios, adaptador en memoria, `PaymentController` y `RefundController`, `GlobalExceptionHandler`, `Idempotency-Key`. |
| 8:30-9:00 | **B · Drill Java conceptual:** constructores, `equals`/`hashCode`, polimorfismo, herencia vs composición, colecciones, jerarquía de excepciones (15 preguntas, con pista antes de la respuesta). |

**Terminado cuando:** los endpoints responden con `curl` con adaptador en memoria, hay 3+ pruebas unitarias del dominio y la API escribe logs que luego parsearás con tus scripts.

**Git (tú):**
- `feature/dia2-puertos-rest`: puertos, servicios, adaptador en memoria, controllers y `GlobalExceptionHandler`. En el PR, pega los `curl` de prueba y sus respuestas.
- `feature/dia2-scripts`: los scripts Bash/Python en `scripts/` → PR aparte.

---

## Día 3 · Martes 27 oct: Postgres, JPA y SQL avanzado

| Bloque | Qué |
|---|---|
| 7:00-8:00 | **A · Drill SQL avanzado:** joins múltiples, GROUP BY/HAVING, subconsultas, CTEs, ROW_NUMBER/RANK/LAG, duplicados e inconsistencias, `EXPLAIN`, índices (20 ejercicios progresivos sobre tus tablas). |
| 8:00-8:40 | **Construcción:** migraciones Flyway (`payments`, `refunds`, `audit_log`, `idempotency_keys`), adaptador JPA con mappers, `@Version`, seed de ~10k pagos con **datos sucios plantados**, dos índices omitidos a propósito. Cambio de memoria a JPA sin tocar `domain`. |
| 8:40-9:00 | **B · Inglés STAR #1:** tu incidente real más complejo, contado de principio a fin en inglés, más la respuesta corta y honesta sobre los "3+ años en soporte". |

**Terminado cuando:** la API funciona contra Postgres, el seed está cargado, y puedes mostrar una consulta con ventana y un `EXPLAIN` antes y después de un índice.

**Git (tú):**
- `feature/dia3-postgres-jpa`: migraciones Flyway, adaptador JPA, mappers y `@Version`. En el PR, explica que el cambio de memoria a JPA no tocó `domain` (enséñalo con `git diff --stat`).
- `feature/dia3-seed`: seed con datos sucios. En el PR, documenta qué datos sucios plantaste y que los 2 índices faltan **a propósito**.

---

## Día 4 · Miércoles 28 oct: Java depurando + despliegue

| Bloque | Qué |
|---|---|
| 7:00-8:00 | **A · Drill Java depurando (el día más pesado de Java):** código con bugs, stack traces (NPE, ClassNotFound, OOM, timeout de conexión), thread dump y heap dump a nivel concepto, Spring Boot básico (controller, service, repository). |
| 8:00-8:45 | **Construcción:** Logback a archivo, perfiles, Actuator (`threaddump`, `heapdump`, `loggers`), `mvn package`, `scp` del JAR, servicio `systemd`, `logrotate`, `cron`. Luego `Dockerfile` y migración de Postgres al contenedor. |
| 8:45-9:00 | **B · Linux repaso:** puntos débiles del diagnóstico del Día 1. |

**Terminado cuando:** entras por SSH, reinicias el servicio, lees logs con `journalctl` y `tail -f`, y la API corre también en contenedor con Postgres migrado y validado.

**Git (tú):**
- `feature/dia4-despliegue`: Logback, perfiles, Actuator, `Dockerfile`, unidad `systemd` y config de `logrotate` (en `deploy/`).
- Fin de la primera etapa entregable: PR `development` → `main` y tag `v0.1.0`.

---

## Día 5 · Jueves 29 oct: Fallas plantadas y RCA

| Bloque | Qué |
|---|---|
| 7:00-7:45 | **Construcción (tú):** plantas 4-6 fallas en la API: NPE en campo opcional, pool Hikari agotado, consulta lenta sin índice, disco lleno por logs, OOM, variable o Secret mal puesto. |
| 7:45-8:45 | **A · Simulacro de incidente L2:** diagnóstico con logs y comandos, hipótesis, mitigación, escalamiento, acción correctiva definitiva y post-mortem. Método STAR con tus casos reales (CrashLoopBackOff, CronJobs fallidos, rollbacks). |
| 8:45-9:00 | **B · Inglés:** el mejor incidente del día, resumido en 2 minutos en inglés. |

**Opcional (solo si vas adelantado):** k3s con `Deployment`, `ConfigMap`, `Secret`, `CronJob` y las mismas fallas dentro del clúster.

**Terminado cuando:** diagnosticaste al menos 3 fallas con evidencia y tienes 2 post-mortems escritos (causa raíz, mitigación, acción correctiva definitiva).

**Git (tú):**
- `incident/dia5-fallas-plantadas`: las fallas van en una rama que **nunca se mergea** a `development` (empújala para tener el historial, pero sin PR).
- Por cada falla diagnosticada: `fix/<falla>` desde `development` con la acción correctiva definitiva → PR que enlace al post-mortem.
- `docs/dia5-postmortems`: post-mortems en `docs/postmortems/` → PR a `development`.

---

## Día 6 · Viernes 30 oct: CI/CD, colaboración y preguntas difíciles

| Bloque | Qué |
|---|---|
| 7:00-7:45 | **Construcción:** workflow simple de GitHub Actions (build + tests) y script de despliegue por SSH con rollback al JAR anterior. |
| 7:45-8:15 | **A · ITIL esencial:** incidente vs problema vs cambio, SLA, MTTR, y cómo dar feedback técnico a Core Development sin culpar (en inglés). |
| 8:15-9:00 | **B · Preguntas difíciles en inglés:** Java sin producción, rol real en soporte, "advanced SQL" con consulta, por qué pasar de desarrollo a L2 sin sonar a paso temporal, "take ownership and lead initiatives in a global team". |

**Terminado cuando:** un push dispara el build y puedes explicar el rollback paso a paso.

**Git (tú):**
- `ci/dia6-github-actions`: workflow en `.github/workflows/` y el script de despliegue con rollback → PR a `development` (el propio PR debe disparar el build).
- Practica también el rollback en Git: `git revert` de un commit ya mergeado, en una rama `fix/` con PR.
- Cierre de etapa: PR `development` → `main` y tag `v1.0.0`.

---

## Día 7 · Sábado 31 oct: Simulacro final

| Bloque | Qué |
|---|---|
| Mañana | Repaso solo de la lista de puntos débiles. Sin construir. |
| Principal | **Simulacro completo de 45 min en inglés:** técnicas, conductuales y salario/disponibilidad. |
| Cierre | Reporte: calificación por área, 5 puntos débiles y qué repasar antes de la llamada. |

**Git (tú):** `docs/dia7-cierre` con la lista final de puntos débiles en este archivo → PR a `development` y luego a `main`. Revisa el historial con `git log --oneline --graph --all`: es lo que vas a mostrar si te preguntan por tu flujo de trabajo.

---

## Preguntas para el reclutador (la publicación no lo dice)

Salario, modalidad (presencial, híbrida o remota en Guadalajara), horarios, turnos y guardias.

## Rutina de cierre de cada sesión (5 min)

1. Qué dominé. 2. Qué falló. 3. Qué repetimos mañana. 4. Actualizar los **puntos débiles**. 5. Commit, push y PR de lo del día (nada queda sin subir).

## Puntos débiles (se actualiza cada día)

- (vacío)

## Reglas del sprint

- Una pregunta o ejercicio a la vez, con pista antes de la solución.
- Si un día se atrasa, se recorta construcción, nunca el drill.
- Cada paso se explica en voz alta; si no puedes justificarlo, no cuenta como aprendido.
- Los bugs del Día 5 los plantas tú, no la otra terminal.
