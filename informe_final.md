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

| Capa | Tecnología | Dominios / Componentes |
|------|-----------|------------------------|
| **Frontend web** | React 18 + Next.js 14, desplegado en AWS Amplify + CloudFront | Todos los dominios con interfaz de usuario |
| **App móvil / tablet** | Flutter 3 (iOS, Android, tablet ≥ 6") + AWS Amplify SDK | Todos los dominios con interfaz de usuario |
| **API Gateway** | Amazon API Gateway (HTTP API) con WAF y autorización JWT | Punto de entrada único al sistema |
| **Mensajería asíncrona** | Amazon MSK (Apache Kafka administrado) | Comunicación entre todos los dominios (eventos) |
| **Service mesh (mTLS)** | Istio 1.x sobre EKS | Comunicación síncrona inter-servicio segura |
| **Microservicios financieros** | Java 21 + Spring Boot 3 en Amazon EKS | D3, D4, D7 — dominios con lógica transaccional y cargas masivas |
| **Microservicios de integración y API** | Node.js 20 + NestJS en Amazon EKS + Fargate | D2, D5, D6 — dominios orientados a I/O y extensibilidad |
| **Identidad y acceso** | Keycloak 24 + NestJS API en Kubernetes on-premise (Colombia) | D1 — IAM, co-localizado con su base de datos regulada |
| **Stream processing (antifraude)** | Amazon Managed Service for Apache Flink | D8 — detección de patrones sospechosos en tiempo real |
| **Ingestión y reportes de auditoría** | Java 21 + Spring Boot 3 / Python 3.12 en K8s on-premise (Colombia) | D8 — Event Ingester y Report Generator |
| **BD relacional on-premise** | PostgreSQL 16 con Patroni (HA) + pgBackRest, on-premise Colombia | D1, D2, D4 — datos regulados: PII, credenciales, transacciones |
| **BD relacional en AWS** | Amazon Aurora PostgreSQL (Multi-AZ, Serverless v2) | D3, D5, D7 — datos de menor sensibilidad regulatoria |
| **Caché distribuida** | Amazon ElastiCache for Redis (cluster Multi-AZ) | D1, D2, D4, D5, D7 — listas antifraude, saldos, idempotencia, sesiones |
| **Histórico de auditoría** | Apache Cassandra 4.1 (RF=3) on-premise Colombia | D8 — audit log inmutable con retención ≥ 5 años |
| **Búsqueda full-text** | Amazon OpenSearch Service | D8 — consultas sobre el histórico de transacciones |
| **Almacenamiento de reportes** | Amazon S3 (SSE-KMS, versionado) | D8 — extractos trimestrales y reportes a la Superintendencia |
| **Gestión de secretos (AWS)** | AWS Secrets Manager + AWS KMS | D3, D5, D6, D7, MSK, Redis — rotación automática de credenciales |
| **Gestión de secretos (on-premise)** | HashiCorp Vault | D1, D2, D4, D8 — cifrado por columna y secretos de BDs reguladas |
| **Seguridad e IDS** | Amazon GuardDuty + AWS WAF + Falco en EKS y K8s on-premise | Transversal — detección de intrusiones y protección OWASP |
| **Orquestación en AWS** | Amazon EKS + Fargate (pods variables) + EC2 (pods de carga fija) | D2–D7 en AWS |
| **Orquestación on-premise** | Kubernetes 1.29 (kubeadm) + MetalLB + Longhorn | D1, D8 on-premise Colombia |
| **Conectividad híbrida** | AWS Direct Connect dedicado 2× 1 Gbps + VPN IPSec overlay | Enlace datacenter Colombia ↔ AWS (`sa-east-1`) |
| **Infraestructura como código** | Terraform (state en S3 + DynamoDB locking) | Gestión de infraestructura AWS y on-premise |
| **Observabilidad** | Amazon CloudWatch + Managed Grafana + AWS X-Ray + OpenTelemetry Collector | Métricas, logs y trazas distribuidas de todos los dominios |
| **CI/CD** | AWS CodePipeline + CodeBuild + Amazon ECR | Pipelines de construcción y despliegue de microservicios |
