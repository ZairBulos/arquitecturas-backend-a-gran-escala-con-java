# System Design

## Diagrama de Arquitectura

![Diagrama de Arquitectura](/assets/01-system-design-architecture-diagram.png)

### 1. Edge

El tráfico del cliente pasa por tres capas antes de llegar a cualquier servicio de negocio:

- **CDN:** entrega assets estáticos (imágenes, JavaScript y CSS) y contenido cacheable, reduciendo la carga sobre la infraestructura interna.
- **WAF:** bloquea tráfico malicioso (SQL injection, XSS y bots) antes de que alcance componentes internos.
- **Load Balancer:** distribuye las solicitudes entre múltiples instancias del API Gateway, permitiendo escalado horizontal y alta disponibilidad.

### 2. Gateway

El **API Gateway** es el punto de entrada único para todas las solicitudes REST síncronas.

- **Autenticación (JWT):** valida la identidad del usuario en cada request.
- **Rate limiting:** protege a los servicios durante lanzamientos de alta demanda, donde pueden aparecer bots intentando acaparar asientos.
- **Routing:** distribuye hacia Search Service, Event Service, Virtual Waiting Queue Service o Booking Service según el endpoint solicitado.

### 3. Servicios

#### 3.1 Search service

Permite buscar eventos por texto libre (nombre del evento, artista/banda o venue), fecha y ubicación. Utiliza un modelo de lectura desnormalizado, actualizado de forma asíncrona.

<mark>AP (Availability + Partition tolerance)</mark>: La consistencia eventual es aceptable para resultados de búsqueda. Un resultado de búsqueda desactualizado por unos segundos no genera daño real.

#### 3.2 Event service

Fuente de verdad del catálogo: evento, recinto, sección, asiento. Expone el mapa de asientos.

<mark>AP (Availability + Partition tolerance)</mark>: Servir el catálogo de un evento levemente desactualizado (ej. un cambio de horario que tarda unos segundos en propagarse) no es crítico. Prioriza estar disponible durante los picos de lectura masivos.

#### 3.3 Virtual waiting queue service

Gate de admisión al flujo de reserva para eventos de alta demanda. El cliente lo consulta a través del API Gateway para obtener su turno y su token de admisión; Booking Service es el único servicio interno que depende de su contrato, validando ese token en cada solicitud de reserva.

<mark>AP (Availability + Partition tolerance)</mark>: Es preferible que la fila siga funcionando (aunque el orden exacto de admisión tenga una pequeña imprecisión bajo carga extrema) a que se caiga y bloquee la entrada de todos los usuarios al flujo de compra.

#### 3.4 Booking service

Dominio core del sistema: hold temporal (10 min), confirmación de reserva y pago. Es la única fuente de verdad del estado del asiento (available, held, booked).

Evitar que dos usuarios reserven el mismo asiento es el requerimiento más crítico del sistema. Se resuelve con dos capas de protección independientes, no una sola:

1. Hold temporal - bloqueo atómico en Redis:

   Al seleccionar asientos, Booking Service ejecuta, por cada asiento, una operación atómica tipo:

   ```sql
   SET seat:{eventId}:{seatId} {userId} NX EX 600
   ```

   - `NX` ("not exists"): la escritura solo tiene éxito si nadie más tiene el asiento bloqueado. Si dos usuarios seleccionan el mismo asiento en simultáneo, Redis garantiza que solo uno gane — es una operación atómica de un único comando, no hay ventana de carrera entre "leer" y "escribir".
   - `EX 600`: TTL de 10 minutos. Si el usuario no completa el pago, Redis libera el asiento automáticamente, sin necesidad de un proceso de limpieza adicional.
   - Si el `SET` falla (el asiento ya estaba tomado), Booking Service responde `409 Conflict` de inmediato — rápido, barato, y evita que la mayoría de los conflictos lleguen siquiera a la base de datos transaccional.

2. Confirmación de pago - bloqueo optimista en PostgreSQL:

   El hold en Redis resuelve el caso común, pero no es la garantía final: existen casos límite (el TTL expira justo cuando el pago se está confirmando, reintentos de red, o un eventual bug en la capa de Redis) donde dos confirmaciones podrían llegar a competir por el mismo asiento. Por eso la confirmación real ocurre con un UPDATE condicional en PostgreSQL:

   ```sql
   UPDATE tickets
   SET
    status = 'booked',
    version = version + 1
   WHERE
    ticket_id = ?
    AND status = 'held'
    AND version = ?;
   ```

   - Si la fila cambió entre que se leyó y se intentó confirmar (por ejemplo, otro proceso ya la liberó o ya la vendió), el WHERE no matchea, el UPDATE afecta 0 filas, y Booking Service detecta el conflicto — nunca hay una venta duplicada comprometida en la base de datos.
   - Si afecta exactamente 1 fila, la transacción ACID de PostgreSQL garantiza que la confirmación es atómica e irreversible.

<mark>CP (Consistency + Partition tolerance)</mark>: Una reserva duplicada es un error de negocio grave e irreversible (dos personas creen que tienen el mismo asiento). Ante una partición de red, el sistema prefiere rechazar la operación antes que arriesgar la duplicación.

#### 3.5 Notification service

Envío de confirmaciones y alertas (email/SMS) tras eventos de negocio (reserva confirmada, hold expirado). No expone API síncrona - es un consumidor puro de eventos.

Dado que una reserva puede incluir varios asientos, Notification Service agrupa los eventos seat.status.changed correspondientes a una misma reserva antes de notificar, en vez de disparar un mensaje por cada asiento - el usuario recibe una única confirmación por compra, no una por butaca.

<mark>AP (Availability + Partition tolerance)</mark>: Es best-effort por naturaleza, una notificación demorada o reintentada no afecta la integridad de ninguna reserva. Nunca debe bloquear al resto del sistema.

### 4. Mensajería

El sistema utiliza Kafka como bus de eventos asíncrono con dos flujos diferenciados.

1. `seat.status.changed` - vía Debezium (CDC) sobre el WAL de PostgreSQL de Booking Service.
   - Booking Service nunca publica manualmente: Debezium captura cada cambio de estado directamente desde el log de la base de datos, garantizando que ningún evento se pierda.
   - Se usa CDC (en vez de publicación directa) porque Booking es el dominio core y su estado no puede perderse ni desincronizarse.
   - Consumidores: Event Service (actualiza su vista del mapa de asientos) y Notification Service (agrupa los eventos de una misma reserva y dispara una única confirmación).

2. `event.catalog.changed` - publicado directamente por Event Service tras escribir en MongoDB (sin CDC).
   - Es aceptable usar publicación directa (con el riesgo de dual-write que eso implica) porque se acepta consistencia eventual para búsqueda - no se justifica la complejidad adicional de CDC sobre MongoDB para este caso.
   - Consumidor: Search Service, que reindexa su modelo de lectura en Elasticsearch.

### 5. Bases de Datos

| Servicio | Base de Datos     | Justificación                                                                                                |
| -------- | ----------------- | ------------------------------------------------------------------------------------------------------------ |
| Search   | **Elasticsearch** | Motor de búsqueda full-text, óptimo para el patrón de lectura extremo (100:1) y consultas por palabra clave. |
| Event    | **MongoDB**       | Esquema flexible: soporta distintos tipos de evento, venues, fechas y artistas sin migraciones rígidas.      |
| Queue    | **Redis**         | Posición en fila y token de admisión, con TTL corto - baja latencia para consultas frecuentes de estado.     |
| Booking  | **Redis**         | Hold temporal de 10 minutos vía TTL nativo - libera asientos automáticamente sin lógica adicional.           |
| Booking  | **PostgreSQL**    | Transacciones ACID para la confirmación de reserva y pago.                                                   |

### 6. Integraciones Externas

- **Payment Gateway:** proveedor de pagos externo (ej. Stripe), llamado de forma síncrona por Booking Service durante la confirmación.
- **Email / SMS:** proveedor externo de comunicaciones, consumido de forma asíncrona por Notification Service.

### 7. Seguridad

- **HTTPS/TLS** en todos los saltos desde el cliente.
- **WAF** como primera línea de defensa contra tráfico malicioso.
- **Autenticación (JWT)** y **rate limiting** en el API Gateway, este último especialmente relevante para prevenir bots durante lanzamientos de alta demanda.

### 8. Tipo de Comunicación

- **Síncrona (REST):** Cliente <-> API Gateway <-> Search/Event/Queue/Booking Service, y Booking <-> Payment Gateway.
- **Asíncrona (Kafka):** Booking -> Event/Notification (vía CDC) y Event -> Search (publish directo).

### 9. Coordinación transaccional (Saga)

El flujo principal (reserva y pago) no necesita Saga. El ticket, booking y pago viven en la misma base de datos (PostgreSQL de Booking Service): la confirmación es una única transacción ACID local, no una transacción distribuida. La llamada a Payment Gateway es Conformist y se resuelve con manejo directo de error (402), sin pasos de compensación multi-servicio.

Esto es una consecuencia directa de la decisión de que Booking Service sea la única fuente de verdad del estado del asiento. Si ese estado estuviera repartido entre Event BC (MongoDB) y Booking BC (PostgreSQL) - por ejemplo, si Event BC reservara el asiento y Booking BC confirmara el pago por separado - sí habría hecho falta un Saga para coordinar ambas escrituras con compensación ante fallos. Centralizar el estado evitó ese problema por completo.

---

## Diagrama de Flujo Principal

![Diagrama de Flujo Principal](/assets/01-system-design-flow-diagram.png)

Dado que se trata de un sistema de gran escala con múltiples dominios y flujos operativos, nos centraremos exclusivamente en el flujo de reserva del cliente.
