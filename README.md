# Workflow Docker — Infraestructura de Proyectos

Este documento describe el sistema completo de infraestructura Docker para gestionar múltiples proyectos en una sola máquina, tanto en desarrollo local como en producción.

---

## Visión general

La infraestructura se organiza en **tres capas independientes**:

```
┌─────────────────────────────────────────────────────────────┐
│  CAPA 1 — Traefik  (~/traefik/)                             │
│  Único punto de entrada. Gestiona dominios y SSL.            │
│  Se instala UNA VEZ por máquina.                            │
├─────────────────────────────────────────────────────────────┤
│  CAPA 2 — db-shared  (~/db-shared/)                         │
│  PostgreSQL + pgAdmin compartidos por todos los proyectos.  │
│  Se instala UNA VEZ por máquina.                            │
├─────────────────────────────────────────────────────────────┤
│  CAPA 3 — Proyectos  (~/mis-proyectos/proyecto-x/)          │
│  Cada proyecto tiene su propio compose.yaml.                │
│  Se integra UNA VEZ por proyecto, usando IA.                │
└─────────────────────────────────────────────────────────────┘
```

**Flujo de una request:**
```
Browser → Traefik (:80/:443) → nginx (proyecto) → app (contenedor)
                                                 ↘ db-shared → PostgreSQL
```

**Filosofía:** Docker es un bus de servicios desechable. El código, los datos y la configuración viven siempre en el host, nunca dentro de los contenedores.

---

## Requisitos de sistema

```bash
# Docker Engine >= 24 y Docker Compose >= v2.22
docker --version
docker compose version

# Ubuntu 24.04 (local y VPS)
```

---

## PARTE 1 — Setup inicial (una sola vez por máquina)

Esta parte se hace **una sola vez**: al preparar tu máquina de desarrollo local y, por separado, al preparar el VPS de producción. No se repite por proyecto.

---

### 1.1 — Traefik

Traefik es el proxy reverso global. Escucha en los puertos 80 y 443 y enruta el tráfico a cada proyecto según su dominio.

#### En desarrollo local

Abre Claude Code en cualquier directorio y ejecuta:

```
/docker-traefik-setup
```

Claude generará los archivos en `~/traefik/`. Luego:

```bash
cd ~/traefik
bash install.sh
```

Esto:
- Crea la red Docker `proxy` (compartida entre todos los proyectos)
- Crea `/srv/traefik/letsencrypt/` para certificados (vacío en dev)
- Arranca Traefik

**Verificación:**
```bash
curl -s http://localhost:8080/api/version   # debe devolver JSON con versión de Traefik
```

El dashboard de Traefik queda disponible en `http://localhost:8080`.

> En desarrollo, los dominios `*.localhost` funcionan nativamente en Chrome y Firefox sin modificar `/etc/hosts`.

#### En producción (VPS)

El proceso es idéntico, pero antes de arrancar debes ajustar `~/traefik/.env`:

```dotenv
TRAEFIK_DASHBOARD=false         # deshabilitar dashboard público
ACME_EMAIL=tu@email.com         # email para Let's Encrypt
LOG_LEVEL=ERROR
```

Y en `~/traefik/compose.yaml`, **descomentar las tres líneas de redirect** HTTP→HTTPS:

```yaml
# Descomentar en producción:
- --entrypoints.web.http.redirections.entrypoint.to=websecure
- --entrypoints.web.http.redirections.entrypoint.scheme=https
- --entrypoints.web.http.redirections.entrypoint.permanent=true
```

Luego:
```bash
cd ~/traefik
bash install.sh
```

---

### 1.2 — PostgreSQL + pgAdmin (db-shared)

Un único PostgreSQL comparte datos con todos los proyectos de la máquina. pgAdmin permite gestionar todas las bases de datos desde una sola interfaz.

**Prerequisito:** Traefik debe estar corriendo (`~/traefik/` levantado).

#### En desarrollo local

```
/docker-db-shared-setup
```

Claude generará los archivos en `~/db-shared/`. Luego:

```bash
cd ~/db-shared
cp .env.example .env
```

Edita `.env` y cambia las contraseñas:

```dotenv
POSTGRES_PASSWORD=una_contraseña_fuerte
PGADMIN_PASSWORD=otra_contraseña
PGADMIN_DOMAIN=pgadmin.localhost    # no cambiar en dev
TRAEFIK_ENTRYPOINT=web              # no cambiar en dev
```

```bash
bash install.sh
```

**Verificación:**
- `http://pgadmin.localhost` → pgAdmin disponible
- Conectar en pgAdmin: Host `postgres`, Port `5432`, User y Password del `.env`

#### En producción (VPS)

Mismo proceso, pero en `.env` cambiar:

```dotenv
POSTGRES_PASSWORD=contraseña_muy_fuerte_diferente_a_dev
PGADMIN_PASSWORD=otra_contraseña_fuerte
PGADMIN_DOMAIN=pgadmin.tudominio.com   # subdominio real
TRAEFIK_ENTRYPOINT=websecure           # activa SSL automático
```

> En producción se recomienda no exponer pgAdmin públicamente. Preferir acceso vía SSH tunnel:
> ```bash
> ssh -L 5050:localhost:5050 user@vps
> # Luego abrir: http://localhost:5050
> ```
> Si lo expones, añade auth básica (ver comentarios en el compose.yaml generado).

---

## PARTE 2 — Integrar Docker en un proyecto existente (por proyecto)

Esta parte se repite **una vez por cada proyecto**. La integración la hace Claude Code analizando el proyecto específico.

### 2.1 — Ejecutar el slash command en el proyecto

Abre Claude Code **dentro del directorio del proyecto** y ejecuta:

```
/docker-project-setup
```

Claude analizará el stack automáticamente (Python/FastAPI, Laravel/PHP, Next.js, React, etc.) y generará todos los archivos de Docker adaptados al proyecto:

```
proyecto/
├── compose.yaml
├── .env.example          ← revisar y copiar a .env
├── .dockerignore
├── Makefile
└── docker/
    ├── Dockerfile
    ├── nginx/
    │   ├── nginx.development.conf
    │   └── nginx.production.conf
    ├── scripts/
    │   ├── init-host-dirs.sh
    │   └── init-secrets.sh
    ├── secrets/           ← gitignoreado
    └── README.docker.md
```

> Si Claude detecta ambigüedad en el stack (ej. el proyecto tiene tanto `requirements.txt` como `package.json`), pedirá aclaración antes de generar.

### 2.2 — Revisar y configurar el .env

```bash
cp .env.example .env
```

Editar `.env` con los valores reales del proyecto:

```dotenv
PROJECT_NAME=mi-proyecto           # sin espacios, usado en dominios y rutas
ENVIRONMENT=development
APP_DOMAIN=mi-proyecto.localhost   # en dev: *.localhost

POSTGRES_DB=mi_proyecto_db         # nombre de la DB a crear
POSTGRES_USER=dbadmin              # debe coincidir con ~/db-shared/.env
POSTGRES_PASSWORD=la_del_db_shared # debe coincidir con ~/db-shared/.env
```

### 2.3 — Crear la base de datos del proyecto

Cada proyecto necesita su propia base de datos dentro del PostgreSQL compartido. Solo la primera vez:

```bash
docker compose -f ~/db-shared/compose.yaml exec postgres \
  psql -U dbadmin -c "CREATE DATABASE mi_proyecto_db;"
```

### 2.4 — Primera vez: inicializar y arrancar

```bash
make setup   # crea /srv/appdata/mi-proyecto/ en el host y genera secretos
make dev     # arranca con live reload (compose watch)
```

La aplicación queda disponible en `http://mi-proyecto.localhost`.

### 2.5 — Flujo de desarrollo diario

```bash
make dev        # arranca el entorno (código se sincroniza automáticamente)
make logs       # ver logs en tiempo real
make shell      # terminal dentro del contenedor de la app
make db-shell   # acceso psql a la base de datos
make stop       # apagar (los datos quedan intactos en /srv/appdata/)
```

> `git pull` funciona normalmente. Docker detecta los cambios automáticamente. No hace falta reiniciar contenedores al hacer pull en dev.

---

## PARTE 3 — Desplegar un proyecto en producción

Una vez que la infraestructura global está lista en el VPS (Parte 1), desplegar un proyecto es simple.

### 3.1 — Clonar el proyecto en el VPS

```bash
git clone git@github.com:ORG/mi-proyecto.git
cd mi-proyecto
cp .env.example .env
```

### 3.2 — Configurar .env para producción

```dotenv
PROJECT_NAME=mi-proyecto
ENVIRONMENT=production
APP_DOMAIN=mi-proyecto.com          # dominio real con DNS apuntando al VPS

POSTGRES_DB=mi_proyecto_db
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=la_del_db_shared_en_vps

APP_SECRET_KEY=clave_secreta_larga_y_distinta_a_dev
APP_DEBUG=false
```

### 3.3 — Crear la base de datos (primera vez)

```bash
docker compose -f ~/db-shared/compose.yaml exec postgres \
  psql -U dbadmin -c "CREATE DATABASE mi_proyecto_db;"
```

### 3.4 — Arrancar en producción

```bash
make setup
make prod    # build + arranque en segundo plano con nginx.production.conf
```

Traefik detecta el proyecto automáticamente y solicita un certificado SSL a Let's Encrypt. En menos de un minuto, `https://mi-proyecto.com` está activo con SSL.

### 3.5 — Actualizar en producción

```bash
make deploy   # git pull + rebuild + restart sin downtime
```

---

## Dónde viven los datos

```
/srv/traefik/letsencrypt/           ← certificados SSL (Traefik)

/srv/shared/postgres/               ← datos de PostgreSQL (todos los proyectos)
/srv/shared/pgadmin/                ← configuración y conexiones de pgAdmin

/srv/appdata/mi-proyecto/
  ├── uploads/                      ← archivos subidos por usuarios
  ├── static/                       ← archivos estáticos
  ├── logs/                         ← logs de la aplicación
  ├── backups/                      ← backups SQL
  ├── redis/                        ← si Redis está activo
  ├── elasticsearch/                ← si Elasticsearch está activo
  └── rabbitmq/                     ← si RabbitMQ está activo
```

**Regla de oro:** `docker compose down` nunca borra datos. Los datos están en el host. Los contenedores son desechables.

**Para migrar a otro servidor:**
1. Copiar `/srv/shared/` y `/srv/appdata/mi-proyecto/` al nuevo servidor
2. Instalar Docker y levantar Traefik + db-shared (Parte 1)
3. Clonar el repo, copiar `.env`, ejecutar `make prod`

---

## Resumen de comandos por etapa

### Setup de máquina (una vez)

| Paso | Comando | Dónde |
|------|---------|-------|
| Instalar Traefik | `/docker-traefik-setup` → `bash install.sh` | Claude Code → `~/traefik/` |
| Instalar db-shared | `/docker-db-shared-setup` → `bash install.sh` | Claude Code → `~/db-shared/` |

### Integración por proyecto (una vez por proyecto)

| Paso | Comando | Notas |
|------|---------|-------|
| Generar archivos Docker | `/docker-project-setup` | Claude Code en el directorio del proyecto |
| Configurar entorno | `cp .env.example .env` | Editar con valores reales |
| Crear base de datos | `docker compose -f ~/db-shared/... exec postgres psql ...` | Solo la primera vez |
| Inicializar host | `make setup` | Crea `/srv/appdata/proyecto/` |
| Arrancar dev | `make dev` | Live reload activo |

### Ciclo diario de desarrollo

| Comando | Acción |
|---------|--------|
| `make dev` | Arranca el entorno con live reload |
| `make logs` | Ver logs en tiempo real |
| `make shell` | Terminal en el contenedor |
| `make db-shell` | Consola psql |
| `make db-backup` | Dump SQL al host |
| `make stop` | Apagar (datos seguros) |
| `make clean` | Apagar + limpiar imágenes (datos seguros) |

### Producción

| Comando | Acción |
|---------|--------|
| `make prod` | Build + arranque en producción |
| `make deploy` | `git pull` + rebuild + restart |
| `make logs` | Ver logs |

---

## Diferencias clave entre desarrollo y producción

| Aspecto | Desarrollo | Producción |
|---------|------------|------------|
| `ENVIRONMENT` | `development` | `production` |
| `APP_DOMAIN` | `proyecto.localhost` | `proyecto.com` |
| SSL | No (HTTP directo) | Automático via Let's Encrypt |
| Redirect HTTP→HTTPS | No | Sí (descomentar en `~/traefik/compose.yaml`) |
| Nginx config | `nginx.development.conf` (sin headers, sin rate limiting) | `nginx.production.conf` (headers de seguridad, gzip, rate limiting) |
| Mailpit (email testing) | Disponible en `proyecto-mail.localhost` | No disponible (profile `dev` excluido) |
| Código | Sincronizado via `compose watch` | Copiado al build (`COPY . /app`) |
| Dashboard Traefik | `http://localhost:8080` | Desactivado |
| pgAdmin | `http://pgadmin.localhost` | SSH tunnel (recomendado) o subdominio con auth |

---

## Troubleshooting

```bash
# Ver estado de todos los contenedores de un proyecto
docker compose ps

# Ver si Traefik está enrutando correctamente
docker compose logs nginx

# Reconstruir desde cero (datos intactos)
make clean && make dev

# Ver las redes del contenedor nginx (debe tener proxy e internal)
docker inspect $(docker compose ps -q nginx) | grep -A 20 Networks

# Verificar que la red proxy existe
docker network inspect proxy

# Verificar que la red db-shared existe
docker network inspect db-shared

# Logs de Traefik (útil para problemas de routing/SSL)
cd ~/traefik && docker compose logs -f

# Backup manual de la base de datos
make db-backup
```
