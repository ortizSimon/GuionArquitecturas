# Dominio 4 — Transferencias y Transacciones

> **Estado:** ✅ Borrador completo
> **Trazabilidad:** Consideraciones 2, 8, 9, 10, 31, 11 → RNF-D4-01…06 → Componentes → Stack → Estrategias de evolución

---

## 4.1 Descripción general

El Dominio 4 es el **motor de movimiento de dinero** de la plataforma. Orquesta cada transferencia desde su inicio hasta su liquidación definitiva, aplicando antifraude en línea antes de ejecutar cualquier movimiento, y manteniendo la integridad de los fondos a través de un patrón Saga con compensación automática.

> **Contrato de existencia:** Toda instrucción de transferencia entra a D4. D4 evalúa el fraude, decide la vía de liquidación (inmediata entre filiales o diferida vía ACH), gestiona los estados y garantiza que no existan fondos perdidos ni transacciones en estado inconsistente.

---

## 4.2 Consideraciones asignadas

| # | Consideración | Prioridad en D4 |
|---|---------------|-----------------|
| 2 | Transferencias con múltiples destinos | Primario |
| 8 | Dualidad de liquidación inmediata vs diferida | Primario |
| 9 | Integración obligatoria con ACH | Secundario (técnico en D6) |
| 10 | Gestión de estados provenientes de ACH (procedente/rechazada) | Primario |
| 31 | Monitoreo de transacciones sospechosas (listas blanca/gris/negra) | Primario |
| 11 | Histórico completo de transacciones | Secundario (persistido en D8) |

---

## 4.3 Actores y responsabilidades

| Actor | Rol en este dominio |
|-------|---------------------|
| Persona natural (cuenta particular) | Inicia transferencias P2P o interbancarias |
| Billetera Digital (D5) | Puede ser origen o destino de transferencia; invoca D4 como si fuera una cuenta |
| D1 — IAM | Autoriza la operación antes de que D4 la procese |
| D2 — Usuarios y Cuentas | D4 consulta saldo disponible y límites operativos de la cuenta origen |
| D6 — Integración | Puente técnico hacia ACH y bancos no filiales; recibe callbacks de ACH y los entrega a D4 |
| D8 — Auditoría | Consume todos los eventos de transferencia para el histórico regulatorio y el motor antifraude |

---

## 4.4 Funciones clave

1. **Validación previa** — verificar cuenta origen (saldo, estado, límites diarios) consultando D2.
2. **Evaluación antifraude en línea** — consultar listas blanca/gris/negra (expuestas por D8) ANTES de aprobar o enviar a ACH. Decisión síncrona; bloqueo inmediato si la cuenta destino está en lista negra.
3. **Enrutamiento por tipo de liquidación:**
   - **Inmediata** (filial–filial): débito en cuenta origen + crédito en cuenta destino en la misma transacción distribuida. Estado final: `SETTLED` en segundos.
   - **Diferida vía ACH** (no filial o internacional): débito provisional en cuenta origen → envío a ACH vía D6 → espera callback → estado `SETTLED` o `FAILED` según respuesta.
4. **Transferencias con múltiples destinos** — un solo `TransferOrder` puede contener N destinos; se crean N transacciones atómicas independientes (cada una con su propia saga) bajo una misma orden padre con trazabilidad compartida.
5. **Gestión de estados** — máquina de estados explícita por transacción (ver §4.5).
6. **Compensación automática** — si un paso de la saga falla, se revierten los pasos previos exitosos (ej.: débito ya aplicado → crédito de compensación automático).
7. **Idempotencia** — todos los endpoints aceptan `idempotency_key`; reintentos del cliente no generan transacciones duplicadas.

---

## 4.5 Máquina de estados de una transferencia

```
                           ┌─────────────────┐
                           │   INITIATED      │
                           └────────┬─────────┘
                                    │ Validar cuenta origen (D2)
                                    ▼
                           ┌─────────────────┐
                           │  FRAUD_CHECK     │ ◄── evaluación listas blanca/gris/negra
                           └────────┬────────┘
                     ┌──────────────┴──────────────┐
                     │ CLEAN                        │ FLAGGED / BLACKLISTED
                     ▼                              ▼
            ┌─────────────────┐          ┌──────────────────┐
            │   APPROVED       │          │    REJECTED       │ (evento: TransferRejected)
            └────────┬─────────┘          └──────────────────┘
                     │
          ┌──────────┴──────────┐
          │ ¿Destino filial?    │
          │                     │
          ▼ SÍ                  ▼ NO
  ┌──────────────┐    ┌─────────────────────┐
  │  SETTLING    │    │  SENT_TO_ACH         │ (evento: TransferSentToACH)
  │ (inmediata)  │    └──────────┬──────────┘
  └──────┬───────┘               │ Callback de ACH (vía D6)
         │                ┌──────┴───────┐
         │                │              │
         ▼                ▼ PROCEDENTE   ▼ RECHAZADA
  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐
  │  SETTLED    │  │   SETTLED    │  │    FAILED    │
  │ (inmediata) │  │ (diferida)   │  │ + compensar  │
  └─────────────┘  └──────────────┘  └──────────────┘

  Estados terminales: SETTLED, REJECTED, FAILED
  Estado especial:    COMPENSATING (rollback en curso)
```

---

## 4.6 Eventos del dominio

### Eventos que produce (publica a Kafka)

| Evento | Disparador | Consumidores principales |
|--------|-----------|--------------------------|
| `TransferInitiated` | Recepción de instrucción válida | D8 |
| `TransferApproved` | Supera validación antifraude | D8 |
| `TransferRejected` | Bloqueada por lista negra o límites | D8, D1 (para bloqueo de sesión si aplica) |
| `TransferSentToACH` | Enviada a ACH vía D6 | D8 |
| `TransferSettled` | Liquidación exitosa (inmediata o ACH) | D8, D2 (actualizar saldo), D5 (si billetera es destino) |
| `TransferFailed` | Error en liquidación o rechazo ACH | D8, D2 (liberar débito provisional) |
| `FraudCheckFlagged` | Destino en lista gris (sospechoso) | D8, D1 (alerta) |

### Eventos que consume

| Evento | Origen | Acción en D4 |
|--------|--------|--------------|
| `ACHResponseReceived` | D6 (callback de ACH) | Transicionar estado: SENT_TO_ACH → SETTLED o FAILED |

---

## 4.7 Comunicación con otros dominios

```
[Cliente] ──HTTP──► D4 (API REST)
                     │
              ┌──────┼─────────────────────┐
              │      │                     │
     síncrono │      │ síncrono            │ síncrono
              ▼      ▼                     ▼
           D1-IAM  D2-Cuentas    D8-Listas antifraude
           (authz) (saldo/límites) (consulta listas B/G/N)

              │ asíncrono (Kafka)
              ▼
           D6-Integración ──► ACH / Bancos externos
              │
              │ callback asíncrono (HTTP webhook o evento Kafka)
              ▼
           D4 actualiza estado de la transacción
              │
              │ asíncrono (Kafka) — eventos de resultado
              ▼
           D8-Auditoría (histórico) / D2 (ajuste saldo) / D5 (billetera)
```

---

## 4.8 RNF del dominio y funciones de ajuste

### RNF-D4-01 — Consistencia transaccional con múltiples destinos

| Campo | Detalle |
|-------|---------|
| **Descripción** | Una instrucción con N destinos debe garantizar que cada sub-transferencia sea atómica e independiente: el fallo de un destino no afecta los demás, y ningún fondo queda en estado intermedio permanente. |
| **Origen** | Consideración 2 (Primario) |
| **Categoría RNF** | Consistencia / Fiabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-01-A | Tasa de compensaciones exitosas | % de sagas en estado COMPENSATING que alcanzan estado final (FAILED con compensación completa) | 100% — 0 fondos perdidos |
| FF-D4-01-B | Tiempo máximo de saga completa (escenario inmediato) | Traza distribuida: desde `TransferInitiated` hasta `TransferSettled` en filiales | P95 < 3 s |
| FF-D4-01-C | 0 transacciones huérfanas | Job de reconciliación periódico: detectar transacciones en estado no terminal por > N minutos | 0 transacciones huérfanas en producción |
| FF-D4-01-D | Idempotencia | Test: enviar la misma instrucción con igual `idempotency_key` dos veces → 1 sola transacción creada | 0 duplicados |

**Tácticas:**
- Patrón **Saga coreografiada**: cada paso (débito, evaluación fraude, crédito, confirmación) emite un evento; el siguiente paso reacciona al evento anterior.
- **Outbox Pattern** para garantizar que el evento se publique junto con la mutación de estado en la misma transacción local.
- **Job de reconciliación** (cron cada 5 min) que detecta transacciones en estados intermedios por más de 10 minutos y las lleva a compensación o alerta al equipo de operaciones.
- Tabla `transfer_saga_state` como log de progreso de la saga, con lock optimista para evitar race conditions.

---

### RNF-D4-02 — Dualidad de liquidación inmediata vs diferida (SLA diferenciado)

| Campo | Detalle |
|-------|---------|
| **Descripción** | El sistema debe distinguir en tiempo de ejecución si una transferencia es entre bancos filiales (liquidación inmediata, reflejo en segundos) o hacia bancos no filiales/internacionales (liquidación diferida vía ACH, puede tomar horas o días hábiles). El usuario debe ser notificado del canal en cada caso. |
| **Origen** | Consideración 8 (Primario) |
| **Categoría RNF** | Rendimiento / Experiencia de usuario / Consistencia eventual |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-02-A | Latencia de liquidación inmediata (filial–filial) | Tiempo desde `TransferApproved` hasta `TransferSettled` | P95 < 500 ms |
| FF-D4-02-B | Latencia de reflejo tras callback ACH | Tiempo desde recepción de `ACHResponseReceived` hasta actualización de estado en D4 y D2 | P95 < 2 s |
| FF-D4-02-C | Clasificación correcta del canal de liquidación | Test: 100 transferencias a cuentas filiales → 100% clasificadas como inmediatas; 100 a no filiales → 100% diferidas | 0 errores de clasificación |
| FF-D4-02-D | Notificación al usuario del canal | Test de integración: verificar que la respuesta HTTP incluye el campo `settlement_type` (`IMMEDIATE` / `DEFERRED`) | 100% de respuestas incluyen el campo |

**Tácticas:**
- Módulo **LiquidationRouter** interno de D4: consulta el registro de bancos filiales de D2 para determinar el canal. Esta consulta se cachea con TTL de 5 min (Redis) para no impactar la latencia.
- Para el canal inmediato: transacción distribuida XA simplificada entre D4 y D2 (o Saga de 2 pasos con compensación en < 1 s).
- Para el canal diferido: débito provisional marcado con flag `HOLD` en D2 → espera callback → liberar o revertir el HOLD según resultado.

---

### RNF-D4-03 — Resiliencia en la integración con ACH

| Campo | Detalle |
|-------|---------|
| **Descripción** | La indisponibilidad temporal del sistema ACH no debe interrumpir las transferencias entre bancos filiales ni bloquear indefinidamente las transferencias diferidas. Las transacciones enviadas a ACH deben poder ser reenviadas de forma segura sin duplicarlas. |
| **Origen** | Consideración 9 (Secundario en D4 / Primario técnico en D6) |
| **Categoría RNF** | Resiliencia / Disponibilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-03-A | Tiempo de detección de ACH no disponible | Circuit Breaker en D6: tiempo entre primera falla y apertura del circuito | < 5 s |
| FF-D4-03-B | Aislamiento del fallo de ACH | Test de caos: simular caída de ACH → transferencias filial–filial no se ven afectadas | 0 impacto en liquidaciones inmediatas |
| FF-D4-03-C | Reenvío seguro a ACH tras recuperación | Test: transacción en SENT_TO_ACH sin callback en N minutos → reenvío idempotente | 0 duplicados en ACH al reenviar |
| FF-D4-03-D | Cobertura de DLQ para eventos fallidos | Verificar que toda falla en la entrega del evento a D6 aterriza en Dead Letter Queue | 100% de eventos fallidos trazables en DLQ |

**Tácticas:**
- **Circuit Breaker** en D6 hacia ACH (Resilience4j); cuando está OPEN, D4 mantiene las transacciones en estado `SENT_TO_ACH` y las reencola para reintento.
- **Idempotency Key** en cada llamada a ACH: D4 envía el `transfer_id` como clave de idempotencia; ACH responde igual si recibe la misma clave dos veces.
- **Dead Letter Queue** en Kafka para mensajes al tópico `ach-outbound` que no puedan ser procesados por D6.
- **Job de retransmisión**: detecta transacciones en `SENT_TO_ACH` sin callback en > 30 min y las reencola (con exponential backoff).

---

### RNF-D4-04 — Gestión completa del ciclo de estados ACH

| Campo | Detalle |
|-------|---------|
| **Descripción** | El sistema debe procesar correctamente todos los estados posibles que ACH puede comunicar (procedente, rechazada, en revisión, timeout) y transicionar la transacción al estado interno correcto en cada caso, liberando o revirtiendo fondos según corresponda. |
| **Origen** | Consideración 10 (Primario) |
| **Categoría RNF** | Correctitud / Fiabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-04-A | Cobertura de estados ACH en la máquina de estados | Test unitario: cada respuesta posible de ACH dispara la transición correcta | Cobertura 100% de casos documentados de ACH |
| FF-D4-04-B | Tiempo de procesamiento del callback de ACH | Traza: desde recepción del webhook de ACH hasta publicación de `TransferSettled` o `TransferFailed` | P95 < 2 s |
| FF-D4-04-C | 0 transacciones en SENT_TO_ACH sin resolución | Job de reconciliación diario: detectar transferencias en SENT_TO_ACH por > 5 días hábiles | Alerta automática + escalación a operaciones |
| FF-D4-04-D | Liberación de fondos ante fallo ACH | Test: simular `ACH_REJECTED` → verificar que el HOLD en D2 se libera en < 5 s | 100% de fallos ACH revierten el HOLD |

**Tácticas:**
- Máquina de estados explícita implementada en código (patrón State o tabla de transiciones en DB) — no lógica condicional distribuida.
- Endpoint dedicado para recibir callbacks de ACH (vía D6), con validación de firma HMAC del payload.
- Cada transición de estado es transaccional: se actualiza el estado y se publica el evento en el mismo Outbox.
- Para respuestas desconocidas de ACH: estado `UNKNOWN_ACH_RESPONSE` + alerta inmediata a operaciones.

---

### RNF-D4-05 — Monitoreo antifraude en línea (listas blanca/gris/negra)

| Campo | Detalle |
|-------|---------|
| **Descripción** | Antes de aprobar cualquier transferencia (filial o no filial), el sistema debe evaluar en tiempo real si las cuentas involucradas están en listas de control de fraude. La evaluación es síncrona y bloquea la transacción si el resultado lo indica. |
| **Origen** | Consideración 31 (Primario) |
| **Categoría RNF** | Seguridad / Cumplimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-05-A | Latencia de evaluación antifraude | Tiempo desde consulta a las listas hasta devolución de decisión | P99 < 200 ms (para no impactar SLA de 2 s) |
| FF-D4-05-B | Tasa de falsos positivos | % de transferencias legítimas bloqueadas incorrectamente | < 0.5% en producción |
| FF-D4-05-C | Cobertura de listas | Test: cuenta en lista negra → 100% rechazada; cuenta en lista blanca → 0% bloqueada | Cobertura 100% de reglas documentadas |
| FF-D4-05-D | Actualización de listas sin redeploy | Test: actualizar lista negra en D8 → nueva transferencia al mismo destino es rechazada en < 60 s | Propagación < 60 s |
| FF-D4-05-E | Auditoría de cada decisión antifraude | Verificar que cada evaluación genera evento `FraudCheckFlagged` o resultado limpio registrado en D8 | 100% de evaluaciones trazadas |

**Tácticas:**
- Las listas se mantienen en D8 y se replican en **caché local de D4** (Redis) con TTL de 60 s para consulta de baja latencia.
- **Reglas de evaluación:**
  - **Lista negra**: bloqueo inmediato → `REJECTED`.
  - **Lista gris**: aprobación con flag → `FraudCheckFlagged` → notificación a D8 y D1 para revisión asíncrona.
  - **Lista blanca**: bypass de revisión adicional → proceso normal.
- La actualización de listas en D8 publica un evento `FraudListUpdated` que invalida el caché de D4 en menos de 60 s.
- Toda decisión antifraude (aprobada, rechazada, sospechosa) queda registrada con `transfer_id`, timestamp y regla aplicada.

---

### RNF-D4-06 — Disponibilidad y rendimiento del motor de transferencias

| Campo | Detalle |
|-------|---------|
| **Descripción** | El motor de transferencias debe operar 24/7 con un tiempo de respuesta menor a 2 segundos para la confirmación al usuario, y debe mantener el SLA incluso durante picos de carga (días de nómina masiva). |
| **Origen** | RNF-01 y RNF-02 globales |
| **Categoría RNF** | Disponibilidad / Rendimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D4-06-A | Tiempo de respuesta end-to-end | Traza desde solicitud HTTP hasta respuesta de confirmación (no espera liquidación diferida) | P95 < 2 s |
| FF-D4-06-B | Disponibilidad del servicio | Health checks en Kubernetes + uptime monitoring | ≥ 99.9% mensual |
| FF-D4-06-C | Degradación controlada bajo carga extrema | Test de carga: simular 5× el volumen pico → verificar que el sistema aplica back-pressure y no falla | Tiempo de respuesta < 5 s (modo degradado); 0 pérdida de datos |
| FF-D4-06-D | Recuperación ante reinicio de pod | K8s liveness/readiness probes → pod nuevo listo para tráfico | < 30 s tiempo de recuperación |

**Tácticas:**
- **Separación de la confirmación al usuario del proceso de liquidación**: la respuesta HTTP retorna cuando la transacción pasa a `APPROVED` (fraude superado, débito registrado), no cuando se liquida en ACH. El usuario recibe el estado final por notificación push/webhook.
- **Escalado horizontal automático (HPA)** del servicio D4 en Kubernetes, configurado para escalar al detectar latencia > 1.5 s o CPU > 70%.
- **Back-pressure**: si la cola interna de sagas supera N elementos, se retorna HTTP 429 con `Retry-After` al cliente en lugar de aceptar sin capacidad de procesar.
- **Replicación activa-activa** del servicio en al menos 2 zonas de disponibilidad.

---

## 4.9 Diagrama interno del dominio

```
                  ┌──────────────────────────────────────────────────────┐
                  │             D4 — Transferencias y Transacciones       │
                  │                                                        │
[Cliente] ─HTTP──►│  ┌───────────────┐    ┌───────────────────────────┐  │
                  │  │  Transfer API  │    │    LiquidationRouter       │  │
                  │  │  (REST)        │───►│  ¿filial? → inmediata     │  │
                  │  └───────┬───────┘    │  ¿no filial? → ACH         │  │
                  │          │             └────────────┬──────────────┘  │
                  │          │ valida                   │                  │
                  │          ▼                          │                  │
                  │  ┌───────────────┐    ┌────────────▼──────────────┐  │
                  │  │  FraudChecker │    │      Saga Orchestrator      │  │
                  │  │  (listas      │    │  - débito (D2)              │  │
                  │  │  B/G/N vía    │    │  - crédito (D2 o ACH/D6)   │  │
                  │  │  Redis caché) │    │  - compensación si falla    │  │
                  │  └───────────────┘    └────────────┬──────────────┘  │
                  │                                     │                  │
                  │                        ┌────────────▼──────────────┐  │
                  │                        │   Transfer State Store      │  │
                  │                        │   (PostgreSQL — ACID)       │  │
                  │                        │   + Outbox Table            │  │
                  │                        └────────────┬──────────────┘  │
                  │                                     │                  │
                  │                        ┌────────────▼──────────────┐  │
                  │                        │   Kafka Publisher (Outbox)  │  │
                  │                        │   TransferInitiated         │──► D8
                  │                        │   TransferSettled           │──► D8, D2, D5
                  │                        │   TransferFailed            │──► D8, D2
                  │                        │   FraudCheckFlagged         │──► D8, D1
                  │                        └───────────────────────────┘  │
                  └──────────────────────────────────────────────────────┘
                                │ síncrono                    ▲
                                ▼                             │ callback (ACHResponseReceived)
                           D6 — Integración ──► ACH externo ─┘
```

---

## 4.10 Stack tecnológico recomendado para D4

| Componente | Tecnología propuesta | Justificación |
|------------|---------------------|---------------|
| API REST | Spring Boot (Java) | Madurez en fintech, integración nativa con Resilience4j y Kafka |
| Motor de saga | Spring State Machine / Axon Framework | Gestión explícita de estados y compensación |
| Base de datos | PostgreSQL | ACID estricto para `transfer_saga_state` y `outbox`; particionamiento por fecha para histórico |
| Caché antifraude | Redis | Latencia sub-milisegundo para consulta de listas, TTL configurable |
| Message Broker | Apache Kafka | Exactly-once delivery, DLQ nativa, replay de eventos |
| Circuit Breaker | Resilience4j | Integración Spring Boot, métricas Micrometer |
| Observabilidad | OpenTelemetry + Jaeger | Trazas distribuidas multi-dominio para debugging de sagas |
| Escalado | Kubernetes HPA | Escalado basado en métricas personalizadas (latencia, tamaño de cola) |

---

## 4.11 Reglas de negocio clave (inmutables)

1. **La evaluación antifraude es SIEMPRE el primer paso tras la validación de cuenta.** No existe vía para saltarla.
2. **El débito en cuenta origen se aplica provisionalmente (`HOLD`) antes de enviar a ACH.** Nunca se envía a ACH dinero no bloqueado.
3. **Una transacción en estado `SETTLED` es inmutable.** Ningún proceso interno puede revertirla; las devoluciones son nuevas transferencias inversas.
4. **El `idempotency_key` tiene vigencia de 24 horas.** Después, la misma clave puede reutilizarse para una nueva operación.
5. **Máximo N destinos por `TransferOrder`:** [POR DEFINIR — ej. 100 destinos simultáneos; valores mayores se tratan como pagos masivos y se delegan a D7].

---

## 4.12 Pendientes / Decisiones abiertas

- [ ] Definir el límite de destinos por `TransferOrder` antes de derivar a D7 (ADR pendiente)
- [ ] Confirmar el esquema de firma HMAC de los callbacks de ACH con el proveedor
- [ ] Definir el timeout de SLA para transacciones diferidas (¿5 días hábiles?) y la política de escalación
- [ ] Confirmar si el motor antifraude es exclusivo de D4 o si D8 expone un microservicio dedicado de evaluación
- [ ] Decidir si la notificación final al usuario (push/email) es responsabilidad de D4 o de un Dominio de Notificaciones separado
