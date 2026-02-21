Introducción a las arquitecturas evolutivas
Universidad EAFIT
Medellín – Colombia
2026-1
Profesor: Carlos Orozco.
Fecha: 21/02/2026
Trabajo final:
Definición de una arquitectura evolutiva de alto nivel.
La empresa X los ha contratado con el propósito de proveer una solución dado el siguiente
conjunto de requisitos de alto nivel:
Contexto: Actualmente, Colombia tiene más de 60 instituciones financieras con una cantidad
aproximada de 35 millones de usuarios activos, cada una con un modelo de negocio establecido.
Sin embargo, de acuerdo con los lineamientos de cada organización, realizar transferencias entre
bancos toma aproximadamente entre 1 y 5 días hábiles. Con el objetivo de facilitar el proceso de
débito y acreditación, la empresa X decidió llevar a cabo la implementación de un sistema que
permita centralizar el proceso de transacción entre diferentes bancos. Para ello, la empresa X ha
planteado construir una aplicación multiplataforma (web, móvil híbrido, tableta) con las
siguientes características:
1. Permitir que un usuario (persona natural o jurídica) conozca el estado de las cuentas que
tenga registradas en diferentes bancos que se encuentren registrados como filiales de la
empresa X.
2. Como persona natural:
a. Con una cuenta particular: Recibir dinero (acreditar) y realizar transferencias
(debitar).
i. A cuentas registradas en el mismo banco.
ii. A cuentas registradas en otros bancos que se encuentren registrados como
filiales en la empresa X.
iii. A cuentas que no se encuentran registradas como filiales en la empresa X.
iv. A cuentas internacionales.
Las transacciones entre bancos que se encuentran registrados en el sistema se ven
reflejadas en el estado de cuenta del usuario de forma inmediata. Por otro lado, las
transferencias entre con bancos que no se encuentran registrados e internacionales se
deben enviar para verificación en un sistema externo (ACH), quien se encarga de
determinar si la transacción puede proceder o es rechazada, una vez se determina el
estado de la transacción, el sistema registra el movimiento.
b. Billetera: Cada usuario debe contar con una “billetera” que actúa como cuenta
emitida por la empresa X y permite acreditar y/o debitar dinero como si fuera una
cuenta particular.
i. La billetera permite mover montos de dinero entre las cuentas activas del
usuario, cuentas externas y/o cuentas internacionales (aplican las mismas
reglas establecidas para una persona natural).
ii. La billetera tiene la capacidad de realizar pagos a terceros.
Un tercero se define como un API’s o servicio expuesto por sistemas externos que
se encuentran asociados con la empresa. Actualmente, se busca que la empresa X
establezca alianzas con empresas para el pago de servicios públicos, facturas,
impuestos, servicios de transporte entre otros. En este sentido, el sistema debe
ser capaz de establecer mecanismos para integrar canales de comunicación dichos
sistemas de forma transparente, por lo cual, se espera que se puedan integrar
terceros a demanda sin afectar el estado del sistema.
iii. Los pagos a terceros se deben realizar utilizando pasarelas de pago (PSE,
DRUO, Apple pay, entre otros). El sistema debe facilitar la integración con
este tipo de sistemas externos.
3. Como empresa:
a. Habilitar mecanismos que permitan realizar pago a empleados de forma
individual.
b. habilitar un mecanismo que permita realizar el pago automático a todos los
empleados que tenga asociados y se encuentren activos en la organización en
fechas específicas.
4. El sistema debe mantener el histórico de todas las transacciones hechas por personas
naturales o empresas.
De acuerdo con el análisis realizado por el equipo de análisis funcional se identificaron los
siguientes aspectos que se deben considerar:
1. Actualmente, la empresa X ya se encuentra aliada con los 4 bancos más grandes del país,
por lo que se espera un aproximado de 25 millones de usuarios activos.
a. El registro de los usuarios será llevado a cabo mediante procesos de carga masiva
programados en los cuales cada banco proveerá la información necesaria para
caracterizar a un usuario en el sistema.
b. El sistema debe realizar procesos de sincronización diarios en los cuales se
actualice la información de los clientes en el sistema.
2. Debido a restricciones jurídicas, el sistema debe enviar un extracto de todos los
movimientos realizados por cada usuario al banco (o bancos) a los cuales se encuentre
adscrito cada tres meses.
3. El sistema debe enviar un reporte a la superintendencia financiera con el detalle de todos
los movimientos realizados por los usuarios del sistema cada seis meses.
4. La empresa X cuenta con un total de 15 empresas aliadas que están dispuestas a usar la
plataforma para realizar el pago a sus empleados. Se espera un total aproximado de 35
mil empleados activos.
a. De manera análoga al caso de los bancos, se realizará un proceso de carga masiva
para registrar solamente la información de las empresas.
b. Por cuestiones de seguridad, la plataforma solo guardará la información básica
que permita obtener la información de un empleado en cada empresa, por lo que
se espera que los procesos de pago se realicen mediante llamados a servicios
expuestos por cada empresa.
c. La plataforma debe mantener la trazabilidad de los pagos realizados a cada
empleado.
d. De acuerdo con el análisis de la dinámica de cada empresa, se esperan pagos
masivos (entre 20 mil y 30 mil transacciones) en fechas específicas.
i. A mitad de mes: Días 14, 15 y 16 de cada mes.
ii. A fin de mes: 29, 30 o 31 de cada mes.
No obstante, se espera que el sistema sea completamente capaz de realizar pagos
masivos en cualquier día del mes.
e. La empresa puede realizar pagos programados o manuales.
f. La empresa puede hacer pagos individuales o masivos.
5. Debido a restricciones estatales, el sistema debe contar con mecanismos de seguridad
que garanticen los siguientes aspectos:
a. Políticas de autenticación y autorización de usuarios en el sistema.
b. Políticas de acceso restringido.
c. Detección de intrusiones.
d. Firmado y cifrado durante todo el ciclo de vida de la información (creación,
manipulación y persistencia).
e. Establecer canales de comunicación seguros en el sistema y con terceros.
f. Monitoreo de registros de actividad.
g. Monitoreo de transacciones sospechosas.
i. Listas blancas.
ii. Listas grises.
iii. Listas negras.
h. Conformidad con los ítems propuestos en el estándar OWASP.
6. Como el sistema debe ser multiplataforma, debe contar con una interfaz de usuario
agradable al usuario que pueda ser utilizado a través de un portal web, smartphones y
tablets de gama media/alta con un tamaño mínimo de seis pulgadas.
7. El sistema debe contar con disponibilidad 24/7.
8. Se espera que el tiempo de respuesta del sistema sea menor a dos segundos.