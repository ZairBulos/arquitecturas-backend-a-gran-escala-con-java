# Data Model

## Event BC

### Event

| Campo                     | Tipo       | Descripción               |
| ------------------------- | ---------- | ------------------------- |
| `_id`                     | ObjectId   | Identificador único       |
| `name`                    | string     | Nombre del evento         |
| `description`             | string     | Descripción del evento    |
| `date`                    | ISODate    | Fecha y hora del evento   |
| `type`                    | string     | concert, sports, theater  |
| `thumbnail_url`           | string     | URL de imagen             |
| `venue_id`                | ObjectId   | Referencia a Venue        |
| `performer_ids`           | ObjectId[] | Referencia a Performer(s) |
| `created_at`,`updated_at` | ISODate    | Auditoría                 |

### Venue

| Campo      | Tipo           | Descripción               |
| ---------- | -------------- | ------------------------- |
| `_id`      | ObjectId       | Identificador único       |
| `name`     | string         | Nombre del recinto        |
| `location` | object         | Ubicación                 |
| `capacity` | int            | Capacidad total           |
| `seat_map` | array embebido | Layout físico de asientos |

### Performer

| Campo      | Tipo     | Descripción              |
| ---------- | -------- | ------------------------ |
| `_id`      | ObjectId | Identificador único      |
| `name`     | string   | Nombre del artista/banda |
| `category` | string   | Género o categoría       |

### SeatAvailability

| Campo        | Tipo     | Descripción                                |
| ------------ | -------- | ------------------------------------------ |
| `event_id`   | ObjectId | Referencia al evento                       |
| `seat_id`    | string   | Referencia al asiento                      |
| `status`     | string   | Copia del estado real de `Ticket.status`   |
| `updated_at` | ISODate  | Marca cuándo llegó la última actualización |

## Search BC

### SearchableEvent

| Campo                | Tipo    | Descripción                             |
| -------------------- | ------- | --------------------------------------- |
| `event_id`           | keyword | Referencia a Event BC                   |
| `name`,`description` | text    | Campos full-text                        |
| `venue_name`,`city`  | text    | Para filtro ubicación                   |
| `date`               | date    | Filtro y orden                          |
| `type`               | keyword | Filtro por tipo                         |
| `performers`         | text[]  | Búsqueedas por artista                  |
| `thumbnail_url`      | keyword | No indexado para búsqueda, solo display |
| `indexed_at`         | date    | Detecta staleness                       |

## Queue BC

### QueueEntry

| Clave                               | Valor  | TTL          | Descripción                |
| ----------------------------------- | ------ | ------------ | -------------------------- |
| `queue:{eventId}:position:{userId}` | int    |              | Posición actual en la fila |
| `queue:{eventId}:token:{userId}`    | string | 300s (5 min) | Token de admisión          |

## Booking BC

### SeatLock

| Clave                     | Valor    | TTL           | Descripción                                              |
| ------------------------- | -------- | ------------- | -------------------------------------------------------- |
| `seat:{eventId}:{seatId}` | `userId` | 600s (10 min) | Libera el asiento automáticamente si no hay confirmación |

### Ticket

| Campo                              | Tipo                              | Descripción                       |
| ---------------------------------- | --------------------------------- | --------------------------------- |
| `id`                               | UUID (PK)                         | Identificador único               |
| `event_id`                         | UUID                              | Referencia a Event BC             |
| `seat_id`,`section`,`row`,`number` | string/int                        | Copia local del asiento           |
| `price`                            | decimal(10,2)                     | Precio al momento de la reserva   |
| `status`                           | enum(`available`,`held`,`booked`) | Estado del ticket                 |
| `user_id`                          | UUID                              | Usuario que tiene el hold/reserva |
| `version`                          | int                               | Bloqueo optimista                 |
| `held_at`,`bookead_at`             | timestamptz                       | Auditoría                         |

### Booking

| Campo                                    | Tipo                                              | Descripción              |
| ---------------------------------------- | ------------------------------------------------- | ------------------------ |
| `id`                                     | UUID (PK)                                         | Identificador único      |
| `user_id`                                | UUID                                              | Comprador                |
| `event_id`                               | UUID                                              | Referencia a Event BC    |
| `status`                                 | enum(`pending`,`confirmed`,`cancelled`,`expired`) | Estado de la reserva     |
| `idempotency_key`                        | string, unique                                    | Evita procesar dos veces |
| `created_at`,`expires_at`,`confirmed_at` | timestamptz                                       | Ciclo de vida del hold   |

### BookingTicket

| Campo        | Tipo      | Descripción          |
| ------------ | --------- | -------------------- |
| `booking_id` | UUID (FK) | Referencia a Booking |
| `ticket_id`  | UUID (FK) | Referencia a Ticket  |

### Payment

| Campo                | Tipo                                            | Descripción                     |
| -------------------- | ----------------------------------------------- | ------------------------------- |
| `id`                 | UUID (PK)                                       | Identificador único             |
| `booking_id`         | UUID                                            | Reserva asociada                |
| `provider_charge_id` | string                                          | ID del cargo en Payment gateway |
| `amount`,`currency`  | decimal/string                                  | Monto cobrado                   |
| `status`             | enum(`pending`,`succeeded`,`failed`,`refunded`) | Estado del pago                 |
| `created_at`         | timestamptz                                     | Auditoría                       |

## Notification BC

### Notification

| Campo      | Tipo                           | Descripción         |
| ---------- | ------------------------------ | ------------------- |
| `id`       | UUID                           | Identificador único |
| `user_id`  | UUID                           | Destinatario        |
| `channel`  | enum(`email`,`sms`)            | Canal de envío      |
| `template` | string                         | Plantilla usada     |
| `status`   | enum(`queued`,`sent`,`failed`) | Estado de envío     |
| `sent_at`  | timestamptz                    | Auditoría           |
