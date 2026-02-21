# Sección 1 — Descripción de la arquitectura seleccionada

> **Estado:** ✅ Borrador completo — pendiente de citas  
> **Trazabilidad:** D1–D8 → Sección 2 (RNF) → Sección 3 (Diagrama) → Sección 4 (Stack) → Sección 5 (Evolución)

---

## 1.1 Justificación de la arquitectura

Para el sistema de la Empresa X se seleccionó una **arquitectura de microservicios orientada a eventos** (*Event-Driven Microservices*). Esta decisión responde directamente a los cuatro factores dominantes del problema:

| Factor | Evidencia en el contexto | Respuesta arquitectónica |
|--------|--------------------------|--------------------------|
| **Escala extrema** | ~25 millones de usuarios activos | Escalado horizontal independiente por servicio |
| **Picos predecibles de carga** | 20K–30K transacciones los días 14–16 y 29–31 de cada mes | Servicios de pagos masivos escalan sin afectar los demás |
| **Extensibilidad a demanda** | Integrar terceros sin afectar el estado del sistema (requisito explícito) | Patrón Adapter por canal externo; nuevos terceros se incorporan sin redespliegue |
| **Disponibilidad 24/7 y < 2 s** | Operación continua + SLA de tiempo de respuesta | Desacoplamiento asíncrono; fallos en ACH no bloquean transferencias entre filiales |

La arquitectura se apoya en tres patrones complementarios:

1. **API Gateway** — punto de entrada único que centraliza autenticación, enrutamiento, rate-limiting y TLS hacia todos los microservicios.
2. **Message Broker / Event Streaming (Kafka)** — desacopla productores y consumidores; garantiza entrega, orden y replay de eventos críticos (transferencias, pagos masivos, sincronizaciones).
3. **Bounded Contexts (DDD)** — cada dominio de negocio se implementa como un microservicio con responsabilidad única y base de datos propia (*Database per Service*), lo que permite evolucionar, versionar o reemplazar un dominio sin impacto en los demás.

**Arquitectura descartada — Monolito modular:** No soporta el requisito de extensibilidad a demanda ni el escalado diferenciado de los módulos de pagos masivos frente a consultas de saldo. Ante picos de nómina, se escalaría innecesariamente todo el sistema.

> **Citas:** [POR DEFINIR — agregar referencias del material del curso: título, autor, unidad]

---

## 1.2 Dominios identificados (Bounded Contexts)

Los dominios se definen usando el concepto de **Bounded Context de Domain-Driven Design (DDD)**: cada dominio agrupa una responsabilidad de negocio bien delimitada, con su propio modelo de datos, su propia base de datos y sus propias reglas. Ningún otro dominio accede directamente a los datos de otro; la comunicación ocurre exclusivamente a través de eventos (Kafka) o contratos de API versionados.

### Criterios de separación de dominios

Un grupo de funcionalidades se separa en un dominio independiente cuando **al menos dos** de las siguientes condiciones se cumplen:

| Criterio | Descripción |
|----------|-------------|
| **Actores distintos** | Los usuarios o sistemas que interactúan con él son diferentes a los de otros dominios (ej. empresas ≠ personas naturales) |
| **Datos propios** | Posee datos que ningún otro dominio debería leer o escribir directamente |
| **Evolución independiente** | Puede modificarse, versionarse o reemplazarse sin afectar al resto del sistema |
| **Ciclo de vida distinto** | Tiene patrones de carga o disponibilidad diferentes (ej. nómina tiene picos predecibles; IAM opera de forma continua y uniforme) |

### Trazabilidad: enunciado → dominio

| Dominio | Origen directo en el enunciado |
|---------|-------------------------------|
| **D1 — IAM** | Sección 5 completa: autenticación, autorización, OWASP, listas negras/grises/blancas |
| **D2 — Usuarios y Cuentas** | Punto 1a/1b: carga masiva de bancos, sincronización diaria, consulta de cuentas |
| **D3 — Empresas y Empleados** | Punto 3 + 4a/4b: 15 empresas aliadas, referencia mínima de empleados |
| **D4 — Transferencias** | Punto 2a: transferencias entre filiales, ACH, internacionales |
| **D5 — Billetera Digital** | Punto 2b: billetera propia de Empresa X, pagos a terceros |
| **D6 — Integraciones** | Punto 2b-iii + requisito explícito: PSE, DRUO, Apple Pay, terceros a demanda sin afectar el sistema |
| **D7 — Pagos Masivos** | Punto 3 + punto 4: nómina empresarial, 20K–30K tx en fechas específicas |
| **D8 — Auditoría** | Punto 4 general + puntos regulatorios 2/3: histórico, extracto trimestral, reporte Superfinanciera |

El sistema se divide en **8 dominios** derivados directamente del enunciado. Cada dominio es implementado como uno o más microservicios cohesionados.

---

### Dominio 1 — Identidad y Acceso (IAM)

**Responsabilidad:** Gestionar la autenticación y autorización de todos los actores del sistema (personas naturales, personas jurídicas, operadores internos, sistemas externos).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural, empresa, sistema externo (bancos, ACH, terceros) |
| **Funciones clave** | Login multifactor, gestión de sesiones, emisión y validación de tokens (JWT/OAuth2), control de acceso basado en roles (RBAC), políticas OWASP |
| **Eventos que produce** | `UserAuthenticated`, `SessionRevoked`, `UnauthorizedAccessAttempt` |
| **Eventos que consume** | `SuspiciousTransactionDetected` (desde D8) |
| **RNF dominante** | Seguridad, Disponibilidad |
| **Observaciones** | Centraliza las políticas de autenticación exigidas por restricciones estatales. Proporciona el token que todos los demás dominios validan vía API Gateway. |

---

### Dominio 2 — Gestión de Usuarios y Cuentas

**Responsabilidad:** Registrar y mantener actualizada la información de personas naturales y sus cuentas bancarias en bancos filiales.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Bancos filiales (proveedores de datos), administrador del sistema |
| **Funciones clave** | Carga masiva desde bancos, sincronización diaria de información de clientes, consulta de estado de cuentas registradas |
| **Eventos que produce** | `UserRegistered`, `AccountLinked`, `AccountSyncCompleted` |
| **Eventos que consume** | — |
| **RNF dominante** | Fiabilidad, Consistencia eventual |
| **Observaciones** | El registro de usuarios NO es autoservicio; es un proceso ETL programado. La sincronización diaria debe ser idempotente para tolerar reintentos. |

---

### Dominio 3 — Gestión de Empresas y Empleados

**Responsabilidad:** Registrar las 15 empresas aliadas y gestionar la información básica que permite identificar a sus empleados en cada nómina.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Empresa aliada, administrador del sistema |
| **Funciones clave** | Carga masiva de empresas, gestión de referencia mínima de empleados (sin PII completa), consulta de empleados activos vía API de la empresa |
| **Eventos que produce** | `CompanyRegistered`, `EmployeeReferenceUpdated` |
| **Eventos que consume** | — |
| **RNF dominante** | Seguridad (mínimo PII almacenado), Fiabilidad |
| **Observaciones** | La plataforma solo guarda la referencia mínima del empleado; los datos completos se consultan en tiempo real al API de cada empresa al momento del pago. |

---

### Dominio 4 — Transferencias y Transacciones

**Responsabilidad:** Orquestar y registrar todas las transferencias de dinero entre cuentas, incluyendo la integración con ACH para operaciones fuera del ecosistema filial.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural (cuenta particular), Billetera Digital (D5) |
| **Funciones clave** | Transferencia inmediata entre bancos filiales, envío a ACH para bancos no filiales e internacionales, actualización inmediata del estado de cuenta en filiales, registro del movimiento tras resolución de ACH |
| **Eventos que produce** | `TransferInitiated`, `TransferCompletedImmediately`, `TransferSentToACH`, `TransferACHResolved` |
| **Eventos que consume** | `ACHResponseReceived` (desde sistema externo ACH vía D6) |
| **RNF dominante** | Consistencia (ACID en filiales), Disponibilidad, Tiempo de respuesta < 2 s |
| **Observaciones** | Patrón **Saga** para coordinar pasos distribuidos (débito → crédito → confirmación). Compensación automática ante fallo parcial. |

---

### Dominio 5 — Billetera Digital

**Responsabilidad:** Administrar la cuenta propia emitida por la Empresa X para cada usuario, habilitando movimientos internos, pagos a terceros y pagos a cuentas externas.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural |
| **Funciones clave** | Acreditar/debitar saldo de la billetera, mover dinero a cuentas activas del usuario, realizar pagos a terceros mediante pasarelas de pago |
| **Eventos que produce** | `WalletCredited`, `WalletDebited`, `ThirdPartyPaymentInitiated` |
| **Eventos que consume** | `PaymentGatewayResult` (desde D6), `TransferACHResolved` (desde D4) |
| **RNF dominante** | Consistencia, Seguridad, Tiempo de respuesta |
| **Observaciones** | Aplican las mismas reglas de transferencia que en una cuenta particular (D4). El pago a terceros se delega al dominio 6. |

---

### Dominio 6 — Integración con Terceros y Pasarelas de Pago

**Responsabilidad:** Abstraer la comunicación con sistemas externos: pasarelas de pago (PSE, DRUO, Apple Pay) y terceros de servicios (servicios públicos, impuestos, transporte, etc.).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Billetera (D5), Pagos Masivos (D7), sistemas externos |
| **Funciones clave** | Adaptar el protocolo de cada pasarela o tercero, integrar nuevos canales a demanda (patrón Plugin/Adapter), gestionar reintentos y timeouts con sistemas externos |
| **Eventos que produce** | `PaymentGatewayResult`, `ThirdPartyServiceResponse`, `ACHResponseReceived` |
| **Eventos que consume** | `ThirdPartyPaymentInitiated`, `MassivePaymentDispatched` |
| **RNF dominante** | Extensibilidad, Disponibilidad, Resiliencia |
| **Observaciones** | Requisito clave del enunciado: integrar terceros sin afectar el estado del sistema. Se implementa con un registro de adaptadores dinámico; añadir un nuevo tercero = desplegar un nuevo adapter sin cambiar el núcleo. |

---

### Dominio 7 — Pagos Masivos a Empleados

**Responsabilidad:** Gestionar el proceso de nómina de las empresas aliadas: pagos individuales, pagos masivos programados/manuales y trazabilidad por empleado.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Empresa aliada, administrador de nómina |
| **Funciones clave** | Programar lotes de pago, consultar información del empleado vía API de la empresa en el momento del pago, ejecutar 20K–30K transacciones en ventanas de alta demanda, mantener trazabilidad individual |
| **Eventos que produce** | `PayrollJobScheduled`, `MassivePaymentDispatched`, `EmployeePaymentCompleted`, `PayrollJobFinished` |
| **Eventos que consume** | `PaymentGatewayResult` (desde D6) |
| **RNF dominante** | Escalabilidad, Fiabilidad, Trazabilidad |
| **Observaciones** | Procesa lotes mediante colas de mensajes con workers escalables. La información del empleado se consulta al API de la empresa en tiempo de ejecución (no se almacena PII). Patrón **Batch + Queue-based load leveling**. |

---

### Dominio 8 — Reportes, Auditoría y Cumplimiento Regulatorio

**Responsabilidad:** Mantener el histórico completo de transacciones y generar los reportes regulatorios exigidos (extracto trimestral a bancos, reporte semestral a la Superintendencia Financiera).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Sistema (scheduler), bancos filiales, Superintendencia Financiera |
| **Funciones clave** | Almacenar histórico inmutable de transacciones, generar y enviar extracto trimestral por usuario/banco, generar reporte semestral a la Superfinanciera, monitoreo de transacciones sospechosas (listas blancas/grises/negras) |
| **Eventos que produce** | `SuspiciousTransactionDetected`, `QuarterlyStatementGenerated`, `SemiannualReportSent` |
| **Eventos que consume** | Todos los eventos de transacción de D4, D5 y D7 |
| **RNF dominante** | Integridad, Trazabilidad, Seguridad, Cumplimiento normativo |
| **Observaciones** | Usa un modelo **Event Sourcing** / append-only para el histórico. Monitoreo de fraude como stream processor (Kafka Streams / Flink) en tiempo real. Las listas negras/grises/blancas retroalimentan al dominio IAM (D1) para bloqueo preventivo. |

---

## 1.3 Mapa de dominios (resumen)

```
┌─────────────────────────────────────────────────────────┐
│                     API Gateway                         │
│         (enrutamiento, autenticación, TLS)              │
└────────────┬────────────────────────┬───────────────────┘
             │                        │
    ┌────────▼──────────┐   ┌─────────▼─────────┐
    │  D1: IAM          │   │  D8: Reportes /   │
    │ Identidad/Acceso  │◄──│   Auditoría       │
    └────────┬──────────┘   └─────────┬─────────┘
             │                        │  (consume eventos)
     ┌───────▼───────────────────┐    │
     │       Message Broker      │◄───┘
     │          (Kafka)          │
     └───┬───┬───┬───┬───┬───┬──┘
         │   │   │   │   │   │
  ┌──────▼─┐ │ ┌─▼───▼┐ │ ┌─▼────────────┐
  │ D2:    │ │ │ D4:  │ │ │ D6:          │
  │Usuarios│ │ │Trans.│ │ │Terceros/     │
  │Cuentas │ │ │y Tx  │ │ │Pasarelas     │
  └────────┘ │ └──────┘ │ └──────────────┘
         ┌───▼──┐   ┌───▼──────┐
         │ D3:  │   │ D5:      │
         │Empr/ │   │Billetera │
         │Empl. │   │Digital   │
         └──────┘   └──────────┘
                   ┌──────────────┐
                   │ D7: Pagos    │
                   │ Masivos      │
                   └──────────────┘
```

---

## Pendientes

- [ ] [POR DEFINIR] Agregar citas del material del curso (títulos, autores, unidad)
- [ ] Confirmar nombre real de la empresa o mantener "Empresa X"
- [ ] Confirmar si se agrega diagrama C4 Level 1 en esta sección o solo en Sección 3
