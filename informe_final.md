# Informe Final — Arquitectura Evolutiva de Alto Nivel
## Empresa X — Sistema de Centralización de Transacciones Financieras

> **Curso:** Introducción a las Arquitecturas Evolutivas
> **Universidad:** EAFIT — Medellín, Colombia
> **Período:** 2026-1
> **Profesor:** Carlos Orozco

---

## 1. Descripción de la arquitectura seleccionada

Para el sistema de la Empresa X se seleccionó una **arquitectura de microservicios orientada a eventos** (*Event-Driven Microservices*), estructurada mediante **Bounded Contexts** del Domain-Driven Design (DDD). Esta decisión responde a cuatro factores dominantes del problema que en conjunto hacen inviable cualquier arquitectura monolítica o de servicios acoplados síncronamente.

La arquitectura se apoya en tres patrones complementarios: un **API Gateway** como punto de entrada único que centraliza autenticación JWT/OAuth2, enrutamiento y terminación TLS; un **Message Broker orientado a eventos (Apache Kafka / Amazon MSK)** que desacopla productores y consumidores y garantiza consistencia eventual entre servicios sin bloqueo sincrónico; y **Bounded Contexts** que implementan cada dominio como un microservicio con responsabilidad única y base de datos propia (*Database per Service*), permitiendo evolucionar o reemplazar un dominio sin afectar a los demás.

### 1.1 Justificación

El sistema maneja aproximadamente **25 millones de usuarios activos** provenientes de los cuatro bancos más grandes del país, lo que exige escalado horizontal independiente por servicio. Adicionalmente, los pagos de nómina generan entre 20.000 y 30.000 transacciones en ventanas específicas de los días 14–16 y 29–31 de cada mes, un pico predecible que solo debe afectar a los dominios de pagos masivos (D7) y transferencias (D4), no al resto del sistema. El enunciado exige también integrar terceros *"sin afectar el estado del sistema"*, requisito que se satisface con el patrón Adapter implementado en D6: añadir un nuevo tercero equivale a desplegar un adaptador independiente sin modificar el núcleo. Finalmente, la disponibilidad 24/7 con tiempo de respuesta menor a 2 segundos se garantiza con el desacoplamiento asíncrono de Kafka, donde los fallos en el sistema ACH externo no bloquean las transferencias entre bancos filiales.

Los microservicios permiten que cada dominio tenga su propio ciclo de despliegue, su propia base de datos y sus propias políticas de escalado, lo que es un requisito implícito cuando se tienen dominios con patrones de carga tan dispares como los de nómina (D7) frente a los de auditoría (D8). Esta separación también es crítica desde el punto de vista de seguridad: dominios como D1 (credenciales de autenticación), D4 (movimiento de fondos) y D8 (histórico inmutable) tienen requisitos y bases de datos radicalmente distintos que no deben compartir ciclo de vida.

### 1.2 Dominios identificados

El sistema se divide en **ocho dominios** derivados directamente del enunciado. Un grupo de funcionalidades se separa en dominio independiente cuando al menos dos de estas condiciones se cumplen: actores distintos, datos propios, posibilidad de evolución independiente o ciclo de vida diferente.

Los dominios identificados son: **D1 — Identidad y Acceso (IAM)**, responsable de autenticación, autorización y políticas OWASP; **D2 — Usuarios y Cuentas**, que gestiona el registro masivo de personas naturales y jurídicas con sus cuentas en bancos filiales; **D3 — Empresas y Empleados**, que registra las 15 empresas aliadas y mantiene solo la referencia mínima de empleados sin almacenar PII; **D4 — Transferencias y Transacciones**, núcleo de movimientos de dinero con soporte para transferencias inmediatas entre filiales y diferidas vía ACH; **D5 — Billetera Digital**, cuenta emitida por la Empresa X con capacidad de acreditar, debitar y pagar a terceros; **D6 — Integración con Terceros y Pasarelas**, que abstrae la comunicación con sistemas externos como ACH, PSE, DRUO y Apple Pay mediante el patrón Adapter; **D7 — Pagos Masivos a Empleados**, que orquesta la nómina empresarial individual y masiva con trazabilidad por empleado; y **D8 — Reportes, Auditoría y Cumplimiento**, que mantiene el histórico inmutable y genera los reportes exigidos por la Superintendencia Financiera. La comunicación entre dominios ocurre exclusivamente mediante eventos en Kafka o contratos de API con mTLS via Istio; ningún dominio accede directamente a la base de datos de otro.

### 1.3 Arquitectura descartada

La alternativa considerada fue el **monolito modular**. Se descartó por tres razones: ante picos de nómina, obligaría a escalar todo el sistema incluyendo módulos que no participan en el proceso; cualquier cambio en auditoría o integraciones requeriría redesplegar la totalidad, afectando la disponibilidad 24/7; y el requisito de integrar terceros sin afectar el sistema es estructuralmente difícil de cumplir cuando cada nuevo adaptador implica compilar y redesplegar el conjunto. Una arquitectura evolutiva requiere que sus partes cambien de forma incremental e independiente, condición que el monolito no satisface con la heterogeneidad de dominios que presenta este sistema.

### 1.4 Modelo de despliegue

El despliegue es **híbrido: on-premise en Colombia + AWS (`sa-east-1`)**. La Superintendencia Financiera exige que los datos sensibles —PII de usuarios, credenciales de autenticación y transacciones financieras— residan en territorio colombiano. Por ello, las bases de datos de D1, D2, D4 y D8 operan on-premise; el cómputo de D2 a D7 corre en Amazon EKS y se conecta a las bases reguladas mediante AWS Direct Connect (2× 1 Gbps, ~15–25 ms), latencia mitigada por caché Redis para consultas de alta frecuencia. La región `us-east-1` actúa como DR.

---

> **Estado del informe:** Secciones 1 y 4 completadas. Pendiente: Sección 2 (RNF y funciones de ajuste), Sección 3 (Diagrama), Sección 5 (Estrategias de evolución), Sección 6 (Referencias completas).

---

## 4. Stack tecnológico

El stack se seleccionó bajo tres criterios: alineación con los RNF del sistema, madurez en el ecosistema financiero colombiano, y cumplimiento del modelo de despliegue híbrido requerido por la Superintendencia Financiera. Los servicios que procesan o persisten datos sensibles —PII de usuarios, credenciales de autenticación y transacciones financieras— residen en un datacenter on-premise en Colombia; el cómputo de los demás dominios corre en Amazon EKS (`sa-east-1`) y se conecta a las bases reguladas mediante AWS Direct Connect. Los datos efímeros como caché y mensajería en tránsito se mantienen en AWS.

**Tabla 2. Stack tecnológico.**

| Herramienta o Tecnología | Descripción | Componente Asociado |
|--------------------------|-------------|---------------------|
| React 18 + Next.js 14 (AWS Amplify + CloudFront) | Aplicación web SPA con patrón BFF; CloudFront distribuye desde edge para cumplir el SLA de < 2 s en Colombia | Frontend web |
| Flutter 3 + AWS Amplify SDK | Un solo codebase para iOS, Android y tablet ≥ 6"; compilación nativa sin WebView | App móvil / tablet |
| Amazon API Gateway (HTTP API) + AWS WAF | Punto de entrada único al sistema: autenticación JWT, enrutamiento, rate-limiting y protección OWASP | API Gateway |
| Amazon MSK (Apache Kafka administrado) | Mensajería asíncrona entre dominios; garantiza entrega, orden y replay de eventos críticos | Comunicación asíncrona entre todos los dominios |
| Istio 1.x sobre EKS | Service mesh con mTLS automático entre pods, circuit breaker y retry policies | Comunicación síncrona inter-servicio segura |
| Java 21 + Spring Boot 3 (Amazon EKS) | Runtime para dominios con lógica transaccional; incluye Spring Batch para cargas masivas y librerías Saga maduras | D3, D4, D7 |
| Node.js 20 + NestJS (Amazon EKS + Fargate) | Alta concurrencia I/O para dominios orientados a integración y API; Fargate elimina la gestión de nodos | D2, D5, D6 |
| Keycloak 24 + NestJS API (K8s on-premise Colombia) | Autenticación MFA, OAuth2/OIDC, gestión de sesiones y RBAC; co-localizado con su BD para cumplimiento regulatorio | D1 — Identidad y Acceso (IAM) |
| Amazon Managed Service for Apache Flink | Stream processing stateful con ventanas de tiempo para detección de patrones de fraude en tiempo real | D8 — Antifraude en tiempo real |
| Java 21 + Spring Boot 3 / Python 3.12 (K8s on-premise Colombia) | Ingestión de eventos en Cassandra (append-only) y generación de extractos trimestrales y reportes regulatorios | D8 — Event Ingester y Report Generator |
| PostgreSQL 16 + Patroni + pgBackRest (on-premise Colombia) | Base de datos regulada con HA automático y backups; datos sensibles nunca salen del territorio colombiano | D1, D2, D4 — PII, credenciales y transacciones |
| Amazon Aurora PostgreSQL (Multi-AZ, Serverless v2) | BD relacional administrada con failover automático < 30 s para datos de menor sensibilidad regulatoria | D3, D5, D7 |
| Amazon ElastiCache for Redis (cluster Multi-AZ) | Caché distribuida sub-milisegundo: listas antifraude (TTL 60 s), bancos filiales (TTL 5 min), saldos e idempotencia | D1, D2, D4, D5, D7 |
| Apache Cassandra 4.1 RF=3 (on-premise Colombia) | Audit log inmutable con escrituras append-only, retención ≥ 5 años y cifrado en reposo; exigido en Colombia | D8 — Histórico de auditoría |
| Amazon OpenSearch Service | Indexación full-text del audit log para búsquedas operacionales y reportes de cumplimiento | D8 — Búsqueda y reportes |
| Amazon S3 (SSE-KMS, versionado habilitado) | Almacenamiento cifrado de extractos trimestrales y reportes semestrales con lifecycle a Glacier | D8 — Reportes regulatorios |
| AWS Secrets Manager + AWS KMS | Rotación automática de credenciales y cifrado en reposo para servicios en AWS | D3, D5, D6, D7, MSK, Redis |
| HashiCorp Vault (on-premise) | Gestión de secretos y cifrado por columna para bases de datos reguladas on-premise | D1, D2, D4, D8 |
| Amazon GuardDuty + AWS WAF + Falco | Detección de intrusiones en AWS y a nivel de contenedor/kernel en ambos clusters; hallazgos centralizados en Security Hub | Transversal — Seguridad e IDS |
| Amazon EKS + Fargate + EC2 node groups | Orquestación de contenedores en AWS; Fargate para pods variables y EC2 para cargas fijas con HPA | D2, D3, D4, D5, D6, D7 en AWS |
| Kubernetes 1.29 + MetalLB + Longhorn (on-premise) | Orquestación on-premise con balanceo L4 y almacenamiento persistente replicado para servicios regulados | D1, D8 on-premise Colombia |
| AWS Direct Connect 2× 1 Gbps + VPN IPSec | Conectividad híbrida dedicada (~15–25 ms Colombia → AWS) con cifrado en tránsito | Enlace datacenter Colombia ↔ AWS |
| Terraform (state en S3 + DynamoDB locking) | Infraestructura como código para gestionar AWS y on-premise desde un único repositorio | Infraestructura completa |
| CloudWatch + Managed Grafana + X-Ray + OpenTelemetry Collector | Observabilidad unificada: métricas, logs y trazas distribuidas de todos los dominios incluyendo on-premise | Transversal — Observabilidad |
| AWS CodePipeline + CodeBuild + Amazon ECR | Pipelines CI/CD con aprobaciones manuales antes de producción e imágenes de contenedor versionadas | Transversal — CI/CD |
