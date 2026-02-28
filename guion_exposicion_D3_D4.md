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

---
---

# Versión corta — 4 minutos

> **Tip:** Habla directo, sin pausas largas. Cada bloque es aproximadamente 1 minuto.

---

## D3 — Empresas y Empleados `[~2 min]`

Mi primer dominio es D3. Su responsabilidad es registrar las **15 empresas aliadas** y mantener únicamente la referencia mínima de cada empleado — solo el ID, la empresa, y si está activo. Ningún dato personal toca esta base de datos.

Tiene tres RNF, cada uno con sus funciones de ajuste:

**Interoperabilidad:** cada empresa aliada tiene su propio protocolo y esquema. D3 no se acopla a ninguna — delega la resolución de datos al dominio D6 vía el patrón Adapter, con un Circuit Breaker aislado por empresa. Si la API de una empresa cae, las demás no se ven afectadas. Las funciones de ajuste que verifican esto son: P95 de resolución del dato del empleado menor a 1.5 segundos, y 0 impacto cruzado entre empresas ante fallo de una de ellas.

**Carga masiva idempotente:** el onboarding de 35.000 registros debe completarse en menos de 10 minutos usando Spring Batch con chunks de 500. La función de ajuste clave es la idempotencia: ejecutar el mismo archivo dos veces produce resultado idéntico — 0 duplicados. Y el SLA global del sistema no se degrada durante el batch porque el job corre de forma asíncrona.

**Minimización de PII:** la base de datos de D3 no tiene ni una columna con nombre, cédula, salario o cuenta bancaria. La función de ajuste es un test automatizado en CI/CD que inspecciona el esquema en cada deploy — si encuentra esas columnas, el deploy falla. Los datos del empleado se resuelven en memoria en tiempo de pago, vía D6, y nunca se persisten.

*[Señalar Figura 4]* El diagrama tiene cinco componentes: la **Company API** para registro y calendarios, la **Employee Ref API** que D7 consulta antes de cada nómina, **Spring Batch** para la carga masiva, **Aurora PostgreSQL** sin PII, y el **Outbox** que publica eventos a D8 para auditoría.

---

## D4 — Transferencias y Transacciones `[~2.5 min]`

D4 es el núcleo de movimientos de dinero. Toda transferencia — P2P, interbancaria, ACH — pasa por aquí. Su base de datos es PostgreSQL on-premise en Colombia, por mandato regulatorio.

Tiene seis RNF agrupados en cuatro ideas, cada uno con funciones de ajuste medibles:

**Consistencia + dualidad de canal:** D4 usa el patrón Saga coreografiada — si algo falla en el proceso de transferencia, el sistema se revierte solo, nadie interviene manualmente. El LiquidationRouter decide en tiempo real si el destino es un banco filial o no: filial → liquida en segundos, no filial → va por ACH diferido. Las funciones de ajuste son: el 95% de las transferencias entre filiales se completan en menos de 500 milisegundos, el 95% de las sagas completas en menos de 3 segundos, y el 100% de las respuestas le dicen al usuario por qué canal va su plata.

**Resiliencia ACH:** si ACH se cae, las transferencias entre filiales no sienten nada — son caminos separados. Las que estaban esperando ACH se reenvían solas cuando vuelve, sin duplicarse. Las funciones de ajuste: ACH caído se detecta en menos de 5 segundos, cero impacto en las inmediatas, cero duplicados, y el 100% de los fallos quedan registrados en la Dead Letter Queue.

**Antifraude en línea:** antes de aprobar cualquier transferencia, se consultan las listas blanca, gris y negra desde Redis. Lista negra — rechazada. Lista gris — pasa pero con alerta. Lista blanca — flujo normal. Las funciones de ajuste: esa consulta responde en menos de 200 milisegundos en el 99% de los casos, y cuando D8 actualiza una lista, D4 la tiene en menos de 60 segundos.

**Disponibilidad:** el sistema responde en menos de 2 segundos para el 95% de las transferencias, con 99.9% de uptime mensual — máximo 43 minutos caído al mes. El HPA de Kubernetes agrega pods automáticamente cuando llega el pico de nómina para no degradarse.

*[Señalar Figura 5]* El pipeline interno es: **Transfer API** → **FraudChecker** → **LiquidationRouter** → **Saga Coordinator** → **Transfer State Store** en PostgreSQL on-premise. Todo cambio de estado se publica a Kafka en la misma transacción ACID, y los consume D8 para auditoría, D6 para ACH, y D5 si la billetera es el destino.

---

## Cierre `[~15 seg]`

D3 provee el directorio de empleados que D7 usa para ejecutar la nómina. Esos pagos de nómina terminan siendo transferencias que procesa D4. Todo queda trazado en D8.

Eso es todo.

---
---

# Notas de estudio — explicaciones ampliadas por punto

> Estas notas son para entender cada concepto antes de exponer. No se leen en voz alta.

---

## D3 — Minimización de PII

**PII** = Información Personal Identificable (nombre, cédula, salario, cuenta bancaria).

D3 conoce a los empleados de las 15 empresas aliadas. Pero el enunciado dice explícitamente: la plataforma solo guarda la información básica — el resto lo obtiene llamando a las APIs de cada empresa.

Entonces D3 guarda únicamente esto por empleado:
- `employee_ref_id` — un ID interno nuestro
- `company_id` — a qué empresa pertenece
- `status` — activo o inactivo

Sin nombre, sin cédula, sin salario, sin cuenta bancaria.

**¿Y cuando se necesitan esos datos para pagar?**
En el momento exacto del pago, D3 llama a D6 → D6 llama a la API de la empresa → los datos vuelven en memoria — se usan para procesar el pago — y se descartan. Nunca se escriben en ninguna base de datos. Si alguien hackea la base de D3, no encuentra nada útil.

**¿Qué es el test automatizado en CI/CD?**
Es una guardia arquitectural. Cada vez que alguien hace un deploy, un test revisa las definiciones de las tablas de D3. Si encuentra una columna llamada `nombre`, `cedula`, `salario` o `cuenta_bancaria` — el deploy falla y nadie puede subir ese cambio a producción.

---

## D3 — Los 5 componentes del diagrama

**1. Company API**
Es la puerta de entrada del dominio. El administrador la usa para registrar una empresa aliada nueva — le manda un archivo con los datos de la empresa y sus empleados. También gestiona los calendarios de nómina: cuándo paga esa empresa (ej. "los días 15 y 30 de cada mes"). Esa configuración queda guardada en la tabla `PayrollSchedule`.

**2. Employee Ref API**
Es la que responde preguntas sobre empleados. El único que la llama es D7 (pagos masivos), justo antes de ejecutar una nómina: le pregunta "dame los empleados activos de la empresa X". D3 responde con los IDs — nada más. Cuando se necesitan los datos completos, esta API llama a D6, D6 llama a la API de la empresa, y los datos vuelven en memoria. Nunca se guardan.

**3. Spring Batch**
Es el motor de carga masiva. Divide el archivo de 35.000 empleados en grupos de 500 y confirma cada grupo por separado. Si algo falla a mitad del archivo, solo se reprocesa ese grupo. Y si alguien manda el mismo archivo dos veces, detecta que ya existen esos registros y no los duplica.

**4. Aurora PostgreSQL (sin PII)**
Guarda tres tablas: `Company`, `EmployeeRef` (solo ID + empresa + status) y `PayrollSchedule`. Sin nombre, sin cédula, sin salario.

**5. Outbox → Kafka**
Cada vez que pasa algo importante en D3, ese evento tiene que llegar a D8 para auditoría. El Outbox garantiza que ese evento no se pierda nunca: se escribe en la misma transacción de base de datos que el cambio que lo originó. Después un proceso lo publica a Kafka, y D8 lo consume.

**Flujo completo:** El Admin carga un archivo → Company API lo recibe → Spring Batch lo procesa en grupos → todo queda en Aurora PostgreSQL → el Outbox avisa a D8. Cuando D7 necesita saber quién cobrar → llama a Employee Ref API → que llama a D6 para los datos completos en tiempo real.

---

## D4 — Consistencia + dualidad de canal

Son dos ideas pegadas en un solo bloque:

**Saga coreografiada — consistencia**
Una transferencia no es una sola operación. Son varios pasos encadenados: debitar la cuenta origen → enviar el dinero → confirmar que llegó. Si el paso 2 falla después de que el paso 1 ya ocurrió, el dinero desapareció del origen pero nunca llegó al destino. La Saga resuelve eso: si algo falla en cualquier punto, el sistema reacciona automáticamente con una compensación — devuelve el débito, revierte lo necesario. Sin intervención manual, sin fondos perdidos.

"Coreografiada" significa que no hay un coordinador central — cada dominio escucha los eventos y reacciona por su cuenta.

**LiquidationRouter — dualidad de canal**
D4 decide en tiempo real por qué camino va cada transferencia:
- Banco filial como destino → liquidación inmediata. El dinero llega en menos de 500 ms.
- Banco no filial o internacional → va a ACH. Puede tardar horas — es diferida.

¿Cómo sabe? Consulta el registro de bancos filiales cacheado en Redis con TTL de 5 minutos.

**El campo `settlement_type`**
Cada respuesta HTTP de D4 incluye ese campo: `IMMEDIATE` o `DEFERRED`. El usuario sabe si el dinero llega en segundos o en horas — sin adivinar.

---

## D4 — Resiliencia ACH

ACH es un sistema externo — no lo controlamos. Es el sistema de compensación interbancaria. Cuando se cae, el sistema no puede caerse con él.

**Transferencias entre filiales cuando ACH cae:** nada. Esas transferencias nunca tocaron ACH — van por canal inmediato directo. Son caminos completamente separados. El Circuit Breaker en D6 detecta que ACH está caído en menos de 5 segundos y deja de intentar enviarle cosas.

**Transferencias diferidas que estaban en cola:** quedan en estado `SENT_TO_ACH` esperando en Kafka. Cuando ACH se recupera, se reenvían. El idempotency key evita duplicados — cada transferencia tiene un ID único que se manda a ACH. Si ACH ya procesó esa ID, la ignora. Da igual cuántas veces se reenvíe.

**Dead Letter Queue (DLQ):** cuando un mensaje no se puede procesar después de varios intentos, va a una cola especial de mensajes fallidos. Ahí queda guardado con alerta. No se pierde silenciosamente.

**Job de reconciliación diario:** revisa si hay transacciones que llevan demasiado tiempo en estado `SENT_TO_ACH` sin respuesta. Si las encuentra, las re-encola o genera una alerta. Ninguna transacción queda zombi para siempre.

---

## D4 — Antifraude en línea

Antes de mover un solo peso, D4 pregunta: ¿esta cuenta está en alguna lista de control de fraude? Esa pregunta tiene que responderse en milisegundos, por eso las listas viven en Redis — memoria pura, ultra-rápido.

**Las tres listas:**
- Lista blanca — cuentas de confianza verificada. Flujo normal.
- Lista gris — cuentas sospechosas pero no bloqueadas. La transferencia pasa, pero se marca con un flag y se avisa a D8.
- Lista negra — cuentas bloqueadas. Rechazada inmediatamente.

**TTL de 60 segundos:** las listas cambian constantemente. El TTL significa que la copia en Redis expira cada minuto y se refresca. El peor caso es que una cuenta recién bloqueada tarde 60 segundos en reflejarse en D4.

**¿Cómo se actualiza cuando D8 detecta algo nuevo?** D8 detecta patrón sospechoso → publica `FraudListUpdated` en Kafka → D4 lo consume → invalida la caché de Redis inmediatamente, sin esperar el TTL. Propagación en menos de 60 segundos.

**P99 < 200 ms:** el 99% de las evaluaciones — incluyendo las más lentas — responden en menos de 200 milisegundos. Como las listas están en Redis local, en la práctica es mucho más rápido.

---

## D4 — Disponibilidad

**P95 end-to-end < 2 segundos:** el 95% de las transferencias responden en menos de 2 segundos desde que el usuario toca "enviar" hasta que recibe confirmación en pantalla. Incluye todo: red, validaciones, fraude, saga, base de datos.

**99.9% de uptime mensual:** D4 puede estar caído máximo 43 minutos al mes. Se logra con réplicas del servicio en múltiples zonas de disponibilidad — si una zona falla, las otras siguen atendiendo. Kubernetes detecta pods caídos y los reinicia en menos de 30 segundos.

**HPA (Horizontal Pod Autoscaler):** los días 14-16 y 29-31 de cada mes, D7 dispara 20.000-30.000 transferencias en ventanas cortas. Sin HPA, D4 se saturaría. Con HPA, Kubernetes detecta que la carga subió y levanta pods adicionales en menos de 60 segundos para absorber el pico.

---

## D4 — El pipeline y los dos caminos del Saga Coordinator

Imagina que cada transferencia pasa por 5 ventanillas en orden:

1. **Transfer API** — puerta de entrada. Valida JWT, consulta D2 para confirmar que la cuenta origen existe y tiene saldo.
2. **FraudChecker** — pregunta a Redis: ¿alguna cuenta está en lista negra o gris? Negra → rechazada. Gris → pasa con flag. Limpia → flujo normal.
3. **LiquidationRouter** — decide el canal. Banco filial → IMMEDIATE. No filial → SENT_TO_ACH. Esta es la ventanilla que diferencia los dos caminos — ANTES de llegar al Saga Coordinator.
4. **Saga Coordinator** — coordina los pasos reales del movimiento de dinero. Ejecuta lo que el LiquidationRouter le dijo.
5. **Transfer State Store (PostgreSQL on-premise)** — guarda el estado con garantía ACID. Está en Colombia por mandato regulatorio.

**¿Por qué del Saga Coordinator salen dos caminos?**

Porque el Saga Coordinator hace SIEMPRE dos cosas al mismo tiempo:

```
Saga Coordinator
    │
    ├──► PostgreSQL  (SIEMPRE — guarda el estado de toda transferencia)
    │
    └──► Kafka → D6 → ACH  (SOLO si es no filial)
```

Para filiales: solo va a PostgreSQL, se resuelve directo, fin.
Para no filiales: va a PostgreSQL Y a Kafka, espera que ACH responda.

Después del Store, los eventos de Kafka los consumen:
- **D8** — guarda todo en el histórico de auditoría
- **D6** — los que son para ACH los envía al sistema externo
- **D5** — si el destino es una billetera digital, D5 acredita el saldo

---

## Kafka / Amazon MSK — qué es

Kafka es un sistema de mensajería asíncrona. Es como una bandeja de entrada gigante y ordenada donde los dominios dejan mensajes, y otros dominios los recogen cuando pueden.

**El problema que resuelve:** sin Kafka, si D4 termina una transferencia y quiere avisarle a D8, tendría que llamarlo directamente. Si D8 está caído, D4 falla también — dos dominios acoplados. Con Kafka, D4 publica el evento y se olvida. D8 lo recoge cuando pueda. Si estaba caído, cuando vuelve los mensajes siguen ahí esperando.

**La analogía:** Kafka es como un tablero de anuncios con memoria. D4 pega un anuncio: "transferencia X completada". D8 pasa más tarde y lo procesa. D6 también pasa, ve que hay algo para ACH, y lo envía. Cada uno a su ritmo.

**Amazon MSK** es simplemente Kafka administrado por Amazon — ellos se encargan de mantenerlo funcionando, escalarlo, hacer backups. Nosotros solo lo usamos.

**En el contexto de D4:** D4 publica `TransferSettled` → D8 lo guarda. `TransferSentToACH` → D6 lo manda a ACH. `TransferSettled` (si billetera es destino) → D5 acredita el saldo. Nadie se llama directamente. Todo pasa por Kafka. Si cualquier dominio se cae y vuelve, retoma desde donde quedó sin perder nada.

---
