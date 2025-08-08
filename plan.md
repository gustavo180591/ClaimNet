# ClaimNet — plan.md
Guía paso a paso para llevar ClaimNet desde local a producción, con calidad, seguridad y métrricas. Copiá/pegá cada bloque y marcá los checkboxes.

---

## 0) Requisitos previos (local)
- [x] Node 20+ y PNPM/NPM
- [ ] Docker + Docker Compose
- [ ] Git configurado
- [ ] OpenSSL (para generar secretos)
- [ ] Cuenta en Vercel / Fly.io / Railway y en un proveedor de DB gestionada

---

## 1) Bootstrap del repo
- [x] `git init` y **repo privado** en GitHub/GitLab.
- [x] Crear estructura base SvelteKit 2 + Tailwind CSS 4.
- [ ] Agregar Prisma y configurar PostgreSQL.
- [x] Añadir ESLint + Prettier + Husky (pre-commit).
- [x] Crear `README.md`, `LICENSE`, `CODE_OF_CONDUCT.md` (opcional).

```bash
npm create svelte@latest claimnet
cd claimnet
npm i -D @tailwindcss/postcss tailwindcss postcss autoprefixer eslint prettier husky
npm i @prisma/client zod
npm i -D prisma
npx prisma init --datasource-provider postgresql
```

---

## 2) Estructura de carpetas (target)
```
src/
  lib/
    server/
      db.ts
      auth.ts
      rate-limit.ts
      storage.ts
      audit.ts
      redact.ts
      mailer.ts
    schemas/
      claim.ts
    components/
    utils/seo.ts
  routes/
    +layout.svelte
    +layout.server.ts
    +page.svelte
    claims/
      +page.svelte
      new/+page.svelte
      new/+page.server.ts
      [id]/+page.svelte
      [id]/+page.server.ts
    admin/
      +layout.server.ts
      login/+page.svelte
      login/+page.server.ts
      claims/+page.svelte
      claims/+page.server.ts
      claims/[id]/+page.svelte
      claims/[id]/+page.server.ts
      stats/+page.svelte
    api/
      health/+server.ts
      version/+server.ts
      export.csv/+server.ts
```

---

## 3) Variables de entorno
- [x] Crear `.env.example` con:
```
DATABASE_URL=postgresql://postgres:postgres@db:5432/claimnet
AUTH_SECRET=change-me
ADMIN_USER=admin
ADMIN_PASS=superseguro
RATE_LIMIT=10:60
STORAGE_PATH=./storage
PUBLIC_BASE_URL=http://localhost:5173
TZ=America/Argentina/Buenos_Aires
COMMIT_SHA=dev
```
- [ ] Copiar a `.env` en local y completar.

---

## 4) Docker (local)
- [ ] Crear `docker-compose.yml` mínimo:
```yaml
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: claimnet
      TZ: ${TZ:-America/Argentina/Buenos_Aires}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports: ["5432:5432"]
volumes:
  pgdata:
```
- [ ] `docker compose up -d`
- [ ] `npx prisma migrate dev`

---

## 5) Base de datos (Prisma)
- [ ] Definir modelos y enums (User, Reporter, Claim, Attachment, Assignment, Note, Tag, ClaimTag, AuditLog).
- [ ] Índices: `status, category, createdAt, search(tsvector)`.
- [ ] Scripts de migración/seed.

```bash
npx prisma migrate dev --name init
npx prisma db seed
```

**Seed inicial (`prisma/seed.ts`):**
- [ ] Crear usuario admin (de `.env`).
- [ ] Cargar categorías base y 10 reclamos demo.
- [ ] Generar `trackingToken` por `Reporter`.

---

## 6) Librerías utilitarias
- [ ] `src/lib/server/db.ts`: Prisma client singleton.
- [ ] `src/lib/server/rate-limit.ts`: LRU/Redis (interface `limit(key, max, windowSec)`).
- [ ] `src/lib/server/storage.ts`: adapter FS/S3 (guardar por `sha256.ext`).
- [ ] `src/lib/server/audit.ts`: registro de acciones.
- [ ] `src/lib/server/redact.ts`: `redactPII`, `redactClaimForPublic`.
- [ ] `src/lib/utils/seo.ts`: helpers de OG/SEO.

---

## 7) Validación y formularios (Zod + actions)
- [ ] `src/lib/schemas/claim.ts` con validaciones de negocio.
- [ ] `src/routes/claims/new/+page.svelte`: UI del form con accesibilidad.
- [ ] `src/routes/claims/new/+page.server.ts`: `actions` + `rateLimit` + `storage.saveAll()` + creación de Claim + Audit.
- [ ] Adjuntos múltiples: validar tamaño/MIME y mostrar previews.

---

## 8) Home (público)
- [ ] `src/+page.svelte`: buscador + categorías.
- [ ] SSR habilitado, metadatos OG básicos.
- [ ] Listado (si aplica) con redacción de PII para anónimos.

---

## 9) Detalle y seguimiento del reclamo (público)
- [ ] `routes/claims/[id]/+page.svelte` + `+page.server.ts`:
  - [ ] Mostrar estado: Enviado → Recibido → En proceso → Resuelto/Rechazado.
  - [ ] Si hay `trackingToken`, permitir ver datos no públicos del reporter.
  - [ ] No exponer adjuntos privados públicamente; usar enlaces firmados si corresponde.

---

## 10) Panel /admin
- [ ] Guard en `admin/+layout.server.ts` (Auth básica segura o Lucia+2FA).
- [ ] Login (`/admin/login`) con rate limit y bloqueo por IP opcional.
- [ ] Bandeja:
  - [ ] Filtros por categoría/estado/fecha, búsqueda y orden.
  - [ ] Paginación en server (DataTable).
  - [ ] Export CSV (endpoint `/api/export.csv`).

- [ ] Detalle de reclamo:
  - [ ] Asignación a operador/responsable.
  - [ ] Cambios de estado con AuditLog.
  - [ ] Notas internas (checkbox `internal=true`).
  - [ ] Etiquetas (CRUD Tag + ClaimTag).
  - [ ] Descarga de evidencias (endpoint protegido).

- [ ] Stats (`/admin/stats`):
  - [ ] Volumen por categoría (últimos 90 días).
  - [ ] Tiempos de resolución p50/p90.
  - [ ] Cumplimiento SLA (72h por defecto).

---

## 11) Seguridad & privacidad
- [ ] HTTPS siempre (producción).
- [ ] CSRF: tokens `double-submit` en formularios sensibles.
- [ ] Rate limit global y por acción.
- [ ] Sanitización de HTML en vistas (no inyectar crudo).
- [ ] Subidas: validar MIME/tamaño, calcular `sha256`, almacenar por hash.
- [ ] Headers duros en `hooks.server.ts`: CSP, X-Frame-Options, Referrer-Policy, Permissions-Policy.
- [ ] Logs sin PII; auditoría centralizada.
- [ ] Términos y Privacidad: `/legal/terms`, `/legal/privacy`.

---

## 12) Accesibilidad y UI (Tailwind 4)
- [ ] Focus visible, `aria-*` correcto, contraste AA.
- [ ] Componentes reutilizables: `Form`, `DataTable`, `Modal`, `Toast`, `Tag`, `BadgeEstado`.
- [ ] OG tags específicos por reclamo y `sitemap.xml`/`robots.txt`.

---

## 13) Hooks globales
- [ ] `sequence(security, limiter)` en `src/hooks.server.ts`.
- [ ] Filtrado de headers serializados para SSR.
- [ ] Rate limiting por ruta crítica.

---

## 14) Endpoints de salud y versión
- [ ] `/api/health`: `200 {status:'ok', time}`.
- [ ] `/api/version`: leer `COMMIT_SHA` del env.
- [ ] Integrar con monitor (UptimeRobot/Healthchecks).

---

## 15) Tests
- [ ] **Unitarios (Vitest)**: `redact.ts`, `rate-limit.ts`, `audit.ts`, utils.
- [ ] **E2E (Playwright)**:
  - [ ] Alta de reclamo (con/ sin adjuntos).
  - [ ] Cambio de estado en admin.
  - [ ] Filtrado/búsqueda en bandeja.
  - [ ] Export CSV descarga OK.
- [ ] **CI (GitHub Actions)**:
  - [ ] Lint + typecheck + vitest.
  - [ ] Correr Playwright en contenedor con Postgres de servicio.

---

## 16) DX
- [ ] Scripts en `package.json`:
```json
{
  "scripts": {
    "dev": "vite dev",
    "dev:docker": "docker compose up -d db && vite dev --host",
    "build": "vite build && prisma generate",
    "preview": "vite preview",
    "migrate": "prisma migrate deploy",
    "seed": "prisma db seed",
    "lint": "eslint .",
    "test": "vitest",
    "e2e": "playwright test"
  }
}
```
- [ ] Husky: `pre-commit` con `lint-staged` (formateo + lint).

---

## 17) Deploy (elige uno)
### Vercel
- [ ] Crear proyecto, setear envs (`DATABASE_URL`, `AUTH_SECRET`, etc.).
- [ ] Build & output SvelteKit (adaptador Vercel).
- [ ] DB gestionada (Neon/Supabase/Planetscale si MySQL).
- [ ] Post-deploy: `prisma migrate deploy`.

### Fly.io
- [ ] `fly launch` (Dockerfile).
- [ ] Secrets: `fly secrets set AUTH_SECRET=...`.
- [ ] Volúmenes solo si usás FS local; preferir S3.
- [ ] Health checks y auto-restart.

### Railway
- [ ] Template 1-click o proyecto manual.
- [ ] Variables de entorno y servicio DB.
- [ ] Deploy + migraciones.

---

## 18) Storage de adjuntos
- [ ] FS local en dev: `STORAGE_PATH=./storage` (gitignored).
- [ ] Producción: S3 compatible (Backblaze/Wasabi/MinIO).
- [ ] Subir por hash, guardar metadatos (path, mime, size, sha256, kind).
- [ ] Endpoints firmados para descarga privada.

---

## 19) Observabilidad
- [ ] Logs estructurados (JSON) con nivel (`info|warn|error`).
- [ ] Métricas: requests, latencia, tasa de error, throughput.
- [ ] Alertas: caídas de `/api/health`, errores 5xx por minuto.

---

## 20) Checklist de “Go-Live”
- [ ] Revisión de seguridad (headers, CSRF, rate limit, secretos rotados).
- [ ] Backups de DB programados.
- [ ] Roles/usuarios creados (ADMIN/OPERATOR).
- [ ] Pruebas E2E verdes en CI.
- [ ] Documentación final del README (setup, migraciones, seeds, troubleshooting).
- [ ] Plan de rollback documentado.

---

## 21) Roadmap (después del MVP)
- [ ] Autenticación con Lucia + 2FA para admin.
- [ ] Vistas públicas opcionales con RLS y cache.
- [ ] Webhooks/Emails de estado al denunciante (opt-in).
- [ ] Integración con GIS (heatmaps de reclamos).
- [ ] Modo multi-tenant.

---

## 22) Comandos de referencia
```bash
# Local (con DB en Docker)
docker compose up -d
cp .env.example .env
npx prisma migrate dev
npm run dev

# Tests
npm run test
npm run e2e

# Build
npm run build

# Producción (post-deploy)
npm run migrate
```

---

## 23) Tabla de estados y workflow
- Enviado → Recibido → En proceso → Resuelto / Rechazado
- Reglas:
  - [ ] Solo ADMIN/OPERATOR pueden cambiar estado.
  - [ ] Registrar `AuditLog` en cada transición (prev/next/actor).
  - [ ] `RESOLVED` exige nota de cierre.

---

## 24) SQL de métricas (Stats)
- [ ] Volumen por categoría (90 días)
- [ ] p50/p90 de resolución
- [ ] SLA 72h

*(Ver queries preparadas en `/admin/stats` service)*

---

**Listo.** Con este plan, clonás, corrés migraciones, levantás con Docker y en 1–2 días cerrás MVP con admin funcional, export, auditoría y métricas básicas.
