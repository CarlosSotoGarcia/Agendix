# 06 — Matriz de Trazabilidad y Consideraciones Técnicas

> Mapea cada requerimiento funcional ([03-requerimientos-funcionales.md](./03-requerimientos-funcionales.md)) a la capa/componente que lo implementa según la arquitectura ya decidida en [01-arquitectura.md](./01-arquitectura.md) (Cloud Firestore Native Mode, subcolección por tenant). Donde el requerimiento excede lo ya modelado (principalmente el módulo `LOY`), se propone la extensión mínima manteniendo el mismo patrón `tenants/{tenant_id}/...` y la separación Router → Service → Repository.

---

## 6.1 Trazabilidad RF → Arquitectura (módulos ya cubiertos por el diseño base)

| RF | Modelo (`models/`) | Repository / ruta Firestore | Service | Router |
|---|---|---|---|---|
| RF-AGE-001 a 004 | `schedule.py` | `schedule_repository.py` → `tenants/{id}/schedules` | `staff_service.py` | `admin/schedules.py` |
| RF-AGE-010 a 012 | `schedule.py` (bloqueos como entradas de excepción) | `schedule_repository.py` → `tenants/{id}/schedules` | `staff_service.py` | `admin/schedules.py` |
| RF-AGE-020, 021 | `appointment.py`, `client` (dentro de `user.py`) | `appointment_repository.py`, `tenant_repository.find_id_by_slug()` → `tenant_slugs/{slug}` | `appointment_service.py` | `public.py` |
| RF-AGE-022 a 024 | `appointment.py` | `appointment_repository.create_if_available()` (transacción Firestore) → `tenants/{id}/appointments` | `appointment_service.py` + `availability_service.py` | `appointments.py` |
| RF-AGE-030 a 034 | `appointment.py` (campo `status`) | `appointment_repository.py` → `tenants/{id}/appointments` | `appointment_service.py` | `appointments.py` |
| RF-TEN-001 a 003 | `tenant.py` | `tenant_repository.py` → `tenants/{id}` + `tenant_slugs/{slug}` | *(nuevo, ver 6.2)* `tenant_service.py` | *(nuevo)* `super_admin/tenants.py` |
| RF-TEN-010 a 013 | `service.py` | `service_repository.py` → `tenants/{id}/services` | `service_service.py` | `admin/services.py` |
| RF-TEN-014 | `tenant.py` (campos de branding) | `tenant_repository.py` → `tenants/{id}` | `tenant_service.py` | `admin/tenant.py` |
| RF-NOT-001 a 007 | *(payload de email, sin modelo persistente propio)* | `notification_log_repository.py` → `tenants/{id}/notifications_log` | `notification_service.py` | *(invocado internamente, sin router propio — fire-and-forget)* |
| RF-CRM-001 a 003 | `appointment.py` (agregación en memoria — sin `$lookup`, ver 6.4) | `appointment_repository.py` → `tenants/{id}/appointments` | *(nuevo)* `client_service.py` | *(nuevo)* `admin/clients.py` |
| RF-CRM-004 a 008 | *(nuevo, ver 6.2)* `client_note.py` | *(nuevo)* `client_note_repository.py` → `tenants/{id}/client_notes` | *(nuevo)* `client_service.py` | *(nuevo)* `admin/clients.py` |

---

## 6.2 Componentes nuevos requeridos (no listados en el árbol original de `backend/agendix/`)

La estructura de directorios de [01-arquitectura.md §1.6](./01-arquitectura.md#16-estructura-de-directorios-del-backend) no incluía `super_admin/tenants.py`, `admin/clients.py`, `tenant_service.py`, `client_service.py` ni el módulo completo de recompensas. Se agregan, respetando la misma regla de capas ("ninguna capa se salta la inmediatamente inferior") y la tabla de "solo importa de" ya definida:

```
backend/agendix/
├── models/
│   ├── client_note.py          # NUEVO — nota privada de staff sobre un cliente
│   ├── loyalty_account.py      # NUEVO — saldo vigente por cliente (LOY)
│   ├── loyalty_transaction.py  # NUEVO — movimiento de puntos/sellos (LOY)
│   ├── reward.py               # NUEVO — ítem del catálogo de premios (LOY)
│   └── redemption.py           # NUEVO — canje de un premio (LOY)
├── repositories/
│   ├── client_note_repository.py       # NUEVO → tenants/{id}/client_notes
│   ├── loyalty_account_repository.py   # NUEVO → tenants/{id}/loyalty_accounts
│   ├── loyalty_transaction_repository.py # NUEVO → tenants/{id}/loyalty_transactions
│   ├── reward_repository.py            # NUEVO → tenants/{id}/rewards_catalog
│   └── redemption_repository.py        # NUEVO → tenants/{id}/redemptions
├── services/
│   ├── tenant_service.py       # NUEVO — CRUD de tenants (super_admin) + gestión de tenant_slugs
│   ├── client_service.py       # NUEVO — perfil, historial, notas (CRM)
│   └── loyalty_service.py      # NUEVO — orquesta otorgamiento y canje (LOY)
└── routers/
    ├── super_admin/
    │   └── tenants.py          # NUEVO — /super-admin/tenants CRUD
    └── admin/
        ├── clients.py          # NUEVO — /admin/clients, /admin/clients/{id}/notes
        └── loyalty.py          # NUEVO — /admin/loyalty/config, /admin/loyalty/catalog
```

`AppointmentService.create()` (ya existente) gana una dependencia opcional hacia `LoyaltyService` para disparar RF-LOY-020 al transicionar una cita a `completed` — mismo patrón fire-and-forget usado para notificaciones (`asyncio.create_task`), de modo que un fallo al otorgar puntos no revierte el cambio de estado de la cita ni bloquea la respuesta del endpoint.

---

## 6.3 Nuevas subcolecciones requeridas (módulo `LOY`)

Siguiendo el mismo criterio de [01-arquitectura.md §1.1 y §1.3](./01-arquitectura.md#11-decisión-de-arquitectura-de-tenancy) — subcolección bajo `tenants/{tenant_id}`, sin campo `tenant_id` redundante porque ya está implícito en la ruta:

```
tenants/{tenant_id}/
  ├─ loyalty_accounts/{client_user_id}      # 1 documento por cliente — ID = user_id del cliente
  ├─ loyalty_transactions/{transaction_id}  # ID determinístico p/idempotencia (ver más abajo)
  ├─ rewards_catalog/{reward_id}
  ├─ redemptions/{redemption_id}
  └─ client_notes/{note_id}
```

### Índices compuestos adicionales (`firestore.indexes.json`)

```json
{
  "collectionGroup": "loyalty_transactions", "queryScope": "COLLECTION",
  "fields": [
    { "fieldPath": "client_id", "order": "ASCENDING" },
    { "fieldPath": "created_at", "order": "DESCENDING" }
  ]
},
{
  "collectionGroup": "redemptions", "queryScope": "COLLECTION",
  "fields": [
    { "fieldPath": "client_id", "order": "ASCENDING" },
    { "fieldPath": "created_at", "order": "DESCENDING" }
  ]
},
{
  "collectionGroup": "client_notes", "queryScope": "COLLECTION",
  "fields": [
    { "fieldPath": "client_id", "order": "ASCENDING" },
    { "fieldPath": "created_at", "order": "DESCENDING" }
  ]
}
```

(`rewards_catalog.status`, `redemptions.status` y `client_notes.is_sensitive` son filtros de igualdad simple sobre un solo campo — cubiertos por el índice automático de campo único de Firestore, sin declaración manual.)

| Colección | Propósito | Campos clave | ID del documento |
|---|---|---|---|
| `loyalty_accounts` | Saldo vigente cacheado por cliente (evita sumar todo el ledger en cada lectura) | `balance`, `updated_at` | `client_user_id` — permite leer el saldo con **una sola lectura de documento**, sin query |
| `loyalty_transactions` | Ledger inmutable de movimientos (otorgado, canjeado, expirado, ajuste manual) | `client_id`, `type`, `amount`, `appointment_id?`, `expires_at?`, `created_by?`, `created_at` | Autogenerado, **excepto** para otorgamientos por cita completada: `appt_{appointment_id}` (ver idempotencia abajo) |
| `rewards_catalog` | Premios canjeables configurados por el negocio | `name`, `cost`, `stock?`, `status` | Autogenerado |
| `redemptions` | Solicitudes de canje y su ciclo de vida | `client_id`, `reward_id`, `status` (`pending`/`delivered`/`rejected`), `created_at` | Autogenerado |
| `client_notes` | Notas privadas de `staff` sobre un `client` | `client_id`, `author_staff_id`, `text`, `is_sensitive`, `created_at` | Autogenerado |

**Decisión de diseño — ledger + saldo cacheado, idempotencia por ID determinístico:** `loyalty_transactions` es la fuente de verdad (append-only, nunca se edita ni borra un movimiento histórico); `loyalty_accounts.balance` es una proyección mantenida transaccionalmente por `LoyaltyService` para que RF-LOY-011 (consulta de saldo) sea una lectura de un solo documento, no un recorrido del historial.

Firestore no tiene el equivalente de un índice único `sparse` de Mongo para garantizar "esta cita solo otorga puntos una vez". El equivalente idiomático es usar **el propio `appointment_id` como ID del documento** de la transacción de otorgamiento (`loyalty_transactions/appt_{appointment_id}`) y escribir con `create()` (falla si el documento ya existe) en vez de `set()`. `LoyaltyService` captura ese error de "ya existe" y lo trata como no-op — mismo comportamiento que el índice único `sparse`, sin necesitar declarar un índice adicional.

El descuento atómico de RF-LOY-012 (canje) se implementa con una **transacción Firestore** que lee `loyalty_accounts/{client_id}`, valida `balance >= cost` y escribe el nuevo balance + el documento de `redemptions` en la misma transacción — Firestore garantiza aislamiento serializable dentro de una transacción, cerrando la condición de carrera descrita en la historia US-06 igual que la transacción de reserva descrita en [01-arquitectura.md §1.4](./01-arquitectura.md#ejemplo-de-flujo-post-appointments-reservar-una-cita).

---

## 6.4 Notas de diseño de consultas (Firestore vs. lo que ofrecía Mongo)

| Necesidad | Enfoque en Mongo (descartado) | Enfoque en Firestore (adoptado) |
|---|---|---|
| Historial agregado de cliente (RF-CRM-001/002) | Aggregation pipeline (`$lookup`, `$group`) en una sola query | El Service (`client_service.py`) hace múltiples lecturas simples (citas del cliente, saldo de loyalty) y agrega en memoria en Python — aceptable dado el volumen por cliente (decenas de citas, no miles) |
| Verificación de solapamiento (RF-AGE-023/024) | `find_one` con filtro compuesto `(staff_id, status, start_time, end_time)` | Query con filtros `staff_id == X`, `status in [...]`, `start_time < end`, más filtro `end_time > start` evaluado en memoria sobre el resultado (pocos documentos), todo dentro de una transacción (ver §1.4) |
| Exportación CSV de clientes (RF-CRM-008) | Query con `$project` que excluye campos sensibles a nivel de DB | El DTO de exportación en `client_service.py` arma explícitamente los campos permitidos — nunca serializa el documento crudo, así que `client_notes` no puede filtrarse por accidente aunque no exista un `$project` de por medio |

---

## 6.5 Matriz RF ↔ RNF (dependencias cruzadas relevantes)

| RF | RNF relacionado | Relación |
|---|---|---|
| RF-AGE-023, RF-AGE-024 | RNF-PERF-001 | La validación de solapamiento debe ser rápida porque corre dentro de una transacción Firestore en el hot path de reserva pública; transacciones más largas aumentan la probabilidad de reintento por contención. |
| RF-AGE-021 | RNF-SEC-008 | Los datos de contacto capturados en reserva pública son PII y deben tratarse como tal, incluso viviendo en una subcolección físicamente aislada por tenant. |
| RF-LOY-012 | Consistencia atómica | El canje debe resolverse sin condición de carrera bajo concurrencia — transacción Firestore (ver §6.3). |
| RF-LOY-020 | Idempotencia | El otorgamiento de puntos por cita completada no debe duplicarse ante reintentos — ID determinístico `appt_{appointment_id}` + `create()` (ver §6.3). |
| RF-CRM-004 a 006 | RNF-SEC-005, RNF-SEC-006 | Las notas privadas/sensibles exigen RBAC más estricto que el resto del sistema, con auditoría de lectura, aplicado en el Service — las Security Rules de Firestore son solo backstop (ver [01-arquitectura.md §1.3](./01-arquitectura.md#reglas-de-seguridad-firestore-security-rules)). |
| RF-NOT-001 a 007 | RNF-DISP-002 | El fallo de notificación nunca debe degradar la disponibilidad percibida del flujo de reserva. |
| Todos los RF que leen/escriben cualquier colección | RNF-SEC-003 | Ningún RF se considera "cumplido" si su implementación en `repositories/` construye una ruta de subcolección fuera de `tenants/{tenant_id}/...` del tenant correcto. |

---

## 6.6 Riesgos técnicos identificados

| Riesgo | Mitigación propuesta |
|---|---|
| Doble otorgamiento de puntos si `AppointmentService` reintenta la transición a `completed` (ej. doble clic) | ID determinístico `appt_{appointment_id}` + `create()` en `loyalty_transactions` (ver §6.3); el segundo intento falla de forma controlada y `LoyaltyService` lo trata como no-op, no como error. |
| Canje concurrente que deje saldo negativo | Transacción Firestore que lee-valida-escribe balance + redemption atómicamente (ver §6.3), nunca lectura-luego-escritura en dos pasos separados. |
| Contención en la transacción de reserva bajo alta demanda de un mismo staff (ej. lanzamiento de un negocio popular) | Mantener la transacción acotada a lecturas del propio staff/rango de tiempo — no incluir lecturas innecesarias dentro de la transacción; monitorear tasa de reintentos vía logs de Cloud Run. |
| Fuga de notas sensibles por endpoint de exportación (RF-CRM-008) | El servicio de exportación arma un DTO explícito que **no** referencia `client_notes` en ningún punto del código, en vez de depender de excluir campos de un documento ya leído. |
| Cambio de `tenant.slug` (RF-TEN-002) rompe links públicos ya compartidos | Restringir la edición a `super_admin`; al cambiar el slug, `tenant_service.py` debe escribir el nuevo `tenant_slugs/{nuevo_slug}` y **conservar** el antiguo apuntando al mismo `tenant_id` marcado como `deprecated` (redirección), en vez de borrarlo. |
| Crecimiento de lecturas/día por negocio muy activo (límite gratuito: 50K lecturas/día) | El saldo cacheado en `loyalty_accounts` y el uso de IDs determinísticos ya reducen lecturas innecesarias; si un tenant se acerca al límite, es la señal misma de que el negocio justifica el plan Blaze (pago por uso, ver [07-plan-implementacion.md §7.2](./07-plan-implementacion.md#72-stack-de-bajo-costo-por-componente)). |
