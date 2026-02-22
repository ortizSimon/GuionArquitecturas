# Sección 2 — Identificación de RNF y Funciones de ajuste (Tabla 1)

> **Estado:** 🔄 En construcción  
> **Trazabilidad:** RNF identificados → Sección 1 (arquitectura) → Sección 3 (componentes) → Sección 4 (stack) → Sección 5 (evolución)

---

## Criterios de identificación

Los RNF se extraen **exclusivamente** del enunciado (`descripcion_del_proyecto.md`). Los marcados como **(H)** son hipótesis razonables no enunciadas explícitamente.

---

## Tabla 1 — Requisitos no funcionales

| # | Requisito no funcional | Descripción del RNF | Funciones de ajuste |
|---|------------------------|---------------------|---------------------|
| RNF-01 | **Disponibilidad** | El sistema debe operar 24/7 sin interrupciones planificadas. | • Replicación activa-activa de servicios críticos (D4, D5, D1) <br>• Health checks y circuit breakers (Resilience4j / Istio) <br>• Failover automático en infraestructura (K8s self-healing) <br>• SLA objetivo: 99.9% uptime |
| RNF-02 | **Rendimiento / Tiempo de respuesta** | El tiempo de respuesta del sistema debe ser menor a 2 segundos para operaciones de usuario. | • Caché distribuida para consultas de saldo (Redis) <br>• Procesamiento asíncrono de operaciones no críticas (Kafka) <br>• CDN para activos estáticos del frontend <br>• Métrica: P95 < 2 s en producción |
| RNF-03 | **Escalabilidad** | El sistema debe soportar ~25 millones de usuarios activos y picos de 20K–30K transacciones en ventanas específicas (días 14–16 y 29–31 de cada mes). | • Escalado horizontal automático (HPA en Kubernetes) <br>• Queue-based load leveling para pagos masivos (D7) <br>• Particionamiento de tópicos Kafka por volumen esperado <br>• Pruebas de carga previas a ventanas de pago |
| RNF-04 | **Seguridad** | El sistema debe cumplir restricciones estatales: autenticación/autorización, cifrado en todo el ciclo de vida, canales seguros, monitoreo de actividad y conformidad OWASP. | • OAuth2 + JWT con MFA (D1-IAM) <br>• TLS 1.3 en todos los canales (internos y externos) <br>• Cifrado en reposo (AES-256) para datos sensibles <br>• WAF + detección de intrusiones (IDS) <br>• Listas blancas/grises/negras en monitoreo de transacciones (D8) <br>• Auditoría de accesos y operaciones (append-only log) <br>• Conformidad OWASP Top 10 |
| RNF-05 | **Extensibilidad** | El sistema debe permitir integrar terceros (pasarelas, servicios de pago) a demanda sin afectar el estado del sistema en producción. | • Patrón Plugin/Adapter por canal externo (D6) <br>• Registro dinámico de adaptadores (sin redespliegue del núcleo) <br>• Contratos de API versionados (OpenAPI 3.x) <br>• Feature flags para activar/desactivar integraciones en caliente |
| RNF-06 | **Trazabilidad y cumplimiento regulatorio** | El sistema debe mantener histórico completo de transacciones, generar extractos trimestrales a bancos y reportes semestrales a la Superintendencia Financiera. | • Event Sourcing / append-only store (D8) <br>• Correlación de eventos con IDs únicos de transacción <br>• Jobs programados para generación y envío de reportes (scheduler) <br>• Retención configurable de datos por normativa |
| RNF-07 | **Fiabilidad / Consistencia transaccional** | Las transferencias entre bancos filiales deben reflejarse de forma inmediata; las operaciones distribuidas deben completarse o compensarse ante fallos. | • Patrón Saga con compensación (D4, D5) <br>• Idempotencia en todos los endpoints de transacción <br>• Exactly-once delivery en Kafka (transacciones Kafka) <br>• Dead letter queues para eventos fallidos |
| RNF-08 | **Multiplataforma / Usabilidad** | La interfaz debe funcionar en web, smartphones y tablets de gama media/alta (mínimo 6 pulgadas) con experiencia de usuario consistente. | • Diseño responsive (CSS Grid / Flexbox) <br>• Progressive Web App (PWA) para móvil híbrido (H) <br>• Pruebas de compatibilidad en dispositivos objetivo <br>• Guía de estilo/design system unificado (H) |
| RNF-09 | **(H) Observabilidad** | El sistema debe permitir monitorear el estado de los servicios, detectar anomalías y depurar incidentes en producción. | • Trazabilidad distribuida (OpenTelemetry / Jaeger) <br>• Métricas por servicio (Prometheus + Grafana) <br>• Logs centralizados (ELK Stack) <br>• Alertas automáticas ante umbrales críticos |
| RNF-10 | **(H) Mantenibilidad / Evolvabilidad** | La arquitectura debe permitir modificar, versionar o reemplazar dominios individuales sin afectar al resto del sistema. | • Bounded contexts con base de datos por servicio (Database per Service) <br>• Contratos de API semánticos + consumer-driven contract testing (Pact) <br>• ADR (Architecture Decision Records) por cada cambio estructural <br>• CI/CD con pipelines por microservicio |
| RNF-D5-01 | **Atomicidad del ledger (D5)** | Toda operación sobre el saldo de la billetera debe registrarse como movimiento de doble entrada (*double-entry bookkeeping*); ningún débito o crédito puede quedar sin su contraparte en la tabla de movimientos. | • Tabla `wallet_entries` append-only con columnas `debit` / `credit` siempre emparejadas en la misma transacción ACID <br>• Constraint de base de datos: `CHECK (debit > 0 XOR credit > 0)` <br>• Prueba de reconciliación automatizada: `SUM(credit) - SUM(debit) = saldo_actual` por cada `wallet_id` |
| RNF-D5-02 | **Compensación transaccional de billetera (D5)** | Si una pasarela de pago (D6) falla tras haberse debitado el saldo de la billetera, el monto debe revertirse automáticamente mediante el mecanismo de compensación de la Saga, sin intervención manual. | • Patrón Saga coreografiado: `WalletDebited` → D6 → `PaymentGatewayFailed` → `WalletCompensationTriggered` <br>• Tiempo máximo de compensación: < 5 s desde la detección del fallo <br>• Dead Letter Queue (DLQ) en Kafka para eventos de compensación fallidos con alerta automática |
| RNF-D6-01 | **Aislamiento de adapters (D6)** | Un fallo o degradación en un adapter externo (ej. DRUO fuera de servicio) no puede impactar la disponibilidad ni el rendimiento de los demás adapters (PSE, ACH, Apple Pay, terceros). | • Cada adapter se despliega como un pod independiente en EKS (fallo de un pod ≠ fallo del servicio completo) <br>• Circuit breaker por adapter con configuración independiente de umbral de error y tiempo de apertura (Resilience4j) <br>• Bulkhead pattern: thread pool separado por adapter para evitar saturación cruzada |
| RNF-D6-02 | **Integración de nuevos terceros sin downtime (D6)** | Registrar y activar un nuevo tercero o pasarela de pago no debe generar indisponibilidad en los adapters existentes ni requerir redespliegue del núcleo del servicio de integraciones. | • Adapter Registry dinámico: los adapters se registran en caliente via configuración en base de datos sin reinicio de la aplicación <br>• Despliegue del nuevo adapter como contenedor independiente (sin tocar el contenedor del núcleo de D6) <br>• Smoke test automatizado post-deploy que valida que todos los adapters existentes siguen respondiendo (health check por adapter en CI/CD) |

---

## Pendientes

- [ ] Revisar con el equipo si hay RNF adicionales identificados en clase
- [ ] Confirmar métricas concretas para RNF-01 (uptime %) y RNF-02 (P95 < 2 s) con el enunciado
- [ ] Añadir citas del material del curso que respalden las funciones de ajuste [POR DEFINIR]
- [ ] Decidir si RNF-09 y RNF-10 (hipótesis) se incluyen en el reporte o se justifican solo internamente
