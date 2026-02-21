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
