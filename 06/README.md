# Cuestionario de Evaluación

## ¿Cómo está protegido tu sistema contra overbooking?

Lo resuelvo con dos capas de protección, no una sola. Con Booking Service como única fuente de verdad del estado del asiento.

### Capa 1 - Hold temporal (Redis)

- `SET seat:{eventId}:{seatId} {userId} NX EX 600`
- `NX` = atómico, un único comando: si dos usuarios seleccionan el mismo asiento en simultáneo, solo uno gana. No hay ventana de carrera entre leer y escribir.
- `EX 600` = TTL de 10 min, libera el asiento automáticamente sin proceso de limpieza.
- Si falla -> 409 Conflict inmediato, barato, ni siquiera toca la base transaccional.

### Capa 2 - Bloqueo optimista (PostgreSQL)

```sql
UPDATE tickets
SET status='booked', version=version+1
WHERE
  ticket_id=?
  AND status='held'
  AND version=?;
```

- Cubre el caso límite que Redis no: el TTL expira justo cuando se confirma, reintentos de red, etc.
- Si 0 filas afectadas -> conflicto detectado, nunca se compromete una venta duplicada.
- Si 1 fila afectada -> transacción ACID, atómica e irreversible.

## ¿Dónde integrás resiliencia, y de qué forma?

- **Rate limiting en API Gateway:** frena bots que intentan acaparar asientos apenas se abre la venta.
- **Circuit breaker en la llamada a la pasarela de pago:** si el proveedor externo empieza a fallar, corto la llamada en vez de dejar que eso tumbe todo Booking Service.
- **Idempotency key en la confirmación de pago:** si el cliente reintenta por un timeout de red, no se cobra ni se reserva dos veces.
- **TTL automático en el hold de Redis:** el asiento se libera solo, sin depender de que nadie avise que se arrepintió.
- **CDC con Debezium en vez de publish manual:** Booking nunca escribe a mano en Kafka; si el publish manual fallara justo después del commit, se perdería el evento. Con CDC eso no puede pasar, se lee directo del log de la base.
- **Kafka como buffer durable:** si Event, Notification o Search se caen, los eventos quedan guardados y se procesan al volver, no se pierden.
- **Servicios de soporte diseñados en AP (Event, Search, Notification):** siguen funcionando aunque Booking esté degradado, no dependen de él.
- **Virtual Waiting Queue:** absorbe la avalancha de gente que quiere entrar, así ese tráfico no le pega directo a Booking.

La resiliencia no es una capa única para todo el sistema, es una decisión distinta por cada servicio, según qué tan grave sería que ese servicio falle.

## ¿Dónde aplica concurrencia en tu sistema?

- **`SET NX` en Redis:** resuelve la exclusión mutua en el momento más caliente del sistema, la selección de asiento, sin necesidad de un lock distribuido más pesado.
- **Bloqueo optimista en PostgreSQL:** en vez de bloquear la fila mientras alguien la lee (bloqueo pesimista), dejo que todos lean libremente y solo reviso si algo cambió al momento de escribir - mejor rendimiento bajo carga alta.
- **Virtual Waiting Queue:** controla cuántas personas entran al flujo de reserva por vez, reduciendo la concurrencia real que le llega a Booking en vez de intentar resolverla toda a nivel de datos.

La concurrencia más peligrosa del sistema está concentrada a propósito en un solo punto (el hold del asiento), para poder resolverla con algo simple y atómico en vez de coordinar varios servicios al mismo tiempo.

## ¿Cómo manejás la llegada masiva de usuarios en un evento muy popular?

1. **CDN:** sirve assets estáticos, ni siquiera llega a tu infraestructura.
2. **WAF:** filtra bots antes del balanceador.
3. **Load Balancer + autoscaling horizontal:** instancias de API Gateway y servicios stateless (Search, Event) escalan elásticamente con la demanda.
4. **Rate limiting en API Gateway:** throttling contra bots que intentan acaparar asientos.
5. **Virtual Waiting Queue:** admite usuarios de forma ordenada al flujo de reserva; es el gate que evita que 5M de personas le peguen a Booking al mismo tiempo.
6. **Redis para el hold:** baja latencia y alto throughput para la operación más caliente del sistema (miles de `SET NX` por segundo).
7. **Kafka como buffer asíncrono:** absorbe el pico de eventos (`seat.status.changed`) sin exigirle a Event/Search/Notification que procesen en tiempo real; se puede procesar todo apenas baje la carga.
8. **Booking Service se mantiene deliberadamente angosto (CP):** todo lo que puede escalar elásticamente (AP) lo hace; lo único que no escala así es, a propósito, el único punto donde no se puede dar el lujo de estar mal (la escritura del estado del asiento).

## Análisis desde la perspectiva del CAP Theorem

En un sistema distribuido las particiones de red van a pasar sí o sí, elijo consistencia o disponibilidad, y esa decisión se toma servicio por servicio, no para todo el sistema por igual.

- **Search Service = AP:** un resultado de búsqueda un poco desactualizado tampoco es grave.
- **Event Service = AP:** un catálogo levemente desactualizado por unos segundos no genera ningún daño real.
- **Virtual Waiting Queue = AP:** prefiero que la fila siga funcionando con una pequeña imprecisión de orden bajo carga extrema, a que se caiga y bloquee la entrada de todos.
- **Booking Service = CP:** una reserva duplicada es un error grave e irreversible (dos personas creen que tienen el mismo asiento). Ante una duda de consistencia, prefiero rechazar la operación.
- **Notification Service = AP:** una notificación que llega tarde no compromete ninguna reserva.

El único lugar donde sacrifico disponibilidad es el dominio principal del sistema (Booking), porque ahí un dato inconsistente se traduce directo en dinero perdido o en una experiencia irreversible para el usuario. Todo lo demás, que es la mayor parte del tráfico, porque se lee mucho más de lo que se escribe, prioriza seguir disponible.
