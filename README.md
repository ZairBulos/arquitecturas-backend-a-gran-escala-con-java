# Arquitecturas Backend a Gran Escala con Java - Código Facilito

## Proyecto

Sistema de Reservaciones de Alta Concurrencia

### Descripción

- Sistema en línea de venta de tickets para eventos de gran escala
- Soporta diferentes venues, tipos de evento, fechas y artistas
- 50M usuarios diarios activos (DAU)
- Debe soportar apertura de venta en eventos masivos (~5M usuarios concurrentes)
- Reto principal: No debe haber overbooking.

### Scope

- Ver eventos
- Buscar eventos
- Fila Virtual
- Reservar (10 minutos) y comprar tickets de un evento

## Entregables

1. **[Diagrama de Arquitectura (System Design):](/01/README.md)**
   Componentes externos, componentes internos, flujo principal, seguridad, interacción, tipo de comunicación
2. **[Diagrama de descomposición de servicios:](/02/README.md)**
   DDD bounded contexts y context map
3. **[API Spec:](/03/README.md)**
   Endpoints, métodos, request/response, data
4. **[Data Model:](/04/README.md)**
   Entidades principales, datos y tipos
5. **[Tech Stack](/05/README.md)**
6. **Cuestionario de evaluación**
7. **[Video de explicación de su diseño]()**
