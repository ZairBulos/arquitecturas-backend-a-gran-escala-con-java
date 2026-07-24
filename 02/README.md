# Diagrama de Descomposición de Servicios

![Context Map](/assets/02-context-map.png)

## Bounded Contexts

| Bounded Context  | Clasificación DDD     | Entidades principales                 |
| ---------------- | --------------------- | ------------------------------------- |
| Booking BC       | Dominio core          | Reserva, Hold, Orden, Ticket, Bloqueo |
| Event BC         | Subdominio de soporte | Evento, Recinto, Sección, Asiento     |
| Search BC        | Subdominio de soporte | Solo lectura (modelo de búsqueda)     |
| Queue BC         | Subdominio de soporte | Turno, Token                          |
| Notification BC  | Subdominio de soporte | Correo y SMS                          |
| Payment provider | Partner externo       | Pago externo                          |

## Relaciones (Context Map)

| Relación                       | Patrón                 | Descripción                                                                                                                                                                    |
| ------------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Event BC -> Booking BC         | **U->D + ACL**         | Event es upstream del catálogo de referencia. Booking, al ser el dominio core, protege su modelo con una capa anticorrupción.                                                  |
| Booking BC -> Event BC         | **Published Language** | Booking publica el estado del asiento vía CDC; Event lo consume para mantener actualizada su vista de lectura del mapa de asientos.                                            |
| Event BC -> Search BC          | **Published Language** | Event publica cambios de catálogo; Search los consume para actualizar su índice.                                                                                               |
| Queue BC -> Booking BC         | **U->D**               | Queue es upstream: emite el token de admisión que ordena el acceso al flujo de reserva durante eventos de alta demanda. Booking depende de ese contrato para admitir usuarios. |
| Booking BC -> Payment provider | **Conformist**         | Booking adopta el modelo del proveedor de pagos externo tal como lo expone, sin una capa de traducción adicional.                                                              |
| Booking BC -> Notification BC  | **Published Language** | Booking publica eventos de negocio (reserva confirmada, hold expirado); Notification los consume para disparar comunicaciones.                                                 |

## Leyenda

- **U->D (Upstream → Downstream, incl. ACL)**: El contexto downstream depende del modelo del upstream; puede incluir una capa anticorrupción cuando el downstream necesita protegerse.
- **Conformist:** El contexto downstream adopta el modelo del upstream sin traducirlo.
- **Published Language (Kafka):** Integración asíncrona vía eventos con un contrato de mensaje explícito.
