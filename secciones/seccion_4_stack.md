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

## Tabla 2 — Stack tecnológico

| Herramienta / Tecnología | Descripción | Componente asociado | RNF que aborda | Ventajas | Riesgos |
|--------------------------|-------------|---------------------|----------------|----------|---------|
| **Kong Gateway** | API Gateway open-source con plugins para auth, rate-limiting, logging y TLS. | API Gateway | RNF-01, RNF-02, RNF-04 | Altamente extensible, soporte nativo OAuth2/JWT, gran ecosistema de plugins | Requiere configuración cuidadosa en alta disponibilidad |
| **Keycloak** | Servidor de identidad open-source: OAuth2, OpenID Connect, MFA, RBAC. | D1 — IAM Service | RNF-04 | Estándar de la industria, integración nativa con LDAP/AD, soporte MFA | Puede ser un single point of failure si no se clusteriza |
| **Spring Boot (Java 21)** | Framework backend para microservicios con soporte nativo para Kafka, JPA, seguridad y resiliencia (Resilience4j). | D2, D3, D4, D5, D7 | RNF-01, RNF-02, RNF-07 | Ecosistema maduro, soporte transacciones ACID, comunidad amplia | Consumo de memoria mayor que alternativas reactivas; mitigable con GraalVM |
| **Node.js (Fastify)** | Runtime JavaScript para servicios de integración y adaptadores con alta concurrencia I/O. | D6 — Integraciones y Pasarelas | RNF-05, RNF-02 | Ideal para I/O intensivo (llamadas a APIs externas), bajo footprint | No adecuado para lógica de negocio compleja; limitado para transacciones ACID |
| **Apache Kafka** | Plataforma de event streaming distribuida: pub/sub, almacenamiento de eventos, exactly-once delivery. | Message Broker | RNF-03, RNF-05, RNF-07 | Altísimo throughput, replay de eventos, particionamiento, ecosistema Kafka Streams/Flink | Operación compleja (ZooKeeper/KRaft), curva de aprendizaje |
| **Apache Flink / Kafka Streams** | Motor de procesamiento de streams para detección de fraude y monitoreo en tiempo real. | D8 — Auditoría y Reportes | RNF-04, RNF-06 | Procesamiento stateful en tiempo real, baja latencia | Flink añade infraestructura adicional; Kafka Streams es más simple pero menos potente |
| **PostgreSQL** | Base de datos relacional ACID para dominios con consistencia fuerte. | D2, D3, D4, D5 | RNF-07 | ACID completo, soporte JSON, replicación nativa, excelente soporte en la nube | Escalado horizontal requiere sharding o Citus |
| **Apache Cassandra** | Base de datos NoSQL de alta disponibilidad para almacenamiento append-only de histórico de transacciones. | D8 — DB Auditoría | RNF-01, RNF-03, RNF-06 | Escalado lineal, excelente para writes masivos, tolerancia a fallos | Consistencia eventual; no adecuada para queries complejas ad-hoc |
| **Redis** | Almacén en memoria para caché de sesiones, saldos frecuentes y rate-limiting del gateway. | Caché compartida | RNF-02, RNF-04 | Latencia sub-milisegundo, soporte TTL, pub/sub, clustering | Datos volátiles; requiere persistencia configurada para sesiones críticas |
| **React (PWA) + React Native** | Frontend web responsivo (PWA) y móvil híbrido con base de código compartida. | Frontend multiplataforma | RNF-08 | Un solo equipo de desarrollo, soporte web/iOS/Android/tablet, gran ecosistema | PWA tiene limitaciones en iOS comparado con app nativa |
| **Kubernetes (K8s)** | Orquestador de contenedores para despliegue, escalado automático (HPA) y self-healing. | Infraestructura de despliegue | RNF-01, RNF-03 | Escalado automático, rolling updates sin downtime, self-healing de pods | Complejidad operacional elevada; requiere equipo con experiencia en K8s |
| **Istio (Service Mesh)** | Malla de servicios para mTLS entre microservicios, circuit breaking, observabilidad de tráfico interno. | Infraestructura de red interna | RNF-01, RNF-04 | mTLS automático, circuit breakers, trazas distribuidas | Overhead de red y complejidad operacional adicional |
| **OpenTelemetry + Jaeger** | Estándar de instrumentación para trazabilidad distribuida entre microservicios. | Observabilidad | RNF-09 | Vendor-agnostic, integración con Kafka y Spring Boot, visualización en Jaeger/Grafana | Requiere instrumentar cada servicio |
| **Prometheus + Grafana** | Stack de métricas y dashboards para monitoreo de SLA (P95, error rate, throughput). | Observabilidad | RNF-02, RNF-09 | Ampliamente adoptado, alertas configurables, integración con K8s | Retención de métricas limitada sin Thanos/Cortex |
| **HashiCorp Vault** | Gestión centralizad de secretos, certificados TLS y claves de cifrado. | Seguridad transversal | RNF-04 | Rotación automática de secretos, auditoría de acceso a secretos, integración K8s | Requiere alta disponibilidad propia |
| **MinIO / AWS S3** | Almacenamiento de objetos para reportes regulatorios (extractos, informes a Superfinanciera) y backups. | D8 — Almacenamiento de reportes | RNF-06 | Alta durabilidad, acceso por políticas IAM, bajo costo para archivos grandes | — |

---

## Pendientes

- [ ] Confirmar si el equipo tiene restricciones de proveedor cloud (AWS, GCP, Azure, on-premise)
- [ ] Definir si se usa Spring Boot reactivo (WebFlux) o bloqueante (MVC) para D4 (impacto en RNF-02)
- [ ] [POR DEFINIR] Citar fuentes del curso que respalden elecciones tecnológicas
- [ ] Validar que cada componente del diagrama (Sección 3) tiene al menos una tecnología asignada
