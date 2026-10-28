# Guía base — monolito Node/Express + React + PostgreSQL desde cero

> **Qué es esto:** una guía para construir, desde cero, una aplicación web monolítica con **Node + Express + React + PostgreSQL**, con estructura de proyecto, patrones de código, línea base de seguridad, estrategia de pruebas y un método de trabajo que deja el proyecto mantenible.
> **Cómo se usa:** léela de arriba abajo la primera vez; úsala como plantilla (copia de carpetas y snippets) y como checklist de decisiones para cada feature nueva.
> **Versión revisada:** incorpora las correcciones de seguridad y operación detectadas en la revisión: GCM con *auth tag*, sesiones hasheadas, rate limit, pruebas contra base `*_test`, guardias de esquema y checklist de producción.

---

## 1. Alcance y principios

Esta guía asume que quieres:

- **Una sola aplicación** (API + web) que se despliega junta, sin microservicios ni CI complejo.
- **Una base de datos PostgreSQL** y acceso con Sequelize (modelos, asociaciones y migraciones).
- **Un panel privado** (autenticado) y, si aplica, una **landing pública**.
- **Multi-tenant**: el cliente es una organización, y cada persona es un usuario con un rol dentro de ella.
- **Integraciones externas que no bloquean el desarrollo**: sin credenciales, la app funciona en modo simulado.
- **Seguridad y operación desde el día uno**: sesiones, rate limit, auditoría, backups y restauración no son “después”.
- **Cambios siempre con evidencia**: pruebas, build y verificación del flujo real.

### Los doce principios que sostienen el proyecto

1. **ORM con Sequelize.** Los modelos y las migraciones viven en Sequelize; el esquema sigue siendo código (migraciones versionadas) y los servicios usan los modelos para CRUD/relaciones, reservando SQL explícito (`sequelize.query`) para reportes, agregaciones o bulk donde el ORM no aporta.
2. **El esquema es código.** Las migraciones de Sequelize son la fuente de verdad y se versionan en el repositorio. Los cambios aditivos van en migraciones; las transformaciones de datos van en migraciones con lógica o scripts `migrate-*.js` con `--dry-run`/`--apply`. En producción nunca se usa `sequelize.sync()`.
3. **La conexión de Sequelize es perezosa y está configurada.** La instancia no abre conexión hasta la primera consulta: los tests pueden fijar la base antes de usarla. Configura límites de pool, timeouts y SSL explícitos.
4. **Las capas no se mezclan**: `routes` (endpoints + guards) → `controllers` (HTTP) → `services` (reglas y modelos) → `db`/`models` (datos).
5. **El tenant sale de la sesión**, nunca del cuerpo de la petición. Si un cliente manda un identificador de organización distinto al suyo, se rechaza.
6. **Cero secretos en el código.** Todo pasa por configuración; los secretos se cifran con **AES-256-GCM autenticado** (IV + *auth tag*) y una clave derivada con KDF, y la API solo devuelve pistas enmascaradas.
7. **Todo proveedor externo tiene modo simulado** y se activa por configuración.
8. **Nada se envía en el momento**: los mensajes y avisos pasan por una bandeja con estado, reintento y *dead letter*.
9. **Los trabajos en segundo plano son idempotentes** (clave de deduplicación con ámbito por organización), así que pueden correr cada hora sin duplicar nada.
10. **Nada se marca hecho sin evidencia**: pruebas en verde, build correcto, verificación del flujo real y, si aplica, migración y restauración probadas.
11. **El servidor es la única autoridad.** Los guards del cliente son UX; nunca autorizan. Toda ruta privada valida sesión, rol y organización en el servidor.
12. **Dinero y datos personales se auditan y se pueden borrar/exportar.** Los eventos de cobro, autenticación y acceso del admin quedan en un registro durable; la retención y el borrado están definidos en la spec.

---

## 2. Stack

| Capa | Tecnología | Versión | Nota |
|---|---|---|---|
| Runtime | Node.js | ^20.19.0 \|\| >=22.12.0 | ESM nativo (`"type": "module"`); `--env-file` disponible |
| Monorepo | npm workspaces | — | un solo `node_modules` raíz |
| API | Express | ^5.3 | routers por dominio, ORM Sequelize |
| Base de datos | PostgreSQL | >= 14 | acceso vía Sequelize (dialecto `postgres`) |
| ORM | Sequelize | ^6.37 | modelos, asociaciones, migraciones y transacciones |
| Frontend | React | ^19.3 | SPA con Vite |
| Build | Vite | ^8.3 | Rolldown + Oxc; `codeSplitting.groups` por familias de dependencias |
| Estilos | Tailwind CSS | ^4.3 | vía `@tailwindcss/vite` (sin `postcss.config`) |
| Componentes | Base UI (`@base-ui/react`) | ^1.9 | *headless*: tú controlas el diseño |
| Iconos | lucide-react | ^1.59 | |
| Routing | react-router-dom | ^7.18 | |
| PWA (opcional) | vite-plugin-pwa | ^2.0 | precache del shell; **sin cache de `/api`** |
| PDF (opcional) | PDFKit | ^0.20 | documentos imprimibles desde el servidor |
| Cabeceras | helmet | ^8.3 | CSP con `upgrade-insecure-requests` desactivado en local |
| Pruebas | `node --test` | nativo | sin jest/vitest ni dependencias de test |
| Método | Markdown + Gherkin + JSON | — | `/specs`, `/plans`, `/features`, `/receipts` |

**Dependencias del servidor:** `express`, `helmet`, `sequelize`, `pg` (driver del dialecto) (+ `pdfkit` si hay documentos). Todo lo demás (hash de contraseñas, cifrado de secretos, cookies, validación, rate limit) se hace con la librería estándar. Desarrollo: `sequelize-cli`.

**Dependencias del cliente:** `react`, `react-dom`, `react-router-dom`, `@base-ui/react`, `lucide-react`, `clsx`, `tailwind-merge`, `class-variance-authority`. En desarrollo: `vite`, `@vitejs/plugin-react`, `tailwindcss`, `@tailwindcss/vite`, `vite-plugin-pwa`; en la raíz, `concurrently`.

**Sin CORS:** en producción todo es el mismo origen y en desarrollo Vite hace proxy de `/api`. No añadas un `cors` permisivo “por si acaso”.

> Las versiones de la tabla son mínimos de referencia. Fija el `package-lock.json`, compruébalas con `npm view` (especialmente `@base-ui/react`, `vite-plugin-pwa` y `pdfkit`) y declara `engines` en `package.json`. Vite 8 exige Node `^20.19.0 || >=22.12.0`; `--env-file` sigue disponible y simplifica las pruebas.

---

## 3. Estructura del repositorio

```text
app/
├── AGENTS.md                 # reglas para agentes: método, evidencia, convenciones
├── README.md                 # documentación de producto para humanos
├── .env.example              # variables documentadas, sin secretos reales
├── .nvmrc                    # Node 20.19+ o 22.12+ (requisito de Vite 8)
├── package.json              # workspaces + scripts orquestadores
├── specs/                    # SDD + contexto de negocio (ODD)
│   ├── negocio.md            # modelo de negocio, palancas, ciclo del dinero
│   ├── <feature>.md          # spec: reglas S#, criterios AC#, contratos
│   └── README.md             # índice y reglas del método
├── plans/                    # TDD: tareas T#, pruebas, evidencia, log
│   ├── <feature>.md
│   └── README.md
├── features/                 # BDD: escenarios Gherkin en español con @s#
│   ├── <feature>.feature
│   └── README.md
├── receipts/                 # RDD: un recibo JSON por tarea T#
│   ├── T1.json
│   └── README.md             # esquema, reglas y ejemplos del recibo
├── server/
│   ├── package.json
│   ├── src/
│   │   ├── index.js          # listen + apagado ordenado
│   │   ├── app.js            # instancia de Express, middlewares, routers
│   │   ├── config/env.js     # carga de .env sin dependencias
│   │   ├── db/
│   │   │   ├── index.js      # instancia Sequelize + conexión + resetSchema() (solo test)
│   │   │   └── migrations/   # migraciones versionadas (sequelize-cli/umzug)
│   │   ├── models/           # modelos Sequelize (uno por entidad) + index.js (asociaciones)
│   │   ├── seeders/          # datos semilla idempotentes (idempotentes por clave única)
│   │   ├── lib/
│   │   │   ├── password.js   # scrypt asíncrono, formato versionado
│   │   │   ├── crypto.js     # AES-256-GCM con tag, KDF, máscaras
│   │   │   ├── validate.js   # validación de entrada sin dependencias
│   │   │   ├── cookies.js    # lectura/escritura de cookies
│   │   │   └── rate-limit.js # limitador en memoria por IP + clave
│   │   ├── middlewares/
│   │   │   ├── auth.js       # sesión, guards de rol y de organización
│   │   │   └── error.js      # manejador central de errores
│   │   ├── services/         # LÓGICA DE NEGOCIO (modelos Sequelize)
│   │   ├── controllers/      # orquestación HTTP (req/res)
│   │   ├── routes/           # definición de endpoints (Express Router)
│   │   └── jobs/             # barridos periódicos in-process
│   ├── tests/
│   │   ├── helpers.js        # cookie jar, login, register, request
│   │   └── *.test.js         # node --test (una API real por archivo)
│   └── .env.test.example     # base *_test, nunca credenciales reales
└── client/
    ├── package.json
    ├── vite.config.js
    ├── public/
    └── src/
        ├── main.jsx
        ├── App.jsx           # rutas y guards por rol (UX, no seguridad)
        ├── api/              # un módulo por dominio; solo fetch
        ├── components/       # pantallas y features de UI
        │   ├── ErrorBoundary.jsx
        │   ├── ui/           # kit propio sobre Base UI
        │   └── layout/       # PublicLayout / PrivateLayout
        ├── context/          # AuthContext, ToastContext
        ├── hooks/            # useResource, useTheme
        └── lib/              # format, geo, utils
```

### Scripts (raíz)

```json
{
  "name": "app",
  "private": true,
  "type": "module",
  "engines": { "node": "^20.19.0 || >=22.12.0" },
  "scripts": {
    "dev": "concurrently -n server,client -c blue,green \"npm:dev:server\" \"npm:dev:client\"",
    "dev:server": "npm run dev --workspace server",
    "dev:client": "npm run dev --workspace client",
    "build": "npm run build --workspace client",
    "start": "npm run start --workspace server",
    "test": "npm run test --workspace server",
    "migrate": "npm run migrate --workspace server",
    "migrate:dry": "npm run migrate:dry --workspace server"
  },
  "workspaces": ["server", "client"]
}
```

- **Dev:** API en `:40400`, web en `:40300` (Vite hace proxy de `/api` → API). El `.env` de desarrollo fija `PORT=40400`.
- **Producción:** `npm start` → Express sirve la API y `client/dist` en el mismo puerto (`PORT`, por defecto `3000`, o el que use el proxy inverso).
- `npm test` → `node --test --test-concurrency=1` (secuencial cuando las pruebas comparten base).
- Instala con `npm ci` en CI/servidor; no uses `npm install` para reproducir un despliegue.

---

## 4. Entorno

`server/src/config/env.js` carga el `.env` de la raíz **sin dependencias** (sube directorios hasta encontrar el `package.json` con `workspaces`):

```js
import { readFileSync, existsSync } from 'node:fs';
import path from 'node:path';
import { fileURLToPath } from 'node:url';

function findMonorepoRoot() {
  let dir = path.dirname(fileURLToPath(import.meta.url));
  const isRoot = (d) =>
    existsSync(path.join(d, 'package.json')) &&
    /"workspaces"/.test(readFileSync(path.join(d, 'package.json'), 'utf8'));
  while (dir !== path.dirname(dir)) {
    if (isRoot(dir)) return dir;
    dir = path.dirname(dir);
  }
  return process.cwd();
}

const envPath = path.join(findMonorepoRoot(), '.env');
try {
  for (const line of readFileSync(envPath, 'utf8').split('\n')) {
    const match = line.match(/^\s*(?:export\s+)?([A-Z0-9_]+)\s*=\s*(.*?)\s*$/);
    // Las líneas que no coinciden (vacías o que empiezan con #) se ignoran.
    if (match && process.env[match[1]] === undefined) {
      process.env[match[1]] = match[2].replace(/^(["'])(.*)\1$/, '$2');
    }
  }
} catch { /* el .env es opcional en desarrollo */ }

export const config = {
  port: Number(process.env.PORT) || 3000,
  env: process.env.NODE_ENV || 'development',
  isProduction: (process.env.NODE_ENV || 'development') === 'production',
  trustProxy: Number(process.env.TRUST_PROXY) || 0,
};
```

- El `.env` **no se versiona**; se versiona `.env.example` con las claves documentadas y valores ficticios.
- En producción, `systemd` puede usar `EnvironmentFile=` con permisos `600`; el `.env` es una comodidad de desarrollo, no el almacén de secretos definitivo.
- `process.env` devuelve strings: parsea booleanos y números explícitamente (`=== 'true'`, `Number(...)`), nunca confíes en valores “truthy”.
- En `app.js`, aplica `app.set('trust proxy', config.trustProxy)` antes de los middlewares. Con nginx, `TRUST_PROXY=1`; sin proxy, `0`.

Variables habituales:

```text
DATABASE_URL        # postgres://usuario:clave@localhost:5432/app
DATABASE_URL_TEST   # opcional en desarrollo; el .env.test usa DATABASE_URL con la base _test
PORT                # puerto del API (dev 40400; prod 3000 o el del proxy)
NODE_ENV            # development | test | production
SETTINGS_SECRET     # clave de cifrado de secretos (32+ bytes aleatorios; si es frase, KDF)
ADMIN_PASSWORD      # contraseña de la cuenta admin semilla (obligatoria y fuerte en producción)
PUBLIC_URL          # URL pública (retornos de pago, enlaces); valida el dominio
TRUST_PROXY         # 1 detrás de nginx; 0 en local
COOKIE_SECURE       # true en producción con TLS
PG_POOL_MAX         # conexiones máximas del pool (por defecto 10)
PGSSL               # true si el proveedor exige SSL
<JOB>_ENABLED / <JOB>_INTERVAL_MINUTES
```

> Si `NODE_ENV=production` y falta `SETTINGS_SECRET` o `ADMIN_PASSWORD`, la app debe **fallar al arrancar** en vez de seguir con secretos en claro o una cuenta débil.

---

## 5. Capa de datos (Sequelize)

`server/src/db/index.js`: instancia de Sequelize, conexión, transacciones y `resetSchema()` (solo tests). El esquema se gestiona con **migraciones versionadas** (`sequelize-cli`/Umzug), no con `sync()` en producción.

```js
// server/src/db/index.js
import { Sequelize } from 'sequelize';

const url = process.env.DATABASE_URL ?? 'postgres://usuario:clave@localhost:5432/app';

export const sequelize = new Sequelize(url, {
  dialect: 'postgres',
  logging: process.env.NODE_ENV === 'development' ? console.log : false,
  pool: { max: Number(process.env.PG_POOL_MAX) || 10, idle: 30_000, acquire: 5_000 },
  dialectOptions: process.env.PGSSL === 'true' ? { ssl: { require: true, rejectUnauthorized: true } } : {},
});

export async function connectDb() { await sequelize.authenticate(); }
export async function closeDb() { await sequelize.close(); }

export function withTransaction(fn) {
  return sequelize.transaction(async (t) => fn(t));
}

export async function resetSchema() {
  const dbName = (() => { try { return new URL(process.env.DATABASE_URL ?? '').pathname.replace(/^\//, ''); } catch { return ''; } })();
  if (process.env.NODE_ENV !== 'test' || !dbName.endsWith('_test')) {
    throw new Error('resetSchema() solo puede ejecutarse con NODE_ENV=test y una base *_test');
  }
  await sequelize.dropSchema('public', { cascade: true });
  await sequelize.createSchema('public');
}
```

- **No uses `sequelize.sync()` en producción**: crea/altera tablas sin control de versiones. Usa migraciones (`npx sequelize-cli db:migrate`) y reserva `sync({ force/alter })` para tests.
- **Transacciones**: pasa el objeto `transaction` (`t`) a los servicios; no anides transacciones, comparte `t`. No hagas HTTP/envíos dentro.
- **Pool**: configurado en la instancia; ajusta `max` según proveedor. Con proveedor gestionado, activa SSL con `rejectUnauthorized`.
- **Modelos**: un archivo por entidad en `server/src/models/` (clase `Model` o `sequelize.define`); las asociaciones se registran en `models/index.js`.
- **Convención**: elige `underscored: true` (camelCase ↔ snake_case) o manual, y mantenla en todos los modelos.

### Migraciones

Las migraciones viven en `server/src/db/migrations/` y son la única forma de cambiar el esquema en producción. Cada una exporta `up`/`down` y corre en transacción (Sequelize lo hace por defecto en Postgres).

```js
// migrations/20260101-create-organizations.cjs
// `.cjs` porque el proyecto es ESM; con Umzug puedes usar `export default`.
module.exports = {
  async up(qi, Sequelize) {
    await qi.createTable('organizations', {
      id: { type: Sequelize.INTEGER, autoIncrement: true, primaryKey: true },
      nombre: { type: Sequelize.TEXT, allowNull: false },
      creado_en: { type: Sequelize.DATE, allowNull: false, defaultValue: Sequelize.literal('now()') },
    });
    await qi.addIndex('organizations', ['nombre']);
  },
  async down(qi) { await qi.dropTable('organizations'); },
};
```

- Para cambios aditivos (columnas, índices) usa migraciones; para transformaciones de datos, migraciones con lógica o scripts `migrate-*.js` que usen los modelos.
- `CREATE INDEX CONCURRENTLY` no puede ir en la transacción de la migración: ejecútalo con `qi.sequelize.query('CREATE INDEX CONCURRENTLY ...', { transaction: null })` o en un script aparte.
- En producción, corre `npx sequelize-cli db:migrate` en el despliegue; en desarrollo puede ejecutarse al arrancar si `NODE_ENV !== 'production'`.

### `resetSchema()` y semillas

`resetSchema()` mantiene la guardia de base `*_test`. Las semillas son idempotentes por clave única (`findOrCreate`), no “si la tabla está vacía”.

```js
// seeders/organizations.js
await Organization.findOrCreate({ where: { nombre: 'Demo' }, defaults: { ... } });
```

### Modelo de ejemplo (multi-tenant)

```js
// models/Entidad.js (ESM)
export default (sequelize, DataTypes) => {
  const Entidad = sequelize.define('Entidad', {
    id: { type: DataTypes.INTEGER, autoIncrement: true, primaryKey: true },
    orgId: { type: DataTypes.INTEGER, allowNull: false, field: 'org_id' },
    nombre: { type: DataTypes.TEXT, allowNull: false },
    estado: { type: DataTypes.ENUM('activo', 'inactivo'), allowNull: false, defaultValue: 'activo' },
    creadoEn: { type: DataTypes.DATE, allowNull: false, defaultValue: DataTypes.NOW, field: 'creado_en' },
    borradoEn: { type: DataTypes.DATE, field: 'borrado_en' },
  }, {
    tableName: 'entidades',
    timestamps: false,
    defaultScope: { where: { borradoEn: null } },
  });
  return Entidad;
};
```

- **`ON DELETE RESTRICT`** se declara en la migración (FK), no en el modelo.
- Indexa `org_id` y las FKs en la migración; Sequelize no los crea automáticamente.
- Usa `defaultScope` para soft delete (`borrado_en IS NULL`) y `unscoped()` cuando necesites ver borrados.
- Para dinero usa `DECIMAL(12,2)` (`DataTypes.DECIMAL(12,2)`) o enteros en unidad mínima, nunca `FLOAT`. Para instantes, `DATE` (timestamptz); para días locales de facturación, `DATEONLY`.
- **Nombres:** elige una convención y respétala; no mezcles `estado` y `status`.

### Consultas y rendimiento

- Evita traer columnas de más: usa `attributes: ['id', 'nombre', ...]`.
- Paginación por keyset: `where: { creadoEn: { [Op.lt]: cursor } }`, `order: [['creadoEn', 'DESC'], ['id', 'DESC']]`, `limit`.
- Evita N+1: usa `include` con `required`/`separate` según el caso y mide con `logging`.
- Para agregaciones pesadas, usa `sequelize.query` con SQL explícito y `replacements`.

### Trampas específicas de Sequelize

- `sequelize.sync({ alter: true })` en producción reescribe tablas y puede perder datos: solo en tests.
- Inserciones masivas sin transacción: usa `bulkCreate` con `transaction`.
- Asociaciones sin FKs explícitas pueden crear constraints inesperadas en tests: decláralas en la migración.
- `paranoid: true` (soft delete de Sequelize) usa `deletedAt`; si el dominio ya usa `borrado_en`, no mezcles ambos: elige uno.
- `underscored: true` mapea camelCase ↔ snake_case; decide y mantenlo en todos los modelos.

---

## 6. Autenticación y guards

`server/src/middlewares/auth.js` — sesión por cookie `httpOnly` + guards reutilizables.

```js
export async function attachUser(req, _res, next) {
  req.sessionToken = getSessionToken(req);        // cookie de sesión (__Host-session en producción)
  req.user = await authService.userForSession(req.sessionToken);
  next();
}

export function requireAuth(req, res, next) { /* 401 si no hay sesión */ }

export function requirePlatformRole(...roles) { /* 403 si req.user.role no está en la lista */ }

export function requireOrg(req, res, next) {
  if (!req.user?.orgId) return res.status(403).json({ error: 'Tu cuenta no tiene una organización activa' });
  next();
}

export function requireMembershipRole(...roles) { /* 403 si el rol de membresía no permite la acción */ }
```

- **Cookie:** `HttpOnly; SameSite=Lax; Path=/; Max-Age=604800` (7 días) y `Secure` en producción. En producción puedes usar el prefijo `__Host-session` (exige `Secure`, `Path=/` y sin `Domain`). `Max-Age` va en **segundos**; si usas `res.cookie`, `maxAge` va en milisegundos.
- **Detrás de nginx:** `app.set('trust proxy', 1)`. Sin esto, `req.ip` es la IP del proxy, el rate limit no sirve y la cookie `Secure` puede no emitirse.
- **Sesiones:** token opaco de 32 bytes en la cookie; en la tabla `sessions` guarda **solo su SHA-256**, no el token en claro. `userForSession` hashea el token recibido y busca por `token_hash`; si se filtra un dump, las sesiones no son reutilizables.

```sql
CREATE TABLE IF NOT EXISTS sessions (
  token_hash text PRIMARY KEY,
  user_id integer NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  creado_en timestamptz NOT NULL DEFAULT now(),
  expira_en timestamptz NOT NULL,
  revocada_en timestamptz
);
CREATE INDEX IF NOT EXISTS sessions_user_idx
  ON sessions (user_id) WHERE revocada_en IS NULL;
CREATE INDEX IF NOT EXISTS sessions_expira_idx
  ON sessions (expira_en) WHERE revocada_en IS NULL;
```

- Rota el token al iniciar sesión (evita *session fixation*), revoca las sesiones al cambiar contraseña o rol, y borra las expiradas con un job.
- **Contraseñas:** `scrypt` asíncrono con sal por usuario, formato versionado (`scrypt$<salt>$<hash>`) y comparación con `timingSafeEqual`. `scryptSync` bloquea el event loop; no lo uses en login.
- **Códigos de error:** **401** sin sesión, **403** con sesión sin permiso, **404** cuando no se quiere revelar la existencia de un recurso ajeno, **429** por rate limit.
- **Rate limit:** obligatorio en login, registro y recuperación. Sin dependencias, un `Map` por `IP + email` con ventana y bloqueo progresivo; detrás del proxy usa `req.ip`. Si hay más de una instancia, el `Map` no se comparte: usa nginx o un almacén compartido.
- **CSRF:** `SameSite=Lax` cubre la mayoría de los POST cruzados; añade comprobación de `Origin`/`Referer` en mutaciones y no uses `GET` para cambiar estado. Si algún día necesitas `SameSite=None`, exige `Secure`.
- **CSRF y CORS no se mezclan:** no abras CORS a cualquier origen con credenciales.

```js
// lib/password.js
import { randomBytes, scrypt as scryptCb, timingSafeEqual } from 'node:crypto';
import { promisify } from 'node:util';
const scrypt = promisify(scryptCb);

export async function hashPassword(password) {
  const salt = randomBytes(16);
  const hash = await scrypt(password, salt, 64);
  return `scrypt$${salt.toString('hex')}$${hash.toString('hex')}`;
}

export async function verifyPassword(password, stored) {
  try {
    const [algo, saltHex, hashHex] = String(stored).split('$');
    if (algo !== 'scrypt' || !saltHex || !hashHex) return false;
    const expected = Buffer.from(hashHex, 'hex');
    const candidate = await scrypt(password, Buffer.from(saltHex, 'hex'), expected.length);
    return candidate.length === expected.length && timingSafeEqual(candidate, expected);
  } catch {
    return false;                       // hash corrupto o formato viejo: no es un 500
  }
}
```

- Valida longitud de contraseña (mínimo 12, máximo 128) y no devuelvas mensajes que permitan enumerar correos.
- **Recuperación de contraseña y verificación de correo** (si el panel es público): token de un solo uso, guardado hasheado, con expiración, invalidación de sesiones y respuesta genérica exista o no la cuenta. El envío va por la bandeja (`message_outbox`), no en línea.
- La cuenta admin semilla solo se crea si no existe; cambiar `ADMIN_PASSWORD` en el entorno **no rota** una contraseña ya creada. Documenta un comando de reseteo (`npm run admin:reset`) o hazlo explícito en el runbook.

---

## 7. Multi-tenant: la organización como cliente y cobro

Es el patrón central para cualquier B2B y evita dos errores caros: cobrar a la persona en vez de al cliente, y filtrar datos por “usuario” en cada consulta nueva.

### Modelo

```text
users 1 ── N memberships N ── 1 organizations 1 ── 1 <entidad principal> 1 ── N <detalles>
                                      │
                                      ├── 1 subscriptions
                                      ├── 0..1 quotas
                                      ├── N documentos
                                      ├── N notifications
                                      └── N message_outbox
```

- `users` = **identidad** (login). `organizations` = **cliente económico y entidad de cobro**.
- `memberships` = usuario ↔ organización con rol (`owner`, `operador`, `lector`) y estado (`activa`, `revocada`).
- Todas las tablas de negocio llevan `org_id NOT NULL` (tras el backfill).
- Restricciones clave:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS memberships_one_active_per_user
  ON memberships (user_id) WHERE estado = 'activa';

CREATE UNIQUE INDEX IF NOT EXISTS memberships_one_owner_per_org
  ON memberships (org_id) WHERE role = 'owner' AND estado = 'activa';

ALTER TABLE memberships
  ADD CONSTRAINT memberships_role_check CHECK (role IN ('owner', 'operador', 'lector')),
  ADD CONSTRAINT memberships_estado_check CHECK (estado IN ('activa', 'revocada'));
```

> Los `ALTER TABLE ... ADD CONSTRAINT` **no son idempotentes**: decláralos en una migración de Sequelize (con `up`/`down`) y no los repitas en el arranque. `CREATE TABLE IF NOT EXISTS` tampoco añade checks a una tabla que ya existe.

> **Decisión de producto:** “una membresía activa por usuario” significa que un usuario no puede pertenecer a dos organizaciones a la vez. Es una limitación consciente y simplifica el tenant; si algún día necesitas multi-organización, elimina ese índice y añade un selector de organización. Documéntalo en la spec.

### Reglas de código

| Regla | Cómo se cumple |
|---|---|
| El tenant sale de la sesión | `req.user.orgId`; si el cliente manda otro `orgId` → `403` |
| Los servicios reciben `orgId` | `entidadesService.listForOrg(orgId)` |
| El admin de plataforma no tiene membresía | opera con un `orgId` explícito en la ruta (`/api/admin/orgs/:orgId/...`), no con un query string ambiguo; sin él → `400` |
| Los datos privados del cliente son 403 para el admin | `requireOrg` en esas rutas; el admin no entra |
| El cobro es de la organización | `subscriptions.org_id`; solo el `owner` paga |
| Toda consulta filtra por organización | `where: { orgId }`; los tests de IDOR lo comprueban |
| Los ids ajenos no revelan existencia | `404` en lugar de `403` cuando el recurso es de otra organización |
| El acceso del admin se audita | fila en `audit_log` con `actor`, `org_id`, acción y motivo |

- **Nunca** aceptes `org_id` del body para decidir el tenant. Si un usuario normal lo envía, ignóralo o recházalo con `403`; el servidor usa el de la sesión.
- `userForSession` resuelve la membresía **activa** en cada petición: revocar una membresía surte efecto inmediato aunque la cookie siga viva.
- Las claves únicas de negocio incluyen el tenant: `UNIQUE (org_id, clave)`, no `UNIQUE (clave)`. Lo mismo para `dedupe_key`.
- Los listados, exportaciones y documentos se filtran siempre por `org_id`; el admin tiene rutas separadas y auditadas.

### Alta de una cuenta (transacción única)

```js
await withTransaction(async (t) => {
  const user = await User.create({ email, name, passHash }, { transaction: t });
  const org = await Organization.create({ name, billingEmail: email }, { transaction: t });
  await Membership.create({ orgId: org.id, userId: user.id, role: 'owner' }, { transaction: t });
  await Subscription.create({ orgId: org.id, userId: user.id, days: 14 }, { transaction: t });
});
```

Si un paso falla, no queda ninguna fila.

### Equipo de una organización (roles)

Un `owner` puede sumar cuentas por correo:

- Si la cuenta existe y no pertenece a otra organización → entra de inmediato.
- Si no existe → queda una **invitación pendiente** que se acepta al registrarse (entra con el rol invitado, sin crear otra organización ni otro periodo de prueba).
- Si ya pertenece a otra organización → `409` con mensaje claro (no se la mueve en silencio).
- El `owner` no se puede quitar ni degradar: la organización no puede quedar sin responsable de cobro.
- La invitación es idempotente por `(org_id, email)` y expira; reenviarla no duplica filas.

### Baja de una organización y retención

- La organización se da de baja **lógicamente** (`borrado_en` no nulo; si el dominio usa un estado, añade `'baja'` a su `CHECK`), no con `DELETE`.
- Las tablas con pagos, facturas y documentos usan `ON DELETE RESTRICT`; la evidencia contable se conserva el tiempo que exija la normativa.
- Los datos personales se pueden anonimizar/exportar sin romper las facturas (por ejemplo, seudonimizar el usuario y conservar el documento fiscal).
- Define en la spec: qué se borra, qué se conserva, durante cuánto tiempo y quién puede solicitarlo.

### Configuración global o por organización

Antes de crear `settings`, decide si las credenciales son **de la plataforma** (el SaaS cobra con su Stripe/WhatsApp) o **de cada organización** (el cliente conecta su propia cuenta). No mezcles ambos mundos en una tabla sin `org_id`:

- `platform_settings`: solo el admin de plataforma las ve y edita.
- `org_settings`: el `owner` de esa organización las ve y edita; nunca se cruzan entre organizaciones.

La precedencia recomendada es: **org → plataforma → entorno → valor por defecto**.

### Migrar un proyecto que usa `user_id` (expand → backfill → contract)

1. **Expand**: crear `organizations`/`memberships` y añadir `org_id` (nullable) a todas las tablas de negocio.
2. **Preflight**: contar filas y **detectar filas de negocio creadas por el admin** (o por usuarios sin organización). Si hay, la migración **aborta** y exige un mapeo explícito — nunca se inventa una organización.
3. **Backfill** (idempotente): una organización por cliente sin membresía (el nombre de su recurso principal, si existe, da contexto) + membresía `owner` + fila de suscripción si falta; y `UPDATE ... SET org_id = ...` por la membresía activa de cada usuario.
4. **Contract**: `ALTER TABLE ... ALTER COLUMN org_id SET NOT NULL`, índices únicos por organización y reemplazo de `user_id` por `org_id` en modelos y consultas.
5. **Migración de Sequelize** o script CLI con `--dry-run` (preflight, no escribe) y `--apply`, más backup `pg_dump` documentado.

**Orden de despliegue seguro** (no lo saltes):

1. Expandir esquema (columnas nullable, triggers compatibles).
2. Desplegar código que escribe el dato viejo y el nuevo.
3. Backfill y verificación de conteos.
4. Cambiar lecturas al dato nuevo.
5. Dejar de escribir el viejo.
6. `SET NOT NULL` y `DROP COLUMN` en una migración posterior.

- Ejecuta cada migración en **una transacción** y con un **advisory lock** para que dos despliegues no la corran a la vez.
- Para tablas grandes, `CREATE INDEX CONCURRENTLY` va **fuera** de la transacción y con ventana de mantenimiento.
- Prueba el rollback/forward-only: si algo falla, el plan es restaurar el backup o aplicar una migración correctiva, no “editar a mano”.

```js
// Patrón de abortar (el error debe ser accionable)
if (orphans.length > 0) {
  const err = new Error(
    `Migración de organizaciones abortada: hay ${orphans.length} fila(s) sin organización`
  );
  err.code = 'TENANT_ORPHANS';
  throw err;
}
```

**Pruebas de migración**: reconstruir el esquema «legacy» (quitar columnas con `ALTER TABLE ... DROP COLUMN ... CASCADE`), insertar filas, ejecutar la migración y comprobar que los conteos de filas de negocio se conservan y que la segunda ejecución no duplica nada.

---

## 8. Configuración y secretos

- Tabla `platform_settings` para lo global; tabla `org_settings` con `PRIMARY KEY (org_id, key)` para lo de cada cliente. Nunca guardes secretos de una organización en la tabla global.

```sql
CREATE TABLE IF NOT EXISTS platform_settings (
  key text PRIMARY KEY,
  value text NOT NULL,
  actualizado_en timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS org_settings (
  org_id integer NOT NULL REFERENCES organizations(id) ON DELETE RESTRICT,
  key text NOT NULL,
  value text NOT NULL,
  actualizado_en timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (org_id, key)
);
```

- Prioridad: **org → plataforma → variable de entorno → valor por defecto**.
- Con `SETTINGS_SECRET` definido, los secretos se cifran con **AES-256-GCM** (clave derivada con KDF, IV aleatorio de 12 bytes por escritura y **auth tag** de 16 bytes) y se guardan como `v1:<iv>:<tag>:<cifrado>`; sin él, en desarrollo se guardan en claro con aviso, y en producción la app **falla al arrancar**.
- La API **nunca** devuelve el secreto: solo una `pista` enmascarada (`••••1234`) y el origen (`org`, `env`, `default`).

```js
// Cifrado: el tag es obligatorio para verificar integridad.
// KDF: si SETTINGS_SECRET es una frase, deriva con scrypt/HKDF (sal + parámetros versionados).
// Aquí se asume una clave aleatoria de 32+ bytes; SHA-256 es aceptable en ese caso.
const key = crypto.createHash('sha256').update(settingsSecret).digest();
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', key, iv);
const ciphertext = Buffer.concat([cipher.update(valor, 'utf8'), cipher.final()]);
const tag = cipher.getAuthTag();
const stored = `v1:${iv.toString('base64')}:${tag.toString('base64')}:${ciphertext.toString('base64')}`;
```

```js
// Descifrado: sin setAuthTag, GCM no autentica nada.
const [version, ivB64, tagB64, ctB64] = stored.split(':');
if (version !== 'v1') throw new Error('Versión de secreto no soportada');
const decipher = crypto.createDecipheriv('aes-256-gcm', key, Buffer.from(ivB64, 'base64'));
decipher.setAuthTag(Buffer.from(tagB64, 'base64'));
const valor = Buffer.concat([
  decipher.update(Buffer.from(ctB64, 'base64')),
  decipher.final(),
]).toString('utf8');
```

- **KDF:** si `SETTINGS_SECRET` es una frase, no uses SHA-256 directo. Usa `scrypt`/`HKDF` con sal y parámetros versionados. Si es una clave aleatoria de 32+ bytes, SHA-256 es aceptable, pero decláralo.
- **Rotación:** el prefijo `v1` permite migrar a `v2`; guarda la versión, descifra con la clave vieja y re-cifra con la nueva en un script controlado. Plan mínimo: (1) backup de `platform_settings`/`org_settings`, (2) `--dry-run` que lista filas a re-cifrar y detecta versiones desconocidas, (3) `--apply` en una transacción con advisory lock, (4) verificar que todas las filas quedan en `v2`, (5) recién entonces cambiar `SETTINGS_SECRET`. No cambies `SETTINGS_SECRET` sin plan de rotación.
- **Nunca** loguees valores descifrados, cookies, cabeceras `Authorization` ni cuerpos con contraseñas. La pista se calcula al vuelo; no se guarda.
- Cada integración expone cuatro operaciones: `view()` (configuración + origen + pista), `save()`, `test()` (probar sin enviar nada) y `bridgeConfig()`. `test()` no debe mutar datos ni enviar mensajes reales.
- Valida la longitud y el formato de claves y valores; una clave arbitraria del cliente no debe romper el panel ni permitir inyección.

---

## 9. Integraciones externas «provider-ready»

Regla: **la aplicación funciona sin credenciales**.

```text
configuración (settings / env) → ¿hay credenciales? → no: modo simulado (se registra y se aprueba al instante)
                                                → sí: checkout real + webhook de confirmación
```

### Pagos

- Montos en **unidad mínima entera** (centavos) o `numeric(12,2)`; nunca `float`. Añade `moneda` (ISO 4217), impuestos y política de redondeo.
- `period_start date` y `period_end date` para el periodo cobrado (días locales, no timestamps); `concept` distingue cobros recurrentes de cobros por recurso.
- **Índice único parcial** para no cobrar dos veces el mismo periodo:

```sql
CREATE UNIQUE INDEX IF NOT EXISTS payments_one_approved_per_period
  ON payments (org_id, concept, period_start)
  WHERE estado = 'aprobado';
```

- Si usas `ON CONFLICT` con un índice parcial, incluye el predicado: `ON CONFLICT (org_id, concept, period_start) WHERE estado = 'aprobado' DO NOTHING`.
- Envía una **idempotency key** al proveedor y guarda su `event_id`; el webhook y el retorno del usuario pueden llegar en cualquier orden.
- **Factura:** si la normativa exige numeración sin huecos, usa un contador en tabla bloqueado con `FOR UPDATE` (las secuencias de PostgreSQL no son transaccionales y dejan huecos); añade datos fiscales, retención legal y PDF con clave de acceso. Si la normativa permite huecos, una secuencia es más simple.
- **Conciliación:** un job compara eventos del proveedor con pagos locales, detecta refunds, contracargos y pagos huérfanos. No confíes solo en el webhook.
- Prueba también los casos de importe cero, moneda distinta, pago duplicado, pago tardío y webhook repetido.

### Webhook

`POST /api/webhooks/payments/:provider` con secreto compartido. **El body crudo es obligatorio** para verificar la firma; `express.json()` lo consume. Monta `express.raw({ type: 'application/json' })` en esa ruta antes del parser JSON, o usa la opción `verify` para guardar `req.rawBody`.

- Verifica `HMAC-SHA256` con `timingSafeEqual`, no una comparación normal.
- Idempotencia por `event_id` del proveedor (índice único) y ventana anti-replay por timestamp.
- Responde `2xx` rápido y procesa el efecto en una transacción/outbox; un webhook lento se reintenta y duplica.
- El estado final lo dicta el proveedor, pero tu base decide si el evento es aplicable (importe, moneda, organización, periodo).
- No pongas el secreto en la URL: queda en logs e historiales.

### Mensajería (SMS/WhatsApp/email)

- Bandeja `message_outbox` con `estado` (`encolado`/`simulado`/`enviado`/`fallido`/`muerto`), `attempts`, `next_attempt_at`, `provider_id`, `error` y **reintento** desde el panel.
- **Idempotencia:** `UNIQUE (org_id, dedupe_key)`, no global. Reintentar el mismo aviso no crea otra fila.
- Backoff exponencial con techo, máximo de intentos y paso a *dead letter*; el panel permite reintentar manualmente.
- Consentimiento y baja: guarda opt-in/opt-out por canal y respeta STOP/desuscripción. Un aviso transaccional no es marketing.
- El mensaje se **encola, no se envía** en la petición HTTP.

### Datos de terceros y llamadas externas

- Degradación elegante con respaldo determinista (por ejemplo, pronóstico sin claves).
- Timeout, reintentos con backoff y, si el proveedor falla seguido, un cortocircuito; no dejes una petición de usuario colgada esperando a un tercero.
- Valida `PUBLIC_URL` y cualquier URL de retorno contra una allowlist; evita *open redirect* y *SSRF*.
- En todos los casos: el proveedor real se activa solo cuando hay credenciales, y el modo simulado se marca claramente en la UI y en los logs.

---

## 10. Barridos (jobs) idempotentes

Jobs in-process con `setInterval` + `unref()` (no impiden el apagado), configurables por entorno y desactivables:

```js
let running = false;

export function startReminderJob() {
  if (process.env.REMINDERS_ENABLED === 'false') return null;
  const timer = setInterval(async () => {
    if (running) return;                       // no solapar barridos
    running = true;
    try {
      await runReminderSweep();
    } catch (err) {
      track('job_error', { job: 'reminders', error: err.message });
    } finally {
      running = false;
    }
  }, reminderIntervalMs());
  timer.unref?.();
  return timer;
}
```

- `reminderIntervalMs()` parsea `REMINDERS_INTERVAL_MINUTES` con un mínimo (por ejemplo, 5 minutos) y devuelve milisegundos; no permitas `0` ni valores negativos.
- **Idempotencia:** cada aviso lleva `dedupe_key` con índice único **por organización** → repetir el barrido no duplica nada.
- El barrido recorre **organizaciones** (no personas): los avisos de una organización se generan una sola vez y se dirigen al `owner`.
- Antes de enviar, se **cancelan** (no se borran) los avisos de hechos que ya pasaron; conserva el histórico para auditoría.
- Si algún día corres más de una instancia, protege el barrido con `pg_try_advisory_lock` o procesa la outbox con `FOR UPDATE SKIP LOCKED`. El `setInterval` no es un coordinador distribuido.
- **Zona horaria:** guarda instantes en `timestamptz` y calcula los días locales con la zona de la organización. No compares fechas como strings.
- **Pruebas de N días exactos:** usa horas absolutas (`make_interval(hours => n * 24)`) para que un cambio de hora no alargue el periodo. **Ciclos mensuales:** usa aritmética de calendario (`+ interval '1 month'`) para facturar el mismo día; no generalices la regla de las horas.
- **Apagado ordenado:** en `SIGTERM`/`SIGINT`, deja de aceptar peticiones, para los timers, espera el barrido en curso y cierra la conexión de Sequelize (`closeDb()`). Un `unref()` no garantiza que el job termine.
- Registra duración, filas procesadas y errores por job; un job silencioso que falla todas las noches es peor que no tenerlo.

---

## 11. Observabilidad

`server/src/services/observability.js` (sin dependencias):

- `track(event, dims)` → línea JSON a stdout + contador.
- `meter(event, dims)` → solo contador (métricas de alta frecuencia).
- `getMetrics()` → snapshot para el panel de administración (embudo, ingresos estimados, tiempos de respuesta).

Pero **stdout + contadores en memoria no son un audit trail durable**. Para dinero, autenticación y accesos del admin, escribe una tabla append-only:

```sql
CREATE TABLE IF NOT EXISTS audit_log (
  id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
  ocurrido_en timestamptz NOT NULL DEFAULT now(),
  actor_user_id integer,
  org_id integer,
  action text NOT NULL,
  entity text,
  entity_id text,
  ip inet,
  user_agent text,
  request_id text,
  antes jsonb,
  despues jsonb
);

CREATE INDEX IF NOT EXISTS audit_log_org_time_idx
  ON audit_log (org_id, ocurrido_en DESC);
```

- Solo `INSERT`; nada de `UPDATE`/`DELETE` desde la app. Revoca esos privilegios al rol de la aplicación y define la retención en la spec.
- `actor_user_id` no lleva FK a propósito: el registro debe sobrevivir al borrado o anonimización del usuario.
- **Nunca** guardes contraseñas, tokens, secretos ni cuerpos completos con PII. Redacta antes de loguear.
- Añade un `request_id` por petición (UUID) y propágalo a logs, errores y auditoría. En el manejador de errores, loguea el stack en el servidor y devuelve un mensaje genérico + `request_id`; nunca el stack al cliente.
- Registra método, ruta, estado, duración, `user_id`, `org_id` y `request_id`. No registres cookies ni `Authorization`.
- `/api/health/live` comprueba el proceso; `/api/health/ready` comprueba la base (`SELECT 1`) y las dependencias críticas. No expongas detalles internos al público.
- `/api/metrics` es solo para el admin y sin caché. Los contadores en memoria se reinician con el proceso y no se comparten entre instancias; si el proyecto crece, sustituye este módulo por OpenTelemetry sin tocar el resto.
- Define retención de logs y auditoría; “para siempre” también es un riesgo de privacidad.

---

## 12. Frontend

### 12.1 Vite, proxy, PWA y servido estático

```js
// client/vite.config.js (extracto)
export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
    VitePWA({
      registerType: 'autoUpdate',
      workbox: {
        globPatterns: ['**/*.{js,css,html,svg,png,ico,woff2}'],
        navigateFallback: '/index.html',
        navigateFallbackDenylist: [/^\/api\//], // la API nunca se cachea
      },
    }),
  ],
  resolve: { alias: { '@': path.resolve(import.meta.dirname, './src') } },
  server: { port: 40300, host: true, proxy: { '/api': 'http://localhost:40400' } },
  build: {
    outDir: 'dist',
    rolldownOptions: {
      output: {
        codeSplitting: {
          groups: [
            { name: 'icons', test: /node_modules[\\/]lucide/ },
            { name: 'ui', test: /node_modules[\\/]@base-ui/ },
            { name: 'react', test: /node_modules[\\/](react|react-dom|react-router)/ },
            { name: 'vendor', test: /node_modules/ },
          ],
        },
      },
    },
  },
});
```

- `codeSplitting.groups` no debe mandar todo `node_modules` a un único chunk enorme; separa `react`, `ui`, `icons` y deja un `vendor` razonable.
- Vite 8 usa **Rolldown** en lugar de Rollup: `build.rollupOptions` pasa a `build.rolldownOptions` y `manualChunks` (forma objeto) ya no existe; usa `codeSplitting.groups`.
- En producción, Express sirve `client/dist` **después** de las rutas `/api`. En Express 5, `app.get('*')` ya no es válido: usa `app.get('/{*splat}', ...)` o un middleware final.

```js
// app.js (extracto de producción)
app.use('/api', apiRouter);
app.use('/api', (_req, res) => res.status(404).json({ error: 'No encontrado' }));

app.use('/assets', express.static(path.join(dist, 'assets'), {
  immutable: true,
  maxAge: '1y',
}));
app.use(express.static(dist, { index: false, maxAge: 0 }));
app.get('/{*splat}', (_req, res) => {
  res.set('Cache-Control', 'no-cache');
  res.sendFile(path.join(dist, 'index.html'));
});
```

- Assets con hash → caché inmutable. `index.html` → sin caché, para que un despliegue se vea. Las rutas `/api` desconocidas responden JSON 404, no el HTML de la SPA.
- PWA: `autoUpdate` puede recargar en mal momento; si hay formularios largos, usa `registerType: 'prompt'` y ofrece actualizar. Nunca cachees `/api`.
- `helmet` con CSP: en local desactiva `upgrade-insecure-requests`; en producción actívalo con TLS/HSTS. No uses `dangerouslySetInnerHTML`; React escapa por defecto.

### 12.2 Capas

- `api/*.js`: **una función por endpoint**, sin estado; único punto de contacto con la red.
- `api/http.js`: helper único con cookie, JSON, timeout, `AbortSignal` y propagación del mensaje **y del estado** del servidor.

```js
export class ApiError extends Error {
  constructor(message, { status = 0, code, data } = {}) {
    super(message);
    this.name = 'ApiError';
    this.status = status;
    this.code = code;
    this.data = data;
  }
}

export async function requestJson(path, {
  method = 'GET', body, signal, timeoutMs = 15_000,
  fallback = 'Error de conexión con la API',
} = {}) {
  const controller = new AbortController();
  let timedOut = false;
  const timer = setTimeout(() => { timedOut = true; controller.abort(); }, timeoutMs);
  const onAbort = () => controller.abort();
  signal?.addEventListener('abort', onAbort, { once: true });

  try {
    const res = await fetch(path, {
      method,
      credentials: 'same-origin',           // solo navegador; en Node no hay cookie jar
      headers: body !== undefined
        ? { 'content-type': 'application/json', accept: 'application/json' }
        : { accept: 'application/json' },
      body: body !== undefined ? JSON.stringify(body) : undefined,
      signal: controller.signal,
    });

    if (res.status === 204) return null;
    const data = await res.json().catch(() => ({}));
    if (!res.ok) {
      throw new ApiError(data.error ?? fallback, { status: res.status, code: data.code, data });
    }
    return data;
  } catch (err) {
    if (err.name === 'AbortError') {
      if (timedOut) throw new ApiError('Tiempo de espera agotado', { status: 0 });
      throw err;
    }
    if (err instanceof ApiError) throw err;
    throw new ApiError(fallback, { status: 0, data: err });
  } finally {
    clearTimeout(timer);
    signal?.removeEventListener('abort', onAbort);
  }
}
```

- `useResource.js`: carga con `data/error/loading`, recarga manual, `setData` y cancelación. No dependas de que el `loader` sea estable: guárdalo en un ref y aborta la petición anterior.

```js
export function useResource(loader) {
  const loaderRef = useRef(loader);
  loaderRef.current = loader;
  const requestRef = useRef(0);
  const controllerRef = useRef(null);
  const [state, setState] = useState({ data: null, error: null, loading: true });

  const reload = useCallback(async () => {
    controllerRef.current?.abort();          // cancela la petición anterior
    const id = ++requestRef.current;
    const controller = new AbortController();
    controllerRef.current = controller;
    setState((s) => ({ ...s, loading: true, error: null }));
    try {
      const data = await loaderRef.current({ signal: controller.signal });
      if (id === requestRef.current) setState({ data, error: null, loading: false });
    } catch (error) {
      if (error.name === 'AbortError') return;
      if (id === requestRef.current) setState({ data: null, error, loading: false });
    }
  }, []);

  useEffect(() => {
    reload();
    return () => {
      requestRef.current += 1;
      controllerRef.current?.abort();
    };
  }, [reload]);

  return {
    ...state,
    reload,
    setData: (data) => setState((s) => ({ ...s, data })),
  };
}
```

- Los loaders deben aceptar `{ signal }` y pasarlo a `requestJson`; si no, la cancelación no llega al `fetch` y la petición anterior sigue viva.
- Si el loader depende de parámetros de ruta, llama a `reload()` desde un `useEffect` con esas dependencias; el hook no adivina cuándo cambió el loader.
- `context/AuthContext.jsx`: sesión global (`undefined` = comprobando, `null` = sin sesión, objeto = usuario). El usuario expone `id`, `email`, `name`, `role` (plataforma), `orgId`, `orgName` y `membershipRole` (dentro de la organización). Si `ApiError.status === 401`, limpia la sesión y redirige al login; no dejes la UI en un estado zombie.
- `context/ToastContext.jsx`: puente **fino** sobre Base UI Toast. Dentro del árbol usa `Toast.useToastManager()`; fuera de React, `Toast.createToastManager()` pasado a `<Toast.Provider toastManager={...}>`. La API oficial es `add({ title, description, type })` —`type` es un string; por convención `success`/`error`/`info`—, **no** `push` ni `variant`. El contexto puede exponer `toast.success(title, description)` como azúcar que llama a `add({ title, description, type: 'success' })`. Si no necesitas esa azúcar, elimina el contexto y usa el manager directo: no reimplementes la cola, los timeouts ni el estado de los toasts.
- `components/ErrorBoundary.jsx`: captura errores de render y muestra una pantalla de recuperación; no dejes la app en blanco.
- `components/layout/`: `PublicLayout` (público) y `PrivateLayout` (sidebar + cabecera + `Outlet`), con navegación filtrada por rol.
- `App.jsx`: tabla de rutas por rol; las rutas privadas sin sesión redirigen al login. **Los guards del cliente son UX**: el servidor vuelve a comprobar todo.
- **Formularios**: 2–3 columnas con la anatomía oficial `Field.Root` + `Field.Label` + `Field.Control` + `Field.Description` + `Field.Error`; `Select` de Base UI para catálogos (o `Combobox` si la lista es grande); `Field.Control` nativo para texto y fecha. Deshabilita el botón mientras envía y muestra el error del servidor con `invalid` en `Field.Root` y `Field.Error match={true}`.
- **Avisos**: toast en cada acción (éxito y error con el mensaje del servidor).
- **Estados**: toda pantalla que carga datos debe tener `loading`, `empty`, `error` y `success`; no muestres una tabla vacía mientras carga.
- **Dos puertas**: una landing pública que explica el servicio sin exponer datos internos, y el panel privado con resumen, listados filtrables, descargas y confirmaciones con la anatomía oficial de `AlertDialog` (`Root` → `Trigger` → `Portal` → `Backdrop` → `Popup` → `Title`/`Description`/`Close`).
- **Seguridad en el cliente:** no guardes tokens en `localStorage` (usa cookie `HttpOnly`); no confíes en `returnTo` sin validarlo; no renderices HTML de usuario sin sanitizar.

### 12.3 Base UI: reglas de uso y kit propio (`client/src/components/ui/`)

Base UI **no es Radix**: no tiene `asChild`, no usa `ToastContext.push`, no existen `FieldLabel` ni `FieldControl`, y los nombres de los subcomponentes son los que publica su documentación. La documentación oficial es la fuente de verdad; si contradice la memoria del modelo o un ejemplo de un blog, gana la documentación. Antes de usar un componente, consulta su página en Markdown (por ejemplo, `https://base-ui.com/react/components/select.md`) o el índice `https://base-ui.com/llms.txt`.

**Paquete e imports**

```bash
npm i @base-ui/react
```

```js
import { Button } from '@base-ui/react/button';
import { Field } from '@base-ui/react/field';
import { Select } from '@base-ui/react/select';
import { AlertDialog } from '@base-ui/react/alert-dialog';
import { Toast } from '@base-ui/react/toast';
```

- El paquete es `@base-ui/react`. `@base-ui-components/react` es el nombre antiguo y **no** debe aparecer en el código.
- Importa por subruta (`@base-ui/react/select`), como hace la documentación; así el bundle solo incluye lo que usas y no dependes del barrel raíz.
- Base UI es *headless*: no trae estilos; el diseño vive en el kit propio con Tailwind.

**Anatomía oficial**

```jsx
// Formularios: el control puede ser Field.Control u otro input de Base UI
<Field.Root name="email" invalid={Boolean(error)}>
  <Field.Label>Correo</Field.Label>
  <Field.Control type="email" required />
  <Field.Description>Solo lo usamos para avisos de cuenta.</Field.Description>
  <Field.Error match="valueMissing">Escribe tu correo.</Field.Error>
  {error && <Field.Error match={true}>{error}</Field.Error>}
</Field.Root>
```

```jsx
// Select: Label + Trigger + Value + Icon + Portal + Positioner + Popup + List
<Select.Root
  name="orgId"
  items={orgs.map((org) => ({ value: org.id, label: org.name }))}
  value={orgId}
  onValueChange={setOrgId}
>
  <Select.Label>Organización</Select.Label>
  <Select.Trigger>
    <Select.Value placeholder="Elige una organización" />
    <Select.Icon><ChevronUpDownIcon /></Select.Icon>
  </Select.Trigger>
  <Select.Portal>
    <Select.Positioner sideOffset={4}>
      <Select.Popup>
        <Select.List>
          {orgs.map((org) => (
            <Select.Item key={org.id} value={org.id}>
              <Select.ItemIndicator><CheckIcon /></Select.ItemIndicator>
              <Select.ItemText>{org.name}</Select.ItemText>
            </Select.Item>
          ))}
        </Select.List>
      </Select.Popup>
    </Select.Positioner>
  </Select.Portal>
</Select.Root>
```

```jsx
// AlertDialog: solo para acciones destructivas o que exigen respuesta
<AlertDialog.Root>
  <AlertDialog.Trigger render={<Button variant="destructive" />}>
    Eliminar factura
  </AlertDialog.Trigger>
  <AlertDialog.Portal>
    <AlertDialog.Backdrop />
    <AlertDialog.Popup>
      <AlertDialog.Title>¿Eliminar la factura?</AlertDialog.Title>
      <AlertDialog.Description>Esta acción no se puede deshacer.</AlertDialog.Description>
      <AlertDialog.Close render={<Button variant="secondary" />}>Cancelar</AlertDialog.Close>
      <AlertDialog.Close render={<Button variant="destructive" />} onClick={onDelete}>
        Eliminar
      </AlertDialog.Close>
    </AlertDialog.Popup>
  </AlertDialog.Portal>
</AlertDialog.Root>
```

`AlertDialog.Viewport` es opcional: aparece en la anatomía de la documentación y se usa cuando el popup debe poder hacer scroll. `AlertDialog.Close` puede envolver tu propio botón con `render`, como en el ejemplo.

```jsx
// Toast: Provider > Portal > Viewport > Root(toast) > Content > Title/Description/Close
<Toast.Provider>
  <App />
  <Toast.Portal>
    <Toast.Viewport>
      <ToastList />
    </Toast.Viewport>
  </Toast.Portal>
</Toast.Provider>

function ToastList() {
  const { toasts } = Toast.useToastManager();
  return toasts.map((toast) => (
    <Toast.Root key={toast.id} toast={toast}>
      <Toast.Content>
        <Toast.Title />
        <Toast.Description />
        <Toast.Close>Descartar</Toast.Close>
      </Toast.Content>
    </Toast.Root>
  ));
}
```

- `Toast.Root` **exige** `toast={toast}`. `Toast.Title` y `Toast.Description` leen el contenido del toast; no los rellenes a mano.
- Para lanzar un toast: `toastManager.add({ title, description, type: 'success' })`. El método es `add` y la propiedad es `type` (un string; por convención `success`/`error`/`info`), no `push`/`variant`. También existen `close`, `update` y `promise`.
- Usa `Toast.useToastManager()` dentro del árbol y `Toast.createToastManager()` para lanzar toasts desde fuera de React; pásalo como `<Toast.Provider toastManager={...}>`.
- El estado del toast se expone con `data-type`, `data-expanded`, `data-limited`, `data-behind`, `data-starting-style` y `data-ending-style`; estílalos con Tailwind en vez de recrear la animación a mano.

**Composición con `render`**

```jsx
<Button render={<Link to="/facturas" />}>Facturas</Button>
<Select.Trigger render={<Button variant="secondary" />} />
```

`render` sustituye el elemento y fusiona props; es la vía oficial para componer con `Link`, triggers y botones del kit. El componente que pongas en `render` debe reenviar `ref` y esparcir las props que recibe en su nodo DOM. **No existe `asChild`**; si copias un ejemplo de Radix, no compila o rompe la semántica.

**Formularios y validación**

- `Field.Root` agrupa etiqueta, control, descripción y error; `Field.Label` se asocia solo al control y `Field.Error` se enlaza con `aria-describedby`.
- Para errores del servidor: `invalid` en `Field.Root` y `<Field.Error match={true}>`; para validación nativa, usa `match="valueMissing"`, `"typeMismatch"`, etc.
- `Field.Control` puede omitirse si usas `Select`, `Checkbox`, `Combobox` u otro input de Base UI: Field los reconoce.
- No escribas `aria-describedby` a mano si Field ya lo gestiona; duplicarlo confunde a lectores de pantalla.

**Select, AlertDialog y Toast: límites**

- `Select` no filtra; para listas grandes usa `Combobox`. `Select.Label` es la etiqueta accesible; si no hay etiqueta visible, pasa `aria-label` al `Select.Trigger`.
- `AlertDialog` no es un modal genérico: úsalo para confirmaciones destructivas o decisiones que exigen respuesta. Para formularios y contenido normal, usa `Dialog`.
- Un toast no sustituye un error de campo ni un mensaje crítico: si el usuario debe actuar, usa `AlertDialog` o un error inline.

**Estilos y portales**

- Estiliza con los atributos de estado documentados (`data-pressed`, `data-invalid`, `data-highlighted`, `data-disabled`, `data-placeholder`, `data-side`, `data-starting-style`, `data-ending-style`…) y con `className` como función `(state) => ...` si necesitas lógica.
- Los popups van en un portal: envuelve la app en un contenedor con `isolation: isolate` para que los `z-index` de la página no los tapen.
- Para iOS 26+ Safari, añade `body { position: relative; }`; los backdrops de los diálogos lo necesitan para cubrir el viewport visual.
- No parchees los estilos internos de Base UI con `!important` ni dependas de clases generadas: usa la API pública y los `data-*`.

**Prohibiciones (rompen la guía)**

| Incorrecto | Correcto |
|---|---|
| `@base-ui-components/react` | `@base-ui/react` |
| `<Button asChild>` | `<Button render={<Link />}>` |
| `toastManager.push({ variant })` | `toastManager.add({ type })` |
| `<FieldLabel>` / `<FieldControl>` | `<Field.Label>` / `<Field.Control>` |
| `<AlertDialog>` genérico | `AlertDialog.Root` + `Trigger` + `Portal` + `Backdrop` + `Popup` + `Title` + `Description` + `Close` |
| `<Select>` genérico | `Select.Root` + `Label` + `Trigger` + `Value` + `Icon` + `Portal` + `Positioner` + `Popup` + `List` + `Item` + `ItemText` + `ItemIndicator` |
| `Toast.Root` sin `toast` | `<Toast.Root toast={toast}>` |

**Versionado**

- Fija `@base-ui/react` con `^1.9` y el `package-lock.json`; comprueba la versión publicada con `npm view @base-ui/react version`.
- Al actualizar, revisa las notas de release y vuelve a leer la página del componente que uses: los nombres y la anatomía pueden cambiar entre versiones menores.
- Si un ejemplo del repositorio y la documentación oficial discrepan, corrige el ejemplo; no adaptes la documentación al código.

**Kit propio**

Envoltorios finos sobre Base UI + `class-variance-authority`:

```text
ui/button.jsx      ui/card.jsx       ui/badge.jsx       ui/input.jsx
ui/select.jsx      ui/field.jsx      ui/alert-dialog.jsx   ui/progress.jsx
```

Convenciones: `variant` (`default`/`secondary`/`ghost`/`destructive`/`link`), `size` (`default`/`sm`/`xs`/`icon-sm`…), `className` con `cn()` (clsx + tailwind-merge) y `render={<Button .../>}` para componer con `Link` o triggers de Base UI. El kit puede simplificar la API, pero **no puede cambiar la anatomía ni la semántica de Base UI**.

### 12.4 Accesibilidad y responsive

- Objetivos táctiles ≥ 44 px, foco visible, `aria-label` en botones de solo icono.
- `prefers-reduced-motion` respetado; tema claro/oscuro con `class` strategy.
- Formularios navegables con teclado; `AlertDialog` para acciones destructivas.
- Contraste suficiente y mensajes de error asociados al campo (`aria-describedby`).

### 12.5 Pruebas de UI

El grueso se prueba en la API, pero el flujo crítico (alta → operación → cobro → documento) necesita un **smoke manual documentado** en cada release. Si el proyecto crece, añade Playwright para 3–5 recorridos; no intentes cubrir cada botón.

---

## 13. Pruebas

- Runner: `node --test` nativo, **sin dependencias de test**.
- Cada archivo levanta la app real y usa `fetch`: se prueba el contrato HTTP de verdad.
- **Nunca** apuntes las pruebas a la base de desarrollo: la URL de test es obligatoria, debe terminar en `_test` y `resetSchema()` se niega a ejecutarse fuera de `NODE_ENV=test`.

### 13.1 Entorno y base de test

No confíes en asignar `process.env` después de los `import`: en ESM los imports estáticos se evalúan **antes** del cuerpo del módulo y la instancia de Sequelize lee `DATABASE_URL` al construirse. Si se inicializa *eager*, las pruebas pueden caer en la base de desarrollo. Usa `--env-file` o un módulo de setup importado primero:

```json
"test": "node --env-file=.env.test --test --test-concurrency=1"
```

```text
# server/.env.test (no se versiona; .env.test.example sí)
NODE_ENV=test
DATABASE_URL=postgres://usuario:clave@localhost:5432/app_test
ADMIN_PASSWORD=test-admin-password
SETTINGS_SECRET=test-only-secret-de-32-bytes-o-mas
```

```js
// tests/setup.js — importado antes que app/db, o sustituido por --env-file
const url = process.env.DATABASE_URL;
if (process.env.NODE_ENV !== 'test' || !url) {
  throw new Error('Las pruebas requieren NODE_ENV=test y DATABASE_URL');
}
const dbName = new URL(url).pathname.replace(/^\//, '');
if (!dbName.endsWith('_test')) {
  throw new Error('Las pruebas solo pueden ejecutarse contra una base *_test');
}
```

- `resetSchema()` borra el **esquema**, no la base. Crea la base `app_test` una vez (`createdb`) y deja que el test limpie el esquema.
- En tests, `createApp()` puede ejecutar `sequelize.sync()` (sin `alter`) sobre la base `_test` tras `resetSchema()`; en producción nunca se usa `sync()`.
- Un archivo de test, una API real: `before` levanta la app en puerto aleatorio (`listen(0)`) y `after` cierra el servidor y la conexión de Sequelize.

```js
import { before, after } from 'node:test';
import { createApp } from '../src/app.js';
import { resetSchema, closeDb } from '../src/db/index.js';

let server;
let baseUrl;

before(async () => {
  await resetSchema();
  server = (await createApp()).listen(0);
  baseUrl = `http://localhost:${server.address().port}`;
});

after(async () => {
  server.closeAllConnections?.();          // evita cuelgues por keep-alive
  await new Promise((resolve) => server.close(resolve));
  await closeDb();
});
```

- En Node 20, `server.close()` cierra las conexiones idle, pero las keep-alive pueden colgar el proceso: llama a `server.closeAllConnections()` antes de cerrar. En versiones anteriores es obligatorio.
- Nombra los archivos `*.test.js` o pasa un glob explícito: el descubrimiento por defecto de `node --test` no es infinito.
- Si los archivos comparten base, `--test-concurrency=1`; si cada uno usa una base propia, puedes paralelizar.

### 13.2 Cookies en las pruebas

`fetch` de Node **no tiene cookie jar**; `credentials: 'same-origin'` es una opción de navegador. Sin un helper que lea `Set-Cookie` y reenvíe `Cookie`, todas las pruebas autenticadas devolverán 401.

```js
// tests/helpers.js
export function createCookieJar() {
  const jar = new Map();
  return {
    header() {
      return [...jar].map(([k, v]) => `${k}=${v}`).join('; ');
    },
    absorb(res) {
      for (const cookie of res.headers.getSetCookie?.() ?? []) {
        const [pair] = cookie.split(';');
        const index = pair.indexOf('=');
        if (index <= 0) continue;
        const name = pair.slice(0, index);
        if (/max-age=0/i.test(cookie)) jar.delete(name);
        else jar.set(name, pair.slice(index + 1));
      }
    },
  };
}

export function makeClient(baseUrl, jar) {
  return async function request(path, options = {}) {
    const res = await fetch(`${baseUrl}${path}`, {
      ...options,
      headers: {
        ...(options.body !== undefined ? { 'content-type': 'application/json' } : {}),
        ...(jar.header() ? { cookie: jar.header() } : {}),
        ...options.headers,
      },
    });
    jar.absorb(res);
    return res;
  };
}
```

### 13.3 Casos que conviene tener siempre

- **Contrato HTTP:** 401 sin sesión; 403 por rol; 404 frente a 403 cuando no se quiere filtrar existencia; 429 por rate limit.
- **Validación:** campos faltantes, tipos incorrectos, longitudes máximas, campos desconocidos y *mass assignment* (no aceptar `role`, `orgId`, `estado` del body).
- **Idempotencia:** repetir la operación no duplica (alta, invitación, webhook, job, cobro).
- **Rollback:** un trigger que lanza excepción dentro de `withTransaction()` → nada queda escrito.
- **Aislamiento entre organizaciones (IDOR):** dos organizaciones y un recurso de cada tipo; el usuario de A no puede leer, editar ni borrar el de B.
- **Sesiones:** rotación al login, revocación al cambiar contraseña/rol, expiración y cookie `HttpOnly`.
- **Contraseñas:** hash corrupto → `false`, no 500; contraseña incorrecta y email inexistente devuelven el mismo mensaje.
- **Rate limit y CSRF:** login repetido → 429; mutación con `Origin` ajeno → 403.
- **Webhook:** firma inválida → 401/403; evento repetido → una sola fila; evento viejo → rechazado.
- **Dinero:** redondeo, moneda, pago duplicado, importe cero, refund/contracargo.
- **Migración:** esquema legacy + doble ejecución + conteos antes/después + huérfanos que abortan.
- **Jobs:** doble barrido no duplica avisos; un barrido solapado no se ejecuta dos veces.
- **Auditoría:** el evento de dinero/auth/admin queda registrado con `request_id`, sin secretos.
- **Paginación:** límite máximo, orden estable y sin duplicados entre páginas.
- **Soft delete:** el recurso borrado no aparece en listados ni se puede editar, pero la evidencia contable sigue existiendo.
- **Frontend:** smoke manual del flujo crítico; si crece, Playwright para 3–5 recorridos.

### 13.4 Lo que no se prueba en la API

Los guards del cliente, el diseño responsive y la accesibilidad necesitan revisión manual o tests de UI. No los des por cubiertos con tests de endpoints.

---

## 14. Método de trabajo (ODD + SDD + BDD + TDD + RDD)

Es lo que hace que el proyecto sea mantenible en el tiempo; vale la pena adoptarlo desde el principio.

| Método | Pregunta que responde | Artefacto | Dónde |
|---|---|---|---|
| **ODD** — Organic Driven Development | ¿Vale la pena? ¿Qué palanca de negocio mueve? ¿Cuánto riesgo tiene? | contexto de negocio + clasificación | `specs/negocio.md` |
| **SDD** — Spec-Driven Development | ¿Qué debe hacer exactamente y cómo sabremos que está bien? | spec con reglas `S#`, criterios `AC#`, contratos | `specs/<feature>.md` |
| **BDD** — Behavior-Driven Development | ¿Cómo se comporta el sistema visto desde el negocio? | escenarios Gherkin en español con `@s#` | `features/<feature>.feature` |
| **TDD** — Test-Driven Development | ¿Cómo lo construimos sin romper lo existente? | tareas `T#`, rojo → verde → refactor, evidencia | `plans/<feature>.md` |
| **RDD** — Receipt-Driven Development (opcional) | ¿La tarea revisada es exactamente la que se entrega y con qué evidencia? | recibo JSON del candidato congelado, checks, hallazgos y veredicto | `receipts/<tarea>.json` |

### Ciclo obligatorio de cada pedido

1. **Autorizar** — investigar o explicar no autoriza cambios de código.
2. **Explorar** el código y los requisitos antes de escribir.
3. **Clasificar**:
   - *Chico* (entendido, riesgo contenido, reanudable con el pedido + `git diff`): se hace directo, sin spec/plan/escenarios; el recibo RDD se decide según el riesgo.
   - *Grande* (dinero, cumplimiento, datos personales, autenticación, suscripciones, migraciones de esquema, integraciones externas): **tres artefactos antes del primer cambio de código** (spec + plan + escenarios), revisión de seguridad y **recibo RDD al cerrar**.
4. **Especificar (SDD)**: la spec cita el pedido literal, declara el impacto de negocio (cobranza · conversión · retención · cumplimiento) y define reglas `S#` con criterios de aceptación. Incluye requisitos no funcionales: seguridad, privacidad, accesibilidad, rendimiento y retención.
5. **Acordar comportamiento (BDD)**: un escenario Gherkin por regla, etiquetado `@s#`, en lenguaje de negocio, sin SQL ni detalles internos.
6. **Implementar (TDD)**: una prueba por regla pedida y una de regresión por cada comportamiento existente que se toque; rojo → verde → refactor.
7. **Cerrar con evidencia**: tests, build, verificación del flujo real, migración probada y, si aplica, restauración de backup. Una tarea sin evidencia **no** se marca hecha; los fallos y pendientes se reportan tal cual.
8. **Revisar y emitir recibo (RDD)**: si RDD está activo, congela el candidato, revisa con las lentes que correspondan, ejecuta los checks y escribe `receipts/<tarea>.json`. Sin `verdict: "approved"` la tarea **no** se cierra.
9. **Actualizar documentos**: spec, plan, `README.md` y `AGENTS.md` si cambió una convención.

### 14.1 RDD: recibo por tarea (`receipts/<tarea>.json`)

RDD es una capa **opcional** de revisión y evidencia. Cuando se adopta, **cada tarea `T#` termina con un recibo**; se recomienda activarlo siempre en tareas de dinero, autenticación, datos personales, migraciones e integraciones externas. El recibo no sustituye los tests ni el code review: los complementa con una foto inmutable de lo revisado.

El flujo es:

1. **Congelar el candidato**: commit base, commit revisado, worktree y hash del diff. Nada cambia durante la revisión.
2. **Clasificar el riesgo**: `low` (lectura estructural, sin lentes), `medium` (una lente de foco) o `high` (4R: `risk`, `resilience`, `readability`, `reliability`).
3. **Revisar y ejecutar checks**: tests, build, smoke y seguridad. Se registra el comando, el `exit_code` y un resumen; la salida completa va a CI o a un log referenciado, nunca dentro del JSON si puede contener secretos o PII.
4. **Corregir una sola vez**: si hay hallazgos bloqueantes, se permite **un** intento acotado; después se vuelve a validar. Si no pasa, el veredicto es `rejected` o `inconclusive`.
5. **Emitir el recibo**: se escribe `receipts/<tarea>.json` y solo un `verdict: "approved"` permite cerrar la tarea.

```json
{
  "task": "T3",
  "feature": "facturacion",
  "created_at": "2026-10-28T14:32:00Z",
  "candidate": {
    "base_commit": "9f1c2a7",
    "head_commit": "4b8d0e1",
    "worktree": "/workspaces/app",
    "diff_sha256": "sha256:6f1c2a..."
  },
  "scope": {
    "files": [
      "server/src/services/invoices.js",
      "server/test/invoices.test.js"
    ]
  },
  "risk": "high",
  "lenses": ["risk", "resilience", "readability", "reliability"],
  "checks": [
    {
      "name": "tests",
      "command": "npm test",
      "cwd": "server",
      "exit_code": 0,
      "summary": "42 passed, 0 failed",
      "evidence": "https://ci.example.com/runs/1842"
    },
    {
      "name": "build",
      "command": "npm run build",
      "cwd": ".",
      "exit_code": 0,
      "summary": "ok",
      "evidence": "https://ci.example.com/runs/1842/build"
    }
  ],
  "findings": [
    {
      "severity": "high",
      "blocking": true,
      "file": "server/src/services/invoices.js",
      "line": 42,
      "message": "La emisión no es idempotente",
      "status": "fixed"
    }
  ],
  "correction": {
    "attempts": 1,
    "max_attempts": 1,
    "result": "passed"
  },
  "verdict": "approved",
  "limitations": [
    "No se probó el webhook real de Stripe; se usó el modo simulado."
  ],
  "author": { "agent": "opencode", "model": "..." },
  "reviewer": { "agent": "reviewer", "model": "..." }
}
```

Campos que no pueden faltar:

- `candidate`: identidad exacta de lo revisado. `base_commit` y `head_commit` delimitan el candidato; si la revisión es previa al commit, `head_commit` coincide con `base_commit` y el diff vive en el worktree. `diff_sha256` es la huella; si el diff cambia, el recibo deja de valer y se regenera.
- `scope.files`: archivos revisados. El commit solo puede tocar esos archivos **más el propio recibo**; si aparece otro, se revisa y se emite un recibo nuevo.
- `risk` y `lenses`: dejan explícito qué nivel de revisión se aplicó. En `low`, `lenses` puede ser `[]`; en `high`, las 4R.
- `checks`: comandos realmente ejecutados. `evidence` apunta a CI o a un log; no pegues salidas enormes ni secretos.
- `findings`: solo hallazgos causados por el candidato. `blocking: true` para `critical`/`high`; `status`: `fixed`, `open` o `accepted`.
- `correction`: como máximo `attempts: 1` con `max_attempts: 1`; `result` es `not-needed`, `passed` o `failed`. Un bucle de correcciones invalida el recibo.
- `verdict`: `approved`, `rejected` o `inconclusive`. Solo `approved` cierra la tarea.
- `limitations`: lo que no se probó o quedó fuera de alcance. Un array vacío es una afirmación fuerte: úsalo solo si de verdad no hay limitaciones.
- `author` y `reviewer`: contextos distintos; idealmente agentes o modelos distintos. El autor no firma su propia revisión.

Reglas de convivencia:

- `<tarea>` es el ID de la tarea del plan (`T3`). Si varias features repiten `T1`, usa `receipts/facturacion-T1.json` y mantén el mismo ID en el plan.
- El recibo se versiona en Git junto al código y es **inmutable** una vez aprobado. Si cambia el candidato, se regenera; no se edita el veredicto.
- El recibo es **evidencia**, no autorización de entrega: commit, push, PR y despliegue siguen las reglas del repositorio y la revisión humana.
- No guardes secretos, tokens ni PII en `receipts/`. Si un check los produce, guarda solo el hash o una versión redactada.
- Usa fechas ISO 8601 en UTC y SHA-256 para las huellas.

### Definition of Done

Una feature no está terminada hasta que:

- [ ] `npm test` en verde y `npm run build` correcto.
- [ ] El flujo real se verificó (smoke) y quedó registrado en el plan.
- [ ] Si RDD está activo: `receipts/<tarea>.json` tiene `verdict: "approved"`, el diff coincide con el candidato revisado y no quedan hallazgos bloqueantes abiertos.
- [ ] Las reglas `S#` tienen su escenario `@s#` y su prueba.
- [ ] Los casos de error (401/403/404/409/429) están cubiertos.
- [ ] No hay secretos, tokens ni PII en logs ni en el repositorio.
- [ ] Si toca dinero, auth o datos personales: revisión de seguridad y auditoría.
- [ ] Si toca esquema: migración con `--dry-run`, backup y prueba de doble ejecución.
- [ ] La UI tiene estados de carga, vacío y error; accesibilidad revisada.
- [ ] `README.md`/`AGENTS.md`/spec actualizados.
- [ ] Commits pequeños y revisables.

### Convenciones

- **Todo en español**: interfaz, mensajes de error, documentos y commits (Conventional Commit en español: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`).
- **Commits pequeños** (≈400 líneas) y revisables; cada tarea registra su commit en el plan.
- Los artefactos del método viven **solo** en `/specs`, `/plans`, `/features` y `/receipts`; no se crean otros directorios de método.
- Cada documento lleva un `## Log` con entradas `L1`, `L2`… y `L1` conserva el **pedido literal**, sin parafrasear.
- Reglas y pruebas se trazan entre sí: `S#` (spec) ↔ `@s#` (escenario) ↔ `T#` (tarea) ↔ nombre de la prueba ↔ `receipts/<tarea>.json`.
- Las decisiones y trade-offs (por ejemplo, “una organización por usuario”) se anotan en la spec; no se dejan solo en la cabeza.

---

## 15. Modelar tu dominio

Antes de escribir la primera tabla, responde estas preguntas:

1. **¿Quién es el cliente económico?** Si es una empresa o un equipo, modela una organización y dale roles; si es una persona, puede bastar el usuario (pero el patrón de organización no estorba y evita rehacerlo).
2. **¿Cuál es la entidad principal** (orden, reserva, ticket, proyecto, campaña…)? Esa tabla y sus detalles llevan `org_id`.
3. **¿Qué se factura?** El plan, el uso, los servicios; el cobro pertenece a la organización, no a la persona.
4. **¿Qué moneda, impuestos, periodo y redondeo?** Define la unidad mínima, la moneda por organización, el tratamiento de impuestos y quién puede cambiar precios.
5. **¿Qué estados tiene el ciclo?** (pendiente → confirmado → en curso → completado → cancelado) y qué evento dispara cada transición. ¿Quién puede dispararla? ¿Queda auditada?
6. **¿Qué datos son privados del cliente?** Esos endpoints exigen `requireOrg`; el admin de plataforma no entra. Dibuja la **matriz de permisos**: rol × recurso × acción.
7. **¿Qué documento debe poder generar el sistema?** (PDF, XML, CSV) y qué datos fiscales necesita. ¿La numeración debe ser sin huecos? ¿Cuánto se conserva?
8. **¿Qué normativa aplica?** Privacidad, retención, consentimiento, exportación y borrado. Cada dato que se exija tiene su campo, su validación y su evidencia.
9. **¿Qué integraciones externas hay y qué pasa sin credenciales?** Cada una en modo simulado desde el día uno, con timeout, reintento y degradación.
10. **¿Qué operaciones deben ser idempotentes?** Cobros, invitaciones, webhooks, jobs, reintentos y altas.
11. **¿Qué pasa al borrar?** Baja lógica, `RESTRICT` en dinero/documentos, anonimización de personas y retención fiscal.
12. **¿Qué se configura a nivel plataforma y qué a nivel organización?** No mezcles credenciales globales con las de un cliente.
13. **¿Cómo se pagina, filtra y ordena?** Límite máximo, orden estable y índices que lo soporten.
14. **¿Qué zona horaria manda?** Instantes en `timestamptz`; días locales de facturación en `date` con la zona de la organización.
15. **¿Qué eventos se auditan?** Login, cambios de rol, cobros, accesos del admin, borrados y exportaciones.

**Catálogos**: si el negocio tiene listas (tipos, estados, categorías, productos), van en tablas de catálogo con su `id` estable y su etiqueta visible; el frontend las consume por `/api/catalog` y no las reescribe.

**Matriz de permisos** (ejemplo mínimo):

| Acción | owner | operador | lector | admin plataforma |
|---|---|---|---|---|
| Ver datos de la organización | sí | sí | sí | no (solo operativo) |
| Crear/editar recursos | sí | sí | no | no |
| Invitar/revocar miembros | sí | no | no | no |
| Ver/gestionar cobro | sí | no | no | sí (soporte auditado) |
| Exportar datos | sí | según política | no | no |

---

## 16. Trampas técnicas (resueltas de antemano)

### 16.1 Datos y SQL

| Trampa | Síntoma | Solución |
|---|---|---|
| `make_interval(days => n)` cruza el cambio de hora | un periodo de 14 días «dura» 15 | para pruebas de N días exactos usa **`make_interval(hours => n * 24)`**; para ciclos mensuales usa `+ interval '1 month'` |
| `UNION ALL` de consultas con joins | `column reference "id" is ambiguous` | calificar **todas** las columnas (`r.id`, `f.user_id AS id`) |
| `SELECT c.id … FROM hijos c JOIN padres p` sin prefijos | `column "nombre" is ambiguous` | prefijar siempre las columnas (`c.nombre`) |
| `INSERT … ON CONFLICT (org_id)` sin índice único | error de inferencia | crear el índice único **antes** de usarlo |
| `ON CONFLICT` con índice parcial | “no unique or exclusion constraint matching…” | repetir el predicado: `ON CONFLICT (org_id, concept, period_start) WHERE estado = 'aprobado'` |
| `NOT NULL` en `org_id` durante la migración | falla si quedan filas huérfanas | backfill primero y **abortar** si sobran filas sin organización |
| Borrado en cascada de pagos/documentos | se pierde evidencia contable | `ON DELETE RESTRICT` + baja lógica de la organización |
| `subscriptions.org_id` único | no hay historial de suscripciones | índice único **parcial** para estados vigentes |
| `float` para dinero | céntimos perdidos, descuadres | `numeric(12,2)` o enteros en unidad mínima |
| FK sin índice | listados y borrados lentos | índice en cada FK, además de `org_id` |
| `actualizado_en` “se actualiza solo” | la fecha miente | trigger `BEFORE UPDATE` o convención obligatoria |
| `CREATE INDEX CONCURRENTLY` dentro de transacción | error de PostgreSQL | ejecutarlo **fuera** de la transacción |
| Dos despliegues ejecutan migraciones a la vez | deadlock o error de catálogo | `pg_advisory_xact_lock` al inicio de la migración |
| `SELECT *` | rompe al añadir columnas y trae datos de más | columnas explícitas |
| `OFFSET` profundo | páginas lentas y saltos | keyset `(creado_en, id)` + `LIMIT` acotado |
| `UNIQUE` con `NULL` | varias filas “vacías” | índice parcial o `NULLS NOT DISTINCT` (PG 15+; en 14, parcial) |
| `dedupe_key` global | colisiones entre organizaciones | `UNIQUE (org_id, dedupe_key)` |
| Factura con `SERIAL` y huecos | incumple numeración legal | contador en tabla con `FOR UPDATE` (las secuencias dejan huecos), o numeración por rango |
| Pool agotado | peticiones colgadas | `pool.max`, `pool.acquire`, `dialectOptions.statement_timeout`; no retengas transacciones |
| Llamada HTTP dentro de transacción | locks y conexiones abiertas | outbox: escribe y procesa fuera |
| `ALTER TABLE` bloqueante | downtime | `CONCURRENTLY`/`NOT VALID` y ventana de mantenimiento |

### 16.2 HTTP, auth y secretos

| Trampa | Síntoma | Solución |
|---|---|---|
| AES-GCM sin *auth tag* | no se detecta corrupción/manipulación | guardar `getAuthTag()` y usar `setAuthTag()` al descifrar |
| SHA-256 como KDF de una frase | clave débil | `scrypt`/`HKDF` con sal; SHA-256 solo para claves aleatorias |
| IV/nonce reutilizado | GCM pierde seguridad | IV aleatorio de 12 bytes por escritura |
| `timingSafeEqual` con longitudes distintas | excepción | comparar longitudes y capturar; devolver `false` |
| `scryptSync` en login | bloquea el event loop | `crypto.scrypt` asíncrono |
| Token de sesión en claro en la base | un dump permite robar sesiones | guardar `SHA-256(token)` |
| Sesión no rotada al login | *session fixation* | token nuevo y revocación del anterior |
| `Max-Age=7d` | cookie inválida | `Max-Age=604800` (segundos) o `maxAge` en ms |
| `Secure` detrás de proxy | cookie no se emite | `app.set('trust proxy', 1)` + TLS |
| `SameSite=None` sin `Secure` | cookie rechazada | exigir `Secure` o usar `Lax` |
| Solo `SameSite=Lax` | CSRF residual | comprobar `Origin`/`Referer` en mutaciones |
| Sin rate limit | fuerza bruta y spam | limitador por IP+email, 429 |
| Sin límite de body | DoS por JSON gigante | `express.json({ limit: '100kb' })` |
| Stack al cliente | filtra rutas y versiones | log interno + mensaje genérico + `request_id` |
| Webhook con `express.json()` | no puedes verificar la firma | `express.raw()` o `verify` para conservar el body |
| Webhook repetido | doble cobro/aviso | `event_id` único + ventana anti-replay |
| Secreto en logs o URL | filtración | cabecera HMAC, redacción y nunca query string |
| `orgId` por query en admin | confusión y logs | ruta explícita `/api/admin/orgs/:orgId/...` + auditoría |
| 403 en recursos ajenos | revela existencia | 404 cuando corresponda |
| Mensaje “email no existe” | enumeración de cuentas | respuesta genérica y mismo tiempo |
| Token de reset en claro | robo de cuenta | hash, un solo uso, expiración, revocar sesiones |
| CORS permisivo con credenciales | CSRF/XSS ampliado | mismo origen; allowlist explícita si hace falta |
| `PUBLIC_URL` sin validar | *open redirect* | allowlist de dominios |
| URL de tercero sin validar | *SSRF* | allowlist y bloqueo de IPs internas |
| `...req.body` en el update | *mass assignment* (`role`, `orgId`) | elegir campos explícitamente |
| `ORDER BY ${columna}` | inyección SQL | lista blanca de columnas |

### 16.3 Frontend

| Trampa | Síntoma | Solución |
|---|---|---|
| `useResource` con loader inline | bucle infinito de fetch | loader en `ref` + `AbortController` |
| Respuesta vieja pisa a la nueva | datos inconsistentes | id de petición y abortar la anterior |
| 401 no manejado | UI zombie | `ApiError.status` + limpieza global de sesión |
| Guard solo en el cliente | cualquiera llama la API | el servidor autoriza; el cliente solo guía |
| Token en `localStorage` | XSS lo roba | cookie `HttpOnly` |
| `dangerouslySetInnerHTML` | XSS | evitar o sanitizar |
| PWA cachea `/api` | datos viejos y privados | `navigateFallbackDenylist` y sin runtime cache de API |
| `autoUpdate` en formulario largo | recarga y pierde datos | `registerType: 'prompt'` |
| `StrictMode` duplica efectos | doble fetch en desarrollo | abortar en cleanup |
| `asChild` de Radix en Base UI | prop ignorada o error | `render={<Componente />}` |
| `toastManager.push({ variant })` | no se muestra el toast | `add({ type })`; `Toast.Root toast={toast}` |
| `@base-ui-components/react` | paquete obsoleto | `@base-ui/react` |
| `FieldLabel` / `FieldControl` | componentes inexistentes | `Field.Label` / `Field.Control` |
| `build.rollupOptions` en Vite 8 | opción deprecada | renombrar a `build.rolldownOptions` |
| `manualChunks` en Vite 8 | objeto no soportado | usar `output.codeSplitting.groups` |
| `__dirname` en `vite.config.js` ESM | `ReferenceError` | `import.meta.dirname` |
| `codeSplitting.groups` manda todo a un chunk | bundle enorme | separar `react`, `ui`, `icons`, `vendor` |
| `app.get('*')` en Express 5 | error de path-to-regexp | `app.get('/{*splat}', ...)` |
| `index.html` cacheado | despliegue invisible | assets inmutables, HTML `no-cache` |
| `fetch` sin timeout | spinner eterno | `AbortSignal.timeout()` |
| Doble submit | duplicados | deshabilitar + idempotency key |
| Formatear dinero a mano | redondeo/formatos raros | `Intl.NumberFormat` y unidad mínima |

### 16.4 Pruebas y operación

| Trampa | Síntoma | Solución |
|---|---|---|
| Pruebas apuntando a la base de desarrollo | datos reales destruidos | `DATABASE_URL` de test `*_test` + guardia en `resetSchema()` |
| `process.env` asignado tras los imports | la base se fija demasiado tarde | `--env-file` o setup importado primero |
| Tests en paralelo sobre la misma base | carreras y drops | `--test-concurrency=1` |
| Archivos que `node --test` no descubre | 0 pruebas y falsa confianza | `*.test.js` o glob explícito |
| `server.close()` colgado | test que no termina | Node 20 cierra idle; versiones viejas, `closeAllConnections()` |
| Jobs solapados | doble envío | flag `running` + lock si hay varias instancias |
| `unref()` y SIGTERM | job a medias | parar timers y esperar el barrido |
| “Audit trail” en stdout | no es durable ni consultable | tabla `audit_log` append-only |
| Métricas en memoria | se pierden al reiniciar | aceptar en monolito; externalizar si crece |
| Health sin comprobar la base | tráfico a una app rota | `/health/ready` con `SELECT 1` |
| Backup nunca restaurado | backup que no sirve | simulacro de `pg_restore` |
| Código nuevo con esquema viejo | errores en despliegue | expand → backfill → contract |
| Migración sin lock | dos ejecuciones | advisory lock + transacción |
| Semilla “si la tabla está vacía” | seeds incompletos | insert-if-missing por clave única |
| Cambiar `ADMIN_PASSWORD` no rota nada | contraseña vieja sigue | comando `admin:reset` documentado |
| systemd sin `Restart` | se queda caído | `Restart=on-failure` y health check |
| Logs sin rotación | disco lleno | journald/logrotate + retención |

---

## 17. Receta de arranque y checklist de producción

### 17.1 Arranque

1. `npm init` en la raíz con `workspaces: ["server", "client"]`, `engines`, `type: module` y los scripts de la sección 3.
2. Crea `.env.example`, `.env.test.example`, `.nvmrc`, `.gitignore` (`.env`, `.env.test`, `node_modules`, `dist`, `*.log`) y el `package-lock.json`.
3. `server/`: `express`, `helmet`, `sequelize`, `pg` (+ `pdfkit` si hay documentos) y dev `sequelize-cli`. `client/`: react, react-dom, react-router-dom, tailwind, base-ui, lucide-react, clsx, tailwind-merge, cva.
4. Copiar `server/src/db/index.js`, `models/`, `config/env.js`, `lib/password.js`, `lib/crypto.js`, `lib/validate.js`, `lib/cookies.js`, `lib/rate-limit.js`, `middlewares/auth.js`, `middlewares/error.js`, `services/observability.js`. Inicializar Sequelize CLI (`npx sequelize-cli init`) y configurar `config/config.json` o `.sequelizerc`.
5. Copiar `client/src/components/ui/*`, `ErrorBoundary.jsx`, `context/ToastContext.jsx`, `hooks/useResource.js`, `api/http.js`, `layout/*` y `App.jsx` como esqueleto.
6. Escribir `specs/negocio.md` (modelo de negocio, palancas, ciclo del dinero, moneda, impuestos, retención).
7. Modelar el dominio (sección 15), la matriz de permisos y la primera spec con reglas `S#`.
8. Escribir los `.feature` y el plan TDD **antes** del código; crear `receipts/README.md` con el esquema del recibo.
9. Implementar por tareas con pruebas primero; `npm test` y `npm run build` en verde en cada fase.
10. Configurar la línea base: rate limit, sesión hasheada, CSRF/Origin, body limit, validación, error handler, `trust proxy`, cabeceras.
11. Base de datos: modelos Sequelize, migraciones versionadas, índices por `org_id`, FKs, `CHECK`, `actualizado_en`, `resetSchema()` con guardia; sin `sync()` en producción.
12. Jobs: `running`, dedupe por organización, apagado ordenado.
13. `npm run dev` y verificar el flujo real de punta a punta (alta → operación → cobro → documento).
14. Documentar el runbook: despliegue, backup, restauración, rotación de secretos y reseteo de admin.

### 17.2 Antes de producción

- [ ] `NODE_ENV=production`, `SETTINGS_SECRET` fuerte y `ADMIN_PASSWORD` fuerte/rotada.
- [ ] TLS + HSTS + `trust proxy` + cookies `Secure`/`HttpOnly`/`SameSite=Lax`.
- [ ] Rate limit en login/registro/reset y límite de body (`express.json({ limit: '100kb' })`).
- [ ] Validación de entrada y errores genéricos sin stack.
- [ ] Sesiones hasheadas, rotación al login, revocación y limpieza de expiradas.
- [ ] Recuperación de contraseña y verificación de correo, si el panel es público.
- [ ] Webhooks con HMAC, body crudo, `event_id` único y anti-replay.
- [ ] Cobros idempotentes, numeración de factura y conciliación.
- [ ] Outbox con backoff, máximo de intentos y *dead letter*; consentimiento de mensajería.
- [ ] Migraciones con `--dry-run`, backup y lock; orden expand → backfill → contract.
- [ ] Backup automático + **restauración probada**; RPO/RTO definidos.
- [ ] Auditoría durable de auth/dinero/admin; logs con `request_id` y sin PII/secretos.
- [ ] `/health/live` y `/health/ready`; `/api/metrics` solo admin.
- [ ] systemd con `Restart=on-failure`, usuario sin privilegios, `EnvironmentFile` 600.
- [ ] Logs con rotación/retención y monitorización de disco/DB/uptime.
- [ ] Smoke del flujo completo y plan de rollback documentado.
- [ ] Recibos RDD de las tareas críticas archivados, con `verdict: "approved"` y evidencia verificable.
- [ ] `README.md`, `AGENTS.md`, spec, `receipts/README.md` y runbook al día.

---

## 18. Glosario

| Término | Significado |
|---|---|
| **tenant** | la organización: dueña de los datos y de la facturación |
| **owner / operador / lector** | roles dentro de la organización (permisos de datos y de cobro) |
| **admin de plataforma** | operador transversal: ve lo operativo de todos y no entra a datos privados |
| **provider-ready** | la integración está escrita pero solo se activa con credenciales; sin ellas, modo simulado |
| **outbox** | bandeja de mensajes salientes con estado, errores y reintentos |
| **dead letter** | mensaje que agotó los reintentos y queda para revisión manual |
| **pista (mask)** | últimos caracteres de un secreto; lo único que la API devuelve |
| **dedupe_key** | clave única (con ámbito de organización) que hace idempotente un aviso o un cobro |
| **idempotency key** | clave enviada a un proveedor para que reintentar no cree otra operación |
| **auth tag** | código de autenticación de GCM; sin él no se verifica integridad |
| **KDF** | función de derivación de clave (scrypt, HKDF); no es un hash cualquiera |
| **HMAC** | firma con clave compartida para verificar webhooks |
| **CSRF / CORS / CSP / HSTS** | protección de peticiones cruzadas / política de orígenes / política de contenido / TLS obligatorio |
| **IDOR** | acceso a un recurso ajeno cambiando un identificador; se previene filtrando por `org_id` |
| **rate limit** | límite de peticiones por IP/clave para evitar fuerza bruta y abuso |
| **request_id / correlation ID** | identificador para seguir una petición en logs, errores y auditoría |
| **audit log** | registro append-only de acciones sensibles; no es un log de aplicación |
| **soft delete** | baja lógica (`borrado_en`) para conservar evidencia |
| **keyset pagination** | paginar por `(creado_en, id)` en vez de `OFFSET` profundo |
| **advisory lock** | lock de PostgreSQL para coordinar migraciones o jobs |
| **readiness / liveness** | ¿puede atender tráfico? / ¿el proceso está vivo? |
| **unidad mínima** | centavos o la unidad entera de la moneda; nunca `float` |
| **expand / backfill / contract** | fases de una migración segura sobre bases existentes |
| **RPO / RTO** | pérdida máxima de datos aceptable / tiempo máximo de recuperación |
| **RDD** | Receipt-Driven Development: revisión de un candidato congelado que produce un recibo verificable antes de cerrar la tarea |
| **recibo** | JSON de evidencia de una tarea: candidato, checks, hallazgos, veredicto y limitaciones |
| **smoke** | comprobación manual del flujo real, con evidencia registrada |


