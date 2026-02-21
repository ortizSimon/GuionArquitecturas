# Dominio 3 — Gestión de Empresas y Empleados

> **Estado:** ✅ Borrador completo
> **Trazabilidad:** Consideraciones 15, 19, 20 → RNF-D3-01…05 → Componentes → Stack → Estrategias de evolución

---

## 3.1 Descripción general

El Dominio 3 es el **custodio** de las 15 empresas aliadas y de las referencias mínimas de sus empleados. Su contrato de existencia se resume en una sola regla de negocio:

> **La plataforma NUNCA almacena datos personales completos del empleado. Solo guarda la referencia mínima necesaria para resolver, en tiempo de ejecución, quién es el empleado y a qué cuenta se le paga, consultando el API de la empresa aliada.**

Esto implica un **modelo de datos híbrido**: la plataforma posee el _stub_ del empleado (ID interno, empresa, estado activo/inactivo); los datos completos (nombre, cuenta, CBU/CLABE/número de cuenta) se resuelven en tiempo real vía los servicios de cada empresa.

---

## 3.2 Consideraciones asignadas

| # | Consideración | Prioridad |
|---|---------------|-----------|
| 15 | Interoperabilidad con empresas aliadas (nómina vía servicios) | Primario |
| 19 | Carga masiva de empresas aliadas | Primario |
| 20 | Modelo de datos híbrido para empleados (no guardar datos completos) | Primario |

---

## 3.3 Actores y responsabilidades

| Actor | Rol en este dominio |
|-------|---------------------|
| Administrador del sistema | Ejecuta la carga masiva inicial de empresas; gestiona el registro de APIs de empresa |
| Empresa aliada | Fuente de verdad de datos del empleado; expone API para consulta en tiempo de pago |
| D6 — Integración | Conector técnico que invoca el API de la empresa para resolver datos del empleado |
| D7 — Pagos Masivos | Consumidor de la lista de empleados activos y del endpoint de resolución de destino |
| D1 — IAM | Autoriza a los _payroll managers_ y administradores corporativos |
| D8 — Auditoría | Consume eventos de cambios empresariales y de altas/bajas de referencias de empleados |

---

## 3.4 Funciones clave

1. **Registro y carga masiva de empresas** — importación inicial de las 15 empresas aliadas desde archivo estructurado (CSV / JSON), incluyendo: nombre, NIT, URL base del API de nómina, mecanismo de autenticación del API (API-Key, OAuth2, mTLS).
2. **Gestión del calendario de nómina** — cada empresa puede configurar una o más fechas de pago programadas (días 14–16 y 29–31 por defecto, o cualquier día del mes).
3. **Gestión de referencias de empleados** — alta, baja y actualización del _stub_ mínimo del empleado: `{ employee_ref_id, company_id, status: activo/inactivo, metadata: {} }`.
4. **Resolución de empleado en tiempo de ejecución** — endpoint interno que, dado un `employee_ref_id`, invoca el API de la empresa (vía D6) y retorna los datos de destino de pago sin persistirlos.
5. **Consulta de empleados activos** — D7 consulta la lista de empleados activos por empresa para construir el lote de pago; D3 devuelve solo los `employee_ref_id` activos, no datos PII.

---

## 3.5 Modelo de datos (híbrido)

```
Company {
  company_id        UUID (PK)
  name              String
  tax_id            String (NIT — cifrado en reposo)
  api_base_url      String
  auth_type         Enum { API_KEY, OAUTH2, MTLS }
  auth_config       JSONB (cifrado en reposo)
  payroll_schedules PayrollSchedule[]
  status            Enum { ACTIVE, SUSPENDED }
  created_at        Timestamp
  updated_at        Timestamp
}

PayrollSchedule {
  schedule_id  UUID (PK)
  company_id   UUID (FK)
  day_of_month Int[]      -- ej: [14, 15, 16, 29, 30, 31]
  type         Enum { AUTOMATIC, MANUAL }
}

EmployeeRef {
  employee_ref_id  UUID (PK)
  company_id       UUID (FK)
  external_emp_id  String   -- ID en el sistema de la empresa (cifrado)
  status           Enum { ACTIVE, INACTIVE }
  created_at       Timestamp
  updated_at       Timestamp
}
-- NOTA: No se almacenan: nombre, CÉDULA, número de cuenta, salario, ni ningún dato bancario del empleado.
```

---

## 3.6 Eventos del dominio

### Eventos que produce (publica a Kafka)

| Evento | Disparador | Consumidores principales |
|--------|-----------|--------------------------|
| `CompanyImported` | Finaliza carga masiva de empresas | D8 (auditoría) |
| `CompanyPayrollScheduleUpdated` | Admin modifica calendario de nómina | D7 (pagos masivos), D8 |
| `EmployeeRefCreated` | Alta de referencia de empleado | D8 |
| `EmployeeRefUpdated` | Cambio de estado activo/inactivo | D7, D8 |

### Eventos que consume

_Este dominio no es consumidor reactivo de eventos externos. Sus operaciones son disparadas por:_
- Peticiones HTTP síncronas desde D7 (consulta de activos)
- Llamadas síncronas desde D6 (resolución de datos del empleado)
- Procesos batch de carga masiva programados

---

## 3.7 Comunicación con otros dominios

```
D3 ──síncrono──► D6: "resuelve datos del empleado X en la empresa Y"
D3 ◄──síncrono── D7: "dame lista de empleados activos de la empresa Z"
D3 ◄──síncrono── D1: "¿tiene el usuario rol payroll_manager para empresa W?"
D3 ──asíncrono─► D8: eventos CompanyImported, EmployeeRefCreated/Updated
```

---

## 3.8 RNF del dominio y funciones de ajuste

### RNF-D3-01 — Interoperabilidad con APIs heterogéneas de empresas aliadas

| Campo | Detalle |
|-------|---------|
| **Descripción** | El dominio debe comunicarse con las APIs de las 15 empresas aliadas, cada una con protocolo, autenticación y esquema de datos potencialmente distinto, sin acoplar la lógica de negocio a ningún proveedor. |
| **Origen** | Consideración 15 (Primario) |
| **Categoría RNF** | Interoperabilidad / Extensibilidad |

**Funciones de ajuste (fitness functions):**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D3-01-A | Tiempo de resolución de datos de empleado | Invocación al API de empresa (vía D6) + retorno a D7 | P95 < 1.5 s en condiciones normales |
| FF-D3-01-B | Tasa de éxito de resolución | % de llamadas exitosas al API externo / total de intentos | ≥ 99% en ventana de pago activa |
| FF-D3-01-C | Adición de nueva empresa sin redespliegue del núcleo | Test de integración: registrar nueva empresa + ejecutar pago de prueba | Debe completarse en < 1 día hábil sin tocar código del núcleo de D3 |
| FF-D3-01-D | Prueba de contrato de adaptador | Consumer-driven contract test entre D3/D6 y el stub de API de empresa | 0 regresiones al actualizar un adaptador |

**Tácticas:**
- Patrón **Adapter** por empresa aliada implementado en D6; D3 solo conoce la interfaz `EmployeeDataPort`.
- Registro dinámico de adaptadores: añadir empresa = desplegar nuevo adapter en D6 sin cambiar D3.
- **Circuit Breaker** (Resilience4j) por empresa: si el API de la empresa falla, se interrumpe el pago del lote de esa empresa sin afectar los demás.
- Timeouts configurables por empresa (default: 3 s) con política de reintentos exponencial (máx. 3 intentos).

---

### RNF-D3-02 — Capacidad de carga masiva de empresas

| Campo | Detalle |
|-------|---------|
| **Descripción** | El proceso de onboarding inicial (y actualizaciones posteriores) debe importar datos de empresas y referencias de empleados de forma masiva, de manera idempotente y con trazabilidad del resultado. |
| **Origen** | Consideración 19 (Primario) |
| **Categoría RNF** | Rendimiento / Fiabilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D3-02-A | Tiempo de importación de empresas | Prueba de carga: importar archivo con 15 empresas + N mil referencias de empleados | Completar en < 10 min para 35 000 registros |
| FF-D3-02-B | Idempotencia de la carga | Ejecutar el mismo archivo dos veces consecutivas | Resultado final idéntico; 0 duplicados |
| FF-D3-02-C | Tasa de errores en carga masiva | Validación de esquema + informe de filas rechazadas | < 0.1% de registros rechazados en una carga limpia |
| FF-D3-02-D | Disponibilidad del sistema durante la carga | Prueba de carga + medición de latencia de endpoints de consulta concurrentes | SLA general del sistema no se degrada durante el batch |

**Tácticas:**
- Job batch asíncrono con cola interna: el proceso de carga no bloquea el hilo principal del servicio.
- Procesamiento en chunks (lotes de 500 registros) con commit transaccional por chunk.
- Informe de resultado al finalizar: registros OK, rechazados, erróneos → evento `CompanyImported` con resumen.
- Carga idempotente: si el `external_emp_id + company_id` ya existe, se actualiza en lugar de insertar.

---

### RNF-D3-03 — Minimización de datos (privacidad y seguridad del empleado)

| Campo | Detalle |
|-------|---------|
| **Descripción** | La plataforma solo debe persistir la referencia mínima del empleado. Ningún dato personal completo (nombre, documento de identidad, cuenta bancaria, salario) debe almacenarse en la base de datos de D3. |
| **Origen** | Consideración 20 (Primario) / Restricción de seguridad explícita del enunciado |
| **Categoría RNF** | Seguridad / Privacidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D3-03-A | Auditoría de esquema de datos | Test automatizado que inspecciona las tablas de D3 en CI | 0 columnas que contengan nombre, cédula, cuenta bancaria o salario |
| FF-D3-03-B | Revisión de logs de respuesta | Análisis estático/dinámico de tráfico: las respuestas de D3 no deben incluir PII | 0 ocurrencias de campos de PII en payloads de respuesta de D3 |
| FF-D3-03-C | Cifrado en reposo de campos sensibles | Verificación de configuración de base de datos (cifrado a nivel columna/disco) | `tax_id` y `auth_config` cifrados con AES-256 |
| FF-D3-03-D | Retención de datos de empleados inactivos | Test de política de retención: EmployeeRef en estado INACTIVE por > N días | Eliminación o anonimización automática según política de retención definida |

**Tácticas:**
- Esquema de base de datos revisado en CI con un test de arquitectura (ArchUnit / custom script) que valida que no existen columnas de PII.
- Los datos resueltos del empleado (nombre, cuenta) se manejan **solo en memoria** durante el procesamiento del pago y nunca se persisten.
- Cifrado en reposo para campos sensibles del registro de empresa (`tax_id`, `auth_config`).
- Política de retención de referencias inactivas documentada en ADR.

---

### RNF-D3-04 — Resiliencia ante indisponibilidad de APIs de empresas aliadas

| Campo | Detalle |
|-------|---------|
| **Descripción** | Si el API de una empresa aliada no responde, el sistema debe aislar el fallo (solo esa empresa ve afectado su lote de pago) sin degradar el servicio global ni bloquear indefinidamente. |
| **Origen** | Derivado de consideraciones 15 y del RNF-01 (Disponibilidad) del sistema global |
| **Categoría RNF** | Resiliencia / Disponibilidad |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D3-04-A | Tiempo de detección de API caída | Circuit breaker: tiempo entre primera falla y apertura del circuito | < 5 s |
| FF-D3-04-B | Aislamiento de fallos entre empresas | Test de caos: simular caída del API de empresa A → verificar que empresa B no se ve afectada | 0 impacto en empresa B |
| FF-D3-04-C | Alerta ante circuit breaker abierto | Métrica Prometheus + alerta Grafana cuando circuito de empresa X está OPEN | Alerta en < 30 s |
| FF-D3-04-D | Recuperación automática (half-open) | Circuit breaker pasa a HALF-OPEN tras timeout configurado | Recuperación automática sin intervención manual |

**Tácticas:**
- Circuit Breaker por empresa en D6 (Resilience4j), configurado con umbrales de error rate (> 50% en 10 llamadas → OPEN).
- Fallback: si el circuito está abierto, el lote de esa empresa queda en estado `PAUSED_API_ERROR`; se reintenta tras la ventana de recuperación.
- Observabilidad: métricas del estado del circuit breaker expuestas a Prometheus, visibles en dashboard de D8.

---

### RNF-D3-05 — Trazabilidad de cambios empresariales y de empleados

| Campo | Detalle |
|-------|---------|
| **Descripción** | Todo cambio en el registro de una empresa (alta, modificación de calendario, suspensión) y en referencias de empleados (alta, baja, cambio de estado) debe quedar registrado en el log de auditoría de D8. |
| **Origen** | RNF-06 (Trazabilidad) del sistema global / Restricción de cumplimiento |
| **Categoría RNF** | Trazabilidad / Cumplimiento |

**Funciones de ajuste:**

| # | Función de ajuste | Mecanismo | Métrica objetivo |
|---|-------------------|-----------|-----------------|
| FF-D3-05-A | Cobertura de eventos publicados | Test de integración: ejecutar cada operación CRUD sobre Company y EmployeeRef → verificar evento en Kafka | 100% de operaciones generan evento correspondiente |
| FF-D3-05-B | Latencia del evento al log | Tiempo entre publicación en Kafka y persistencia en D8 | P95 < 500 ms |
| FF-D3-05-C | Inmutabilidad del log | Test de intento de modificación de registro histórico en D8 | Operación rechazada (append-only) |

**Tácticas:**
- Publicación de eventos en el mismo contexto transaccional de la operación (Outbox Pattern): el evento se persiste en tabla local antes de enviarse a Kafka, garantizando entrega al menos una vez.
- D8 consume los eventos y los escribe en el append-only store.

---

## 3.9 Diagrama interno del dominio

```
                    ┌─────────────────────────────────────┐
                    │       D3 — Empresas & Empleados      │
                    │                                      │
  [Admin] ─HTTP──►  │  ┌──────────────┐                   │
                    │  │ Company API  │ ◄── carga masiva   │
  [D7-Pagos] ─HTTP──► │  ┌────────────────────┐            │
                    │  │ Employee Ref API│                  │
                    │  └────────────────────┘              │
                    │           │                           │
                    │  ┌────────▼──────────┐               │
                    │  │   DB (PostgreSQL)  │               │
                    │  │  Company           │               │
                    │  │  EmployeeRef       │               │
                    │  │  PayrollSchedule   │               │
                    │  └───────────────────┘               │
                    │           │                           │
                    │  ┌────────▼──────────┐               │
                    │  │  Outbox → Kafka    │               │
                    │  │  CompanyImported   │──► D8         │
                    │  │  EmployeeRefCreated│──► D8         │
                    │  └───────────────────┘               │
                    └─────────────────────────────────────┘
                                  │
                    síncrono vía D6 ─► APIs externas empresas
```

---

## 3.10 Stack tecnológico recomendado para D3

> Alineado con el stack global del proyecto (Sección 4). Proveedor de nube: **AWS** (`sa-east-1` como región primaria).

| Componente | Tecnología propuesta | Justificación |
|------------|---------------------|---------------|
| API REST del dominio | Java 21 + Spring Boot 3 en **Amazon EKS** (EC2 node groups) | Consistente con D4 y D7 (dominios financieros); madurez para ACID, integración nativa con Resilience4j |
| Base de datos | **Amazon Aurora PostgreSQL** (Multi-AZ, Serverless v2) + **AWS KMS** (cifrado en reposo) | Aurora provee hasta 5× throughput de PostgreSQL estándar; KMS gestiona cifrado de columnas sensibles (`tax_id`, `auth_config`) sin pgcrypto propio |
| Message Broker | **Amazon MSK** (Managed Streaming for Apache Kafka) — Outbox Pattern | Elimina gestión del plano de control de Kafka; integración nativa con IAM y CloudWatch; entrega garantizada de eventos a D8 |
| Circuit Breaker | Resilience4j (librería) | Integración nativa con Spring Boot; métricas expuestas vía Micrometer → CloudWatch |
| Batch / carga masiva | Spring Batch en EKS | Procesamiento por chunks, reintentos configurables, informe de resultado; mismo runtime que el servicio principal |
| Observabilidad | Micrometer + **Amazon CloudWatch** + **Amazon Managed Grafana** + **AWS X-Ray** (via OpenTelemetry SDK) | Stack de observabilidad unificado con el resto del proyecto; CloudWatch recolecta métricas nativas de EKS y Aurora; X-Ray provee trazas distribuidas |

---

## 3.11 Pendientes / Decisiones abiertas

- [ ] Confirmar el formato del archivo de carga masiva de empresas (CSV, JSON, Excel)
- [ ] Definir la política de retención de `EmployeeRef` en estado INACTIVE (ADR pendiente)
- [ ] Confirmar los campos exactos que el API de cada empresa expone (esquema mínimo esperado)
- [ ] Decidir si el endpoint de resolución de empleado es expuesto directamente por D3 o delegado completamente a D6
