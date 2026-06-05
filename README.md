# Tarjeta de Fidelidad para Cafeteria - Apple Wallet

App propia para crear, distribuir y actualizar tarjetas de fidelidad (`.pkpass`)
en Apple Wallet. Modelo "10 sellos = 1 cafe gratis", con codigo QR, niveles de
cliente y actualizacion en tiempo real via notificaciones push.

## Que incluye

- **Generador de passes** firmados (`.pkpass`) con tus certificados de Apple.
- **Web service de Apple Wallet** (registro, baja y actualizacion de passes).
- **Notificaciones push (APNs)** para actualizar el saldo de sellos al instante.
- **Cola de push (BullMQ + Redis)** con reintentos automaticos y workers escalables.
- **API de administracion** para crear clientes, sumar sellos y canjear premios.
- **Base de datos PostgreSQL** (escalable a miles de cafeterias y varios servidores).

## Arquitectura

```
Cliente compra cafe
      │
      ▼
POST /admin/passes/:serial/sellos   ← tu POS o panel suma 1 sello
      │
      ▼
Se actualiza PostgreSQL  ──►  Cola (Redis)  ──►  Worker  ──►  Push APNs  ──►  iPhone pide el pass actualizado
                                              │
                                              ▼
                              GET /v1/passes/...  devuelve el .pkpass nuevo
```

## Requisitos previos

1. **Cuenta Apple Developer** (99 USD/ano): https://developer.apple.com/programs/
2. **Node.js 18+** (usa `http2` y `--watch` nativos).
3. **PostgreSQL 13+** (local con Docker o un servicio gestionado).
4. Certificados de Apple (ver seccion "Certificados").

## Instalacion

```powershell
npm install
copy .env.example .env   # luego edita .env con tus datos
npm run init-db
```

## Base de datos PostgreSQL

La app usa PostgreSQL para poder crecer a miles de cafeterias y repartir la carga
entre varios servidores.

### Opcion A: PostgreSQL local con Docker (recomendado para empezar)

```powershell
docker run --name fidelidad-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=fidelidad -p 5432:5432 -d postgres:16
```

Eso deja la base disponible en `localhost:5432`. En tu `.env`:

```
DATABASE_URL=postgres://postgres:postgres@localhost:5432/fidelidad
PGSSL=false
PG_POOL_MAX=10
```

### Opcion B: Servicio gestionado (produccion)

Crea una base en Render, Supabase, Neon o Amazon RDS y copia la cadena de
conexion que te dan en `DATABASE_URL`. Si exigen SSL, pon `PGSSL=true`.

### Crear las tablas

Con `DATABASE_URL` configurado, ejecuta una sola vez:

```powershell
npm run init-db
```

Esto crea todas las tablas (cafeterias, usuarios, passes, dispositivos y
registros). Si vuelves a ejecutarlo no borra nada: solo crea lo que falte.

## Cola de notificaciones push (Redis)

Cuando sumas un sello, el servidor no espera a Apple: deja el aviso en una **cola
en Redis** y responde al instante. Un **worker** se encarga de hablar con APNs en
segundo plano, con **reintentos automaticos** si algo falla. Asi el panel sigue
rapido aunque envies miles de notificaciones.

### Levantar Redis (Docker)

```powershell
docker run --name fidelidad-redis -p 6379:6379 -d redis:7
```

En tu `.env`:

```
REDIS_URL=redis://localhost:6379
PUSH_WORKER_CONCURRENCY=5
START_WORKER=false
```

### Arrancar el worker

En produccion, el servidor web y el worker corren por separado (puedes lanzar
varios workers para repartir la carga):

```powershell
npm run start     # servidor web (API)
npm run worker    # procesa la cola de push (en otra terminal)
```

Para desarrollo, si quieres un solo proceso que haga todo, pon `START_WORKER=true`
en el `.env` y el propio servidor procesara la cola.

> Si dejas `REDIS_URL` vacio, todo sigue funcionando: los push se envian de forma
> directa, sin cola (modo simple, recomendado solo para pruebas).

## Certificados (paso clave)

Necesitas convertir tus certificados de Apple a formato `.pem` y dejarlos en la
carpeta `certs/`.

### 1. Pass Type ID y certificado

1. En [developer.apple.com](https://developer.apple.com/account/resources) crea un
   **Pass Type ID**: `pass.com.tucafeteria.fidelidad`.
2. Genera un **Pass Type ID Certificate** (subiendo un CSR) y descarga el `.cer`.
3. En **Keychain Access** (Mac), exporta el certificado + clave privada como
   `Certificates.p12`.

### 2. Convertir a PEM

```bash
# Certificado de firma
openssl pkcs12 -in Certificates.p12 -clcerts -nokeys -out certs/signerCert.pem
# Clave privada
openssl pkcs12 -in Certificates.p12 -nocerts -out certs/signerKey.pem
```

### 3. Certificado intermedio WWDR de Apple

Descarga el "Apple WWDR Certificate" (G4) desde
https://www.apple.com/certificateauthority/ y conviertelo:

```bash
openssl x509 -inform DER -in AppleWWDRCAG4.cer -out certs/wwdr.pem
```

Tu carpeta `certs/` debe quedar con: `signerCert.pem`, `signerKey.pem`, `wwdr.pem`.

> En Windows puedes instalar OpenSSL o usar WSL para estos comandos.

## Imagenes de marca

Coloca tus imagenes (logo, icono) en `passModels/fidelidad.pass/`.
Ver [passModels/fidelidad.pass/IMAGENES.md](passModels/fidelidad.pass/IMAGENES.md)
para los tamanos exactos requeridos.

## Configurar el `.env`

Edita `.env` con:

- `PASS_TYPE_IDENTIFIER` y `TEAM_IDENTIFIER` de tu cuenta Apple.
- `WEB_SERVICE_URL` = la URL **publica HTTPS** de tu servidor.
- `AUTHENTICATION_TOKEN` y `ADMIN_API_KEY` con valores secretos aleatorios.

Genera tokens seguros:

```powershell
node -e "console.log(require('crypto').randomBytes(24).toString('hex'))"
```

## Probar sin publicar (generar una tarjeta demo)

```powershell
npm run demo
```

Crea `data/output/CAFE-DEMO0001.pkpass`. Arrastralo al **Simulator de iOS** o
abrelo en un Mac para ver como queda la tarjeta.

## Levantar el servidor

```powershell
npm start
# o en desarrollo con recarga:
npm run dev
```

## Modo SaaS (varias cafeterias)

La app es **multi-tenant**: una sola instancia y un solo certificado de Apple
sirven a muchas cafeterias. Cada cafeteria (tenant) tiene su marca (colores,
logo, nombre), sus reglas (sellos para recompensa) y su propia **API key**, y
sus clientes quedan **aislados** de las demas.

Hay dos niveles de acceso:

| Nivel | Header | Para que |
|-------|--------|----------|
| **Plataforma** (tu, operador SaaS) | `x-platform-key: <PLATFORM_ADMIN_KEY>` | Dar de alta y configurar cafeterias en `/platform`. |
| **Cafeteria** (cada negocio) | `x-api-key: <API_KEY de la cafeteria>` | Gestionar sus clientes en `/admin`. |

Ademas de estas claves para integraciones, existe **login con usuarios y roles**
(ver seccion "Login con cuentas de usuario").

### 1. Dar de alta una cafeteria (operador)

```bash
curl -X POST http://localhost:3000/platform/businesses \
  -H "x-platform-key: TU_PLATFORM_KEY" \
  -H "Content-Type: application/json" \
  -d '{
        "name": "Cafe Central",
        "logoText": "Cafe Central",
        "backgroundColor": "rgb(40, 30, 20)",
        "foregroundColor": "rgb(255, 250, 240)",
        "labelColor": "rgb(200, 160, 120)",
        "sellosParaRecompensa": 8,
        "ownerEmail": "dueno@cafecentral.com",
        "ownerPassword": "ContrasenaSegura123"
      }'
```

La respuesta incluye la **`apiKey` de esa cafeteria** (se muestra **una sola
vez**). Si envias `ownerEmail` y `ownerPassword`, tambien se crea la cuenta del
**propietario** (rol `business_owner`) para que entre por login.

Otros endpoints de plataforma:

| Metodo | Ruta | Que hace |
|--------|------|----------|
| GET   | `/platform/businesses` | Lista las cafeterias |
| GET   | `/platform/businesses/:id` | Ver una cafeteria |
| PATCH | `/platform/businesses/:id` | Cambiar marca/reglas |
| POST  | `/platform/businesses/:id/rotate-key` | Generar nueva API key |

## Login con cuentas de usuario (roles)

Cada persona puede entrar con email y contrasena en lugar de manejar claves. Hay
tres roles:

| Rol | Quien | Puede |
|-----|-------|-------|
| `platform_owner` | Tu (operador SaaS) | Todo `/platform` |
| `business_owner` | Dueno de una cafeteria | Todo `/admin` + gestionar personal |
| `business_staff` | Cajero/empleado | `/admin` (clientes y sellos), sin gestionar personal ni claves |

### Crear el operador de la plataforma (una vez)

```powershell
npm run create-owner -- tu@correo.com "TuContrasenaSegura"
```

### Iniciar sesion

```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"tu@correo.com","password":"TuContrasenaSegura"}'
```

Devuelve un `token`. Envialo en las siguientes peticiones con el header
`Authorization: Bearer <token>` (valido 12 horas). `GET /auth/me` devuelve tu
usuario actual.

### Gestionar el personal de una cafeteria (solo `business_owner`)

```bash
# Crear un cajero
curl -X POST http://localhost:3000/admin/staff \
  -H "Authorization: Bearer TOKEN_DEL_DUENO" \
  -H "Content-Type: application/json" \
  -d '{"email":"cajero@cafecentral.com","password":"ClaveCajero123","name":"Ana"}'
```

| Metodo | Ruta | Quien |
|--------|------|-------|
| GET   | `/admin/staff` | propietario |
| POST  | `/admin/staff` | propietario |
| PATCH | `/admin/staff/:userId` | propietario |
| POST  | `/admin/staff/:userId/password` | propietario |

## Uso de la API de administracion (cada cafeteria)

Las rutas `/admin` aceptan **dos formas de acceso**: el login de un usuario de la
cafeteria (`Authorization: Bearer <token>`) **o** la `x-api-key` de la cafeteria
(para integraciones). Los datos quedan aislados: una cafeteria solo ve y modifica
sus propios clientes.

### Crear un cliente

```bash
curl -X POST http://localhost:3000/admin/passes \
  -H "Authorization: Bearer TU_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"customerName":"Juan Perez","email":"juan@mail.com"}'
```

Respuesta: incluye el `serial_number` (ej. `CAFE-AB12CD34`).

### Descargar la tarjeta para entregarla al cliente

```
GET http://localhost:3000/admin/passes/CAFE-AB12CD34/download
```

Entregala por email, WhatsApp o un QR de descarga.

### Sumar un sello (en cada compra)

```bash
curl -X POST http://localhost:3000/admin/passes/CAFE-AB12CD34/sellos \
  -H "x-api-key: TU_ADMIN_KEY" \
  -H "Content-Type: application/json" \
  -d '{"cantidad":1}'
```

Al llegar a 10 sellos se otorga 1 recompensa automaticamente y el iPhone del
cliente se actualiza solo (si el servidor es publico con HTTPS).

### Canjear una recompensa

```bash
curl -X POST http://localhost:3000/admin/passes/CAFE-AB12CD34/canjear \
  -H "x-api-key: TU_ADMIN_KEY"
```

## Endpoints del protocolo Apple Wallet (automaticos)

Estos los llama el iPhone, no tu. No necesitas invocarlos manualmente:

| Metodo | Ruta |
|--------|------|
| POST   | `/v1/devices/:deviceId/registrations/:passTypeId/:serial` |
| DELETE | `/v1/devices/:deviceId/registrations/:passTypeId/:serial` |
| GET    | `/v1/devices/:deviceId/registrations/:passTypeId` |
| GET    | `/v1/passes/:passTypeId/:serial` |
| POST   | `/v1/log` |

## Pasar a produccion

1. Publica el servidor con **HTTPS valido** (Apple exige TLS para `webServiceURL`
   y APNs). Opciones: un VPS con Nginx + Let's Encrypt, o servicios como Render,
   Railway, Fly.io.
2. Cambia `WEB_SERVICE_URL` a tu dominio real.
3. Asegura `certs/`, `.env` y `data/` (no subirlos a git; ya estan en `.gitignore`).
4. Integra el `POST /sellos` con tu punto de venta (POS) o una app simple para el
   personal que escanee el QR del cliente.

## Seguridad (datos de clientes)

Como esta app maneja informacion personal de clientes (PII), incluye estas
protecciones:

| Medida | Que hace |
|--------|----------|
| **Cifrado en reposo (AES-256-GCM)** | Nombre y email se guardan cifrados en la base de datos. Requiere `ENCRYPTION_KEY`. |
| **Comparaciones timing-safe** | El token de Wallet y la API key se comparan sin filtrar informacion por tiempo. |
| **Rate limiting** | Limita peticiones por IP para frenar fuerza bruta y abuso. |
| **Cabeceras de seguridad (Helmet)** | HSTS, no-sniff y otras cabeceras endurecidas. |
| **HTTPS obligatorio en produccion** | Rechaza trafico sin TLS cuando `NODE_ENV=production`. |
| **Validacion de entrada** | Valida nombre, email, serial y cantidad de sellos. |
| **Sin filtrado de secretos** | La API nunca devuelve el `auth_token`; los errores no exponen detalles internos. |
| **Validacion de arranque** | En produccion el servidor no arranca si detecta claves por defecto o falta `ENCRYPTION_KEY`. |

### Generar la clave de cifrado (obligatoria)

```powershell
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Copia el resultado en `ENCRYPTION_KEY` dentro de tu `.env`.

> **Guarda esta clave en un gestor de secretos seguro.** Si la pierdes, no podras
> descifrar los datos de tus clientes. Si la cambias, los datos existentes quedan
> ilegibles. Respalda la clave por separado de la base de datos.

### Recomendaciones adicionales para produccion

- Sirve siempre detras de **HTTPS** con `NODE_ENV=production` y `TRUST_PROXY=true`
  si usas un proxy (Nginx, Render, etc.).
- Restringe el acceso a `/admin` por **IP** o **VPN** a nivel de firewall.
- Rota periodicamente `ADMIN_API_KEY` y `AUTHENTICATION_TOKEN`.
- Haz **backups cifrados** de `data/` y guarda `certs/`, `.env` y `ENCRYPTION_KEY`
  fuera del repositorio (ya estan en `.gitignore`).
- Cumple la normativa de proteccion de datos aplicable (consentimiento, derecho
  al borrado del cliente, etc.).

## Solucion de problemas

- **El pass no se agrega a Wallet**: revisa que `passTypeIdentifier` y
  `teamIdentifier` coincidan con el certificado, y que `wwdr.pem` sea el actual.
- **No se actualiza solo**: el `webServiceURL` debe ser HTTPS publico y el push
  APNs requiere certificados validos. Prueba primero con `npm run demo`.
- **Error de firma**: verifica que `signerKey.pem` corresponda a `signerCert.pem`
  y que el passphrase en `.env` sea correcto.

## Estructura del proyecto

```
.
├── certs/                      # Tus certificados .pem (no se versionan)
├── data/                       # passes generados (no se versiona)
├── passModels/
│   └── fidelidad.pass/         # Plantilla: pass.json + imagenes
├── src/
│   ├── config.js               # Carga de .env y rutas
│   ├── server.js               # Servidor Express
│   ├── db/database.js          # PostgreSQL (clientes y registros)
│   ├── queue/
│   │   ├── connection.js       # Conexion Redis compartida
│   │   ├── pushQueue.js        # Cola de push (productor)
│   │   └── pushWorker.js       # Worker que envia a APNs (consumidor)
│   ├── routes/
│   │   ├── wallet.js           # Protocolo Apple Wallet
│   │   └── admin.js            # API de administracion
│   ├── services/
│   │   ├── passGenerator.js    # Genera el .pkpass firmado
│   │   └── apns.js             # Notificaciones push
│   └── scripts/
│       ├── initDb.js
│       ├── worker.js           # Arranca el worker de push
│       └── createDemoPass.js
├── .env.example
└── package.json
```
