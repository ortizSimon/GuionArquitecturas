# Guión de exposición — D3 y D4

> **Dominos asignados:** D3 — Empresas y Empleados · D4 — Transferencias y Transacciones
> **Tiempo estimado total:** 10–12 minutos
> **Estructura:** Intro → D3 (RNF + diagrama) → D4 (RNF + diagrama) → Cierre
> **Tip:** Las marcas `[~X min]` son orientativas. Lo que está en *cursiva* son notas internas, no se dice en voz alta.

---

## INTRO `[~30 seg]`

Buenas, mi parte cubre los dominios **D3 — Empresas y Empleados** y **D4 — Transferencias y Transacciones**.

D3 es el dominio que registra a las empresas aliadas y mantiene la referencia mínima de sus empleados. D4 es el núcleo de movimientos de dinero — toda transferencia, ya sea entre personas, entre bancos, o hacia el exterior, pasa por D4.

Voy a explicar primero los requisitos no funcionales de cada dominio, y luego el diagrama con sus componentes internos.

---

## D3 — EMPRESAS Y EMPLEADOS

### ¿Qué hace este dominio? `[~30 seg]`

D3 tiene una responsabilidad muy concreta: registrar las **15 empresas aliadas** y guardar solo la **referencia mínima** de cada empleado — nada más que un identificador, el ID de la empresa a la que pertenece, y su estado activo/inactivo.

¿Por qué tan poco? Porque los datos personales del empleado — nombre, cédula, salario, cuenta bancaria — no los almacenamos nosotros. Eso lo resuelve D6 en tiempo real, directo con la API de cada empresa, justo en el momento en que se ejecuta un pago. D3 actúa como el "directorio" mínimo; D6 es quien va a buscar el dato completo cuando se necesita.

Tecnológicamente corre en **Java 21 + Spring Boot 3** sobre **Amazon EKS**, con una base de datos **Aurora PostgreSQL** que vive completamente en AWS.

---

### RNF de D3 `[~2.5 min]`

D3 tiene tres requisitos no funcionales propios:

---

**RNF-D3-01 — Interoperabilidad con las APIs de las empresas aliadas**

El problema aquí es que cada empresa tiene su propio protocolo, su propio esquema de autenticación, y su propio formato de datos. No podemos acoplar D3 a ninguna de ellas directamente.

La función de ajuste que usamos es el **Patrón Adapter** implementado en D6: D3 solo conoce una interfaz genérica llamada `EmployeeDataPort`, y D6 se encarga de traducir esa interfaz al lenguaje de cada empresa aliada.

La métrica objetivo es que la resolución del dato del empleado — el viaje completo D3 → D6 → API de la empresa y vuelta — responda en menos de **1.5 segundos** en P95.

Y la táctica de resiliencia clave es el **Circuit Breaker por empresa** con Resilience4j: si la API de la empresa A cae, eso no afecta ni al servicio de la empresa B ni al sistema en general. Cada empresa tiene su propio breaker, completamente aislado.

---

**RNF-D3-02 — Carga masiva idempotente**

Cuando una empresa aliada se incorpora al sistema, hay que importar un volumen grande de datos — hablamos de unos **35 000 registros** de referencia de empleados — y esto tiene que completarse en menos de **10 minutos** sin bajar el SLA global del sistema.

La táctica que usamos es **Spring Batch** con chunks de 500 registros y commit transaccional por chunk. Cada chunk se procesa y confirma de forma independiente, lo que permite recuperarse de un fallo sin reprocesar todo el archivo.

La idempotencia se garantiza con un `upsert` por clave natural (`external_emp_id + company_id`): si alguien ejecuta el mismo archivo dos veces, el resultado es idéntico y sin duplicados. El job corre de forma asíncrona para no bloquear el hilo principal del servicio.

---

**RNF-D3-03 — Minimización de datos PII del empleado**

Este es quizás el RNF más importante de D3 desde el punto de vista regulatorio. El enunciado es claro: la plataforma solo puede persistir la referencia mínima del empleado. Ningún dato personal — nombre, cédula, cuenta bancaria, salario — puede estar en la base de datos de D3.

La función de ajuste que lo verifica es un **test automatizado en CI/CD** que inspecciona el esquema de la base de datos de D3 y falla si encuentra alguna columna que contenga esos atributos. Zero tolerancia.

Los datos del empleado se resuelven en tiempo real vía D6 y se manejan únicamente en memoria durante el pago; nunca se persisten.

Eso sí, los campos sensibles de la propia empresa — como el `tax_id` o la configuración de autenticación — sí están en D3, pero **cifrados en reposo con AES-256 usando AWS KMS**.

---

### Diagrama de componentes — D3 `[~2.5 min]`

*[Señalar el diagrama / Figura 4 en el reporte]*

D3 tiene cinco componentes principales:

**Company API** — es el punto de entrada REST para el administrador. Cuando se registra una empresa nueva, recibe el archivo estructurado, lo delega a Spring Batch para la carga masiva, y además llama a D2 por Istio mTLS para resolver qué cuenta bancaria tiene registrada esa empresa, que será la cuenta origen de sus futuros pagos de nómina.

**Employee Ref API** — gestiona el stub mínimo del empleado. D7 — el dominio de pagos masivos — la consulta por Istio mTLS para saber qué empleados están activos antes de procesar la nómina. Cuando se necesitan los datos completos del empleado — solo en el momento del pago — esta API llama a D6, que a su vez llama a la API de la empresa; los datos vuelven en memoria y no se persisten.

**Spring Batch** — corre en paralelo al API, no bloquea requests. Procesa en chunks de 500, hace upsert por clave natural y genera al final un evento `CompanyImported` con el resumen del resultado.

**Aurora PostgreSQL (D3)** — almacena `Company`, `EmployeeRef` y `PayrollSchedule`. Insisto: sin ninguna columna PII. Los campos sensibles de empresa están cifrados con KMS. Esta base vive completamente en AWS.

**Outbox → Kafka** — garantiza entrega at-least-once de los eventos a D8 dentro de la misma transacción ACID. Los eventos que publica son `CompanyImported` y `EmployeeRefCreated/Updated`, que D8 consume para auditoría.

En resumen, D3 se comunica hacia afuera con cuatro actores: recibe del Admin vía API Gateway, es consultado por D7 para empleados activos, llama a D2 para cuentas bancarias, y llama a D6 para datos completos del empleado.

---

## D4 — TRANSFERENCIAS Y TRANSACCIONES

### ¿Qué hace este dominio? `[~30 seg]`

D4 es el corazón del sistema. Todo movimiento de dinero — transferencias P2P, interbancarias, ACH hacia bancos no filiales, pagos a billetera, con uno o múltiples destinos — pasa por aquí.

Corre también en **Java 21 + Spring Boot 3** sobre **Amazon EKS**, pero a diferencia de D3, su base de datos está en **PostgreSQL 16 on-premise, en Colombia**, conectada vía Direct Connect. Esto es por mandato regulatorio: los datos de transacciones financieras no pueden salir del territorio colombiano.

---

### RNF de D4 `[~3.5 min]`

D4 tiene seis requisitos no funcionales específicos:

---

**RNF-D4-01 — Consistencia transaccional con múltiples destinos**

Si alguien hace una transferencia a tres destinos distintos, cada sub-transferencia debe ser atómica e independiente: el fallo de un destino no puede afectar a los otros, y ningún fondo puede quedar en estado intermedio de forma permanente.

La táctica es el **Patrón Saga coreografiada** con compensación automática, combinada con el **Outbox Pattern** para garantizar que cada evento de estado se persiste en la misma transacción ACID en la que ocurre el cambio. La idempotencia se garantiza con un `idempotency_key` único por transacción — si se intenta duplicar, se detecta y se rechaza.

Métrica: P95 de la saga completa en escenario inmediato menor a **3 segundos**, y **0 fondos perdidos** verificado por un job de reconciliación periódico.

---

**RNF-D4-02 — Dualidad de liquidación inmediata vs. diferida**

El sistema tiene que distinguir en tiempo real si una transferencia va hacia un banco filial — en cuyo caso es inmediata — o hacia un banco no filial o internacional — en cuyo caso es diferida vía ACH.

El usuario debe ser notificado del canal en cada caso. Por eso el 100% de las respuestas HTTP incluyen el campo `settlement_type`, que puede ser `IMMEDIATE` o `DEFERRED`.

Métricas: liquidación filial-filial de `TransferApproved` a `TransferSettled` en menos de **500 milisegundos** en P95. Reflejo tras callback de ACH en menos de **2 segundos** en P95.

---

**RNF-D4-03 — Resiliencia en la integración con ACH**

ACH puede caerse. Y cuando se cae, no puede afectar para nada a las transferencias entre bancos filiales, que son inmediatas. Y las transferencias diferidas que ya están en cola no pueden perderse ni duplicarse cuando ACH se recupere.

La detección de que ACH está caído ocurre en menos de **5 segundos** gracias al Circuit Breaker que D6 tiene sobre ACH. Las transferencias en estado `SENT_TO_ACH` sin callback en más de 30 minutos se re-encolan automáticamente con idempotency key, garantizando **0 duplicados**. Los eventos fallidos van a una **Dead Letter Queue** en Kafka para su trazabilidad.

---

**RNF-D4-04 — Gestión completa del ciclo de estados ACH**

ACH puede responder con varios estados: procedente, rechazada, en revisión, timeout. El sistema debe manejar correctamente todos ellos y transicionar la transacción al estado interno correcto, liberando o revirtiendo los fondos según corresponda.

La táctica es una **máquina de estados explícita** — una tabla de transiciones en base de datos — donde cada transición es transaccional: el nuevo estado y el evento correspondiente se escriben en el mismo Outbox.

Métrica: **100% de los estados ACH documentados** cubiertos por tests unitarios. P95 del procesamiento de un callback menor a **2 segundos**. Y 0 transacciones sin resolución, vigilado por un job de reconciliación diario con alerta automática.

---

**RNF-D4-05 — Monitoreo antifraude en línea**

Antes de aprobar cualquier transferencia, se evalúa en tiempo real si las cuentas involucradas están en alguna lista de control de fraude: blanca, gris o negra. Esta evaluación es síncrona y bloquea la transacción si el resultado lo indica.

La táctica clave es tener las **listas replicadas en caché Redis de D4 con TTL de 60 segundos**. Así la evaluación no depende de una llamada a D8 en tiempo real — que añadiría latencia inaceptable.

Cuando D8 detecta un nuevo patrón sospechoso y actualiza las listas, publica el evento `FraudListUpdated`, que D4 consume e invalida la caché. La propagación ocurre en menos de **60 segundos**.

Métrica principal: P99 de la evaluación antifraude menor a **200 milisegundos**. Menos del **0.5% de falsos positivos**. Y el 100% de las evaluaciones trazadas con `transfer_id`, timestamp y la regla aplicada.

---

**RNF-D4-06 — Disponibilidad y rendimiento del motor de transferencias**

D4 debe operar 24/7 con P95 end-to-end menor a **2 segundos** para la confirmación al usuario. Y esto aplica también durante los picos de nómina — días 14 al 16 y 29 al 31 de cada mes — cuando el volumen de transacciones se dispara.

Las tácticas: **separación entre la confirmación al usuario y la liquidación** — el usuario recibe confirmación al llegar a estado `APPROVED`, sin esperar que el dinero llegue al destino. **HPA en Kubernetes** que escala pods automáticamente cuando la latencia supera 1.5 s o el CPU supera 70%. Y **replicación activa-activa** en al menos dos zonas de disponibilidad.

SLA objetivo: **99.9% de disponibilidad mensual**. Recuperación ante reinicio de pod en menos de 30 segundos.

---

### Diagrama de componentes — D4 `[~3 min]`

*[Señalar el diagrama / Figura 5 en el reporte]*

D4 tiene cinco componentes que forman un pipeline de validación y coordinación:

**Transfer API** — el punto de entrada REST. Recibe la instrucción de transferencia del usuario vía API Gateway, valida el JWT, y hace una llamada síncrona a D2 por Istio mTLS para verificar que las cuentas existen y que los límites permiten la operación. Si todo está bien, pasa al siguiente componente. Retorna la confirmación al usuario cuando la transacción llega a estado `APPROVED` — sin esperar la liquidación.

**FraudChecker** — evalúa en tiempo real las listas blanca, gris y negra desde ElastiCache Redis con TTL de 60 segundos. Si la cuenta está en lista negra, rechaza inmediatamente. Si está en lista gris, aprueba pero marca con flag y alerta a D8. Si está en lista blanca o no está en ninguna, flujo normal.

**LiquidationRouter** — consulta el registro de bancos filiales en ElastiCache Redis con TTL de 5 minutos. Si el destino es un banco filial, el canal es `IMMEDIATE` — la liquidación ocurre en el mismo flujo sincrónicamente. Si el destino es un banco no filial o internacional, el canal es `DEFERRED` y la transacción pasa a `SENT_TO_ACH` para ser procesada de forma asíncrona.

**Saga Coordinator** — coordina los pasos del pago: débito en la cuenta origen, liquidación en el destino, crédito confirmado. Ante cualquier fallo en cualquier paso, ejecuta compensación automática. El progreso se registra en `transfer_saga_state` con lock optimista para evitar condiciones de carrera. Para el canal ACH, publica `TransferSentToACH` a Kafka, D6 lo consume y lo envía al sistema ACH. Cuando ACH responde, D6 publica `ACHResponseReceived` de vuelta a Kafka, y el Saga Coordinator finaliza la transacción.

**Transfer State Store (PostgreSQL on-premise)** — almacena el estado ACID de cada transacción. Está en Colombia — por mandato regulatorio — y D4 se conecta a él vía Direct Connect con una latencia de 15 a 25 milisegundos, mitigada por el caché Redis para las consultas frecuentes. La tabla Outbox está co-ubicada aquí: garantiza que el evento de cambio de estado se publica exactamente cuando ocurre la transacción, dentro del mismo `COMMIT`.

Los eventos que D4 publica a Kafka son: `TransferInitiated`, `TransferApproved`, `TransferSettled`, `TransferFailed`, `FraudCheckFlagged` y `TransferSentToACH`. Esos eventos los consumen D8 para auditoría, D6 para el canal ACH, y D5 si la billetera digital es el destino de la transferencia.

---

## CIERRE `[~30 seg]`

Para cerrar, quiero mencionar el vínculo entre mis dos dominios y el resto del sistema.

D3 y D4 no operan solos. D3 alimenta a D7 con la lista de empleados activos. D7 toma esa lista y genera las instrucciones de pago masivo. Esas instrucciones terminan llegando a D4, que es quien ejecuta cada transferencia individual al empleado.

D4, por su parte, depende de D2 para validar cuentas y de D6 para comunicarse con el sistema ACH externo. Y todo lo que pasa — cada estado, cada evento de fraude, cada compensación — queda registrado en D8 para auditoría y cumplimiento regulatorio.

Eso es todo de mi parte.

---

> **Referencias en el reporte:**
> - RNF de D3: Tabla 1, filas RNF-D3-01 a RNF-D3-03 (Sección 2)
> - RNF de D4: Tabla 1, filas RNF-D4-01 a RNF-D4-06 (Sección 2)
> - Diagrama D3: Figura 4 (Sección 3)
> - Diagrama D4: Figura 5 (Sección 3)
> - Stack tecnológico D3/D4: Sección 4.3 y 4.9/4.10
