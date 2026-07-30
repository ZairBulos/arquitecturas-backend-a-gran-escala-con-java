# Cuestionario de Evaluación

Cuestionario de Evaluación

## 1. Dado que el sistema de reservaciones puede recibir una demanda muy alta de usuarios simultáneamente, ¿de qué manera garantizarás que dos usuarios no puedan reservar el mismo asiento?

Garantizo que dos usuarios no puedan reservar el mismo asiento mediante dos capas de protección, siendo Booking Service la única fuente de verdad sobre el estado del asiento.

### Capa 1: Bloqueo atómico en Redis

Cuando un usuario selecciona un asiento, utilizo en Redis el comando atómico `SET seat:{eventId}:{seatId} {userId} NX EX 600`. Si dos usuarios intentan seleccionar el mismo asiento al mismo tiempo, únicamente uno de ellos logra crear la clave.

Al tratarse de una operación atómica, elimino la condición de carrera que existiría entre comprobar si el asiento está disponible y marcarlo como retenido.

### Capa 2: Bloqueo optimista en PostgreSQL

Como garantía final, al confirmar el pago ejecuto un `UPDATE` que solo se aplica si el ticket continúa en estado `held` y su versión no ha cambiado desde que fue leída. Si otro proceso modifica el registro mientras tanto, el `UPDATE` no afecta ninguna fila, lo que me permite detectar el conflicto antes de comprometer la venta.

## 2. ¿Cómo funciona exactamente la reservación temporal de 10 minutos que implementaste en tu solución? Recuerda que se debe garantizar que, pasados exactamente 10 minutos, el asiento vuelva a estar disponible y que el usuario que lo tenía ya no pueda reservarlo.

Al crear el hold, el asiento se marca como retenido en Redis con un TTL de 600 segundos (`SET seat:{eventId}:{seatId} {userId} NX EX 600`). Cuando se cumple ese tiempo, Redis elimina automáticamente la clave y el asiento queda disponible para otros usuarios, sin necesidad de ningún proceso adicional de limpieza.

## 3. ¿Qué sucede si el usuario paga exactamente cuando vence su reservación?

Si el bloqueo del usuario caduca en el minuto 10, pero su pago se completa luego, otro usuario podría haber obtenido el bloqueo entretanto. En este caso excepcional, la transacción de la base de datos fallará para uno de ellos, ya que el bloqueo optimista garantiza que solo se realice una escritura con éxito, y se debera emitir un reembolso automático por la reserva fallida.

## 4. ¿Cómo garantizas que las solicitudes duplicadas no generen dos o más compras?

Utilizo idempotencia en la operación de confirmación de compra. El cliente envía una `idempotency_key` única en cada intento de pago. Antes de procesar la solicitud, verifico si ya existe un resultado asociado a esa clave.

- Si es la primera vez que recibo esa `idempotency_key`, proceso el pago normalmente y almaceno el resultado asociado a esa clave.
- Si recibo nuevamente la misma `idempotency_key` (por ejemplo, porque el cliente reintentó la solicitud tras un problema de red o un timeout), no vuelvo a ejecutar la operación. En su lugar, devuelvo el resultado previamente almacenado, evitando un doble cobro y la creación de una segunda compra.

## 5. Explica los beneficios que has identificado al utilizar una "fila virtual" en este escenario con 5 millones de usuarios concurrentes.

- Evita que millones de usuarios intenten acceder a Booking Service al mismo tiempo, controlando el ingreso de manera ordenada.
- Permite que Booking Service mantenga su garantía de consistencia fuerte, ya que nunca recibe más carga de la que puede procesar de forma segura.
- Aísla el punto de mayor contención del sistema, protegiendo indirectamente a otros servicios de los efectos de una posible saturación en Booking Service.
- Mejora la experiencia del usuario bajo carga extrema: en lugar de recibir errores o tiempos de espera indefinidos, el usuario visualiza su posición en la fila y un tiempo estimado de espera.

## 6. ¿Qué características de resiliencia son importantes para este tipo de escenario y en qué parte de tu solución las aplicarías?

- **Rate limiting en API Gateway:** limita la cantidad de solicitudes por cliente para reducir el impacto de bots o usuarios que intentan acaparar asientos apenas se habilita la venta.
- **Circuit breaker en la integración con el proveedor de pagos:** si el proveedor externo comienza a fallar o responde con alta latencia, el circuito se abre y evita que esos problemas consuman recursos de Booking Service y afecten el resto del sistema.
- **Idempotency key en la confirmación de pago:** garantiza que múltiples reintentos de una misma solicitud produzcan un único resultado, evitando dobles cobros o reservas duplicadas.
- **TTL automático en el hold de Redis:** si el usuario abandona el proceso de compra o nunca completa el pago, el hold expira automáticamente y el asiento vuelve a estar disponible sin intervención manual.
- **CDC con Debezium en lugar de publicación manual en Kafka:** Booking Service solo confirma la transacción en la base de datos. Luego, Debezium captura ese cambio desde el log de transacciones y lo publica en Kafka.
- **Kafka como buffer durable:** si servicios como Event, Search o Notification dejan de estar disponibles temporalmente, los eventos permanecen almacenados en Kafka y se procesan cuando los consumidores se recuperan, sin pérdida de información.
- **Servicios de soporte desacoplados mediante eventos:** Event Service, Search Service y Notification Service consumen eventos de Kafka y no dependen de llamadas síncronas a Booking Service. Esto evita que una degradación en uno de ellos afecte el flujo crítico de reservas.
- **Virtual Waiting Queue:** absorbe los picos de tráfico cuando se abre la venta de entradas y controla la cantidad de usuarios que pueden acceder simultáneamente a Booking Service, evitando su saturación.

## 7. ¿En qué partes de tu solución utilizarías eventual consistency y en qué otras strong consistency?

Utilizo **strong consistency** en Booking Service, ya que es la única fuente de verdad sobre el estado de los asientos. Cualquier inconsistencia en este servicio podría provocar una doble reserva o confirmar una compra sobre un asiento que ya no está disponible.

En cambio, utilizo **eventual consistency** en Event Service, Search Service, Virtual Waiting Queue y Notification Service. En estos casos, que la información tarde unos segundos en propagarse no genera un problema crítico.

La regla que utilizo para decidir es: si una inconsistencia puede generar una operación incorrecta o una pérdida de dinero, utilizo strong consistency. Si el peor escenario es mostrar información temporalmente desactualizada, utilizo eventual consistency, ya que a cambio obtengo mayor disponibilidad y escalabilidad en los servicios con mayor volumen de lecturas.

## 8. ¿Qué sucede si alguno de tus componentes deja de responder? Caché, base de datos, sistema de mensajería, proveedor de pagos.

- **Si Redis deja de responder:**
  - Booking Service deja de aceptar nuevas reservas temporales para evitar inconsistencias.
  - Las compras de reservas ya confirmadas continúan siendo válidas porque la fuente de verdad es PostgreSQL.
- **Si Kafka deja de responder:**
  - La compra puede completarse correctamente porque la transacción crítica termina al persistir la información en PostgreSQL.
  - Con el patrón Transactional Outbox + Debezium (CDC), los eventos quedan registrados en la base de datos y serán publicados en Kafka cuando este vuelva a estar disponible.
  - Los servicios consumidores (Search, Event y Notification) verán la actualización con retraso, pero no se pierde información.
- **Si PostgreSQL deja de responder:**
  - Booking Service no puede confirmar reservas ni pagos, ya que es la base de datos transaccional.
  - Se devuelve un error temporal al cliente y este puede reintentar posteriormente.
- **Si el proveedor de pagos deja de responder o presenta alta latencia:**
  - El Circuit Breaker abre el circuito tras detectar múltiples fallos consecutivos.
  - El usuario recibe un mensaje indicando que el pago no pudo procesarse y puede reintentarlo posteriormente utilizando la misma idempotency_key, evitando dobles cobros.
