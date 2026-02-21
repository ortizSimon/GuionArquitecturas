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
|----------|----------|
| **Actores** | Persona natural, empresa, sistema externo (bancos, ACH, terceros) |
| **Funciones clave** | Login multifactor, gestión de sesiones, emisión y validación de tokens (JWT/OAuth2), control de acceso basado en roles (RBAC), políticas OWASP |
| **Eventos que produce** | `UserAuthenticated`, `SessionRevoked`, `UnauthorizedAccessAttempt` |
| **Eventos que consume** | `SuspiciousTransactionDetected` (desde D8) |
| **RNF dominante** | Seguridad, Disponibilidad |
| **Observaciones** | Centraliza las políticas de autenticación exigidas por restricciones estatales. Proporciona el token que todos los demás dominios validan vía API Gateway. |
| **Comunicación con otros dominios** | **Expone:**<br>- Validación de identidad (login, token, MFA)<br>- Autorización (roles/permisos, scopes)<br><br>**Lo consumen (síncrono):**<br>- D2 Usuarios y Cuentas (para consultar saldos/estados)<br>- D4 Transferencias (para ejecutar transferencias)<br>- D5 Billetera (para movimientos/pagos)<br>- D7 Pagos Masivos (para operaciones empresariales)<br>- D6 Integraciones (para autenticación entre servicios y claves/credenciales de integraciones)<br><br>**Comunicación recomendada:**<br>- Síncrona: `validateToken()`, `authorize(action, subject)`<br>- Asíncrona (opcional): `UserAuthenticated`, `SuspiciousLoginDetected` → D8 |
| **Eventos típicos (publica)** | `UserAuthenticated` (opcional)<br>`SuspiciousLoginDetected` (opcional) |

---

### Dominio 2 — Gestión de Usuarios y Cuentas

**Responsabilidad:** Registrar y mantener actualizada la información de personas naturales y sus cuentas bancarias en bancos filiales.
| Aspecto | Detalle |
|----------|----------|
| **Actores** | Bancos filiales (proveedores de datos), administrador del sistema |
| **Funciones clave** | Carga masiva desde bancos, sincronización diaria de información de clientes, consulta de estado de cuentas registradas |
| **Eventos que produce** | `UserRegistered`, `AccountLinked`, `AccountSyncCompleted` |
| **Eventos que consume** | — |
| **RNF dominante** | Fiabilidad, Consistencia eventual |
| **Observaciones** | El registro de usuarios NO es autoservicio; es un proceso ETL programado. La sincronización diaria debe ser idempotente para tolerar reintentos. |
| **Consideraciones asignadas** | 1. Gestión multibanco (consulta unificada) (Primario)<br>16. Volumen masivo de usuarios (~25M) (Primario)<br>17. Registro de usuarios mediante carga masiva (Primario)<br>18. Sincronización diaria de clientes (Primario) |
| **Comunicación con otros dominios** | Responsabilidad: usuarios, cuentas bancarias afiliadas, estados de cuenta agregados, sincronización masiva.<br><br>Se comunica con:<br>- D1 IAM (síncrono): validar identidad/permiso<br>- D6 Integraciones (asíncrono o batch): sincronizar cuentas con bancos afiliados; procesos de carga masiva<br>- D4 Transferencias (síncrono): validar cuentas origen/destino (existencia, estado, límites)<br>- D5 Billetera (síncrono): mostrar saldo combinado / mover fondos desde/hacia billetera (según diseño)<br>- D8 Auditoría (asíncrono): registrar eventos de consulta crítica, cambios en perfil, carga/sync completada |
| **Eventos típicos (publica)** | `UserImported`, `UserUpdated`, `AccountLinked`, `AccountStatusChanged`, `DailySyncCompleted` |

---

### Dominio 3 — Gestión de Empresas y Empleados

**Responsabilidad:** Registrar las 15 empresas aliadas y gestionar la información básica que permite identificar a sus empleados en cada nómina.

| Aspecto | Detalle |
|----------|----------|
| **Actores** | Empresa aliada, administrador del sistema |
| **Funciones clave** | Carga masiva de empresas, gestión de referencia mínima de empleados (sin PII completa), consulta de empleados activos vía API de la empresa |
| **Eventos que produce** | `CompanyRegistered`, `EmployeeReferenceUpdated` |
| **Eventos que consume** | — |
| **RNF dominante** | Seguridad (mínimo PII almacenado), Fiabilidad |
| **Observaciones** | La plataforma solo guarda la referencia mínima del empleado; los datos completos se consultan en tiempo real al API de cada empresa al momento del pago. |
| **Consideraciones asignadas** | 15. Interoperabilidad con empresas aliadas (nómina vía servicios) (Primario)<br>19. Carga masiva de empresas aliadas (Primario)<br>20. Modelo de datos híbrido para empleados (no guardar datos completos) (Primario)<br><br>Nota: el “pago” como tal queda en D7, pero la gestión/consulta de empleados y empresas queda aquí. |
| **Comunicación con otros dominios** | Responsabilidad: empresas aliadas, empleados (solo referencia mínima), endpoints para obtener datos del empleado desde la empresa.<br><br>Se comunica con:<br>- D6 Integraciones (síncrono): “conector” hacia APIs de empresas para obtener datos de empleados<br>- D7 Pagos Masivos (síncrono): entregar lista de empleados activos / endpoints para resolver “empleado → destino de pago”<br>- D1 IAM (síncrono): control de acceso empresarial (admins, payroll managers)<br>- D8 Auditoría (asíncrono): registro de cambios empresariales, altas/bajas de empleados (referencias), resultados de consulta |
| **Eventos típicos (publica)** | `CompanyImported`<br>`EmployeeRefCreated`<br>`EmployeeRefUpdated`<br>`CompanyPayrollScheduleUpdated` |
| **Extra (mínimo)** | Restricción clave: almacenar solo referencias mínimas de empleado y resolver datos vía servicios de la empresa. |

---

### Dominio 4 — Transferencias y Transacciones

**Responsabilidad:** Orquestar y registrar todas las transferencias de dinero entre cuentas, incluyendo la integración con ACH para operaciones fuera del ecosistema filial.

| Aspecto | Detalle |
|----------|----------|
| **Actores** | Persona natural (cuenta particular), Billetera Digital (D5) |
| **Funciones clave** | Transferencia inmediata entre bancos filiales, envío a ACH para bancos no filiales e internacionales, actualización inmediata del estado de cuenta en filiales, registro del movimiento tras resolución de ACH |
| **Eventos que produce** | `TransferInitiated`, `TransferCompletedImmediately`, `TransferSentToACH`, `TransferACHResolved` |
| **Eventos que consume** | `ACHResponseReceived` (desde sistema externo ACH vía D6) |
| **RNF dominante** | Consistencia (ACID en filiales), Disponibilidad, Tiempo de respuesta < 2 s |
| **Observaciones** | Patrón **Saga** para coordinar pasos distribuidos (débito → crédito → confirmación). Compensación automática ante fallo parcial. |
| **Consideraciones asignadas** | 2. Transferencias con múltiples destinos (Primario)<br>8. Dualidad de liquidación inmediata vs diferida (Primario)<br>9. Integración obligatoria con ACH (Secundario; Primario en D6 por ser integración)<br>10. Gestión de estados provenientes de ACH (procedente/rechazada) (Primario)<br>31. Monitoreo de transacciones sospechosas (listas blanca/gris/negra) (Primario)<br>11. Histórico completo de transacciones (Secundario; Primario en D8) |
| **Comunicación con otros dominios** | Responsabilidad: ejecutar transferencias P2P / interbancarias, estados, reglas, antifraude en línea.<br><br>Se comunica con:<br>- D1 IAM (síncrono): autorización<br>- D2 Usuarios y Cuentas (síncrono): validar cuenta origen/destino y límites<br>- D6 Integraciones (asíncrono y/o síncrono): enviar a ACH / bancos no filiales, recibir callback/estado<br>- D8 Auditoría (asíncrono): registrar transacción, cambios de estado, decisiones antifraude<br>- D5 Billetera (síncrono/asíncrono): si la billetera es origen/destino de transferencia |
| **Eventos típicos (publica)** | `TransferInitiated`<br>`TransferApproved`<br>`TransferRejected`<br>`TransferSentToACH`<br>`TransferSettled`<br>`TransferFailed`<br>`FraudCheckFlagged` |

---

### Dominio 5 — Billetera Digital

**Responsabilidad:** Administrar la cuenta propia emitida por la Empresa X para cada usuario, habilitando movimientos internos, pagos a terceros y pagos a cuentas externas.

| Aspecto | Detalle |
|----------|----------|
| **Actores** | Persona natural |
| **Funciones clave** | Acreditar/debitar saldo de la billetera, mover dinero a cuentas activas del usuario, realizar pagos a terceros mediante pasarelas de pago |
| **Eventos que produce** | `WalletCredited`, `WalletDebited`, `ThirdPartyPaymentInitiated` |
| **Eventos que consume** | `PaymentGatewayResult` (desde D6), `TransferACHResolved` (desde D4) |
| **RNF dominante** | Consistencia, Seguridad, Tiempo de respuesta |
| **Observaciones** | Aplican las mismas reglas de transferencia que en una cuenta particular (D4). El pago a terceros se delega al dominio 6. |
| **Consideraciones asignadas** | 3. Billetera virtual emitida por Empresa X (Primario)<br>4. Movimientos desde la billetera (Primario)<br>5. Pagos a terceros desde la billetera (Primario; Secundario en D6 por pasarelas/terceros) |
| **Comunicación con otros dominios** | Responsabilidad: ledger de billetera, movimientos internos, pagos a terceros (lógica billetera).<br><br>Se comunica con:<br>- D1 IAM (síncrono): autorización<br>- D2 Usuarios y Cuentas (síncrono): mover fondos entre billetera ↔ cuentas (si aplica)<br>- D6 Integraciones (síncrono): pasarelas de pago / terceros<br>- D8 Auditoría (asíncrono): registrar movimientos, pagos, reversos<br>- D4 Transferencias (secundario): si billetera participa en transferencias |
| **Eventos típicos (publica)** | `WalletCreated`<br>`WalletDebited`<br>`WalletCredited`<br>`ThirdPartyPaymentInitiated`<br>`ThirdPartyPaymentCompleted`<br>`ThirdPartyPaymentFailed` |

---

### Dominio 6 — Integración con Terceros y Pasarelas de Pago

**Responsabilidad:** Abstraer la comunicación con sistemas externos: pasarelas de pago (PSE, DRUO, Apple Pay) y terceros de servicios (servicios públicos, impuestos, transporte, etc.).

| Aspecto | Detalle |
|----------|----------|
| **Actores** | Billetera (D5), Pagos Masivos (D7), sistemas externos |
| **Funciones clave** | Adaptar el protocolo de cada pasarela o tercero, integrar nuevos canales a demanda (patrón Plugin/Adapter), gestionar reintentos y timeouts con sistemas externos |
| **Eventos que produce** | `PaymentGatewayResult`, `ThirdPartyServiceResponse`, `ACHResponseReceived` |
| **Eventos que consume** | `ThirdPartyPaymentInitiated`, `MassivePaymentDispatched` |
| **RNF dominante** | Extensibilidad, Disponibilidad, Resiliencia |
| **Observaciones** | Requisito clave del enunciado: integrar terceros sin afectar el estado del sistema. Se implementa con un registro de adaptadores dinámico; añadir un nuevo tercero = desplegar un nuevo adapter sin cambiar el núcleo. |
| **Consideraciones asignadas** | 12. Terceros a demanda (plug-and-play) (Primario)<br>13. Transparencia en integración de APIs (Primario)<br>14. Pasarelas de pago (PSE/DRUO/Apple Pay, etc.) (Primario)<br>9. Integración obligatoria con ACH (Primario)<br>26. Canales de comunicación seguros (interno y terceros) (Secundario transversal; aquí aplica fuerte por integraciones) |
| **Comunicación con otros dominios** | Responsabilidad: adaptadores/anti-corruption layer para ACH, bancos, pasarelas, terceros, empresas.<br><br>Se comunica con:<br>- D4 Transferencias (asíncrono): envío/recepción de estados ACH, bancos no filiales, internacionales<br>- D5 Billetera (síncrono/asíncrono): pasarelas PSE/DRUO/ApplePay y terceros<br>- D2 Usuarios y Cuentas (batch/asíncrono): cargas masivas y sincronización diaria<br>- D3 Empresas y Empleados (síncrono): llamados a servicios de empresas para datos de empleados<br>- D8 Auditoría (asíncrono): logs de integración, latencias, errores, reintentos, SLAs |
| **Eventos típicos (publica)** | `ACHRequestSent`<br>`ACHResponseReceived`<br>`BankSyncFailed`<br>`ThirdPartyIntegrationAdded`<br>`PaymentGatewayCallbackReceived` |


---

### Dominio 7 — Pagos Masivos a Empleados

**Responsabilidad:** Gestionar el proceso de nómina de las empresas aliadas: pagos individuales, pagos masivos programados/manuales y trazabilidad por empleado.

| Aspecto | Detalle |
|----------|----------|
| **Actores** | Empresa aliada, administrador de nómina |
| **Funciones clave** | Programar lotes de pago, consultar información del empleado vía API de la empresa en el momento del pago, ejecutar 20K–30K transacciones en ventanas de alta demanda, mantener trazabilidad individual |
| **Eventos que produce** | `PayrollJobScheduled`, `MassivePaymentDispatched`, `EmployeePaymentCompleted`, `PayrollJobFinished` |
| **Eventos que consume** | `PaymentGatewayResult` (desde D6) |
| **RNF dominante** | Escalabilidad, Fiabilidad, Trazabilidad |
| **Observaciones** | Procesa lotes mediante colas de mensajes con workers escalables. La información del empleado se consulta al API de la empresa en tiempo de ejecución (no se almacena PII). Patrón **Batch + Queue-based load leveling**. |
| **Consideraciones asignadas** | 6. Pagos empresariales (nómina: individual/masivo, manual/programado) (Primario)<br>21. Pagos masivos en fechas críticas (20k–30k) (Primario)<br>22. Ventanas específicas (14–16 y 29–31) (Primario)<br>23. Capacidad de pagos masivos cualquier día (Primario)<br>24. Escalabilidad sostenida ante alta transaccionalidad (Primario por driver de nómina/picos) |
| **Comunicación con otros dominios** | Responsabilidad: orquestación de nómina (individual/masiva), scheduling, colas/lotes, idempotencia masiva.<br><br>Se comunica con:<br>- D1 IAM (síncrono): autorización de operaciones empresariales<br>- D3 Empresas y Empleados (síncrono): obtener empleados activos (referencias) y resolver datos necesarios vía la empresa<br>- D4 Transferencias (asíncrono o síncrono controlado): ejecutar pagos como transacciones (cada pago es una transferencia)<br>- D8 Auditoría (asíncrono): trazabilidad por empleado, por lote, resultados y fallos<br>- D6 Integraciones (secundario): si pagos salen a bancos no filiales / ACH |
| **Eventos típicos (publica)** | `PayrollBatchCreated`<br>`PayrollBatchStarted`<br>`PayrollPaymentInitiated`<br>`PayrollPaymentSucceeded`<br>`PayrollPaymentFailed`<br>`PayrollBatchCompleted` |

---

### Dominio 8 — Reportes, Auditoría y Cumplimiento Regulatorio

**Responsabilidad:** Mantener el histórico completo de transacciones y generar los reportes regulatorios exigidos (extracto trimestral a bancos, reporte semestral a la Superintendencia Financiera).


| Aspecto | Detalle |
|----------|----------|
| **Actores** | Sistema (scheduler), bancos filiales, Superintendencia Financiera |
| **Funciones clave** | Almacenar histórico inmutable de transacciones, generar y enviar extracto trimestral por usuario/banco, generar reporte semestral a la Superfinanciera, monitoreo de transacciones sospechosas (listas blancas/grises/negras) |
| **Eventos que produce** | `SuspiciousTransactionDetected`, `QuarterlyStatementGenerated`, `SemiannualReportSent` |
| **Eventos que consume** | Todos los eventos de transacción de D4, D5 y D7 |
| **RNF dominante** | Integridad, Trazabilidad, Seguridad, Cumplimiento normativo |
| **Observaciones** | Usa un modelo **Event Sourcing** / append-only para el histórico. Monitoreo de fraude como stream processor (Kafka Streams / Flink) en tiempo real. Las listas negras/grises/blancas retroalimentan al dominio IAM (D1) para bloqueo preventivo. |
| **Consideraciones asignadas** | 7. Trazabilidad de nómina (Primario)<br>11. Histórico completo de transacciones (Primario)<br>30. Monitoreo de actividad (auditoría de acciones/eventos/logs) (Primario)<br>33. Extractos trimestrales a bancos (Primario)<br>34. Reporte semestral a Superintendencia Financiera (Primario)<br>35. Automatización del calendario regulatorio (Primario)<br>25. Cifrado y firmado en todo el ciclo de vida (Secundario/transversal; lo anclamos aquí por cumplimiento y evidencia) |
| **Comunicación con otros dominios** | Responsabilidad: histórico, evidencia, trazabilidad, reportes regulatorios, calendario regulatorio, monitoreo/auditoría de acciones.<br><br>Se comunica con:<br>- Recibe eventos de todos (asíncrono): D2/D3/D4/D5/D6/D7/D1<br>- Envía (asíncrono/batch): reportes a bancos y Superintendencia (vía D6 Integraciones)<br>- Consulta (síncrono opcional): para generar extractos/consultas internas, dashboards |
| **Eventos típicos (consume)** | Todos los eventos anteriores; su función es persistir, correlacionar, firmar, generar reportes. | 

---
### Transversales

| Aspecto | Detalle |
|----------|----------|
| **Naturaleza** | Requisitos y capacidades que impactan a múltiples dominios; no pertenecen exclusivamente a uno funcional. |
| **Funciones clave** | Seguridad operativa, experiencia de usuario multiplataforma, rendimiento bajo carga, alta disponibilidad, protección contra intrusiones. |
| **RNF dominante** | Seguridad, Disponibilidad, Rendimiento, Usabilidad |
| **Observaciones** | Estos elementos deben implementarse como capacidades de plataforma (cross-cutting concerns) mediante infraestructura compartida, middleware, API Gateway, observabilidad y SRE. No deben acoplarse a un único dominio. |
| **Ítems transversales y ubicación recomendada** | 29. Detección de intrusiones → D8 (Primario), D1/D6 (Secundario).<br><br>36. Multiplataforma → D2 (Secundario) y D5 (Secundario) (capa de presentación).<br><br>37. Optimización gama media/alta → Transversal (presentación + rendimiento).<br><br>38. Pantalla mínima 6 pulgadas → Transversal (presentación).<br><br>39. UX consistente y agradable → Transversal (presentación).<br><br>40. Disponibilidad 24/7 → Transversal (SRE/Plataforma; si debes asignar: D8 Secundario, D7 Secundario).<br><br>41. Respuesta < 2 segundos → Transversal (si debes asignar: D4/D2/D7 según caso de uso).<br><br>42. Desempeño bajo picos → D7 (Primario), D4 (Secundario). |
| **Comunicación / Impacto arquitectónico** | - 29 impacta monitoreo centralizado (SIEM) y bloqueo preventivo en IAM.<br>- 36–39 impactan capa de presentación y APIs de consulta.<br>- 40 requiere arquitectura altamente disponible (replicación, balanceo, autoescalado).<br>- 41 y 42 impactan D4 (transferencias) y D7 (pagos masivos) principalmente, además de caching en D2. |


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
