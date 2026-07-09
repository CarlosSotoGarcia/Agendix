# 04 — Requerimientos No Funcionales (RNF)

> Todos los RNF asumen la infraestructura ya decidida en [01-arquitectura.md](./01-arquitectura.md): FastAPI + Motor sobre MongoDB single-DB, Angular 20 + PrimeNG, despliegue en GCP (Cloud Run + Firebase Hosting) vía `release.py`.

---

## 4.1 Rendimiento, Escalabilidad y Disponibilidad

| ID | Requerimiento |
|---|---|
| RNF-PERF-001 | El cálculo de disponibilidad (`AvailabilityService.is_slot_available`) debe resolver en **< 200 ms p95** apoyándose en el índice compuesto `(tenant_id, staff_id, start_time)` sobre `appointments`, el más sensible según [01-arquitectura.md §1.3](./01-arquitectura.md#13-índices-mongodb-por-colección). |
| RNF-PERF-002 | El endpoint público `GET /public/{tenant_slug}/availability` debe responder en **< 500 ms p95** bajo carga normal, dado que es la puerta de entrada de clientes anónimos sensibles a la fricción. |
| RNF-PERF-003 | El envío de notificaciones (confirmación, recordatorio) es **fire-and-forget** (RF-NOT-001/007) y **no debe** contarse dentro del SLA de latencia del endpoint que lo origina. |
| RNF-ESC-001 | La estrategia single-DB + `tenant_id` (ver [01-arquitectura.md §1.1](./01-arquitectura.md#11-decisión-de-arquitectura-de-tenancy)) debe soportar crecimiento horizontal de tenants sin cambio de esquema; la migración a DB-per-tenant para un cliente enterprise específico debe seguir siendo viable sin romper el modelo de datos. |
| RNF-ESC-002 | El backend (Cloud Run) debe escalar horizontalmente sin estado compartido en proceso — cualquier estado de sesión vive en JWT o en MongoDB, nunca en memoria del contenedor. |
| RNF-DISP-001 | Disponibilidad objetivo: **99.5%** mensual para el backend en producción (excluye ventanas de mantenimiento anunciadas), verificable vía `python release.py status`. |
| RNF-DISP-002 | Un fallo del proveedor de notificaciones (SMTP/SMS) no debe degradar la disponibilidad del flujo de reserva — la creación de la cita se confirma en base de datos antes de intentar notificar (orden ya reflejado en el ejemplo de [01-arquitectura.md §1.4](./01-arquitectura.md#ejemplo-de-flujo-post-appointments-reservar-una-cita)). |

---

## 4.2 Seguridad y Privacidad de Datos

| ID | Requerimiento |
|---|---|
| RNF-SEC-001 | Toda contraseña se almacena con hash `bcrypt` (nunca en texto plano ni con hash reversible), consistente con `core/security.py` de la arquitectura base. |
| RNF-SEC-002 | Autenticación vía JWT de acceso (15 min) + refresh token (7 días); ningún endpoint fuera de `BYPASS_PATHS`/`BYPASS_PREFIXES` debe aceptar requests sin token válido o sin `X-Tenant-ID` (ver `TenantMiddleware`, [01-arquitectura.md §1.2](./01-arquitectura.md#12-identificación-del-tenant-en-fastapi)). |
| RNF-SEC-003 | **Aislamiento de datos entre tenants**: toda query de `repositories/` debe incluir `tenant_id` en el filtro — no debe existir ningún path de código que lea/escriba un documento de un tenant sin verificar pertenencia (ver responsabilidad de capa en [01-arquitectura.md §1.4](./01-arquitectura.md#responsabilidad-de-cada-capa)). Se recomienda test de regresión dedicado (`test_multitenancy.py`, ya listado en la estructura de `tests/`) que intente fugas cross-tenant y falle el build si alguna pasa. |
| RNF-SEC-004 | RBAC estricto por rol (`require_role()`): un `client` nunca debe poder invocar endpoints de `business_admin`/`staff` (gestión de servicios, staff, catálogo de recompensas, notas privadas), verificado a nivel de router (Capa 1), no solo ocultado en el frontend. |
| RNF-SEC-005 | Las **notas privadas del proveedor** (RF-CRM-004/005) deben ser inaccesibles para el rol `client` en cualquier endpoint, incluidos los de exportación (RF-CRM-008), que deben excluirlas explícitamente. |
| RNF-SEC-006 | Para negocios de vertical "consultorio" con notas marcadas como **sensibles** (RF-CRM-006, datos de salud), el sistema debe: (a) restringir su lectura al `staff` que la creó y al `business_admin` propietario del tenant — no a todo `staff` del negocio por defecto; (b) registrar en log de auditoría cada acceso de lectura a una nota sensible (quién, cuándo). Este control es más estricto que el RBAC genérico de RNF-SEC-004 y responde al mayor estándar de confidencialidad esperado para datos clínicos, aun cuando el sistema no busca certificación HIPAA formal en el MVP. |
| RNF-SEC-007 | Toda comunicación cliente-servidor debe viajar sobre HTTPS (impuesto por Cloud Run + Firebase Hosting en producción); `ALLOWED_ORIGINS`/`ALLOWED_ORIGIN_REGEX` deben restringir CORS al dominio del frontend oficial. |
| RNF-SEC-008 | Los datos de contacto del `client` (email, teléfono) capturados en la reserva pública (RF-AGE-021) deben tratarse como PII: no se exponen en ningún endpoint público de otro tenant ni en logs de aplicación en texto plano. |

---

## 4.3 Usabilidad y Dispositivos Móviles

| ID | Requerimiento |
|---|---|
| RNF-USA-001 | La página pública de reserva (`/:tenantSlug/reservar`) debe ser **responsive-first**: el flujo completo (servicio → staff → slot → confirmación) debe ser usable sin scroll horizontal en viewport de 360px de ancho. |
| RNF-USA-002 | La página pública de reserva debe implementarse como **PWA** (manifest + service worker básico) para permitir "agregar a inicio" desde el móvil del cliente final, dado que es el punto de contacto más frecuente y con menor tolerancia a fricción. |
| RNF-USA-003 | Los paneles de `business_admin`/`staff` (gestión de agenda, clientes, recompensas) deben ser usables en tablet como mínimo; no se exige paridad completa con mobile-first para estas vistas administrativas. |
| RNF-USA-004 | El flujo de reserva pública no debe requerir más de 4 pasos (servicio, staff/horario, datos de contacto, confirmación) para minimizar abandono, alineado con el objetivo de negocio de < 90 segundos (ver [02-vision-general.md §2.1](./02-vision-general.md#objetivos-de-negocio-medibles)). |
| RNF-USA-005 | Todos los estados de carga y error de disponibilidad (slot recién tomado por otro cliente, RF-AGE-023) deben comunicarse con mensajes claros en el momento, no con errores genéricos de HTTP. |

---

## 4.4 Observabilidad y Mantenibilidad

| ID | Requerimiento |
|---|---|
| RNF-OBS-001 | Todo cambio de estado de cita (RF-AGE-034) y todo acceso a nota sensible (RNF-SEC-006) deben quedar en un log auditable, independiente de `notifications_log`. |
| RNF-OBS-002 | El endpoint `/health` debe reportar estado de conexión a MongoDB, consumido por `python release.py status`. |
| RNF-OBS-003 | La cobertura de tests debe incluir explícitamente casos de solapamiento de horario y límites de agenda (`test_availability.py`) y fuga cross-tenant (`test_multitenancy.py`), ya previstos en la estructura de directorios de [01-arquitectura.md §1.6](./01-arquitectura.md#16-estructura-de-directorios-del-backend). |
