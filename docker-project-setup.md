# /docker-project-setup
# Slash command para Claude Code
# Compatible: Python · FastAPI · Flask · Laravel · PHP 8.3 · Next.js · React
# Target: Ubuntu 24.04 (dev local y VPS producción)
# Requiere: /docker-traefik-setup y /docker-db-shared-setup ejecutados previamente
# use context7
# ══════════════════════════════════════════════════════════════════════════════

Analiza el proyecto completo. Detecta el stack automáticamente.
Si hay ambigüedad, pregunta UNA VEZ antes de generar cualquier archivo.

---

## EL CONTRATO FUNDAMENTAL

Docker en este proyecto es un bus de servicios desechable y reemplazable.
Su único trabajo es levantar los servicios que la aplicación necesita.
Traefik es el único que escucha en los puertos 80/443 de la máquina —
este proyecto se conecta a él y declara su dominio en 3 líneas. Nada más.

**Lo que Docker NUNCA debe hacer aquí:**
- Poseer el código fuente
- Poseer los datos de la base de datos
- Impedir un `git pull`
- Romper el entorno si se elimina y recrea
- Comportarse diferente entre la PC del dev y el VPS de producción
- Exponer los puertos 80 o 443 directamente al host (eso es trabajo de Traefik)

**La promesa que este setup garantiza:**
- `docker compose down` → sin miedo. Los datos están en el host, no en Docker.
- Mismo stack en cualquier Ubuntu 24.04 con Traefik instalado → funciona igual, siempre.
- `git pull` en el host → el contenedor lo ve instantáneamente en dev.
- Un dev nuevo: clona el repo, 3 comandos, entorno completo en su dominio `.localhost`.
- Para migrar a otro VPS: copiar `/srv/appdata/` y clonar el repo. Listo.

**Prerequisitos:** Verificar que existen las redes Docker `proxy` y `db-shared` antes de continuar.
Si no están, indicar al usuario que ejecute primero:
  1. `/docker-traefik-setup`
  2. `/docker-db-shared-setup`

---

## ESTRUCTURA DE ARCHIVOS — regla absoluta

La raíz del proyecto debe quedar limpia.

```
proyecto/                        ← raíz del repo git
├── compose.yaml                 ← Docker Compose lo busca aquí
├── .env                         ← todas las tools lo esperan aquí
├── .env.example                 ← visible al clonar, sin valores reales
├── .dockerignore                ← Docker lo lee desde el build context
├── Makefile                     ← interfaz de uso diario
│
└── docker/                      ← TODO lo demás de Docker vive aquí
    ├── Dockerfile                ← multi-stage: base → development → production
    ├── nginx/
    │   ├── nginx.development.conf ← proxy reverso sin SSL
    │   └── nginx.production.conf  ← headers de seguridad, gzip, rate limiting
    ├── scripts/
    │   ├── init-host-dirs.sh     ← crea /srv/appdata/ en el host
    │   └── init-secrets.sh       ← genera archivos de secretos
    ├── secrets/                  ← gitignoreado
    │   └── .gitkeep
    └── README.docker.md          ← única documentación que necesita el dev
```

---

## REGLA 1 — Imágenes precompiladas. Nunca construir desde fuente para servicios.

El primer `docker compose up` debe ser rápido.

| Servicio        | Imagen                       | Tamaño aprox. |
|-----------------|------------------------------|---------------|
| PostgreSQL      | `postgres:16-alpine`           | ~85 MB        | **obligatorio** |
| pgAdmin         | `dpage/pgadmin4:latest`        | ~350 MB       | **obligatorio** |
| **Nginx**       | `nginx:alpine`                 | ~25 MB        | **obligatorio** |
| Redis           | `redis:7-alpine`               | ~16 MB        | opcional        |
| Elasticsearch   | `elasticsearch:8.13.4`         | ~600 MB (JVM) | opcional        |
| RabbitMQ        | `rabbitmq:3-management-alpine` | ~95 MB        | opcional        |
| Mailpit (dev)   | `axllent/mailpit:latest`       | ~35 MB        | opcional (dev)  |

Base para el contenedor de la aplicación:

| Stack           | Imagen base              |
|-----------------|--------------------------|
| Python 3.12     | `python:3.12-slim`       |
| PHP 8.3/Laravel | `php:8.3-fpm-alpine`     |
| Node 20 / Next  | `node:20-alpine`         |
| React SPA       | `nginx:alpine`           |

Elasticsearch: JVM, no puede ser ligero. Limitar con `ES_JAVA_OPTS=-Xms256m -Xmx256m`.

---

## REGLA 2 — Traefik es el punto de entrada. Siempre. En dev y en prod.

Ningún servicio de este proyecto expone los puertos 80 o 443 al host.
Traefik ya los tiene. Este proyecto se conecta a la red `proxy` de Traefik
y declara su dominio con exactamente 3 líneas de labels en el servicio nginx.

### Las 3 líneas — desarrollo

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.${PROJECT_NAME}.rule=Host(`${APP_DOMAIN}`)
  - traefik.http.routers.${PROJECT_NAME}.entrypoints=web
```

### Las 3 líneas — producción (agrega SSL automático con Let's Encrypt)

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.${PROJECT_NAME}.rule=Host(`${APP_DOMAIN}`)
  - traefik.http.routers.${PROJECT_NAME}.entrypoints=websecure
  - traefik.http.routers.${PROJECT_NAME}.tls.certresolver=le
```

> En producción son 4 líneas porque SSL requiere especificar el certresolver.

### El dominio por entorno

En `.env`:
```dotenv
# Desarrollo:
APP_DOMAIN=myapp.localhost     # funciona nativamente en Chrome/Firefox

# Producción:
APP_DOMAIN=myapp.com           # dominio real, SSL automático via Let's Encrypt
```

Un solo campo. Traefik hace el resto.

### Redes del proyecto

Cada proyecto tiene DOS redes:
- `proxy` — red externa compartida con Traefik (solo nginx la necesita)
- `internal` — red interna aislada (todos los servicios del proyecto)

```yaml
networks:
  proxy:
    external: true      # la red de Traefik, ya existe en la máquina
  internal:             # red privada de este proyecto
    driver: bridge
```

Esto garantiza que los servicios internos (db, redis) **no son accesibles
desde otros proyectos** aunque compartan la red proxy de Traefik.

---

## REGLA 3 — Nginx es transversal. Presente en dev y en prod.

Nginx no es opcional. Garantiza que el comportamiento de red es idéntico
en ambos entornos y elimina la fuente principal de bugs entre dev y producción.

La app nunca expone su puerto al host. Todo el tráfico entra por Nginx → Traefik.

### `docker/nginx/nginx.development.conf`

```nginx
server {
    listen 80;
    server_name localhost;

    client_max_body_size 100M;

    location / {
        proxy_pass         http://app:${APP_PORT:-8000};
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 300s;
    }

    location /static/ {
        alias  /app/static/;
        expires off;
    }

    location /uploads/ {
        alias  /app/uploads/;
        expires off;
    }
}
```

### `docker/nginx/nginx.production.conf`

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;

server {
    listen 80;
    server_name _;
    client_max_body_size 100M;

    # Headers de seguridad
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options            SAMEORIGIN always;
    add_header X-Content-Type-Options     nosniff always;
    add_header Referrer-Policy            strict-origin-when-cross-origin always;

    # Gzip
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_min_length 1000;

    location / {
        proxy_pass         http://app:${APP_PORT:-8000};
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto https;
        proxy_read_timeout 300s;
        limit_req          zone=api burst=20 nodelay;
    }

    location /static/ {
        alias  /app/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /uploads/ {
        alias  /app/uploads/;
        expires 7d;
    }
}
```

---

## REGLA 4 — Los datos viven en el host. Para siempre.

```
/srv/appdata/{PROJECT_NAME}/
  ├── postgres/
  ├── redis/
  ├── elasticsearch/
  ├── uploads/
  ├── static/
  ├── logs/
  └── backups/
```

Sintaxis obligatoria (tipo explícito):
```yaml
volumes:
  - type: bind
    source: /srv/appdata/${PROJECT_NAME}/postgres
    target: /var/lib/postgresql/data
```

Generar `docker/scripts/init-host-dirs.sh`:
```bash
#!/bin/bash
set -e
source .env
sudo mkdir -p /srv/appdata/${PROJECT_NAME}/{uploads,static,logs,backups}
sudo chown -R 1000:1000 /srv/appdata/${PROJECT_NAME}
echo "✓ Directorios creados en /srv/appdata/${PROJECT_NAME}"
```

---

## REGLA 5 — El código vive en el repo. `compose watch` lo sincroniza.

```yaml
develop:
  watch:
    - action: sync
      path: ./src
      target: /app/src
      initial_sync: true
      ignore: [__pycache__, "*.pyc", node_modules, vendor, .git, "*.log"]
    - action: sync+restart
      path: ./config
      target: /app/config
    - action: rebuild
      path: requirements.txt     # ajustar: composer.json / package.json
```

En producción: `COPY --chown=appuser:appuser . /app` en el Dockerfile.

---

## REGLA 6 — Un solo `.env`. Sin repeticiones.

```dotenv
# ── Proyecto ─────────────────────────────────────────────────────
PROJECT_NAME=myapp
ENVIRONMENT=development          # development | production

# ── Dominio (Traefik lo lee de aquí) ─────────────────────────────
APP_DOMAIN=myapp.localhost       # dev: *.localhost | prod: dominio real

# ── PostgreSQL (compartido en ~/db-shared/) ──────────────────────
POSTGRES_HOST=postgres           # nombre del servicio en db-shared
POSTGRES_PORT=5432
POSTGRES_DB=myapp_db             # DB exclusiva de este proyecto
POSTGRES_USER=dbadmin            # user del db-shared
POSTGRES_PASSWORD=cambia_esto_ya # password del db-shared

# ── Redis ────────────────────────────────────────────────────────
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=cambia_esto_ya

# ── Elasticsearch ────────────────────────────────────────────────
ELASTIC_HOST=elasticsearch
ELASTIC_PORT=9200
ELASTIC_PASSWORD=cambia_esto_ya
ES_JAVA_OPTS=-Xms256m -Xmx256m

# ── RabbitMQ (opcional) ──────────────────────────────────────────
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=myapp_user
RABBITMQ_PASSWORD=cambia_esto_ya
RABBITMQ_VHOST=myapp

# ── Aplicación ───────────────────────────────────────────────────
APP_PORT=8000
APP_DEBUG=false
APP_SECRET_KEY=cambia_esto_ya_minimo_32_caracteres

# ── Rutas host ───────────────────────────────────────────────────
DATA_BASE_PATH=/srv/appdata/${PROJECT_NAME}
```

Generar `.env.example` (igual que `.env` pero con valores de ejemplo, sin secretos reales):

```dotenv
# ── Proyecto ─────────────────────────────────────────────────────
PROJECT_NAME=myapp
ENVIRONMENT=development          # development | production

# ── Dominio (Traefik lo lee de aquí) ─────────────────────────────
APP_DOMAIN=myapp.localhost       # dev: *.localhost | prod: dominio real

# ── PostgreSQL (compartido en ~/db-shared/) ──────────────────────
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=myapp_db
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=CHANGE_ME

# ── Redis ────────────────────────────────────────────────────────
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=CHANGE_ME

# ── Aplicación ───────────────────────────────────────────────────
APP_PORT=8000
APP_DEBUG=false
APP_SECRET_KEY=CHANGE_ME_min_32_chars
```

Secretos en producción → `docker/secrets/*.txt` (gitignoreados).

Generar `docker/scripts/init-secrets.sh`:
```bash
#!/bin/bash
set -e
mkdir -p docker/secrets
openssl rand -base64 32 > docker/secrets/db_password.txt
openssl rand -base64 48 > docker/secrets/app_secret_key.txt
openssl rand -base64 24 > docker/secrets/redis_password.txt
chmod 600 docker/secrets/*.txt
echo "✓ Secretos generados en docker/secrets/"
```

---

## compose.yaml — referencia completa

```yaml
# Sin campo "version" — Compose v2 no lo necesita

secrets:
  app_secret_key:
    file: ./docker/secrets/app_secret_key.txt

networks:
  proxy:
    external: true       # red de Traefik
  db-shared:
    external: true       # red de PostgreSQL + pgAdmin compartidos
  internal:              # red privada de este proyecto (redis, elastic, etc.)
    driver: bridge

services:

  # ── PostgreSQL — usar el compartido en ~/db-shared/ ────────────
  # No se define aquí. El servicio `postgres` vive en la red db-shared.
  # La app se conecta a él a través de esa red.
  # POSTGRES_HOST=postgres  (nombre del servicio en db-shared)
  # POSTGRES_PORT=5432
  #
  # Para crear la DB de este proyecto (solo la primera vez):
  #   docker compose -f ~/db-shared/compose.yaml exec postgres \
  #     psql -U dbadmin -c "CREATE DATABASE ${PROJECT_NAME};"

  # ── Redis (opcional — descomentar si el proyecto lo usa) ────────
  # redis:
  #   image: redis:7-alpine
  #   restart: unless-stopped
  #   command: redis-server --requirepass ${REDIS_PASSWORD} --save 60 1
  #   volumes:
  #     - type: bind
  #       source: /srv/appdata/${PROJECT_NAME}/redis
  #       target: /data
  #   networks:
  #     - internal
  #   healthcheck:
  #     test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
  #     interval: 10s
  #     timeout: 5s
  #     retries: 5
  #   deploy:
  #     resources:
  #       limits:
  #         memory: 128M

  # ── Elasticsearch (opcional — descomentar si el proyecto lo usa)
  # elasticsearch:
  #   image: elasticsearch:8.13.4
  #   restart: unless-stopped
  #   environment:
  #     discovery.type:          single-node
  #     ELASTIC_PASSWORD:        ${ELASTIC_PASSWORD}
  #     ES_JAVA_OPTS:            ${ES_JAVA_OPTS:--Xms256m -Xmx256m}
  #     xpack.security.enabled:  "true"
  #   volumes:
  #     - type: bind
  #       source: /srv/appdata/${PROJECT_NAME}/elasticsearch
  #       target: /usr/share/elasticsearch/data
  #   networks:
  #     - internal
  #   healthcheck:
  #     test: ["CMD-SHELL", "curl -sf -u elastic:${ELASTIC_PASSWORD} http://localhost:9200/_cluster/health || exit 1"]
  #     interval: 20s
  #     timeout: 10s
  #     retries: 10
  #   deploy:
  #     resources:
  #       limits:
  #         memory: 512M

  # ── RabbitMQ (opcional — descomentar si el proyecto lo usa) ────
  # rabbitmq:
  #   image: rabbitmq:3-management-alpine
  #   restart: unless-stopped
  #   env_file: .env
  #   environment:
  #     RABBITMQ_DEFAULT_USER:  ${RABBITMQ_USER}
  #     RABBITMQ_DEFAULT_PASS:  ${RABBITMQ_PASSWORD}
  #     RABBITMQ_DEFAULT_VHOST: ${RABBITMQ_VHOST}
  #   volumes:
  #     - type: bind
  #       source: /srv/appdata/${PROJECT_NAME}/rabbitmq
  #       target: /var/lib/rabbitmq
  #   networks:
  #     - internal
  #   healthcheck:
  #     test: ["CMD", "rabbitmq-diagnostics", "ping"]
  #     interval: 15s
  #     timeout: 10s
  #     retries: 5
  #   labels:
  #     - traefik.enable=true
  #     - traefik.http.routers.${PROJECT_NAME}-rabbit.rule=Host(`${PROJECT_NAME}-rabbit.localhost`)
  #     - traefik.http.routers.${PROJECT_NAME}-rabbit.entrypoints=web
  #     - traefik.http.services.${PROJECT_NAME}-rabbit.loadbalancer.server.port=15672
  #   deploy:
  #     resources:
  #       limits:
  #         memory: 256M

  # ── Aplicación ──────────────────────────────────────────────────
  app:
    build:
      context: .
      dockerfile: ./docker/Dockerfile
      target: ${ENVIRONMENT:-production}
    restart: unless-stopped
    env_file: .env
    secrets:
      - app_secret_key
    expose:
      - "${APP_PORT:-8000}"    # solo red interna, Nginx lo proxea
    volumes:
      - type: bind
        source: /srv/appdata/${PROJECT_NAME}/uploads
        target: /app/uploads
      - type: bind
        source: /srv/appdata/${PROJECT_NAME}/logs
        target: /app/logs
    networks:
      - internal
      - db-shared    # acceso al PostgreSQL compartido
    # depends_on:                      # descomentar si Redis u otros servicios internos están activos
    #   redis:
    #     condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:${APP_PORT:-8000}/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    develop:
      watch:
        - action: sync
          path: ./src
          target: /app/src
          initial_sync: true
          ignore: [__pycache__, "*.pyc", node_modules, vendor, .git]
        - action: rebuild
          path: requirements.txt

  # ── Nginx + Traefik labels ──────────────────────────────────────
  nginx:
    image: nginx:alpine
    restart: unless-stopped
    # ↓ Las 3 líneas que conectan este proyecto con Traefik
    labels:
      - traefik.enable=true
      - traefik.http.routers.${PROJECT_NAME}.rule=Host(`${APP_DOMAIN}`)
      - traefik.http.routers.${PROJECT_NAME}.entrypoints=${TRAEFIK_ENTRYPOINT:-web}
      # SSL automático en producción (descomentado cuando ENVIRONMENT=production):
      # - traefik.http.routers.${PROJECT_NAME}.tls.certresolver=le
    volumes:
      - type: bind
        source: ./docker/nginx/nginx.${ENVIRONMENT:-development}.conf
        target: /etc/nginx/conf.d/default.conf
        read_only: true
      - type: bind
        source: /srv/appdata/${PROJECT_NAME}/uploads
        target: /app/uploads
        read_only: true
      - type: bind
        source: /srv/appdata/${PROJECT_NAME}/static
        target: /app/static
        read_only: true
    networks:
      - proxy        # conectado a Traefik
      - internal     # conectado a la app
    depends_on:
      app:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 64M

  # ── pgAdmin — usar el global en ~/db-shared/ ───────────────────
  # No se define aquí. Acceder en: http://pgadmin.localhost
  # Ver /docker-db-shared-setup para detalles.

  # ── Mailpit — solo dev (profile: dev) ──────────────────────────
  mailpit:
    image: axllent/mailpit:latest
    profiles: [dev]
    labels:
      - traefik.enable=true
      - traefik.http.routers.${PROJECT_NAME}-mail.rule=Host(`${PROJECT_NAME}-mail.localhost`)
      - traefik.http.routers.${PROJECT_NAME}-mail.entrypoints=web
      - traefik.http.services.${PROJECT_NAME}-mail.loadbalancer.server.port=8025
    networks:
      - proxy
      - internal
```

---

## docker/Dockerfile — multi-stage

```dockerfile
# ── base ────────────────────────────────────────────────
FROM python:3.12-slim AS base
RUN groupadd -g 1000 appuser \
 && useradd -u 1000 -g appuser -s /bin/bash appuser
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ── development ─────────────────────────────────────────
FROM base AS development
RUN pip install --no-cache-dir watchdog
USER appuser
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--reload"]

# ── production ──────────────────────────────────────────
FROM base AS production
COPY --chown=appuser:appuser . /app
USER appuser
CMD ["python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--workers", "2"]
```

---

## Makefile

```makefile
.PHONY: help setup dev prod build deploy logs shell db-shell db-backup stop clean

help:         ## Muestra este menú
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
	  awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-14s\033[0m %s\n", $$1, $$2}'

setup:        ## Primera vez: crea directorios en host y archivos de secretos
	bash docker/scripts/init-host-dirs.sh
	bash docker/scripts/init-secrets.sh

dev:          ## Desarrollo con sincronización de archivos en vivo
	ENVIRONMENT=development COMPOSE_PROFILES=dev docker compose watch

prod:         ## Producción en segundo plano
	ENVIRONMENT=production docker compose up -d --build

build:        ## Construye la imagen de la app
	ENVIRONMENT=production docker compose build app

deploy:       ## VPS: git pull + rebuild + restart
	git pull
	make build
	ENVIRONMENT=production docker compose up -d

logs:         ## Logs en tiempo real
	docker compose logs -f

shell:        ## Terminal dentro de la app
	docker compose exec app bash

db-shell:     ## psql al PostgreSQL compartido
	@set -a; . .env; set +a; \
	docker compose -f ~/db-shared/compose.yaml exec postgres \
	  psql -U $$POSTGRES_USER $$POSTGRES_DB

db-backup:    ## Dump SQL al host
	@set -a; . .env; set +a; \
	docker compose -f ~/db-shared/compose.yaml exec -T postgres \
	  pg_dump -U $$POSTGRES_USER $$POSTGRES_DB \
	  > /srv/appdata/$$PROJECT_NAME/backups/backup_$$(date +%Y%m%d_%H%M%S).sql \
	  && echo "✓ Backup en /srv/appdata/$$PROJECT_NAME/backups/"

stop:         ## Apaga contenedores (datos seguros en host)
	docker compose down

clean:        ## Apaga + elimina imágenes locales (datos seguros)
	docker compose down --rmi local --remove-orphans
```

---

## REGLA 7 — Generar docker/README.docker.md (obligatorio)

Escríbelo en el idioma del proyecto. Directo, sin párrafos largos.
Rellénalo con valores reales del proyecto — sin placeholders.

```markdown
# Docker — Guía rápida

## Prerequisito
Traefik y db-shared deben estar corriendo en la máquina.
Si no: ejecutar `/docker-traefik-setup` y `/docker-db-shared-setup` primero.

## Requisitos
- Ubuntu 24.04
- Docker Engine >= 24
- Docker Compose >= v2.22
- Traefik corriendo (~/traefik/)
- db-shared corriendo (~/db-shared/)

## Primera vez

git clone git@github.com:ORG/PROYECTO.git
cd PROYECTO
cp .env.example .env     # completar valores marcados con "cambia_esto"

# Crear la DB de este proyecto en el PostgreSQL compartido:
docker compose -f ~/db-shared/compose.yaml exec postgres \
  psql -U dbadmin -c "CREATE DATABASE nombre_proyecto;"

make setup

## Desarrollo

make dev

| Servicio      | URL                            |
|---------------|--------------------------------|
| Aplicación    | http://PROYECTO.localhost      |
| pgAdmin       | http://pgadmin.localhost       |  ← global, todos los proyectos
| Mailpit       | http://PROYECTO-mail.localhost |
| PostgreSQL    | vía db-shared (interno)        |
| Redis         | interno (si activo)            |
| Elasticsearch | interno (si activo)            |

Los cambios en el código se reflejan automáticamente.
git pull funciona normalmente — Docker lo detecta solo.

## Producción

# En .env cambiar:
ENVIRONMENT=production
APP_DOMAIN=dominio-real.com

make prod

Nginx activo. Mailpit NO disponible. pgAdmin en http://pgadmin.tudominio.com
SSL automático via Let's Encrypt.

## Actualizar en VPS

make deploy

## Comandos

make help       → ver todos los comandos
make logs       → logs en tiempo real
make shell      → terminal en la app
make db-shell   → psql al PostgreSQL compartido
make db-backup  → backup SQL
make stop       → apagar (datos NO se borran)
make clean      → apagar + limpiar imágenes (datos NO se borran)

## Dónde viven los datos

/srv/shared/postgres/               ← PostgreSQL (todos los proyectos)
/srv/shared/pgadmin/                ← configuración pgAdmin
/srv/appdata/PROYECTO/uploads/      ← archivos de usuarios
/srv/appdata/PROYECTO/backups/      ← backups SQL

Migrar a otro servidor:
  1. Copiar /srv/shared/ y /srv/appdata/PROYECTO/ al nuevo servidor
  2. Levantar ~/traefik/ y ~/db-shared/ en el nuevo servidor
  3. Clonar el repo
  4. Copiar .env
  5. make prod

## Problemas comunes

docker compose ps                       # estado de contenedores
docker compose logs nginx               # ver si Traefik conecta
docker compose build --no-cache app     # reconstruir desde cero
make clean && make prod                 # reset completo (datos intactos)
```

---

## Ajustes automáticos por stack (aplicar sin preguntar)

### Laravel / PHP 8.3
- Base image: `php:8.3-fpm-alpine`
- Nginx usa `fastcgi_pass app:9000` (no `proxy_pass`)
- Watch paths: `./app`, `./routes`, `./config`, `./resources`
- Rebuild trigger: `composer.json`
- Agregar servicio `queue` en compose.yaml

### Python / FastAPI / Flask
- Base image: `python:3.12-slim`
- Nginx usa `proxy_pass http://app:8000`
- Rebuild trigger: `requirements.txt`
- Dev: uvicorn `--reload` | Prod: uvicorn `--workers 2`

### Next.js
- Base image: `node:20-alpine`
- Nginx usa `proxy_pass http://app:3000`
- Rebuild trigger: `package.json`
- Agregar volumen anónimo `/app/node_modules`

### React SPA pura
- No contenedor de runtime en dev (Vite corre en el host)
- Prod: `npm run build` → servido por `nginx:alpine`
- Las 3 líneas de Traefik van en el servicio nginx que sirve el `dist/`

---

## Checklist de validación

1. `docker compose config` — cero errores
2. `make setup` — dirs en `/srv/appdata/`, secretos generados
3. `make dev` — todos los servicios en estado `healthy`
4. `http://${APP_DOMAIN}` responde vía Traefik → Nginx → app
5. Editar un archivo fuente → cambio visible sin restart
6. `make stop` → datos en `/srv/appdata/` intactos
7. `make prod` — nginx carga `nginx.production.conf`, SSL activo si dominio real
8. `docker inspect <nginx>` → conectado a redes `proxy` e `internal`
9. `docker inspect <db>` → conectado SOLO a red `internal` (no expuesto)
10. `docker/README.docker.md` tiene URLs reales con dominios `.localhost` del proyecto
