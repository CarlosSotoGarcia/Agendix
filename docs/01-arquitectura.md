# 01 — Arquitectura y Estrategia Multi-Tenant (Agendix)

> Base técnica extraída de Zity (mismo stack Novex). El dominio se adaptó: los "tenants" aquí son **negocios** (barberías, spas, consultorios) en vez de colonias, y el flujo central es **reservar una cita**, no verificar un acceso.

## 1.1 Decisión de Arquitectura de Tenancy

Se adopta, igual que en Zity, la estrategia **"Single Database + Discriminator Field"**:

- Un único cluster MongoDB.
- Una única base de datos `agendix_db`.
- Cada documento en **todas** las colecciones contiene el campo `tenant_id` (ObjectId = negocio), indexado como compound index junto con el campo de búsqueda principal.

### Justificación

| Criterio | Single DB + tenant_id | DB por tenant |
|---|---|---|
| Costo inicial | Bajo | Alto |
| Complejidad operativa | Baja | Alta (N conexiones) |
| Aislamiento lógico | Garantizado por app | Garantizado por motor |
| Escalabilidad horizontal | Alta | Media |
| Cumplimiento de privacidad (MVP) | Suficiente | Óptimo |

Igual que en Zity, la migración a DB-per-tenant sigue siendo posible sin cambios de esquema gracias al campo `tenant_id` uniforme.

---

## 1.2 Identificación del Tenant en FastAPI

El `tenant_id` se resuelve en el mismo orden que Zity:

```
1. Claim "tenant_id" dentro del JWT (fuente principal tras login)
2. Header HTTP personalizado "X-Tenant-ID" (uso pre-login, app autenticada)
3. Slug en la ruta pública, ej. /api/v1/public/{tenant_slug}/... (ver 1.2.1)
```

### Middleware de Resolución

```python
# agendix/middleware/tenant.py

from fastapi import Request, HTTPException, status
from starlette.middleware.base import BaseHTTPMiddleware
from agendix.core.jwt import decode_token

BYPASS_PATHS = {"/health", "/docs", "/redoc", "/openapi.json"}
BYPASS_PREFIXES = ("/api/v1/auth", "/api/v1/public")

class TenantMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        if request.url.path in BYPASS_PATHS or request.url.path.startswith(BYPASS_PREFIXES):
            return await call_next(request)

        tenant_id: str | None = None

        auth_header = request.headers.get("Authorization", "")
        if auth_header.startswith("Bearer "):
            try:
                payload = decode_token(auth_header.split(" ")[1])
                tenant_id = payload.get("tenant_id")
            except Exception:
                pass

        if not tenant_id:
            tenant_id = request.headers.get("X-Tenant-ID")

        if not tenant_id:
            raise HTTPException(
                status_code=status.HTTP_400_BAD_REQUEST,
                detail="tenant_id no identificado. Incluya X-Tenant-ID o un JWT válido."
            )

        request.state.tenant_id = tenant_id
        return await call_next(request)
```

### 1.2.1 Diferencia clave con Zity: página pública de reservas

Zity es un sistema cerrado — todo requiere tenant resuelto por header/JWT porque solo lo usan colonos, vigilantes y admins ya autenticados.

Agendix necesita además una **página pública de reservas** (`agendix.app/:tenantSlug`, estilo Calendly) donde un cliente anónimo ve disponibilidad y agenda sin cuenta previa. Esas rutas viven bajo `/api/v1/public/{tenant_slug}/...`, quedan fuera del `TenantMiddleware` y resuelven el tenant explícitamente en el router vía `TenantRepository.find_by_slug(tenant_slug)` — no dependen de JWT ni header. Al confirmar la reserva se crea (o reutiliza) un `client` mínimo con email/teléfono, sin contraseña obligatoria en el MVP.

### Dependencia Inyectable

```python
# agendix/dependencies/tenant.py

from fastapi import Request

def get_tenant_id(request: Request) -> str:
    return request.state.tenant_id
```

---

## 1.3 Índices MongoDB por Colección

```javascript
// agendix/db/indexes.py

db.tenants.create_index([("slug", ASCENDING)], unique=True)

db.users.create_index([("tenant_id", ASCENDING), ("email", ASCENDING)], unique=True)

db.services.create_index([("tenant_id", ASCENDING), ("name", ASCENDING)])

db.staff.create_index([("tenant_id", ASCENDING), ("user_id", ASCENDING)])

db.schedules.create_index([("tenant_id", ASCENDING), ("staff_id", ASCENDING), ("day_of_week", ASCENDING)])

db.appointments.create_index([("tenant_id", ASCENDING), ("staff_id", ASCENDING), ("start_time", ASCENDING)])
db.appointments.create_index([("tenant_id", ASCENDING), ("client_id", ASCENDING), ("start_time", DESCENDING)])
db.appointments.create_index([("tenant_id", ASCENDING), ("status", ASCENDING), ("start_time", ASCENDING)])

db.notifications_log.create_index([("tenant_id", ASCENDING), ("appointment_id", ASCENDING)])
```

El índice `(tenant_id, staff_id, start_time)` es el más sensible: sobre él corre la verificación de solapamiento de horarios (ver 1.4).

---

## 1.4 Arquitectura en Capas: Router → Service → Repository

Misma separación estricta de Zity. Ninguna capa se salta la inmediatamente inferior.

```
┌─────────────────────────────────────────────────────────┐
│                       ROUTER                            │
│  • Recibe y valida el request HTTP (Pydantic schemas)   │
│  • Ejecuta guards de autenticación y RBAC                │
│  • Delega la lógica al Service correspondiente          │
│  • NO conoce MongoDB, NO escribe queries                │
└─────────────────────┬───────────────────────────────────┘
                      │ llama a
┌─────────────────────▼───────────────────────────────────┐
│                      SERVICE                            │
│  • Contiene toda la lógica de negocio                   │
│  • AvailabilityService calcula slots libres por staff    │
│  • AppointmentService valida solapamiento antes de crear │
│  • NO importa Motor ni AsyncIOMotorClient directamente   │
└─────────────────────┬───────────────────────────────────┘
                      │ llama a
┌─────────────────────▼───────────────────────────────────┐
│                    REPOSITORY                           │
│  • Único punto de contacto con MongoDB (Motor async)     │
│  • Aplica siempre el filtro tenant_id en sus queries      │
│  • Convierte documentos BSON ↔ modelos Pydantic          │
│  • NO contiene lógica de negocio                        │
└─────────────────────────────────────────────────────────┘
```

### Ejemplo de flujo `POST /appointments` (reservar una cita)

```python
# CAPA 1 — Router (agendix/routers/appointments.py)
@router.post("/appointments", response_model=AppointmentPublic, status_code=201)
async def create_appointment(
    body: AppointmentCreate,
    tenant_id: str = Depends(get_tenant_id),
    current_user: UserInDB = Depends(require_role(UserRole.CLIENT, UserRole.BUSINESS_ADMIN)),
    appointment_svc: AppointmentService = Depends(get_appointment_service),
):
    return await appointment_svc.create(body, tenant_id, current_user)

# CAPA 2 — Service (agendix/services/appointment_service.py)
class AppointmentService:
    def __init__(self, repo: AppointmentRepository, availability_svc: AvailabilityService):
        self.repo = repo
        self.availability_svc = availability_svc

    async def create(self, data: AppointmentCreate, tenant_id: str, user: UserInDB) -> AppointmentPublic:
        is_free = await self.availability_svc.is_slot_available(
            tenant_id, data.staff_id, data.start_time, data.end_time
        )
        if not is_free:
            raise HTTPException(status_code=409, detail="El horario ya no está disponible")
        appointment = await self.repo.insert(data, tenant_id, user)
        asyncio.create_task(self._notify_confirmation(appointment))  # fire-and-forget
        return appointment

# CAPA 3 — Repository (agendix/repositories/appointment_repository.py)
class AppointmentRepository:
    def __init__(self, db: AsyncIOMotorDatabase):
        self._col = db["appointments"]

    async def find_overlapping(self, tenant_id: str, staff_id: str, start: datetime, end: datetime):
        return await self._col.find_one({
            "tenant_id": tenant_id,
            "staff_id": staff_id,
            "status": {"$in": ["pending", "confirmed"]},
            "start_time": {"$lt": end},
            "end_time": {"$gt": start},
        })
```

La regla de negocio central de Agendix — **no permitir dos citas solapadas para el mismo staff** — vive en `AvailabilityService`, nunca en el router ni directamente en Mongo queries sueltas.

---

## 1.5 Diagrama de Componentes (Fase 1)

```
┌─────────────────────────────────────────────────────────┐
│                        CLIENTES                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Cliente final │  │ Staff        │  │ Admin negocio │  │
│  │ (reserva)     │  │ (atiende)    │  │ (configura)   │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────│────────────────│────────────────│────────────┘
          │  Angular 20    │                │
          └────────────────┴────────────────┘
                           │ HTTPS + JWT (o público por slug)
                ┌──────────▼──────────┐
                │   FastAPI Gateway   │
                │  TenantMiddleware   │
                │  AuthMiddleware     │
                │  RBAC Dependency    │
                └──────────┬──────────┘
                           │
           ┌───────────────┼───────────────┬───────────────┐
           ▼               ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
    │   ROUTERS  │  │   ROUTERS  │  │   ROUTERS  │  │   ROUTERS  │
    │   /auth    │  │/appointments│ │  /public   │  │ /admin/*   │
    └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  CAPA 1
    ──────┼───────────────┼───────────────┼───────────────┼─────────
          ▼               ▼               ▼               ▼        CAPA 2
    ┌────────────┐  ┌──────────────┐ ┌──────────────┐ ┌────────────┐
    │ AuthService│  │ Appointment  │ │ Availability │ │ Notification│
    │ UserService│  │ Service      │ │ Service      │ │ Service     │
    └─────┬──────┘  └─────┬────────┘ └─────┬────────┘ └─────┬───────┘
          │               │                │                │
    ──────┼───────────────┼────────────────┼────────────────┼────────
          ▼               ▼                ▼                ▼        CAPA 3
    ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐
    │ UserRepo   │  │ Appointment│  │ Schedule   │  │ NotificationLog │
    │ TenantRepo │  │ Repository │  │ Repository │  │ Repository      │
    └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬───────────┘
          └───────────────┴───────────────┴────────────────┘
                           │ Motor async
                ┌──────────▼──────────┐
                │   MongoDB 7+        │
                │   agendix_db        │
                │   ├─ tenants        │
                │   ├─ users          │
                │   ├─ services       │
                │   ├─ staff          │
                │   ├─ schedules      │
                │   ├─ appointments   │
                │   └─ notifications_log │
                └─────────────────────┘
```

---

## 1.6 Estructura de Directorios del Backend

```
backend/
├── agendix/
│   ├── main.py                        # FastAPI app, registro de routers y middlewares
│   ├── core/
│   │   ├── config.py                  # Settings con pydantic-settings
│   │   ├── jwt.py                     # Creación y decodificación de tokens
│   │   └── security.py                # Hash de contraseñas (bcrypt)
│   ├── db/
│   │   ├── client.py                  # Motor AsyncIOMotorClient singleton + get_database()
│   │   └── indexes.py                 # Creación de índices al startup
│   ├── middleware/
│   │   └── tenant.py                  # TenantMiddleware
│   ├── dependencies/
│   │   ├── tenant.py                  # get_tenant_id()
│   │   ├── auth.py                    # get_current_user(), require_role()
│   │   └── services.py                # Fábrica de services y repositories via Depends
│   ├── models/
│   │   ├── tenant.py                  # Negocio: nombre, slug, horario general, config
│   │   ├── user.py                    # super_admin / business_admin / staff / client
│   │   ├── service.py                 # Servicio ofrecido: nombre, duración, precio
│   │   ├── staff.py                   # Perfil de staff (vincula a user)
│   │   ├── schedule.py                # Disponibilidad recurrente por staff/día
│   │   └── appointment.py             # Cita: staff_id, service_id, client_id, start/end, status
│   ├── repositories/
│   │   ├── base_repository.py
│   │   ├── tenant_repository.py
│   │   ├── user_repository.py
│   │   ├── service_repository.py
│   │   ├── staff_repository.py
│   │   ├── schedule_repository.py
│   │   ├── appointment_repository.py
│   │   └── notification_log_repository.py
│   ├── services/
│   │   ├── auth_service.py            # Login, refresh, hashing
│   │   ├── appointment_service.py      # Reglas de reserva, orquesta repos
│   │   ├── availability_service.py     # Cálculo de slots libres + detección de solapamiento
│   │   ├── notification_service.py     # Confirmaciones y recordatorios (fire-and-forget)
│   │   ├── service_service.py          # CRUD de catálogo de servicios
│   │   └── staff_service.py            # CRUD de staff y sus horarios
│   └── routers/
│       ├── auth.py                    # /auth/login, /auth/refresh, /auth/me
│       ├── appointments.py            # /appointments CRUD, /appointments/{id}/cancel
│       ├── public.py                  # /public/{tenant_slug}/availability, /public/{tenant_slug}/book
│       └── admin/
│           ├── services.py            # /admin/services CRUD
│           ├── staff.py               # /admin/staff CRUD
│           └── schedules.py           # /admin/schedules CRUD
├── scripts/
│   └── seed.py                        # Datos iniciales de desarrollo
├── tests/
│   ├── conftest.py
│   ├── test_auth.py
│   ├── test_appointments.py
│   ├── test_availability.py           # casos de solapamiento y límites de horario
│   └── test_multitenancy.py
├── Dockerfile
├── requirements.txt
└── .env.example
```

### Responsabilidad de cada capa

| Carpeta | Capa | Solo importa de |
|---------|------|----------------|
| `routers/` | Router | `models/`, `services/` vía `dependencies/services.py` |
| `services/` | Service | `models/`, `repositories/`, `core/` |
| `repositories/` | Repository | `models/`, `db/client.py` |
| `models/` | Schemas | Solo Pydantic (sin imports del proyecto) |
| `core/` | Transversal | Solo stdlib y librerías externas |

---

## 1.7 Frontend (Angular 20 + PrimeNG)

Misma convención que Zity: componentes **standalone**, rutas lazy-loaded, dos interceptors (`auth`, `tenant`), `StorageService` para `access_token`/`refresh_token`/`tenant_id`/`tenant_slug`, signals para estado reactivo.

**Ruta pública nueva propia de Agendix**: `/:tenantSlug/reservar` — página de reserva sin `authGuard`, consume `/api/v1/public/{tenant_slug}/*`. Es el equivalente al `LandingComponent` público de Zity, pero interactivo (selecciona servicio → staff → slot → confirma).

### Sistema de marca Novex

Reutilizar `novex-brand.css` (mismo archivo fuente que Zity, copiado desde `novexlandig/assets/novex-brand.css`) y definir un preset propio `NovexAgendixPreset` en `app.config.ts` sobre Aura, igual que `NovexZityPreset`. Se recomienda un acento distinto al azul de Zity para diferenciar el producto en la familia Novex (p. ej. cian/teal, consistente con el mockup de "Novex Book"), manteniendo los tokens `--nx-navy-*` para dark mode y `--nx-success/error/warning` sin cambios.

---

## 1.8 Variables de Entorno Requeridas

```env
# .env.example
MONGODB_URL=mongodb://localhost:27017
MONGODB_DB_NAME=agendix_db

JWT_SECRET_KEY=cambia_esto_en_produccion_min32chars
JWT_ALGORITHM=HS256
JWT_ACCESS_TOKEN_EXPIRE_MINUTES=15
JWT_REFRESH_TOKEN_EXPIRE_DAYS=7

ENVIRONMENT=development
ALLOWED_ORIGINS=http://localhost:4200
ALLOWED_ORIGIN_REGEX=

FRONTEND_URL=http://localhost:4200
EMAIL_HOST=
EMAIL_PORT=
EMAIL_USER=
EMAIL_PASSWORD=
EMAIL_FROM=
```

No hay equivalente a `QR_SIGNING_SECRET` (Agendix no firma códigos QR); en cambio, en fases posteriores se añadirán credenciales de proveedor de SMS/WhatsApp para recordatorios (`TWILIO_*` o similar).

---

## 1.9 Qué se reutiliza de Zity vs. qué es nuevo

| Reutilizado tal cual | Nuevo / específico de Agendix |
|---|---|
| Single DB + `tenant_id` discriminator | Modelo de dominio: `service`, `staff`, `schedule`, `appointment` |
| `TenantMiddleware` (JWT → header) | Ruta pública por slug sin tenant en header (`/public/{tenant_slug}`) |
| Patrón Router → Service → Repository | `AvailabilityService` — cálculo de slots y solapamiento |
| JWT access/refresh, `require_role()` | Roles: `business_admin`, `staff`, `client` (sin `guard`) |
| Interceptors Angular, `StorageService`, signals | Página pública de reserva tipo Calendly |
| Brand system `--nx-*` / PrimeNG preset | Notificaciones de recordatorio (no solo confirmación) |
| Deploy GCP (Cloud Run + Firebase) vía `release.py` | — |
