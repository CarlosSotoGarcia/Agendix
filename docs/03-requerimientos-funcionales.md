# 03 — Requerimientos Funcionales (RF)

> Actores y roles según [02-vision-general.md](./02-vision-general.md). Cada RF indica el/los actor(es) que lo disparan y, cuando aplica, la capa de la arquitectura ([01-arquitectura.md](./01-arquitectura.md)) que lo resuelve. La trazabilidad completa hacia modelos/servicios/repositorios está en [06-trazabilidad.md](./06-trazabilidad.md).

Convención de IDs: `RF-<MÓDULO>-<NNN>`. Módulos: `AGE` (Agendamiento), `TEN` (Multitenancy/Personalización), `LOY` (Loyalty/Recompensas), `NOT` (Notificaciones), `CRM` (Clientes).

---

## 3.1 Módulo de Agendamiento y Citas (`AGE`)

### Configuración de horarios por proveedor

| ID | Requerimiento | Actor |
|---|---|---|
| RF-AGE-001 | El sistema debe permitir a un `staff` (o a un `business_admin` en su nombre) definir un horario recurrente semanal por día de la semana (`day_of_week`, `start_time`, `end_time`), persistido en la colección `schedules`. | staff, business_admin |
| RF-AGE-002 | El sistema debe permitir múltiples bloques de disponibilidad por día (ej. mañana y tarde con pausa de almuerzo entre medio). | staff, business_admin |
| RF-AGE-003 | El sistema debe permitir definir la zona horaria del negocio a nivel de `tenant`, y todos los `start_time`/`end_time` de `schedules` y `appointments` deben interpretarse en esa zona horaria. | business_admin |
| RF-AGE-004 | El sistema debe permitir asociar cada `service` a uno o más `staff` habilitados para prestarlo (no todo el personal ofrece todos los servicios). | business_admin |

### Bloqueo de agenda

| ID | Requerimiento | Actor |
|---|---|---|
| RF-AGE-010 | El sistema debe permitir a un `staff` crear bloqueos puntuales de agenda (vacaciones, incapacidad, evento personal) sobre un rango de fecha/hora, que excluyan esos slots del cálculo de disponibilidad. | staff |
| RF-AGE-011 | Un bloqueo de agenda que se solape con citas ya `confirmed` o `pending` debe requerir confirmación explícita y disparar notificación de posible reprogramación/cancelación a los clientes afectados. | staff, business_admin |
| RF-AGE-012 | El `business_admin` debe poder bloquear la agenda de todo el negocio (ej. día feriado) sin tener que bloquear staff por staff. | business_admin |

### Reservas públicas y privadas

| ID | Requerimiento | Actor |
|---|---|---|
| RF-AGE-020 | El sistema debe exponer una página pública por `tenant_slug` (`/:tenantSlug/reservar`) donde un visitante sin cuenta consulta disponibilidad real (servicio → staff → slot) y confirma una reserva, resolviendo el tenant vía `TenantRepository.find_id_by_slug()` (ver [01-arquitectura.md §1.2.1](./01-arquitectura.md#121-resolución-de-slug--tenant_id-página-pública-de-reservas)). | client (guest) |
| RF-AGE-021 | Al confirmar una reserva pública, el sistema debe crear o reutilizar (por email/teléfono) un registro `client` mínimo, sin exigir contraseña en el MVP. | client (guest) |
| RF-AGE-022 | Un `business_admin` o `staff` debe poder crear una cita manualmente "a nombre de" un cliente existente o nuevo (reserva privada, ej. tomada por teléfono). | staff, business_admin |
| RF-AGE-023 | Antes de persistir cualquier cita (pública o privada), el sistema debe validar disponibilidad vía `AvailabilityService.is_slot_available()` y rechazar con `409 Conflict` si el slot ya no está libre — esta regla vive únicamente en el Service, nunca en el router ni en queries sueltas (ver [01-arquitectura.md §1.4](./01-arquitectura.md#14-arquitectura-en-capas-router--service--repository)). | sistema |
| RF-AGE-024 | El sistema no debe permitir dos citas `pending`/`confirmed` solapadas para el mismo `staff_id` dentro del mismo `tenant_id`. | sistema |

### Estados de cita

| ID | Requerimiento | Actor |
|---|---|---|
| RF-AGE-030 | Toda cita debe tener un campo `status` con máquina de estados: `pending → confirmed → completed`, con transiciones alternativas a `cancelled` (desde `pending`/`confirmed`) y a `no_show` (desde `confirmed`, tras la hora de inicio). | sistema |
| RF-AGE-031 | Un `client` solo puede cancelar o reprogramar sus propias citas, y solo mientras estén en `pending` o `confirmed`, sujeto a la política de anticipación mínima configurada por el `tenant` (ver RF-TEN-013). | client |
| RF-AGE-032 | Solo `staff` o `business_admin` pueden marcar una cita como `completed` o `no_show`. | staff, business_admin |
| RF-AGE-033 | Al pasar una cita a `completed`, el sistema debe disparar (fire-and-forget) el otorgamiento de puntos de fidelización si el negocio tiene el módulo de recompensas activo (ver RF-LOY-020). | sistema |
| RF-AGE-034 | El sistema debe registrar auditoría mínima de cada cambio de estado (`status`, `changed_by`, `changed_at`) para trazabilidad ante disputas. | sistema |

**Diagrama de estados:**

```
   ┌─────────┐   confirma    ┌───────────┐   se cumple    ┌────────────┐
   │ pending │──────────────▶│ confirmed │────────────────▶│ completed  │
   └────┬────┘                └─────┬─────┘                └────────────┘
        │ cancela                   │ cancela / no llega
        ▼                           ▼
   ┌───────────┐              ┌───────────┐
   │ cancelled │              │  no_show  │
   └───────────┘              └───────────┘
```

---

## 3.2 Módulo de Multitenancy y Personalización (`TEN`)

| ID | Requerimiento | Actor |
|---|---|---|
| RF-TEN-001 | El sistema debe permitir a un `super_admin` dar de alta un nuevo `tenant` con `slug` único (validado vía escritura condicional del documento `tenant_slugs/{slug}`, ver [01-arquitectura.md §1.2.1](./01-arquitectura.md#121-resolución-de-slug--tenant_id-página-pública-de-reservas)), nombre comercial, vertical (barbería/consultorio/gimnasio/estética) y datos de contacto. | super_admin |
| RF-TEN-002 | El `slug` debe ser editable solo por `super_admin` (cambiarlo rompe enlaces públicos ya compartidos por el negocio), con advertencia explícita en UI. | super_admin |
| RF-TEN-003 | El sistema debe permitir a un `super_admin` suspender un `tenant` (ej. por impago), bloqueando el acceso de `business_admin`/`staff`/`client` de ese tenant sin borrar datos. | super_admin |
| RF-TEN-010 | El `business_admin` debe poder crear, editar y desactivar `services` con: nombre, descripción, duración (minutos), precio, categoría y estado (activo/inactivo). | business_admin |
| RF-TEN-011 | Un `service` desactivado no debe aparecer en la página pública de reserva, pero debe conservarse en citas históricas ya creadas. | sistema |
| RF-TEN-012 | El `business_admin` debe poder asignar/desasignar `staff` a cada `service` (ver RF-AGE-004) y definir un precio/duración distinto por combinación staff-servicio si el negocio lo requiere (override opcional). | business_admin |
| RF-TEN-013 | El `business_admin` debe poder configurar, a nivel de `tenant`, la política de cancelación/reprogramación (horas mínimas de anticipación) aplicada en RF-AGE-031. | business_admin |
| RF-TEN-014 | El `business_admin` debe poder personalizar la marca visible en su página pública (logo, color de acento) reutilizando el sistema de tokens `--nx-*` descrito en [01-arquitectura.md §1.7](./01-arquitectura.md#17-frontend-angular-20--primeng), sin alterar tokens de estado (`--nx-success/error/warning`). | business_admin |

---

## 3.3 Módulo de Sistema de Recompensas y Fidelización (`LOY`)

> Módulo nuevo respecto a la base heredada de Zity — no existe equivalente en el dominio de control de acceso. Requiere extender el modelo de datos (ver [06-trazabilidad.md §6.3](./06-trazabilidad.md#63-nuevas-colecciones-requeridas-módulo-loy)) con nuevas colecciones que siguen el mismo patrón `tenant_id` discriminator.

### Configuración de la mecánica (por tenant)

| ID | Requerimiento | Actor |
|---|---|---|
| RF-LOY-001 | El `business_admin` debe poder elegir, por tenant, entre dos mecánicas mutuamente excluyentes: **(a) puntos por monto gastado** (ej. 1 punto por cada $1.000) o **(b) sellos por visita** (ej. 1 sello por cita completada, sin importar monto). | business_admin |
| RF-LOY-002 | El `business_admin` debe poder configurar la **expiración de puntos/sellos** en días desde su otorgamiento (0 = sin expiración). | business_admin |
| RF-LOY-003 | El `business_admin` debe poder restringir qué `services` generan puntos/sellos (ej. servicios promocionales no acumulan). | business_admin |
| RF-LOY-004 | El `business_admin` debe poder activar/desactivar el módulo completo de recompensas para su tenant sin perder el histórico ya acumulado por sus clientes. | business_admin |

### Catálogo de premios y canje

| ID | Requerimiento | Actor |
|---|---|---|
| RF-LOY-010 | El `business_admin` debe poder crear, editar y desactivar ítems del catálogo de premios (`rewards_catalog`): nombre, descripción, costo en puntos/sellos, stock opcional, estado. | business_admin |
| RF-LOY-011 | Un `client` debe poder consultar su saldo de puntos/sellos vigente (ya descontada la porción expirada) y el catálogo de premios disponibles para su tenant. | client |
| RF-LOY-012 | Un `client` debe poder solicitar el canje de un premio si su saldo vigente ≥ costo del premio; el sistema debe descontar el saldo de forma atómica para evitar doble canje por condición de carrera. | client, sistema |
| RF-LOY-013 | Todo canje debe quedar en estado `pending` hasta que un `staff`/`business_admin` lo confirme en el momento de la entrega física del premio (evita canjes fantasma sin entrega real). | staff, business_admin |
| RF-LOY-014 | El sistema no debe permitir canjes que dejen el saldo del cliente en negativo, ni canjes de premios sin stock disponible (cuando el `business_admin` definió stock finito). | sistema |

### Otorgamiento de puntos

| ID | Requerimiento | Actor |
|---|---|---|
| RF-LOY-020 | Al marcar una cita como `completed` (RF-AGE-033), el sistema debe otorgar puntos/sellos automáticamente según la configuración del tenant, registrando la transacción en `loyalty_transactions` con referencia a `appointment_id` (idempotente: una misma cita nunca otorga puntos dos veces). | sistema |
| RF-LOY-021 | El `business_admin` debe poder otorgar puntos manualmente a un cliente (ajuste, promoción especial), quedando registrado quién lo autorizó. | business_admin |
| RF-LOY-022 | El sistema debe exponer al `client` un historial cronológico de movimientos de su cuenta de puntos (otorgados, canjeados, expirados). | client |

---

## 3.4 Módulo de Notificaciones y Recordatorios (`NOT`)

| ID | Requerimiento | Actor |
|---|---|---|
| RF-NOT-001 | Al confirmarse una cita (pública o privada), el sistema debe enviar una notificación de confirmación por email de forma **fire-and-forget** (`asyncio.create_task`, sin bloquear la respuesta del `POST /appointments`), tal como se ilustra en [01-arquitectura.md §1.4](./01-arquitectura.md#ejemplo-de-flujo-post-appointments-reservar-una-cita). | sistema |
| RF-NOT-002 | El sistema debe enviar un recordatorio automático 24 horas antes del inicio de la cita, y opcionalmente un segundo recordatorio configurable (ej. 2 horas antes). *(Fase 2 según roadmap de [docs/README.md](./README.md)).* | sistema |
| RF-NOT-003 | El sistema debe notificar al `client` ante cualquier cancelación o reprogramación de su cita, sea iniciada por el negocio o por el propio cliente (confirmación). | sistema |
| RF-NOT-004 | El sistema debe notificar al `staff` responsable cuando se le asigna una nueva cita o cuando una existente se cancela/reprograma. | sistema |
| RF-NOT-005 | Todo envío (exitoso o fallido) debe quedar registrado en `notifications_log` con `tenant_id`, `appointment_id`, canal, estado y timestamp, para poder auditar entregabilidad. | sistema |
| RF-NOT-006 | El `business_admin` debe poder elegir el/los canal(es) activos por tenant: Email (MVP), y en fase 2 SMS/WhatsApp vía proveedor externo (`TWILIO_*` o similar, ver [01-arquitectura.md §1.8](./01-arquitectura.md#18-variables-de-entorno-requeridas)). | business_admin |
| RF-NOT-007 | Un fallo en el envío de notificación **no debe** revertir ni bloquear la operación de negocio que la originó (reserva, cancelación, cambio de estado) — el envío es asíncrono y desacoplado por diseño. | sistema |

---

## 3.5 Módulo de Clientes (CRM Básico) (`CRM`)

| ID | Requerimiento | Actor |
|---|---|---|
| RF-CRM-001 | El sistema debe mantener, por `client`, un historial completo de citas (`pending`/`confirmed`/`completed`/`cancelled`/`no_show`) visible para `staff` y `business_admin` de su tenant. | staff, business_admin |
| RF-CRM-002 | El sistema debe mostrar, por cliente, el listado de servicios contratados a lo largo del tiempo (derivado de `appointments.service_id` + `completed`), útil para detectar patrones de consumo. | staff, business_admin |
| RF-CRM-003 | El sistema debe mostrar el balance de puntos/sellos vigente del cliente (integración con módulo `LOY`, ver RF-LOY-011) en la misma vista de perfil. | staff, business_admin |
| RF-CRM-004 | El sistema debe permitir a un `staff` (o `business_admin`) agregar **notas privadas** de texto libre asociadas a un cliente (ej. preferencias, alergias, observaciones clínicas), no visibles para el propio `client`. | staff, business_admin |
| RF-CRM-005 | Las notas privadas deben quedar restringidas por RBAC: únicamente `staff`/`business_admin` del mismo `tenant_id` pueden leerlas; nunca se exponen en endpoints públicos ni en el perfil que ve el `client`. | sistema |
| RF-CRM-006 | Para negocios de vertical "consultorio", el sistema debe permitir marcar una nota como **sensible** (dato de salud), lo que activa los controles adicionales descritos en [04-requerimientos-no-funcionales.md §4.2](./04-requerimientos-no-funcionales.md#42-seguridad-y-privacidad-de-datos). | staff, business_admin |
| RF-CRM-007 | Un `client` autenticado debe poder ver y editar sus propios datos de contacto (nombre, email, teléfono), pero nunca las notas internas del proveedor. | client |
| RF-CRM-008 | El `business_admin` debe poder exportar (CSV) el listado de clientes de su tenant con su historial agregado (nombre, # citas completadas, # no-show, saldo de puntos), para campañas de retención fuera del sistema. | business_admin |

---

## 3.6 Fuera de alcance del MVP (explícito)

Para evitar ambigüedad en el sprint de diseño, quedan **fuera** de esta versión de requerimientos (candidatos a fase 2/3 según roadmap en [docs/README.md](./README.md)):

- Pagos en línea / depósitos por adelantado al reservar.
- Lista de espera automática cuando un slot se libera por cancelación.
- Recordatorios por SMS/WhatsApp (solo email en MVP).
- Reglas de recompensas combinadas (puntos **y** sellos simultáneos por tenant).
- App móvil nativa (el cliente final usa PWA responsive, ver RNF de usabilidad).
