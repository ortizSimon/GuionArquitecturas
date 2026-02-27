# Sección 4 — Stack tecnológico (Tabla 2)

> **Estado:** 🔄 En construcción  
> **Trazabilidad:** Componentes (Sección 3) → RNF (Sección 2) → Justificación de elección tecnológica

---

## Criterios de selección

Las tecnologías se seleccionan con base en:
1. Restricciones explícitas del enunciado (multiplataforma, seguridad estatal, integraciones externas).
2. Alineación con los RNF identificados (Sección 2).
3. Madurez en el ecosistema financiero / bancario colombiano.
4. [POR DEFINIR] Restricciones adicionales del equipo o del curso.

---

## 4. Stack tecnológico (Tabla 2)

La siguiente tabla mapea cada capa del sistema a su tecnología recomendada. **Modelo de despliegue: híbrido (on-premise Colombia + AWS).** La Superintendencia Financiera confirmó que el alojamiento de datos regulados en Brasil (`sa-east-1`) no es aceptable; por lo tanto, los datos sensibles (autenticación, PII de usuarios, transacciones financieras e histórico de auditoría) residen en un **datacenter on-premise en Colombia**. El cómputo de D1 (IAM) y D8 (Event Ingester / Report Generator) se co-localiza on-premise con sus bases de datos. Los servicios de D2, D3, D4, D5, D6 y D7 se ejecutan en **Amazon EKS** (`sa-east-1`); D2 y D4 conectan a sus bases de datos on-premise vía **AWS Direct Connect** (2× 1 Gbps, ~15–25 ms). Los servicios administrados de AWS (MSK, ElastiCache Redis, CloudFront, OpenSearch, S3, observabilidad) se mantienen en `sa-east-1` para datos efímeros y operacionales. `us-east-1` actúa como región de DR.

### 4.1 Presentación y acceso

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Web SPA + BFF** | React 18 + Next.js 14 desplegado en **AWS Amplify + CloudFront** | Angular 17 + S3/CloudFront | Amplify automatiza el pipeline de despliegue del frontend; CloudFront distribuye desde edge locations para < 2 s en Colombia. Next.js provee el patrón BFF con API Routes. Angular es alternativa si el equipo ya lo domina. | Usabilidad, Tiempo de respuesta < 2 s |
| **App Móvil / Tablet** | Flutter 3 (Dart) + **AWS Amplify SDK** | React Native + Amplify | Un solo codebase para iOS, Android y tablet ≥ 6"; compilación nativa sin WebView; Amplify SDK integra llamadas API y notificaciones push; autenticación vía Keycloak (OIDC) a través de la librería `flutter_appauth` compatible con OAuth2/OpenID Connect. React Native tiene mayor ecosistema JS pero más overhead en el puente nativo. | Usabilidad, Compatibilidad multiplataforma |
| **API Gateway** | **Amazon API Gateway (HTTP API)** | Kong Gateway (self-hosted en EKS) | Fully managed: throttling, autorización JWT (validación de tokens emitidos por Keycloak via JWT authorizer), WAF integrado y trazabilidad con X-Ray incluidos; escala automática a millones de peticiones. Kong da mayor control sobre plugins personalizados pero requiere operar el gateway y un cluster de K8s adicional. | Seguridad, Disponibilidad, Escalabilidad |

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
| **Dominios financieros — D3, D4, D7** | Java 21 + Spring Boot 3 en **Amazon EKS** (node groups EC2) | Kotlin + Spring Boot en EKS | ACID nativo con Spring Data JPA; Spring Batch para cargas masivas de D3; librerías Saga maduras (Eventuate Tram, Axon Framework); EC2 node groups permiten elegir instancias optimizadas para cómputo intensivo en picos de nómina. **D4 conecta a PostgreSQL on-premise vía Direct Connect** (~15–25 ms mitigados por caché Redis en AWS); D3 y D7 usan Aurora PostgreSQL en AWS. Kotlin reduce verbosidad pero añade curva de adopción al equipo. | Consistencia, Fiabilidad, Trazabilidad |
| **Dominios API / integración — D2, D5, D6** | Node.js 20 + NestJS en **Amazon EKS + Fargate** | Go (Golang) en EKS | Alta concurrencia I/O para servicios orientados a API; Fargate elimina la gestión de nodos para pods de escala variable; TypeScript garantiza tipado estricto. **D2 conecta a PostgreSQL on-premise vía Direct Connect** (~15–25 ms mitigados por caché Redis de saldo); D5 y D6 usan Aurora PostgreSQL en AWS. Go ofrece menor latencia pero con ecosistema de librerías financieras más reducido. | Tiempo de respuesta < 2 s, Extensibilidad |
| **Identidad y acceso — D1** | **Keycloak 24** (Java) + **NestJS** thin API, desplegados en **cluster K8s on-premise (Colombia)** | Amazon Cognito + Lambda | Co-localizado con su PostgreSQL on-premise para eliminar latencia inter-DC en operaciones de autenticación (P95 login < 1.5 s). Keycloak maneja autenticación, MFA, OAuth2/OIDC y gestión de sesiones; el servicio NestJS publica `UserAuthenticated` / `SuspiciousLoginDetected` a MSK vía Direct Connect y gestiona políticas RBAC custom. Los datos de autenticación nunca salen del territorio colombiano. | Seguridad, Disponibilidad, Cumplimiento normativo |
| **Stream processing — Fraude (D8)** | **Amazon Managed Service for Apache Flink** | Kafka Streams en EKS | Flink administrado sobre MSK: procesamiento stateful con ventanas de tiempo para detección de patrones sospechosos; escala automática sin gestión de cluster; métricas nativas en CloudWatch. Kafka Streams es más simple pero con menor expresividad para CEP complejo. | Seguridad, Trazabilidad en tiempo real |
| **Event Ingester (D8)** | Java 21 + Spring Boot 3 en **cluster K8s on-premise (Colombia)** | Node.js + NestJS | Co-localizado con Apache Cassandra on-premise: consume eventos de D4/D5/D7 desde MSK vía Direct Connect, los persiste en Cassandra (append-only) sin cruce inter-DC en la escritura. Reenvía a OpenSearch (AWS) vía Kinesis Data Firehose a través de Direct Connect para indexación full-text. Java comparte modelos de dominio con Flink. | Trazabilidad, Integridad, Cumplimiento normativo |
| **Fraud List Manager (D8)** | Java 21 + Spring Boot 3 en **Amazon EKS** (Fargate) | Incluido en Event Ingester | Separado del Event Ingester para mantenerlo en AWS junto a Flink: consume `SuspiciousPatternDetected` desde MSK, actualiza listas antifraude y propaga a Redis (D4) directamente sin cruce de Direct Connect. Publica `FraudListUpdated` a MSK. | Seguridad, Trazabilidad en tiempo real |
| **Report Generator (D8)** | **Python 3.12** en **cluster K8s on-premise (Colombia)** (ejecución bajo demanda) | Java + Spring Batch | Co-localizado con Cassandra on-premise: lee directamente del audit store sin cruce inter-DC; genera extractos trimestrales y reportes semestrales (PDF/CSV para bancos, XML para Superintendencia); sube reportes a S3 (AWS) vía Direct Connect con cifrado SSE-KMS. Python ofrece ecosistema maduro para generación de documentos (ReportLab, openpyxl, lxml). | Cumplimiento normativo, Trazabilidad |
| **Dashboard API (D8)** | Node.js 20 + NestJS en **Amazon EKS + Fargate** | Go (Golang) | API REST protegida por D1 que expone consultas sobre OpenSearch (búsqueda por correlation_id, alertas de fraude, estado de lotes). Comparte stack con D2/D5/D6 para consistencia tecnológica. | Observabilidad, Trazabilidad |

---

### 4.4 Almacenamiento

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **BD relacional on-premise — D1, D2, D4** | **PostgreSQL 16 self-managed on-premise (Colombia)** con Patroni (HA automático) + pgBackRest (backups) | Amazon RDS for PostgreSQL vía VPN | Datos regulados (PII de ~25M usuarios, transacciones financieras, credenciales de autenticación) permanecen en territorio colombiano. Patroni gestiona failover automático entre nodos primario/réplica en el datacenter. Streaming replication asíncrona a réplica de lectura en AWS (`sa-east-1`) como DR. Cifrado por columna con pgcrypto + claves en HashiCorp Vault. | Cumplimiento normativo, Consistencia, Integridad |
| **BD relacional AWS — D3, D5, D7** | **Amazon Aurora PostgreSQL** (Multi-AZ, Serverless v2) | Amazon RDS for PostgreSQL Multi-AZ | Datos de menor sensibilidad regulatoria (empresas aliadas, billetera, pagos masivos): Aurora ofrece hasta 5× el throughput de PostgreSQL estándar; failover automático < 30 s; réplicas de lectura para CQRS; cifrado en reposo con KMS integrado. | Consistencia, Integridad, Disponibilidad 24/7 |
| **Caché distribuida — D1, D2, D4, D5, D7** | **Amazon ElastiCache for Redis** (cluster mode, Multi-AZ) | ElastiCache Serverless | Lecturas sub-milisegundo para múltiples casos de uso: proxy de saldo en tiempo real (D2, TTL corto), listas antifraude y registro de bancos filiales (D4, TTL 60 s y 5 min respectivamente), idempotencia de operaciones (D5, TTL 24 h), contadores de intentos de login (D1), validación de idempotencia de pagos de nómina (D7, TTL 24 h). Replicación Multi-AZ con failover automático; integración VPC nativa. Serverless simplifica la operación pero introduce latencia en cold-start inaceptable para el SLA de < 2 s. | Tiempo de respuesta < 2 s, Escalabilidad, Seguridad |
| **Event Sourcing / histórico inmutable on-premise — D8** | **Apache Cassandra 4.1 self-managed on-premise (Colombia)** con replicación multi-rack (RF=3) | Amazon Keyspaces (serverless en AWS) | El audit log completo con firmas digitales y hash chains permanece en territorio colombiano (exigencia de la Superfinanciera). Cassandra provee escrituras append-only, CQL nativo y escalabilidad lineal. TTL deshabilitado (retención ≥ 5 años). Cifrado en reposo con Transparent Data Encryption + claves en HashiCorp Vault. Keyspaces es alternativa serverless si la regulación cambiara, pero actualmente no cumple el requisito de soberanía. | Cumplimiento normativo, Integridad, Trazabilidad |
| **Búsqueda y reportes — D8** | **Amazon OpenSearch Service** (Multi-AZ) | OpenSearch self-managed en EKS | Managed: actualizaciones automáticas, Multi-AZ, indexación full-text sobre el audit log; OpenSearch Dashboards para reportes de cumplimiento regulatorio; ingesta vía Amazon Kinesis Firehose desde CloudWatch Logs. Self-managed da más control pero aumenta carga operacional. | Trazabilidad, Cumplimiento normativo |
| **Almacenamiento de reportes regulatorios — D8** | **Amazon S3** (SSE-KMS, versionado habilitado) | Azure Blob Storage | Durabilidad 99.999999999% (11 nines); cifrado con KMS para extractos trimestrales y reportes semestrales; versionado para mantener histórico de envíos; políticas de lifecycle (S3 Intelligent-Tiering → Glacier) para optimizar costos de retención a largo plazo (>5 años). Azure Blob Storage es alternativa multi-cloud pero sin integración nativa con el ecosistema AWS. | Cumplimiento normativo, Seguridad, Trazabilidad |

---

### 4.5 Seguridad e identidad

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Identity Provider — D1** | **Keycloak 24 self-hosted en K8s on-premise (Colombia)** | Amazon Cognito | Keycloak desplegado en el cluster K8s on-premise garantiza que los datos de autenticación (hashes, MFA, sesiones de ~25M usuarios) nunca salen del territorio colombiano. MFA, RBAC avanzado, OAuth2/OIDC y SSO nativos; amplio uso en sector financiero LATAM. Cognito es fully managed pero los datos residen en AWS, lo cual fue rechazado por la Superfinanciera. | Seguridad, Cumplimiento normativo |
| **Gestión de secretos y cifrado — modelo dual** | **AWS Secrets Manager + AWS KMS** (servicios en AWS: D3, D5, D6, D7, Redis, MSK) + **HashiCorp Vault on-premise** (servicios on-premise: D1, D2-BD, D4-BD, D8) | Solo KMS / Solo Vault | Modelo dual: Secrets Manager rota automáticamente credenciales de Aurora (D3/D5/D7), MSK y APIs de terceros; KMS cifra datos en reposo en AWS. Vault on-premise gestiona secretos y cifrado de las BDs on-premise (D1, D2, D4, D8): Transit engine para cifrado por columna (pgcrypto), PKI engine para certificados mTLS del cluster on-premise, rotación automática de credenciales PostgreSQL y Cassandra. Vault se sincroniza con Secrets Manager vía Vault AWS Secrets Engine para credenciales cross-environment. | Seguridad (cifrado en reposo y tránsito, firmado) |
| **Detección de intrusiones (IDS)** | **Amazon GuardDuty + AWS Security Hub + AWS WAF** + **Falco** en EKS y K8s on-premise | Wazuh SIEM | GuardDuty analiza VPC Flow Logs, CloudTrail y DNS para detectar amenazas en AWS; Security Hub centraliza hallazgos; WAF protege el API Gateway de ataques OWASP Top 10. Falco se despliega en ambos clusters (EKS y K8s on-premise) para detección a nivel de contenedor/kernel; los hallazgos on-premise se envían a Security Hub vía Direct Connect. Wazuh añade correlación SIEM más avanzada pero con mayor carga operacional. | Seguridad (IDS/IPS), Trazabilidad |

---

### 4.6 Infraestructura y orquestación

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Orquestación de contenedores — AWS** | **Amazon EKS** (plano de control administrado) + **AWS Fargate** para pods variables + **EC2 node groups** para pods de carga fija | EKS con solo EC2 Auto Scaling | EKS gestiona el plano de control K8s sin intervención manual; Fargate para servicios de integración y API (D2, D5, D6, D8 Dashboard API, D8 Fraud List Manager); EC2 para D3/D4/D7 y D8 Flink donde se necesita control de instancia. | Disponibilidad 24/7, Escalabilidad |
| **Orquestación de contenedores — On-premise (Colombia)** | **Kubernetes 1.29** (kubeadm) + **MetalLB** (balanceo L4) + **Longhorn** (almacenamiento persistente) | Rancher / RKE2 | Cluster K8s on-premise dedicado para D1 (Keycloak + NestJS API) y D8 (Event Ingester + Report Generator). kubeadm provee control total sobre el plano de control sin dependencia de proveedor; MetalLB asigna IPs del rango del datacenter; Longhorn provee volúmenes replicados para PostgreSQL y Cassandra. Rancher/RKE2 simplifica la gestión pero añade dependencia de SUSE. Mínimo 3 nodos master + 3 nodos worker. | Cumplimiento normativo, Disponibilidad |
| **Conectividad híbrida** | **AWS Direct Connect** dedicado (2× 1 Gbps, rutas físicas distintas) + **VPN Site-to-Site** (IPSec overlay) | VPN sobre internet público | Direct Connect provee latencia predecible (~15–25 ms Colombia → São Paulo) y ancho de banda garantizado para tráfico entre servicios AWS y BDs on-premise (D2→PG, D4→PG, Event Ingester→MSK). Dos conexiones por rutas distintas garantizan resiliencia ante corte de fibra. VPN IPSec añade cifrado en tránsito adicional al TLS de la aplicación. VPN sobre internet es más económica pero con latencia variable e inaceptable para SLA < 2 s. | Disponibilidad, Seguridad, Tiempo de respuesta < 2 s |
| **Infraestructura como Código** | **Terraform** (AWS provider + Kubernetes provider + Vault provider, state en S3 + DynamoDB locking) | AWS CDK + Ansible | Declarativo y multi-entorno: gestiona infraestructura AWS (EKS, MSK, Aurora, Redis) y on-premise (K8s manifests, Vault config, PostgreSQL/Cassandra provisioning) desde un solo repositorio. State remoto en S3 con locking en DynamoDB; entornos reproducibles para auditorías de cumplimiento. CDK es más acoplado a AWS y no gestiona on-premise nativamente. | Evolución del sistema, Cumplimiento |

---

### 4.7 Observabilidad

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Métricas y alertas** | **Amazon CloudWatch** + **Amazon Managed Grafana** | Prometheus self-managed + Grafana | CloudWatch recolecta métricas de todos los servicios AWS de forma nativa (EKS, Aurora, MSK, API Gateway); Managed Grafana conecta CloudWatch como datasource para dashboards operacionales. Prometheus self-managed ofrece mayor flexibilidad en PromQL pero requiere operar el scraper y el almacenamiento. | Disponibilidad, Tiempo de respuesta < 2 s |
| **Logs centralizados / Audit Log** | **Amazon CloudWatch Logs + Amazon OpenSearch Service** (ingesta vía Kinesis Data Firehose) | ELK Stack self-managed en EKS | CloudWatch Logs centraliza logs de EKS, Aurora, API Gateway y Lambda; Firehose enruta hacia OpenSearch para búsqueda full-text del audit log; retención y cifrado con KMS configurables por política regulatoria. ELK self-managed tiene mayor personalización pero aumenta la carga operacional significativamente. | Trazabilidad, Cumplimiento normativo |
| **Trazas distribuidas** | **AWS X-Ray** (instrumentación via OpenTelemetry SDK) | Jaeger en EKS | X-Ray integra nativamente con API Gateway, EKS y todos los servicios AWS; Service Map visual identifica latencia entre microservicios para el SLA < 2 s; sin infraestructura adicional que operar. Jaeger ofrece sampling adaptativo más fino pero requiere despliegue y almacenamiento propio. | Tiempo de respuesta < 2 s, Disponibilidad |
| **Bridge de observabilidad on-premise → AWS** | **OpenTelemetry Collector on-premise** + **Prometheus node-exporter** con remote-write a **Amazon Managed Prometheus (AMP)** | Prometheus + Grafana self-managed on-premise | OTel Collector desplegado en K8s on-premise exporta métricas, logs y trazas de D1/D8 a CloudWatch, X-Ray y AMP vía Direct Connect, consolidando la observabilidad en los mismos dashboards de Managed Grafana. Prometheus node-exporter monitorea salud de nodos físicos (CPU, disco, red). La alternativa self-managed on-premise requiere operar un stack de observabilidad paralelo con mayor carga operacional. | Observabilidad, Disponibilidad |

---

### 4.8 CI/CD

| Capa / Componente | Tecnología recomendada | Alternativa | Justificación y trade-offs | RNF vinculado |
|---|---|---|---|---|
| **Pipelines CI/CD** | **AWS CodePipeline + AWS CodeBuild + Amazon ECR** | GitLab CI/CD con runners en EKS | Stack fully managed: CodeBuild construye y testea imágenes de contenedores, ECR las almacena, CodePipeline orquesta stages con aprobaciones manuales antes de producción; integración nativa con EKS y Terraform. GitLab CI/CD tiene mayor madurez para flujos complejos de revisión y pipelines multi-etapa, preferible si el repositorio ya está en GitLab. | Evolución del sistema, Disponibilidad |

---

### 4.9 Detalle de stack — D3: Empresas y Empleados

| Componente | Tecnología | Alternativa | Por qué se eligió | RNF vinculado |
|---|---|---|---|---|
| **Carga masiva de empresas y empleados** | **Spring Batch** en EKS (mismo runtime Java/Spring Boot del servicio) | AWS Glue (ETL serverless) | Procesamiento en chunks de 500 registros con commit transaccional por chunk, reintentos configurables e idempotencia nativa (`upsert` por `external_emp_id + company_id`). Reutiliza el runtime del servicio sin infraestructura adicional. AWS Glue simplifica el ETL pero no ofrece el nivel de control transaccional requerido para garantizar idempotencia a nivel de registro. | RNF-D3-02 |
| **Cifrado de campos sensibles de empresa** | **AWS KMS** (cifrado por columna en Aurora para `tax_id` y `auth_config`) | Cifrado solo a nivel de disco | `tax_id` (NIT de la empresa) y `auth_config` (credenciales del API de la empresa aliada) se cifran por columna: incluso con acceso al disco, los campos críticos requieren la clave KMS para descifrarse. Garantiza minimización de PII sin alterar el esquema relacional. | RNF-D3-03 |
| **Circuit Breaker por empresa aliada** | **Resilience4j** (instancia independiente por empresa, integrado en Spring Boot) | Istio (nivel de malla de red) | Cada empresa aliada tiene su propio circuito con umbral de error y tiempo de apertura configurables. Ante caída del API de empresa X, el circuito de X se abre y su lote queda en `PAUSED_API_ERROR` sin afectar a las otras 14 empresas. Istio actúa a nivel de red con menor granularidad por empresa aliada. | RNF-D3-01 |

---

### 4.10 Detalle de stack — D4: Transferencias y Transacciones

| Componente | Tecnología | Alternativa | Por qué se eligió | RNF vinculado |
|---|---|---|---|---|
| **Motor de saga y compensación** | **Axon Framework / Eventuate Tram** (librería sobre Spring Boot) | Temporal.io (workflow engine) | Gestión explícita de estados de saga (`transfer_saga_state`) y compensación automática integrada con Spring Boot y Amazon MSK (vía Direct Connect). El Outbox Pattern garantiza que cada mutación de estado y su evento Kafka se persistan en la misma transacción ACID contra **PostgreSQL on-premise**. Temporal añade mayor expresividad pero requiere operar su propio cluster. | RNF-D4-01 |
| **Caché de listas antifraude** | **Amazon ElastiCache for Redis** (cluster mode, Multi-AZ, TTL 60 s) | ElastiCache Serverless | Latencia sub-milisegundo para consultar listas blanca/gris/negra, garantizando P99 < 200 ms sin impactar el SLA de 2 s. TTL de 60 s asegura propagación rápida de actualizaciones desde D8. ElastiCache Serverless introduce latencia de cold-start incompatible con ese P99. | RNF-D4-05 |
| **Caché del registro de bancos filiales** | **Amazon ElastiCache for Redis** (TTL 5 min — registro de bancos filiales para LiquidationRouter) | Consulta síncrona a D2 por cada transferencia | El LiquidationRouter necesita determinar en < 50 ms si el banco destino es filial o no. Cachear este registro con TTL de 5 min elimina una llamada síncrona a D2 del camino crítico sin riesgo de inconsistencia, dado que el conjunto de bancos filiales cambia con muy poca frecuencia. | RNF-D4-02 |
| **Escalado horizontal en picos de nómina** | **Amazon EKS + HPA** con métricas personalizadas vía **CloudWatch Adapter** | KEDA (Kubernetes Event-Driven Autoscaling) | HPA escala los pods de D4 al detectar latencia de cola > 1.5 s o CPU > 70%; EC2 node groups permiten elegir instancias de mayor capacidad en los días de pago masivo (14–16 y 29–31). KEDA añade mayor granularidad por tamaño de cola Kafka pero con mayor complejidad operacional. | RNF-D4-01, RNF-D4-02 |

---

### 4.11 Detalle de stack — D5: Billetera Digital

| Componente | Tecnología | Alternativa | Por qué se eligió | RNF vinculado |
|---|---|---|---|---|
| **Base de datos del ledger** | **Aurora PostgreSQL** (tabla `wallet_entries`, solo se insertan registros, nunca se modifican) | PostgreSQL propio en contenedor | Aurora garantiza alta disponibilidad y recuperación automática ante fallos. Al no permitir modificar registros, el historial de movimientos nunca se puede alterar. | RNF-D5-01 |
| **Saga coreografiada (consumidor de eventos)** | **NestJS** con consumidores Kafka (mismo lenguaje que el resto de D5) | Eventuate Tram (Java) | Cada paso de la saga se dispara por eventos: D5 publica `WalletDebited` → D6 reacciona y procesa el pago → si D6 publica `PaymentGatewayFailed`, D5 consume el evento y revierte el débito automáticamente. No hay coordinador central; cada servicio escucha y reacciona de forma independiente. Usa el mismo stack Node.js del dominio para no agregar complejidad tecnológica. | RNF-D5-02 |
| **Control de operaciones duplicadas** | **ElastiCache Redis** (guarda un identificador único por operación durante 24 h) | Restricción `UNIQUE` directamente en Aurora | Si el usuario o la red reintenta una misma operación, Redis la detecta y la rechaza antes de tocar la base de datos, evitando dobles débitos. | RNF-D5-01, RNF-D5-02 |

---

### 4.12 Detalle de stack — D6: Integraciones y Pasarelas de Pago

| Componente | Tecnología | Alternativa | Por qué se eligió | RNF vinculado |
|---|---|---|---|---|
| **Registro de pasarelas (Adapter Registry)** | **NestJS con patrón Strategy** (cada pasarela es un módulo independiente que se activa/desactiva en caliente) | Spring Cloud (Java) | Permite agregar o quitar una pasarela (por ejemplo, un nuevo proveedor de pagos) sin apagar el servicio. La configuración de qué pasarelas están activas se guarda en Aurora. | RNF-D6-02 |
| **Aislamiento de fallos por pasarela** | **`opossum`** (circuit breaker para Node.js), una instancia por cada pasarela | Resilience4j (Java) | Si PSE falla repetidamente, solo se corta el circuito de PSE; DRUO, ACH y Apple Pay siguen funcionando con normalidad. Es la librería de circuit breaker más usada en el ecosistema Node.js. | RNF-D6-01 |
| **Pruebas de integración sin depender de externos** | **OpenAPI 3.x** (contrato de cada pasarela) + **WireMock** (simula PSE, DRUO, ACH en el pipeline de CI) | Pact | Los tests de integración corren en CI sin necesitar conexión real a PSE o ACH. El contrato OpenAPI detecta automáticamente si una pasarela cambió su API. | RNF-D6-01, RNF-D6-02 |
| **Credenciales de cada pasarela** | **AWS Secrets Manager** (una clave por pasarela, rotación automática) | HashiCorp Vault | Las claves de API de PSE, DRUO, ACH y Apple Pay se rotan automáticamente sin reiniciar ningún pod. Si se revoca una pasarela, sus credenciales no afectan a las demás. | RNF-D6-01 |

---

### 4.13 Detalle de stack — D7: Pagos Masivos a Empleados

| Componente | Tecnología | Alternativa | Por qué se eligió | RNF vinculado |
|---|---|---|---|---|
| **Scheduler transaccional** | **ShedLock** (lock distribuido sobre Aurora PostgreSQL en AWS) + `@Scheduled` de Spring Boot | Quartz Scheduler | ShedLock garantiza que un job programado se ejecute exactamente una vez, incluso si hay múltiples réplicas del servicio: adquiere un lock en Aurora (AWS, D7 permanece completamente en AWS) antes de ejecutar y lo libera al terminar. Más ligero que Quartz; no requiere tablas adicionales complejas. | RNF-D7-04 |
| **Worker pool escalable** | **Spring Boot consumers** sobre Amazon MSK + **HPA** con métricas de lag de Kafka (CloudWatch Adapter) | KEDA (Kubernetes Event-Driven Autoscaling) | Cada worker consume de una partición Kafka dedicada por empresa; HPA escala pods al detectar lag > umbral configurable. EC2 node groups permiten instancias de mayor capacidad en días de pago (14–16, 29–31). KEDA ofrece mayor granularidad pero con complejidad operacional adicional. | RNF-D7-01, RNF-D7-05 |
| **Aislamiento por empresa** | **Kafka topic particionado por `company_id`** + **Resilience4j** (circuit breaker por empresa) | Colas SQS separadas por empresa | Cada empresa tiene su propia partición (o conjunto de particiones); un fallo en la API de empresa A solo abre el circuito de A sin afectar a las demás 14 empresas. Kafka mantiene orden y garantía de entrega; SQS es alternativa más simple pero con menor control sobre particionamiento y replay. | RNF-D7-05, RNF-D7-02 |
| **Idempotencia de pagos** | **`payroll_payment_id`** como idempotency key + constraint `UNIQUE` en Aurora + validación previa en Redis (TTL 24 h) | Solo constraint en BD | Redis detecta duplicados antes de tocar Aurora, evitando contención en la BD durante picos de 30K pagos. El constraint en Aurora actúa como red de seguridad final. | RNF-D7-02 |
| **Trazabilidad de lotes** | **`payroll_batch_id`** como correlation ID propagado en todos los eventos Kafka + registro en D8 | Logging manual | Cada pago individual hereda el `payroll_batch_id` de su lote padre, permitiendo consultar en D8/OpenSearch el estado completo de un lote y el detalle de cada pago. | RNF-D7-03 |

---

### ADR-001 — Modelo híbrido adoptado (on-premise Colombia + AWS)

> **Estado:** Adoptado  
> **Fecha:** Febrero 2026  
> **Decisores:** Equipo de arquitectura + asesoría legal  

#### Contexto

AWS no cuenta con región en Colombia; la región más cercana es `sa-east-1` (São Paulo, Brasil). La Superintendencia Financiera de Colombia confirmó que el alojamiento de datos regulados (PII de usuarios, transacciones financieras, credenciales de autenticación e histórico de auditoría) fuera del territorio nacional **no es aceptable** bajo la Circular Externa 007 de 2018 y sus actualizaciones.

#### Decisión

Se adopta un **modelo híbrido** donde los datos sensibles y el cómputo de los dominios más críticos residen en un datacenter on-premise en Colombia, conectado a AWS (`sa-east-1`) mediante AWS Direct Connect.

**On-premise (datacenter Colombia):**

| Componente | Dominio | Datos / Cómputo |
|---|---|---|
| **Keycloak 24 + NestJS API** | D1 — IAM | Cómputo co-localizado con su BD |
| **PostgreSQL 16 (Patroni HA)** | D1 | Hashes de contraseñas, tokens MFA, sesiones (~25M usuarios) |
| **PostgreSQL 16 (Patroni HA)** | D2 | PII: nombres, cédulas, cuentas bancarias (~25M usuarios) |
| **PostgreSQL 16 (Patroni HA)** | D4 | Montos, cuentas origen/destino, estados de transacción |
| **Apache Cassandra 4.1 (RF=3)** | D8 | Audit log inmutable, firmas digitales, hash chains (retención ≥ 5 años) |
| **D8 Event Ingester** (Java/Spring Boot) | D8 | Cómputo co-localizado con Cassandra |
| **D8 Report Generator** (Python) | D8 | Cómputo co-localizado con Cassandra |
| **HashiCorp Vault** | Transversal | Secretos y cifrado de BDs on-premise |
| **Kubernetes 1.29** (kubeadm + MetalLB + Longhorn) | Infraestructura | Orquestación de contenedores on-premise |

**AWS (`sa-east-1`):**

| Componente | Justificación |
|---|---|
| **Amazon EKS** (D2, D3, D4, D5, D6, D7 services + D8 Dashboard API, Flink, Fraud List Manager) | Cómputo de servicios: procesan datos en tránsito sin persistirlos localmente; D2 y D4 conectan a PostgreSQL on-premise vía Direct Connect. |
| **Amazon MSK** | Eventos en tránsito cifrados (TLS 1.3 + KMS); retención temporal (7–14 días) antes de persistirse en Cassandra on-premise. |
| **ElastiCache Redis** | Datos efímeros (caché, TTL corto); no constituyen almacenamiento permanente de PII. |
| **Aurora PostgreSQL** (D3, D5, D7) | Datos de menor sensibilidad regulatoria que pueden residir en AWS. |
| **Amazon OpenSearch Service** | Indexación full-text del audit log para búsquedas operacionales (datos derivados, no fuente de verdad). |
| **Amazon S3** | Almacenamiento cifrado (SSE-KMS) de reportes regulatorios generados on-premise. |
| **CloudFront + Amplify** | CDN y frontend; no almacenan datos sensibles. |
| **CloudWatch, Managed Grafana, X-Ray** | Métricas operacionales; los logs con PII se enrutan al audit store on-premise. |
| **AWS Secrets Manager + KMS** | Secretos y cifrado de servicios en AWS (D3, D5, D6, D7, MSK, Redis). |

**Conectividad:**

| Aspecto | Detalle |
|---|---|
| **Enlace** | AWS Direct Connect dedicado, 2× 1 Gbps por rutas físicas distintas |
| **Latencia** | ~15–25 ms (Colombia → São Paulo) |
| **Cifrado** | VPN Site-to-Site sobre Direct Connect (IPSec) + TLS 1.3 de la aplicación |
| **Costo** | ~$1,500–$3,000 USD/mes por puerto + transferencia de datos |

#### Consecuencias aceptadas

| Aspecto | Impacto | Mitigación |
|---|---|---|
| **Latencia D2/D4 → BD on-premise** | +15–25 ms por query SQL | Caché Redis en AWS (proxy de saldo TTL corto para D2; listas antifraude TTL 60 s y bancos filiales TTL 5 min para D4). Patrón CQRS para lecturas frecuentes. |
| **Carga operacional on-premise** | Equipo de operaciones debe gestionar PostgreSQL, Cassandra, Vault, K8s, hardware | Patroni automatiza failover de PostgreSQL; Longhorn replica volúmenes; Terraform gestiona ambos entornos; runbooks documentados. |
| **Disponibilidad del datacenter** | Depende del proveedor colombiano | SLA contractual ≥ 99.9% con generadores, UPS y redundancia de red. |
| **Disaster Recovery** | PostgreSQL on-premise es fuente de verdad | Streaming replication asíncrona a réplica de lectura en AWS; promoción a primaria en caso de desastre del datacenter colombiano. RPO < 1 min, RTO < 30 min. |
| **Escalabilidad on-premise** | Escalar hardware es más lento que escalar en AWS | Dimensionar para picos esperados (días de nómina 14–16, 29–31); capacidad de reserva de 2× la carga pico. |

---


*Próxima sección: **Sección 5 — Estrategias para facilitar la evolución***
