# 06 — Matriz de Trazabilidad y Consideraciones Técnicas

> Mapea cada requerimiento funcional ([03-requerimientos-funcionales.md](./03-requerimientos-funcionales.md)) a la capa/componente que lo implementa según la arquitectura ya decidida en [01-arquitectura.md](./01-arquitectura.md). Donde el requerimiento excede lo ya modelado (principalmente el módulo `LOY`), se propone la extensión mínima manteniendo el mismo patrón `tenant_id` discriminator y la separación Router → Service → Repository.

---

## 6.1 Trazabilidad RF → Arquitectura (módulos ya cubiertos por el diseño base)

| RF | Modelo (`models/`) | Repository | Service | Router |
|---|---|---|---|---|
| RF-AGE-001 a 004 | `schedule.py` | `schedule_repository.py` | `staff_service.py` | `admin/schedules.py` |
| RF-AGE-010 a 012 | `schedule.py` (bloqueos como entradas de excepción) | `schedule_repository.py` | `staff_service.py` | `admin/schedules.py` |
| RF-AGE-020, 021 | `appointment.py`, `client` (dentro de `user.py`) | `appointment_repository.py`, `tenant_repository.find_by_slug()` | `appointment_service.py` | `public.py` |
| RF-AGE-022 a 024 | `appointment.py` | `appointment_repository.find_overlapping()` | `appointment_service.py` + `availability_service.py` | `appointments.py` |
| RF-AGE-030 a 034 | `appointment.py` (campo `status`) | `appointment_repository.py` | `appointment_service.py` | `appointments.py` |
| RF-TEN-001 a 003 | `tenant.py` | `tenant_repository.py` | *(nuevo, ver 6.2)* `tenant_service.py` | *(nuevo)* `super_admin/tenants.py` |
| RF-TEN-010 a 013 | `service.py` | `service_repository.py` | `service_service.py` | `admin/services.py` |
| RF-TEN-014 | `tenant.py` (campos de branding) | `tenant_repository.py` | `tenant_service.py` | `admin/tenant.py` |
| RF-NOT-001 a 007 | *(payload de email, sin modelo persistente propio)* | `notification_log_repository.py` | `notification_service.py` | *(invocado internamente, sin router propio — fire-and-forget)* |
| RF-CRM-001 a 003 | `appointment.py` (agregación) | `appointment_repository.py` | *(nuevo)* `client_service.py` | *(nuevo)* `admin/clients.py` |
| RF-CRM-004 a 008 | *(nuevo, ver 6.2)* `client_note.py` | *(nuevo)* `client_note_repository.py` | *(nuevo)* `client_service.py` | *(nuevo)* `admin/clients.py` |

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
│   ├── client_note_repository.py       # NUEVO
│   ├── loyalty_account_repository.py   # NUEVO
│   ├── loyalty_transaction_repository.py # NUEVO
│   ├── reward_repository.py            # NUEVO
│   └── redemption_repository.py        # NUEVO
├── services/
│   ├── tenant_service.py       # NUEVO — CRUD de tenants (super_admin)
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

## 6.3 Nuevas colecciones requeridas (módulo `LOY`)

Siguiendo el mismo criterio de [01-arquitectura.md §1.1](./01-arquitectura.md#11-decisión-de-arquitectura-de-tenancy) — single DB, `tenant_id` como discriminador en todas las colecciones — y el mismo estilo de índices de [01-arquitectura.md §1.3](./01-arquitectura.md#13-índices-mongodb-por-colección):

```javascript
// agendix/db/indexes.py — extensión propuesta

db.loyalty_accounts.create_index([("tenant_id", ASCENDING), ("client_id", ASCENDING)], unique=True)

db.loyalty_transactions.create_index([("tenant_id", ASCENDING), ("client_id", ASCENDING), ("created_at", DESCENDING)])
db.loyalty_transactions.create_index([("tenant_id", ASCENDING), ("appointment_id", ASCENDING)], unique=True, sparse=True)
// ^ el índice único sobre appointment_id es la garantía de idempotencia de RF-LOY-020:
//   una misma cita no puede generar dos transacciones de "otorgado por visita".

db.rewards_catalog.create_index([("tenant_id", ASCENDING), ("status", ASCENDING)])

db.redemptions.create_index([("tenant_id", ASCENDING), ("client_id", ASCENDING), ("created_at", DESCENDING)])
db.redemptions.create_index([("tenant_id", ASCENDING), ("status", ASCENDING)])

db.client_notes.create_index([("tenant_id", ASCENDING), ("client_id", ASCENDING), ("created_at", DESCENDING)])
db.client_notes.create_index([("tenant_id", ASCENDING), ("is_sensitive", ASCENDING)])
```

| Colección | Propósito | Campos clave |
|---|---|---|
| `loyalty_accounts` | Saldo vigente cacheado por cliente (evita recalcular sumando todas las transacciones en cada lectura) | `tenant_id`, `client_id`, `balance`, `updated_at` |
| `loyalty_transactions` | Ledger inmutable de movimientos (otorgado, canjeado, expirado, ajuste manual) | `tenant_id`, `client_id`, `type`, `amount`, `appointment_id?`, `expires_at?`, `created_by?` |
| `rewards_catalog` | Premios canjeables configurados por el negocio | `tenant_id`, `name`, `cost`, `stock?`, `status` |
| `redemptions` | Solicitudes de canje y su ciclo de vida | `tenant_id`, `client_id`, `reward_id`, `status` (`pending`/`delivered`/`rejected`), `created_at` |
| `client_notes` | Notas privadas de `staff` sobre un `client` | `tenant_id`, `client_id`, `author_staff_id`, `text`, `is_sensitive`, `created_at` |

**Decisión de diseño — ledger + saldo cacheado:** `loyalty_transactions` es la fuente de verdad (append-only, nunca se edita ni borra un movimiento histórico); `loyalty_accounts.balance` es una proyección mantenida transaccionalmente por `LoyaltyService` para que RF-LOY-011 (consulta de saldo) no tenga que recorrer todo el historial en cada request. El descuento atómico de RF-LOY-012 se implementa con `find_one_and_update` condicionado a `balance >= cost` (operación atómica de MongoDB), evitando la condición de carrera descrita en la historia US-06.

---

## 6.4 Matriz RF ↔ RNF (dependencias cruzadas relevantes)

| RF | RNF relacionado | Relación |
|---|---|---|
| RF-AGE-023, RF-AGE-024 | RNF-PERF-001 | La validación de solapamiento debe ser rápida porque bloquea la creación de la cita en el hot path de reserva pública. |
| RF-AGE-021 | RNF-SEC-008 | Los datos de contacto capturados en reserva pública son PII y deben tratarse como tal. |
| RF-LOY-012 | RNF-PERF-001 (implícito), consistencia atómica | El canje debe resolverse sin condición de carrera bajo concurrencia (ver 6.3). |
| RF-CRM-004 a 006 | RNF-SEC-005, RNF-SEC-006 | Las notas privadas/sensibles exigen RBAC más estricto que el resto del sistema, con auditoría de lectura. |
| RF-NOT-001 a 007 | RNF-DISP-002 | El fallo de notificación nunca debe degradar la disponibilidad percibida del flujo de reserva. |
| Todos los RF que leen/escriben cualquier colección | RNF-SEC-003 | Ningún RF se considera "cumplido" si su implementación en `repositories/` omite el filtro `tenant_id`. |

---

## 6.5 Riesgos técnicos identificados

| Riesgo | Mitigación propuesta |
|---|---|
| Doble otorgamiento de puntos si `AppointmentService` reintenta la transición a `completed` (ej. doble clic) | Índice único `sparse` en `loyalty_transactions.appointment_id` (ver 6.3) hace que el segundo intento de inserción falle de forma controlada; `LoyaltyService` debe capturar ese error de duplicado y tratarlo como no-op, no como fallo. |
| Canje concurrente que deje saldo negativo | `find_one_and_update` atómico condicionado a saldo suficiente (ver 6.3), nunca lectura-luego-escritura en dos pasos. |
| Fuga de notas sensibles por endpoint de exportación (RF-CRM-008) | El servicio de exportación debe usar un DTO explícito que excluya `client_notes` por diseño, no un `$lookup` genérico que las incluya por accidente. |
| Cambio de `tenant.slug` (RF-TEN-002) rompe links públicos ya compartidos | Restringir la edición a `super_admin`, mostrar advertencia explícita; evaluar en fase posterior un histórico de slugs con redirección 301. |
