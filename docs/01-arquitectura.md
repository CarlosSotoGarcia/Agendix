# 01 — Arquitectura y Estrategia Multi-Tenant (Agendix)

> Base técnica extraída de Zity (mismo stack Novex) **con una divergencia deliberada**: Zity persiste en MongoDB (Motor); Agendix persiste en **Cloud Firestore (Native Mode)**. La razón es puramente de costo/operación: al desplegar en GCP (Cloud Run + Firebase Hosting), Firestore vive en el mismo proyecto, sin cuenta externa, sin cadena de conexión que proteger, con un free tier permanente (no expira como una prueba) y con IAM/Application Default Credentials en vez de whitelist de IP. Todo lo demás —capas Router → Service → Repository, roles, JWT, Angular, deploy vía `release.py`— se mantiene idéntico entre ambos productos Novex (ver comparación completa en [§1.9](#19-qué-se-reutiliza-de-zity-vs-qué-es-nuevo)).
>
> El dominio también se adaptó: los "tenants" aquí son **negocios** (barberías, spas, consultorios) en vez de colonias, y el flujo central es **reservar una cita**, no verificar un acceso.

## 1.1 Decisión de Arquitectura de Tenancy

Se adopta la estrategia **"Un solo proyecto Firestore + subcolección por tenant"**:

- Un único proyecto GCP, una única base de datos Firestore (Native Mode, `(default)`).
- Cada negocio (`tenant`) es un documento en la colección raíz `tenants/{tenant_id}`.
- Todos los datos operativos del negocio (servicios, staff, horarios, citas, notas, recompensas) viven como **subcolecciones** de ese documento: `tenants/{tenant_id}/appointments/{appointment_id}`, `tenants/{tenant_id}/services/{service_id}`, etc.

### Justificación (vs. discriminador `tenant_id` estilo Mongo)

| Criterio | Subcolección por tenant (Firestore) | Campo `tenant_id` discriminador (Mongo, usado en Zity) |
|---|---|---|
| Costo inicial | $0 (free tier permanente: 1 GiB, 50K lecturas/día, 20K escrituras/día) | $0 (Atlas M0, 512 MB, sin límite de días pero sí de tamaño) |
| Aislamiento físico entre tenants | Garantizado por la **ruta del documento** (`tenants/A/...` nunca puede leer `tenants/B/...` por accidente de query) | Garantizado por **disciplina de código**: toda query debe incluir `tenant_id` en el filtro |
| Cuentas/credenciales a administrar | Ninguna — mismo proyecto GCP que Cloud Run, autenticación vía IAM/Service Account | Cuenta MongoDB Atlas separada, connection string como secreto, whitelist de IP |
| Índices compuestos necesarios | Menos — el `tenant_id` no participa como campo de índice (ya está en la ruta) | Más — todo índice compuesto debe empezar por `tenant_id` |
| Modelo de consultas | Sin agregaciones tipo `$lookup`/pipeline; consultas por colección + filtros simples, `in`, rango | Rico (aggregation pipeline), pero no se usa a fondo en el diseño actual de Agendix |
| Migración a aislamiento físico total (DB-per-tenant) | Ya es el nivel más fuerte posible sin salir de Firestore | Requiere migrar a cluster dedicado por tenant |

Firestore no ofrece agregaciones complejas ni transacciones multi-documento tan flexibles como Mongo, pero el dominio de Agendix (CRUD + validación de solapamiento + contadores de puntos) no las necesita. A cambio, se gana aislamiento físico por tenant de fábrica y cero infraestructura externa que mantener.

---

## 1.2 Identificación del Tenant en FastAPI

El `tenant_id` se resuelve en el mismo orden conceptual que Zity — la diferencia es que, al resolverse, determina la **ruta de subcolección** que usará el Repository, no un valor de filtro:

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

Este middleware es **idéntico** al de Zity: solo determina *cuál* es el tenant. La diferencia con Mongo aparece un nivel más abajo, en el Repository (§1.4), que traduce ese `tenant_id` en `db.collection("tenants").document(tenant_id).collection(...)` en vez de en un filtro `{"tenant_id": tenant_id}`.

### 1.2.1 Resolución de slug → tenant_id (página pública de reservas)

Agendix necesita, además, una **página pública de reservas** (`agendix.app/:tenantSlug`, estilo Calendly) donde un cliente anónimo ve disponibilidad y agenda sin cuenta previa. Esas rutas viven bajo `/api/v1/public/{tenant_slug}/...`, quedan fuera del `TenantMiddleware` y resuelven el tenant explícitamente en el router.

Para que esa resolución sea **una sola lectura de documento** (no una query, ni siquiera con índice) se mantiene una colección raíz de lookup separada del documento de negocio:

```
tenant_slugs/{slug}          →  { tenant_id: "abc123" }
tenants/{tenant_id}          →  { name, slug, vertical, status, timezone, branding, ... }
```

```python
# agendix/repositories/tenant_repository.py
class TenantRepository:
    async def find_id_by_slug(self, slug: str) -> str | None:
        doc = await self._db.collection("tenant_slugs").document(slug).get()
        return doc.get("tenant_id") if doc.exists else None
```

Esta indirección (`tenant_slugs` como colección puente) es la razón por la que **cambiar el slug de un negocio** (RF-TEN-002) es una operación administrativa explícita — reescribe el documento de lookup — y no un rename trivial de un ID de documento (Firestore no permite renombrar el ID de un documento con subcolecciones sin recrear todo el árbol).

Al confirmar la reserva pública se crea (o reutiliza, por email/teléfono) un `client` mínimo en `tenants/{tenant_id}/users/{user_id}`, sin contraseña obligatoria en el MVP.

### Dependencia Inyectable

```python
# agendix/dependencies/tenant.py

from fastapi import Request

def get_tenant_id(request: Request) -> str:
    return request.state.tenant_id
```

---

## 1.3 Modelo de Datos y Colecciones en Firestore

```
tenants/{tenant_id}                              # documento raíz del negocio
  ├─ (campos: name, slug, vertical, status, timezone,
  │           cancellation_policy_hours, loyalty_config, branding)
  ├─ users/{user_id}                             # business_admin, staff, client de ESTE tenant
  ├─ services/{service_id}
  ├─ staff/{staff_id}
  ├─ schedules/{schedule_id}
  ├─ appointments/{appointment_id}
  ├─ notifications_log/{log_id}
  ├─ client_notes/{note_id}
  ├─ loyalty_accounts/{client_user_id}           # 1 doc por cliente, ID = user_id (lectura O(1) de saldo)
  ├─ loyalty_transactions/{transaction_id}       # ID determinístico p/idempotencia, ver 06-trazabilidad
  ├─ rewards_catalog/{reward_id}
  └─ redemptions/{redemption_id}

tenant_slugs/{slug}                              # lookup raíz, ver §1.2.1
platform_admins/{user_id}                        # super_admin — global, fuera de cualquier tenant
```

### Índices compuestos requeridos (`firestore.indexes.json`)

A diferencia de Mongo, el `tenant_id` **no** participa como primer campo del índice — ya está resuelto por la ruta de la subcolección. Solo se declaran los campos que realmente combinan filtros/orden dentro de una misma colección:

```json
{
  "indexes": [
    { "collectionGroup": "appointments", "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "staff_id", "order": "ASCENDING" },
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "start_time", "order": "ASCENDING" }
      ] },
    { "collectionGroup": "appointments", "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "client_id", "order": "ASCENDING" },
        { "fieldPath": "start_time", "order": "DESCENDING" }
      ] },
    { "collectionGroup": "schedules", "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "staff_id", "order": "ASCENDING" },
        { "fieldPath": "day_of_week", "order": "ASCENDING" }
      ] },
    { "collectionGroup": "loyalty_transactions", "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "client_id", "order": "ASCENDING" },
        { "fieldPath": "created_at", "order": "DESCENDING" }
      ] }
  ]
}
```

`queryScope: COLLECTION` hace que cada índice aplique a **cualquier** subcolección con ese nombre bajo cualquier `tenants/{tenant_id}` — se declara una vez y sirve para todos los negocios, presentes y futuros. Los campos filtrados por igualdad simple (`email`, `status`, `is_sensitive`) usan el índice automático de campo único que Firestore crea por defecto, sin declaración manual.

El índice `(staff_id, status, start_time)` de `appointments` es el más sensible: sobre él corre la verificación de solapamiento de horarios (ver §1.4).

### Reglas de seguridad (Firestore Security Rules)

El frontend Angular **nunca habla directo con Firestore** — todo pasa por FastAPI (Router → Service → Repository), igual que con Mongo. El backend usa el **Admin SDK** con credenciales de Service Account, que por diseño **bypassa** las Security Rules. Por eso la responsabilidad de aislamiento sigue viviendo en la capa Service/Repository (RNF-SEC-003), exactamente igual que en Zity — Firestore Security Rules se despliegan igualmente como **backstop** de defensa en profundidad, por si en el futuro se habilita algún acceso directo cliente→Firestore (ej. listeners en tiempo real):

```
// firestore.rules (backstop — el acceso normal va vía backend con Admin SDK)
match /tenants/{tenantId}/{document=**} {
  allow read, write: if false; // todo acceso real pasa por FastAPI, no por el cliente
}
```

---

## 1.4 Arquitectura en Capas: Router → Service → Repository

Misma separación estricta de Zity. Ninguna capa se salta la inmediatamente inferior.

```
┌─────────────────────────────────────────────────────────┐
│                       ROUTER                            │
│  • Recibe y valida el request HTTP (Pydantic schemas)   │
│  • Ejecuta guards de autenticación y RBAC                │
│  • Delega la lógica al Service correspondiente          │
│  • NO conoce Firestore, NO construye referencias de doc  │
└─────────────────────┬───────────────────────────────────┘
                      │ llama a
┌─────────────────────▼───────────────────────────────────┐
│                      SERVICE                            │
│  • Contiene toda la lógica de negocio                   │
│  • AvailabilityService calcula slots libres por staff    │
│  • AppointmentService valida solapamiento antes de crear │
│  • NO importa google.cloud.firestore directamente        │
└─────────────────────┬───────────────────────────────────┘
                      │ llama a
┌─────────────────────▼───────────────────────────────────┐
│                    REPOSITORY                           │
│  • Único punto de contacto con Firestore (Admin SDK)     │
│  • Construye siempre la ruta tenants/{tenant_id}/...     │
│  • Convierte documentos Firestore ↔ modelos Pydantic     │
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
        # create() corre dentro de una transacción Firestore: valida solapamiento
        # y escribe el documento en la misma transacción, cerrando la condición de
        # carrera que existiría con un simple "leer, luego escribir".
        appointment = await self.repo.create_if_available(data, tenant_id)
        if appointment is None:
            raise HTTPException(status_code=409, detail="El horario ya no está disponible")
        asyncio.create_task(self._notify_confirmation(appointment))  # fire-and-forget
        return appointment

# CAPA 3 — Repository (agendix/repositories/appointment_repository.py)
class AppointmentRepository:
    def __init__(self, db: firestore.AsyncClient):
        self._db = db

    def _collection(self, tenant_id: str):
        return self._db.collection("tenants").document(tenant_id).collection("appointments")

    async def create_if_available(self, data: AppointmentCreate, tenant_id: str):
        @firestore.async_transactional
        async def _txn(transaction):
            query = (
                self._collection(tenant_id)
                .where("staff_id", "==", data.staff_id)
                .where("status", "in", ["pending", "confirmed"])
                .where("start_time", "<", data.end_time)
            )
            # Firestore permite un número limitado de filtros de desigualdad combinados;
            # el cruce final (end_time > start) se valida en memoria sobre un result set
            # ya acotado por staff_id + status + start_time (pocos documentos).
            candidates = [doc async for doc in query.get(transaction=transaction)]
            overlaps = any(c.get("end_time") > data.start_time for c in candidates)
            if overlaps:
                return None
            doc_ref = self._collection(tenant_id).document()
            transaction.set(doc_ref, data.model_dump())
            return doc_ref

        transaction = self._db.transaction()
        doc_ref = await _txn(transaction)
        return await doc_ref.get() if doc_ref else None
```

La regla de negocio central de Agendix — **no permitir dos citas solapadas para el mismo staff** — sigue viviendo en `AvailabilityService`/`AppointmentRepository`, nunca en el router. La transacción de Firestore es, de hecho, una garantía **más fuerte** contra condiciones de carrera que el `find_one` + `insert` no transaccional que usaba la versión sobre Mongo.

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
                           │ google-cloud-firestore (Admin SDK, async)
                ┌──────────▼──────────┐
                │  Cloud Firestore    │
                │  (Native Mode)      │
                │   ├─ tenant_slugs   │
                │   ├─ platform_admins│
                │   └─ tenants/{id}/  │
                │       ├─ users          │
                │       ├─ services       │
                │       ├─ staff          │
                │       ├─ schedules      │
                │       ├─ appointments   │
                │       ├─ notifications_log │
                │       ├─ client_notes   │
                │       └─ loyalty_*      │
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
│   │   └── client.py                  # firestore.AsyncClient singleton + get_firestore()
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
│   │   ├── base_repository.py         # Helpers comunes de referencia a subcolección de tenant
│   │   ├── tenant_repository.py       # + find_id_by_slug() vía tenant_slugs
│   │   ├── user_repository.py
│   │   ├── service_repository.py
│   │   ├── staff_repository.py
│   │   ├── schedule_repository.py
│   │   ├── appointment_repository.py  # create_if_available() transaccional
│   │   └── notification_log_repository.py
│   ├── services/
│   │   ├── auth_service.py            # Login (requiere slug/tenant), refresh, hashing
│   │   ├── appointment_service.py      # Reglas de reserva, orquesta repos
│   │   ├── availability_service.py     # Cálculo de slots libres + detección de solapamiento
│   │   ├── notification_service.py     # Confirmaciones y recordatorios (fire-and-forget)
│   │   ├── service_service.py          # CRUD de catálogo de servicios
│   │   └── staff_service.py            # CRUD de staff y sus horarios
│   └── routers/
│       ├── auth.py                    # /auth/login (slug + email + password), /auth/refresh, /auth/me
│       ├── appointments.py            # /appointments CRUD, /appointments/{id}/cancel
│       ├── public.py                  # /public/{tenant_slug}/availability, /public/{tenant_slug}/book
│       └── admin/
│           ├── services.py            # /admin/services CRUD
│           ├── staff.py               # /admin/staff CRUD
│           └── schedules.py           # /admin/schedules CRUD
├── firestore.rules                    # Security Rules (backstop, ver §1.3)
├── firestore.indexes.json             # Índices compuestos (ver §1.3)
├── scripts/
│   └── seed.py                        # Datos iniciales de desarrollo (contra el emulador)
├── tests/
│   ├── conftest.py                    # Fixture que apunta a Firestore Emulator, no a prod
│   ├── test_auth.py
│   ├── test_appointments.py
│   ├── test_availability.py           # casos de solapamiento y límites de horario
│   └── test_multitenancy.py           # intentos de fuga cross-tenant (deben fallar)
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

**Nota de login:** a diferencia de un sistema con lookup global por email, el login de Agendix pide **slug del negocio + email + password** (como "workspace" en Slack), porque cada `user` vive dentro de la subcolección `tenants/{tenant_id}/users` de su propio negocio. Esto evita necesitar una collection-group query (más cara y con índice adicional) para ubicar en qué tenant vive un email, y es coherente con que la propia página pública de reserva ya usa el slug como punto de entrada.

---

## 1.7 Frontend (Angular 20 + PrimeNG)

Misma convención que Zity: componentes **standalone**, rutas lazy-loaded, dos interceptors (`auth`, `tenant`), `StorageService` para `access_token`/`refresh_token`/`tenant_id`/`tenant_slug`, signals para estado reactivo. El frontend consume **exclusivamente** la API de FastAPI — nunca el SDK de Firestore directamente (ver §1.3).

**Ruta pública nueva propia de Agendix**: `/:tenantSlug/reservar` — página de reserva sin `authGuard`, consume `/api/v1/public/{tenant_slug}/*`. Es el equivalente al `LandingComponent` público de Zity, pero interactivo (selecciona servicio → staff → slot → confirma).

**Login**: `/:tenantSlug/login` (o selector de negocio si el usuario no llega desde un link directo), consistente con el modelo de login por slug de §1.6.

### Sistema de marca Novex

Reutilizar `novex-brand.css` (mismo archivo fuente que Zity, copiado desde `novexlandig/assets/novex-brand.css`) y definir un preset propio `NovexAgendixPreset` en `app.config.ts` sobre Aura, igual que `NovexZityPreset`. Se recomienda un acento distinto al azul de Zity para diferenciar el producto en la familia Novex (p. ej. cian/teal, consistente con el mockup de "Novex Book"), manteniendo los tokens `--nx-navy-*` para dark mode y `--nx-success/error/warning` sin cambios.

---

## 1.8 Variables de Entorno Requeridas

```env
# .env.example
GOOGLE_CLOUD_PROJECT=agendix-prod                # también auto-detectado en Cloud Run
FIRESTORE_EMULATOR_HOST=                         # ej. localhost:8080 — solo en desarrollo local

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

No existe `MONGODB_URL` ni `MONGODB_DB_NAME`: la conexión a Firestore en producción no usa cadena de conexión ni secreto — el contenedor de Cloud Run se autentica automáticamente vía la **Service Account** adjunta al servicio (Application Default Credentials). En desarrollo local, `FIRESTORE_EMULATOR_HOST` redirige el SDK al **Firestore Emulator** (parte del Firebase CLI, gratuito y offline) en vez de a un proyecto real.

Tampoco hay equivalente a `QR_SIGNING_SECRET` (Agendix no firma códigos QR); en cambio, en fases posteriores se añadirán credenciales de proveedor de SMS/WhatsApp para recordatorios (`TWILIO_*` o similar).

---

## 1.9 Qué se reutiliza de Zity vs. qué es nuevo

| Reutilizado tal cual | Nuevo / específico de Agendix |
|---|---|
| `TenantMiddleware` (JWT → header) | **Persistencia: Cloud Firestore (Native Mode) en vez de MongoDB/Motor** |
| Patrón Router → Service → Repository | Modelo de tenancy: subcolección por tenant en vez de discriminador `tenant_id` |
| JWT access/refresh, `require_role()` | Modelo de dominio: `service`, `staff`, `schedule`, `appointment` |
| Interceptors Angular, `StorageService`, signals | Ruta pública por slug sin tenant en header (`/public/{tenant_slug}`) + lookup `tenant_slugs` |
| Brand system `--nx-*` / PrimeNG preset | `AvailabilityService` — cálculo de slots y solapamiento, ahora con transacción Firestore |
| Deploy GCP (Cloud Run + Firebase) vía `release.py` | Roles: `business_admin`, `staff`, `client` (sin `guard`) |
| RBAC a nivel de router/dependencia | Página pública de reserva tipo Calendly |
| — | Notificaciones de recordatorio (no solo confirmación) |
| — | Login por slug de negocio (no lookup global por email) |

> **Por qué diverge el motor de datos entre productos Novex:** Zity ya estaba construido sobre Mongo antes de que Agendix existiera, y migrar Zity no está en alcance. Para un producto **nuevo** desplegado desde el día uno en GCP, Firestore elimina una cuenta externa (Atlas), un secreto de conexión y una whitelist de IP, a cambio de un modelo de consultas más simple que el dominio de Agendix no necesita superar. Ver el detalle de costos y fases en [07-plan-implementacion.md](./07-plan-implementacion.md).
