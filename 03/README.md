# API Spec

Cada servicio documenta su contrato completo en OpenAPI (YAML).

## Servicios

| Servicio                                    | Rol                                  | CAP | Auth |
| ------------------------------------------- | ------------------------------------ | --- | ---- |
| [Search service](/03/search-service.yaml)   | Búsqueda de eventos                  | AP  | No   |
| [Event service](/03/event-service.yaml)     | Catálogo y mapa de asientos          | AP  | Sí   |
| [Queue service](/03/queue-service.yaml)     | Gate de admisión al flujo de reserva | AP  | Sí   |
| [Booking service](/03/booking-service.yaml) | Dominio core — reserva, hold y pago  | CP  | Sí   |

## Endpoints

| Método | Path                                     | Servicio | Descripción                        |
| ------ | ---------------------------------------- | -------- | ---------------------------------- |
| GET    | `/search`                                | Search   | Buscar eventos                     |
| GET    | `/events/{eventId}`                      | Event    | Detalle de un evento               |
| GET    | `/events/{eventId}/seats`                | Event    | Mapa de asientos en tiempo real    |
| POST   | `/queue/{eventId}/join`                  | Queue    | Unirse a la fila virtual           |
| GET    | `/queue/{eventId}/status`                | Queue    | Consultar posición en la fila      |
| POST   | `/bookings/reservations`                 | Booking  | Reservar asientos (hold 10 min)    |
| DELETE | `/bookings/reservations/{reservationId}` | Booking  | Cancelar un hold                   |
| POST   | `/bookings`                              | Booking  | Completar pago y confirmar reserva |
| GET    | `/bookings/{bookingId}`                  | Booking  | Consultar estado de una reserva    |
