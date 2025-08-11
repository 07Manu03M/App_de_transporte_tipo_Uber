# Diseño reflexivo de endpoints — App de transporte tipo "CarroTurbo"

**Autor:** Manuel

---

1. **¿Qué es un endpoint?**
   Un endpoint es una URL (ruta) expuesta por la API que permite al cliente interactuar con un recurso o acción (ej. `POST /api/rides` crea una solicitud de viaje).

2. **Endpoint público vs privado**

   * **Público:** Es accesible sin autenticación (ej. documentación, registro).
   * **Privado:** Se requiere credenciales limitado según rol (ej. aceptar viaje sólo para conductores).

3. **Información confidencial de usuario**
   Contraseñas, tokens, datos de pago (números de tarjeta completos), número de documento completo y ubicación continua en tiempo real (log de ubicaciones). Estos no deben exponerse.

4. **Importancia de definir métodos HTTP**
   Define intención semántica (GET=leer, POST=crear, PUT/PATCH=actualizar, DELETE=borrar), ayuda a cachés, seguridad y mantiene la API RESTful coherente.

5. **Información que requiere autenticación**
   Perfil completo, crear/aceptar/cancelar viajes, actualizar ubicación, pagos, ver historial y calificaciones, endpoints administrativos.

6. **Seguridad de la ubicación**
   Transmitir via HTTPS, limitar frecuencia de actualización, solo exponer ubicación aproximada a terceros, anonimizar rutas históricas, y solicitar permiso del usuario.

7. **Si no hay conductores disponibles**
   La API responde 200 con `status: no_drivers` o 204 con cuerpo explicativo y `estimated_wait`/sugerencias; también puede encolar la solicitud o sugerir alternativas.

8. **Recursos principales**
   `users`, `drivers` (subtipo de users), `rides`, `payments`, `ratings`/`reviews`, `vehicles`, `sessions` (auth), `admin`.

9. **Ventajas de versionar la API**
   Permite cambios compatibles e incompatibles sin romper clientes existentes; facilita migraciones y pruebas (`/v1/...`, `/v2/...`).

10. **Documentar respuestas de error**
    Importante para que el cliente gestione fallos correctamente; evita comportamiento inesperado y facilita debugging.

---

## Nombre de la aplicacion y autor

 * CarroTurbo
 * Manuel :V

## Roles y permisos

* **Pasajero (passenger)**

  * Registrarse, editar su perfil, solicitar viajes, pagar, calificar conductor, ver historial.

* **Conductor (driver)**

  * Registrarse/activar perfil de conductor, actualizar disponibilidad y ubicación, aceptar/rechazar viajes, iniciar/terminar viajes, ver historial y recibir pagos.

* **Administrador (admin)**

  * Ver/modificar usuarios, ver todos los viajes y pagos, bloquear/unblock usuarios, generar reportes.

---

## Recursos principales

* `users` (incluye pasajeros y conductores)
* `drivers` (datos específicos de conductor/vehículo)
* `rides` (viajes)
* `payments` (transacciones)
* `reviews` / `ratings` (calificaciones y comentarios)
* `vehicles` (info de vehículos)
* `auth` / `sessions` (login, refresh tokens)
* `admin` (panel y reportes)

---

## Tabla de endpoints (24 endpoints)

> **Nota:** todas las rutas van prefijadas con `/api/v1`.

| Método | Ruta                               |                                 Descripción | Params / Body                                                   | Autenticación                                   |                     |
| ------ | ---------------------------------- | ------------------------------------------: | --------------------------------------------------------------- | ----------------------------------------------- | ------------------- |
| POST   | `/api/v1/auth/register`            |      Registrar usuario (passenger o driver) | Body: `{full_name,email,phone,password,role}`                   | Público                                         |                     |
| POST   | `/api/v1/auth/login`               |    Login -> devuelve access & refresh token | Body: `{email,password}`                                        | Público                                         |                     |
| POST   | `/api/v1/auth/refresh`             |                   Refrescar token de acceso | Body: `{refresh_token}`                                         | Público (token válido requerido)                |                     |
| GET    | `/api/v1/users/me`                 |         Obtener perfil del usuario logueado | -                                                               | Requiere token                                  |                     |
| PUT    | `/api/v1/users/me`                 |                           Actualizar perfil | Body: `{full_name,phone,profile_picture_url}`                   | Requiere token                                  |                     |
| GET    | `/api/v1/drivers/available`        |            Listar drivers disponibles cerca | Query: `lat,lng,radius_km,page,limit`                           | Público (recomendado token opcional)            |                     |
| PUT    | `/api/v1/drivers/:id/availability` |        Actualizar disponibilidad del driver | Path: `id`; Body: `{is_available,current_lat,current_lng}`      | Token (role: driver)                            |                     |
| PATCH  | `/api/v1/drivers/location`         | Actualizar ubicación del driver (frecuente) | Body: `{current_lat,current_lng}`                               | Token (role: driver)                            |                     |
| POST   | `/api/v1/rides`                    |        Crear solicitud de viaje (passenger) | Body: `{origin:{lat,lng},destination:{lat,lng},payment_method}` | Token (role: passenger)                         |                     |
| GET    | `/api/v1/rides/:id`                |                  Obtener detalle de un ride | Path: `id`                                                      | Token (driver o passenger involucrado, o admin) |                     |
| POST   | `/api/v1/rides/:id/cancel`         |                              Cancelar viaje | Path: `id`; Body opc: `{reason}`                                | Token (passenger o driver involucrado)          |                     |
| POST   | `/api/v1/rides/:id/accept`         |                      Conductor acepta viaje | Path: `id`                                                      | Token (role: driver)                            |                     |
| POST   | `/api/v1/rides/:id/start`          |                               Iniciar viaje | Path: `id`                                                      | Token (role: driver)                            |                     |
| POST   | `/api/v1/rides/:id/complete`       |           Finalizar viaje y calcular tarifa | Path: `id`; Body: `{distance_km,duration_min}`                  | Token (role: driver)                            |                     |
| POST   | `/api/v1/rides/:id/pay`            |                     Pagar viaje (passenger) | Path: `id`; Body: `{method,amount_cents,card_token?}`           | Token (role: passenger)                         |                     |
| GET    | `/api/v1/users/:id/rides`          |            Historial de rides de un usuario | Path: `id`; Query: `status,page,limit`                          | Token (owner o admin)                           |                     |
| POST   | `/api/v1/rides/:id/review`         |              Crear calificación (post-ride) | Body: `{rating,comment}`                                        | Token (participante)                            |                     |
| GET    | `/api/v1/users/:id/reviews`        |               Obtener reviews de un usuario | Path: `id`; Query: `page,limit`                                 | Público (resumen)                               |                     |
| GET    | `/api/v1/payments/:id/receipt`     |                      Obtener recibo de pago | Path: `id`                                                      | Token (owner o admin)                           |                     |
| GET    | `/api/v1/admin/rides`              |          Listar todos los rides con filtros | Query: `status,from,to,page,limit`                              | Token (role: admin)                             |                     |
| PATCH  | `/api/v1/admin/users/:id/block`    |                Bloquear/Desbloquear usuario | Path: `id`; Body: \`{is\_blocked\:true                          | false}\`                                        | Token (role: admin) |
| POST   | `/api/v1/vehicles`                 |                 Registrar vehículo (driver) | Body: `{plate,model,color,year}`                                | Token (role: driver)                            |                     |
| GET    | `/api/v1/health`                   |                       Healthcheck de la API | -                                                               | Público                                         |                     |
| DELETE | `/api/v1/users/me`                 |               Eliminar cuenta (soft-delete) | Body opc: `{reason}`                                            | Token                                           |                     |

---

## Estructura de request/responses (ejemplos breves)

* **Crear ride (POST /api/v1/rides)**

  * Body: `{ "origin": {"lat":4.710989,"lng":-74.072090}, "destination": {"lat":4.611,"lng":-74.070}, "payment_method":"card" }`
  * Response 201: `{ "ride_id":"uuid","status":"requested","estimated_price_cents":12000,"estimated_time_min":12 }`

* **Aceptar ride (POST /api/v1/rides/\:id/accept)**

  * Auth driver. Response 200: `{ "ride_id":"uuid","status":"accepted","driver_id":"uuid" }`

* **Completar ride (POST /api/v1/rides/\:id/complete)**

  * Body: `{distance_km: 5.4, duration_min: 14}` -> Response 200: `{ "ride_id":"uuid","status":"completed","price_cents":15000 }`

---

## Flujos de uso (mínimo 3)

### Flujo A — Solicitud, aceptación y finalización de un viaje

1. Pasajero: `POST /api/v1/rides` con origen/destino. API responde `requested` con `ride_id` y estimados.
2. Sistema: busca drivers disponibles cerca (consulta interna) y notifica a drivers.
3. Conductor: `POST /api/v1/rides/:id/accept` -> si éxito status = `accepted` y `driver_id` asignado.
4. Conductor: `POST /api/v1/rides/:id/start` al iniciar — status `in_progress`.
5. Durante el viaje: driver actualiza ubicación frecuentemente `PATCH /api/v1/drivers/location`.
6. Al terminar: driver `POST /api/v1/rides/:id/complete` con `distance_km` y `duration_min`.
7. Sistema calcula `price_cents`, crea `payments` con `payment_status: pending`.
8. Pasajero: `POST /api/v1/rides/:id/pay` -> si éxito `payment_status: completed`.
9. Ambos pueden `POST /api/v1/rides/:id/review` para calificar.

### Flujo B — Cancelación y posible reembolso parcial

1. Pasajero solicita `POST /api/v1/rides` -> `requested`.
2. Antes de asignar conductor, pasajero cancela `POST /api/v1/rides/:id/cancel` -> status `cancelled`, sin cobro.
3. Si conductor ya aceptó y pasa la ventana X minutos, política aplica cancel fee: crear `payment` con `amount_cents` equal a fee; enviar notificación y cobrar según método disponible.
4. API devuelve 200 con `{ status: 'cancelled', refund_required: true/false, fee_cents }` según caso.

### Flujo C — Calificación mutua y actualización de promedio

1. Después del viaje completado, pasajero `POST /api/v1/rides/:id/review` con `{rating,comment}`.
2. API crea `review` y recalcula `drivers.rating_avg` (media ponderada).
3. Conductor puede hacer lo mismo para pasajero.
4. Reviews visibles en `GET /api/v1/users/:id/reviews`.

---

## Decisiones de diseño y justificación

* **Prefijo `/api/v1`:** permite versionar y evolucionar sin romper clientes.
* **JWT para auth:** fácil de usar en móviles; usar refresh tokens para sesiones largas.
* **Separación `users` / `drivers` / `vehicles`:** mantiene la tabla `users` genérica y tablas relacionadas para datos específicos (normalización).
* **Endpoints de ubicación en PATCH:** ubicación cambia frecuentemente, usar PATCH y endpoints livianos para evitar cargar logs innecesarios.
* **Pagos:** la API crea registros de `payments` y estados (`pending`, `completed`, `failed`) y delega cobro real a pasarela externa (webhooks).
* **Manejo de disponibilidad:** `drivers.is_available` + `current_lat/current_lng` para matchmaking.
* **Seguridad y privacidad:** https obligatorio, bcrypt para contraseñas, no retornar datos sensibles (ej. `password_hash`), throttling y validación de input.

---

## Manejo de errores (esquema)

**Formato estándar de error (JSON):**

```
HTTP/1.1 <codigo>
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Descripción x :v",
    "details": { /* opcional: campo, validaciones */ }
  }
}
```

**Códigos y ejemplos (5 errores reales):**

1. `400 Bad Request` — `INVALID_COORDINATES` — cuerpo con lat/lng fuera de rango.
2. `401 Unauthorized` — `INVALID_TOKEN` — token ausente o expirado.
3. `403 Forbidden` — `ROLE_FORBIDDEN` — driver intenta usar endpoint admin.
4. `404 Not Found` — `RIDE_NOT_FOUND` — id de ride inexistente o no pertenece al usuario.
5. `409 Conflict` — `RIDE_ALREADY_ACCEPTED` — dos drivers intentan aceptar el mismo ride; segundo recibe 409.

Adicionales: `429 Too Many Requests` para rate-limiting; `500 Internal Server Error` para fallos inesperados.

---

## Propuestas de mejora (futuras funcionalidades)

1. **Tracking en tiempo real con WebSockets** para ubicación y actualizaciones instantáneas (mejor UX).
2. **Integración con pasarela de pagos completa** (tokenización de tarjetas, 3DS, webhooks de reembolso).
3. **Matching avanzado**: algoritmo que considere rating, tiempo estimado, preferencia de viajes y tarifas dinámicas.
4. **Soporte multi-idioma y multi-moneda**.
5. **Funcionalidad de promociones y wallet interno** (créditos, descuentos, historiales).

---

 * GRACIAS :D
 * TRABAJO ECHO X MANUEL UWU


