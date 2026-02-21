# Trabajo Final — Arquitecturas Evolutivas
## Definición de una arquitectura evolutiva de alto nivel
**Universidad EAFIT · 2026-1 · Profesor: Carlos Orozco**

---

## 1. Descripción de la arquitectura seleccionada

### 1.1 Justificación de la arquitectura

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

### 1.2 Dominios identificados (Bounded Contexts)

El sistema se divide en **8 dominios** derivados directamente del enunciado. Cada dominio es implementado como uno o más microservicios cohesionados.

---

#### Dominio 1 — Identidad y Acceso (IAM)
**Responsabilidad:** Gestionar la autenticación y autorización de todos los actores del sistema (personas naturales, personas jurídicas, operadores internos, sistemas externos).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural, empresa, sistema externo (bancos, ACH, terceros) |
| **Funciones clave** | Login multifactor, gestión de sesiones, emisión y validación de tokens (JWT/OAuth2), control de acceso basado en roles (RBAC), políticas OWASP |
| **Eventos que produce** | `UserAuthenticated`, `SessionRevoked`, `UnauthorizedAccessAttempt` |
| **RNF dominante** | Seguridad, Disponibilidad |
| **Observaciones** | Centraliza las políticas de autenticación exigidas por restricciones estatales. Proporciona el token que todos los demás dominios validan vía API Gateway. |

---

#### Dominio 2 — Gestión de Usuarios y Cuentas
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

#### Dominio 3 — Gestión de Empresas y Empleados
**Responsabilidad:** Registrar las 15 empresas aliadas y gestionar la información básica que permite identificar a sus empleados en cada nómina.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Empresa aliada, administrador del sistema |
| **Funciones clave** | Carga masiva de empresas, gestión de referencia mínima de empleados (sin PII completa), consulta de empleados activos |
| **Eventos que produce** | `CompanyRegistered`, `EmployeeReferenceUpdated` |
| **RNF dominante** | Seguridad (mínimo PII almacenado), Fiabilidad |
| **Observaciones** | La plataforma solo guarda la referencia mínima del empleado; los datos completos se consultan en tiempo real al API de cada empresa al momento del pago. |

---

#### Dominio 4 — Transferencias y Transacciones
**Responsabilidad:** Orquestar y registrar todas las transferencias de dinero entre cuentas, incluyendo la integración con ACH para operaciones fuera del ecosistema filial.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural (cuenta particular), billetera |
| **Funciones clave** | Transferencia inmediata entre bancos filiales, envío a ACH para bancos no filiales e internacionales, actualización inmediata del estado de cuenta en filiales, registro del movimiento tras resolución de ACH |
| **Eventos que produce** | `TransferInitiated`, `TransferCompletedImmediately`, `TransferSentToACH`, `TransferACHResolved` |
| **Eventos que consume** | `ACHResponseReceived` |
| **RNF dominante** | Consistencia (ACID en filiales), Disponibilidad, Tiempo de respuesta < 2 s |
| **Observaciones** | Patrón **Saga** para coordinar pasos distribuidos (débito → crédito → confirmación). Compensación automática ante fallo parcial. |

---

#### Dominio 5 — Billetera Digital
**Responsabilidad:** Administrar la cuenta propia emitida por la Empresa X para cada usuario, habilitando movimientos internos, pagos a terceros y pagos a cuentas externas.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Persona natural |
| **Funciones clave** | Acreditar/debitar saldo de la billetera, mover dinero a cuentas activas del usuario, realizar pagos a terceros mediante pasarelas de pago |
| **Eventos que produce** | `WalletCredited`, `WalletDebited`, `ThirdPartyPaymentInitiated` |
| **Eventos que consume** | `PaymentGatewayResult`, `TransferACHResolved` |
| **RNF dominante** | Consistencia, Seguridad, Tiempo de respuesta |
| **Observaciones** | Aplican las mismas reglas de transferencia que en una cuenta particular (dominio 4). El pago a terceros se delega al dominio 6. |

---

#### Dominio 6 — Integración con Terceros y Pasarelas de Pago
**Responsabilidad:** Abstraer la comunicación con sistemas externos: pasarelas de pago (PSE, DRUO, Apple Pay) y terceros de servicios (servicios públicos, impuestos, transporte, etc.).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Billetera, Pagos Masivos, sistemas externos |
| **Funciones clave** | Adaptar el protocolo de cada pasarela o tercero, integrar nuevos canales a demanda (patrón Plugin/Adapter), gestionar reintentos y timeouts con sistemas externos |
| **Eventos que produce** | `PaymentGatewayResult`, `ThirdPartyServiceResponse` |
| **Eventos que consume** | `ThirdPartyPaymentInitiated`, `MassivePaymentDispatched` |
| **RNF dominante** | Extensibilidad, Disponibilidad, Resiliencia |
| **Observaciones** | **Requisito clave del enunciado:** integrar terceros sin afectar el estado del sistema. Se implementa con un registro de adaptadores dinámico; añadir un nuevo tercero = desplegar un nuevo adapter sin cambiar el núcleo. |

---

#### Dominio 7 — Pagos Masivos a Empleados
**Responsabilidad:** Gestionar el proceso de nómina de las empresas aliadas: pagos individuales, pagos masivos programados/manuales y trazabilidad por empleado.

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Empresa aliada, administrador de nómina |
| **Funciones clave** | Programar lotes de pago, consultar información del empleado vía API de la empresa en el momento del pago, ejecutar 20K–30K transacciones en ventanas de alta demanda, mantener trazabilidad individual |
| **Eventos que produce** | `PayrollJobScheduled`, `MassivePaymentDispatched`, `EmployeePaymentCompleted`, `PayrollJobFinished` |
| **Eventos que consume** | `PaymentGatewayResult` |
| **RNF dominante** | Escalabilidad, Fiabilidad, Trazabilidad |
| **Observaciones** | Procesa lotes mediante colas de mensajes con workers escalables. La información del empleado se consulta al API de la empresa en tiempo de ejecución (no se almacena PII). Patrón **Batch + Queue-based load leveling**. |

---

#### Dominio 8 — Reportes, Auditoría y Cumplimiento Regulatorio
**Responsabilidad:** Mantener el histórico completo de transacciones y generar los reportes regulatorios exigidos (extracto trimestral a bancos, reporte semestral a la Superintendencia Financiera).

| Aspecto | Detalle |
|---------|---------|
| **Actores** | Sistema (scheduler), bancos filiales, Superintendencia Financiera |
| **Funciones clave** | Almacenar histórico inmutable de transacciones, generar y enviar extracto trimestral por usuario/banco, generar reporte semestral a la Superfinanciera, monitoreo de transacciones sospechosas (listas blancas/grises/negras) |
| **Eventos que consume** | Todos los eventos de transacción de los dominios 4, 5 y 7 |
| **RNF dominante** | Integridad, Trazabilidad, Seguridad, Cumplimiento normativo |
| **Observaciones** | Usa un modelo **Event Sourcing** / append-only para el histórico. El monitoreo de fraude puede implementarse como un stream processor (Kafka Streams / Flink) sobre los eventos en tiempo real. Los datos de listas negras/grises/blancas alimentan al dominio IAM para bloqueo preventivo. |

---

### 1.3 Mapa de dominios (resumen)

```
┌─────────────────────────────────────────────────────────┐
│                     API Gateway                         │
│         (enrutamiento, autenticación, TLS)              │
└────────────┬────────────────────────┬───────────────────┘
             │                        │
    ┌────────▼──────────┐   ┌─────────▼─────────┐
    │  D1: IAM          │   │  D8: Reportes /   │
    │ Identidad/Acceso  │   │   Auditoría       │
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
         │Empr. │   │Billetera │
         │Empl. │   │Digital   │
         └──────┘   └──────────┘
                   ┌──────────────┐
                   │ D7: Pagos    │
                   │ Masivos      │
                   └──────────────┘
```

---

### Pendientes Sección 1

- [ ] Agregar citas del material del curso (títulos, autores, unidad) — marcado como [POR DEFINIR]
- [ ] Revisar si el nombre de la empresa tiene nombre real o se mantiene como "Empresa X"
- [ ] Confirmar si el diagrama de la sección 1 es suficiente o se quiere un C4 Level 1 aquí también

---

*Próxima sección: **Sección 2 — RNF y Funciones de ajuste (Tabla 1)***


## 4. Stack tecnológico (Tabla 2)

La siguiente tabla mapea cada capa del sistema a su tecnología recomendada. **Proveedor de nube principal: AWS** (`sa-east-1` São Paulo como región primaria, `us-east-1` como región de DR). Las decisiones priorizan servicios administrados de AWS para reducir carga operacional y maximizar integración nativa; donde AWS no tiene un equivalente sólido, se mantiene la opción self-hosted sobre EKS.

### 4.1 Presentación y acceso

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Web SPA + BFF** | React 18 + Next.js 14 desplegado en **AWS Amplify + CloudFront** | Angular 17 + S3/CloudFront | Amplify automatiza el pipeline de despliegue del frontend; CloudFront distribuye desde edge locations para < 2 s en Colombia. Next.js provee el patrón BFF con API Routes. Angular es alternativa si el equipo ya lo domina. | Usabilidad, Tiempo de respuesta < 2 s |
| **App Móvil / Tablet** | Flutter 3 (Dart) + **AWS Amplify SDK** | React Native + Amplify | Un solo codebase para iOS, Android y tablet ≥ 6"; compilación nativa sin WebView; Amplify SDK integra autenticación Cognito, llamadas API y notificaciones push directamente desde el cliente móvil. React Native tiene mayor ecosistema JS pero más overhead en el puente nativo. | Usabilidad, Compatibilidad multiplataforma |
| **API Gateway** | **Amazon API Gateway (HTTP API)** | Kong Gateway (self-hosted en EKS) | Fully managed: throttling, autorización JWT/Cognito, WAF integrado y trazabilidad con X-Ray incluidos; escala automática a millones de peticiones. Kong da mayor control sobre plugins personalizados pero requiere operar el gateway y un cluster de K8s adicional. | Seguridad, Disponibilidad, Escalabilidad |

---

### 4.2 Mensajería y comunicación asíncrona

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Message Broker** | **Amazon MSK** (Managed Streaming for Apache Kafka) | Confluent Cloud | MSK elimina la gestión del plano de control de Kafka; integración nativa con IAM, VPC, CloudWatch y MSK Connect para sink/source connectors hacia otros servicios AWS. Confluent añade Schema Registry y mayor tooling de monitoreo pero con costo adicional y dependencia de tercero. | Escalabilidad, Fiabilidad, Consistencia eventual |
| **Service Mesh (mTLS inter-servicio)** | **Istio 1.x sobre EKS** | AWS App Mesh | Istio sobre EKS provee mTLS automático entre pods, circuit breaker, retry policies y métricas de tráfico; integración con OpenTelemetry para trazas. AWS App Mesh es más ligero y nativo en AWS pero con menor madurez de comunidad y roadmap incierto. | Seguridad (canales inter-servicio), Disponibilidad |

---

### 4.3 Microservicios — Runtime

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Dominios financieros — D4, D7** | Java 21 + Spring Boot 3 en **Amazon EKS** (node groups EC2) | Kotlin + Spring Boot en EKS | ACID nativo con Spring Data JPA; librerías Saga maduras (Eventuate Tram, Axon Framework); EC2 node groups permiten elegir instancias optimizadas para cómputo intensivo en picos de nómina. Kotlin reduce verbosidad pero añade curva de adopción al equipo. | Consistencia, Fiabilidad, Trazabilidad |
| **Dominios API / integración — D1, D2, D5, D6** | Node.js 20 + NestJS en **Amazon EKS + Fargate** | Go (Golang) en EKS | Alta concurrencia I/O para servicios orientados a API; Fargate elimina la gestión de nodos para pods de escala variable; TypeScript garantiza tipado estricto. Go ofrece menor latencia pero con ecosistema de librerías financieras más reducido. | Tiempo de respuesta < 2 s, Extensibilidad |
| **Stream processing — Fraude (D8)** | **Amazon Managed Service for Apache Flink** | Kafka Streams en EKS | Flink administrado sobre MSK: procesamiento stateful con ventanas de tiempo para detección de patrones sospechosos; escala automática sin gestión de cluster; métricas nativas en CloudWatch. Kafka Streams es más simple pero con menor expresividad para CEP complejo. | Seguridad, Trazabilidad en tiempo real |

---

### 4.4 Almacenamiento

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **BD relacional — D1, D2, D3, D4, D5, D7** | **Amazon Aurora PostgreSQL** (Multi-AZ, Serverless v2) | Amazon RDS for PostgreSQL Multi-AZ | Aurora ofrece hasta 5× el throughput de PostgreSQL estándar; failover automático < 30 s; réplicas de lectura para CQRS; cifrado en reposo con KMS integrado. RDS PostgreSQL es más económico para cargas bajas y predecibles pero con menor capacidad de escala automática. | Consistencia, Integridad, Disponibilidad 24/7 |
| **Caché / CQRS Query Side — D2** | **Amazon ElastiCache for Redis** (cluster mode, Multi-AZ) | ElastiCache Serverless | Lecturas sub-milisegundo para consultas de estado de cuenta; replicación Multi-AZ con failover automático; integración VPC nativa. Serverless simplifica la operación pero introduce latencia en cold-start inaceptable para el SLA de < 2 s. | Tiempo de respuesta < 2 s, Escalabilidad |
| **Event Sourcing / histórico inmutable — D8** | **Amazon Keyspaces** (Apache Cassandra compatible, serverless) | Amazon DynamoDB | Serverless con compatibilidad CQL nativa: escala automática sin gestión de cluster; TTL nativo para retención regulatoria; escrituras append-only ideales para Event Sourcing. DynamoDB tiene integración AWS más profunda pero requiere rediseño del modelo de datos (no CQL). | Integridad, Trazabilidad, Escalabilidad |
| **Búsqueda y reportes — D8** | **Amazon OpenSearch Service** (Multi-AZ) | OpenSearch self-managed en EKS | Managed: actualizaciones automáticas, Multi-AZ, indexación full-text sobre el audit log; OpenSearch Dashboards para reportes de cumplimiento regulatorio; ingesta vía Amazon Kinesis Firehose desde CloudWatch Logs. Self-managed da más control pero aumenta carga operacional. | Trazabilidad, Cumplimiento normativo |

---

### 4.5 Seguridad e identidad

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Identity Provider — D1** | **Keycloak 24 self-hosted en EKS** | Amazon Cognito | Keycloak garantiza soberanía de datos de autenticación en infraestructura propia (crítico bajo regulación colombiana); MFA, RBAC avanzado, OAuth2/OIDC y SSO nativos; amplio uso en sector financiero LATAM. Cognito es fully managed e integra con Amplify/API Gateway, pero los datos de usuarios residen en AWS, lo que puede requerir aprobación regulatoria explícita. | Seguridad, Cumplimiento normativo |
| **Gestión de secretos y cifrado** | **AWS Secrets Manager + AWS KMS** | HashiCorp Vault en EKS | Secrets Manager rota automáticamente credenciales de Aurora, MSK y APIs de terceros; KMS gestiona claves de cifrado en reposo para todos los servicios AWS con integración transparente (RDS, S3, EKS). Vault ofrece más flexibilidad multi-cloud y PKI avanzada pero requiere operar el cluster y gestionar su alta disponibilidad. | Seguridad (cifrado en reposo y tránsito, firmado) |
| **Detección de intrusiones (IDS)** | **Amazon GuardDuty + AWS Security Hub + AWS WAF** + Falco en EKS | Wazuh SIEM | GuardDuty analiza VPC Flow Logs, CloudTrail y DNS para detectar amenazas de red y acceso; Security Hub centraliza hallazgos; WAF protege el API Gateway de ataques OWASP Top 10. Falco complementa con detección a nivel de contenedor/kernel (capa que GuardDuty no cubre). Wazuh añade correlación SIEM más avanzada pero con mayor carga operacional. | Seguridad (IDS/IPS), Trazabilidad |

---

### 4.6 Infraestructura y orquestación

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Orquestación de contenedores** | **Amazon EKS** (plano de control administrado) + **AWS Fargate** para pods variables + **EC2 node groups** para pods de carga fija | EKS con solo EC2 Auto Scaling | EKS gestiona el plano de control K8s sin intervención manual; Fargate para servicios de integración y API (D6, D2); EC2 para D4/D7 donde se necesita control de instancia. Solo EC2 da más control sobre hardware pero aumenta la carga de gestión de nodos. | Disponibilidad 24/7, Escalabilidad |
| **Infraestructura como Código** | **Terraform** (AWS provider, state en S3 + DynamoDB locking) | AWS CDK | Declarativo, multi-cloud, ecosistema de módulos AWS maduros; state remoto en S3 con locking en DynamoDB; entornos reproducibles para auditorías de cumplimiento. CDK usa lenguajes de programación y genera CloudFormation nativamente, más acoplado a AWS pero con menor portabilidad. | Evolución del sistema, Cumplimiento |

---

### 4.7 Observabilidad

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Métricas y alertas** | **Amazon CloudWatch** + **Amazon Managed Grafana** | Prometheus self-managed + Grafana | CloudWatch recolecta métricas de todos los servicios AWS de forma nativa (EKS, Aurora, MSK, API Gateway); Managed Grafana conecta CloudWatch como datasource para dashboards operacionales. Prometheus self-managed ofrece mayor flexibilidad en PromQL pero requiere operar el scraper y el almacenamiento. | Disponibilidad, Tiempo de respuesta < 2 s |
| **Logs centralizados / Audit Log** | **Amazon CloudWatch Logs + Amazon OpenSearch Service** (ingesta vía Kinesis Data Firehose) | ELK Stack self-managed en EKS | CloudWatch Logs centraliza logs de EKS, Aurora, API Gateway y Lambda; Firehose enruta hacia OpenSearch para búsqueda full-text del audit log; retención y cifrado con KMS configurables por política regulatoria. ELK self-managed tiene mayor personalización pero aumenta la carga operacional significativamente. | Trazabilidad, Cumplimiento normativo |
| **Trazas distribuidas** | **AWS X-Ray** (instrumentación via OpenTelemetry SDK) | Jaeger en EKS | X-Ray integra nativamente con API Gateway, EKS y todos los servicios AWS; Service Map visual identifica latencia entre microservicios para el SLA < 2 s; sin infraestructura adicional que operar. Jaeger ofrece sampling adaptativo más fino pero requiere despliegue y almacenamiento propio. | Tiempo de respuesta < 2 s, Disponibilidad |

---

### 4.8 CI/CD

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Pipelines CI/CD** | **AWS CodePipeline + AWS CodeBuild + Amazon ECR** | GitLab CI/CD con runners en EKS | Stack fully managed: CodeBuild construye y testea imágenes de contenedores, ECR las almacena, CodePipeline orquesta stages con aprobaciones manuales antes de producción; integración nativa con EKS y Terraform. GitLab CI/CD tiene mayor madurez para flujos complejos de revisión y pipelines multi-etapa, preferible si el repositorio ya está en GitLab. | Evolución del sistema, Disponibilidad |

---

### Nota sobre soberanía de datos

AWS no cuenta con región en Colombia; la región más cercana es `sa-east-1` (São Paulo, Brasil). Para los datos de autenticación (Keycloak) y transaccionales críticos, se recomienda verificar con la Superintendencia Financiera si el alojamiento en Brasil cumple los requisitos de soberanía de datos, o evaluar la opción de mantener componentes sensibles en infraestructura on-premise con conectividad AWS Direct Connect.

---

### Pendientes Sección 4

- [ ] Confirmar con la Superfinanciera si `sa-east-1` (São Paulo) satisface requisitos de residencia de datos o si se requiere on-premise para componentes críticos
- [ ] Definir política de retención de datos en Keyspaces y OpenSearch (periodicidad de purga vs. exigencia regulatoria)
- [ ] Decidir si Keycloak self-hosted en EKS es suficiente o si se requiere proveedor IAM certificado por regulador colombiano

---

*Próxima sección: **Sección 2 — RNF y Funciones de ajuste (Tabla 1)** · Sección 3 — Diagrama C4 (Figura 1)***

