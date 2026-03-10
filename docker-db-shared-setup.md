# /docker-db-shared-setup
# Slash command para Claude Code
# Se ejecuta UNA SOLA VEZ por máquina (dev local o VPS de producción)
# Nunca se repite por proyecto — es infraestructura global
# Parte del trio: /docker-traefik-setup + /docker-db-shared-setup + /docker-project-setup
# ══════════════════════════════════════════════════════════════════════════════

Crea el setup global de PostgreSQL + pgAdmin en `~/db-shared/` en esta máquina.
Este directorio NO es un repo de proyecto — es infraestructura permanente de la máquina.

---

## QUÉ ES ESTO Y POR QUÉ

Un único PostgreSQL y un único pgAdmin comparten todos los proyectos de la máquina.
Cada proyecto declara la red `db-shared` como externa y se conecta directamente.
pgAdmin se accede desde un solo dominio y puede ver todas las bases de datos.

```
proyecto-a  ──┐
proyecto-b  ──┼──→  red Docker "db-shared"  →  PostgreSQL + pgAdmin
proyecto-c  ──┘
```

**En desarrollo:** `pgadmin.localhost` — un único panel para todas las DBs
**En producción:** `pgadmin.tudominio.com` protegido con auth básica, o acceso
                   exclusivamente vía SSH tunnel (recomendado)

**Prerequisito:** Traefik debe estar corriendo (`/docker-traefik-setup` ejecutado primero).

---

## ARCHIVOS A GENERAR

```
~/db-shared/
├── compose.yaml
├── .env
├── .env.example
├── scripts/
│   └── init-host-dirs.sh
└── README.md
```

---

## ~/db-shared/compose.yaml

```yaml
networks:
  proxy:
    external: true      # red de Traefik
  db-shared:
    name: db-shared     # red que los proyectos declaran como external
    driver: bridge

services:

  # ── PostgreSQL ──────────────────────────────────────────────────
  postgres:
    image: postgres:16-alpine
    restart: unless-stopped
    env_file: .env
    environment:
      POSTGRES_USER:     ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB:       postgres        # DB de sistema, cada proyecto crea la suya
    volumes:
      - type: bind
        source: /srv/shared/postgres
        target: /var/lib/postgresql/data
    networks:
      - db-shared
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 512M

  # ── pgAdmin ─────────────────────────────────────────────────────
  pgadmin:
    image: dpage/pgadmin4:latest
    restart: unless-stopped
    env_file: .env
    environment:
      PGADMIN_DEFAULT_EMAIL:    ${PGADMIN_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_PASSWORD}
      PGADMIN_CONFIG_SERVER_MODE: "False"          # modo desktop, sin login múltiple
      PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: "False"
    volumes:
      - type: bind
        source: /srv/shared/pgadmin
        target: /var/lib/pgadmin
    labels:
      - traefik.enable=true
      - traefik.http.routers.pgadmin.rule=Host(`${PGADMIN_DOMAIN}`)
      - traefik.http.routers.pgadmin.entrypoints=${TRAEFIK_ENTRYPOINT:-web}   # dev: web | prod: websecure
      # Producción: descomentar para SSL + auth básica
      # - traefik.http.routers.pgadmin.entrypoints=websecure
      # - traefik.http.routers.pgadmin.tls.certresolver=le
      # - traefik.http.routers.pgadmin.middlewares=pgadmin-auth
      # - traefik.http.middlewares.pgadmin-auth.basicauth.users=${PGADMIN_BASIC_AUTH}
    networks:
      - proxy       # Traefik lo expone
      - db-shared   # puede ver el PostgreSQL
    depends_on:
      postgres:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: 256M
```

---

## ~/db-shared/.env.example

```dotenv
# ── PostgreSQL compartido ────────────────────────────────────────
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=CHANGE_ME

# ── pgAdmin ──────────────────────────────────────────────────────
PGADMIN_EMAIL=admin@local.dev
PGADMIN_PASSWORD=CHANGE_ME

# ── Dominio pgAdmin ──────────────────────────────────────────────
PGADMIN_DOMAIN=pgadmin.localhost

# ── Traefik entrypoint ───────────────────────────────────────────
TRAEFIK_ENTRYPOINT=web          # dev: web | prod: websecure
```

---

## ~/db-shared/.env

```dotenv
# ── PostgreSQL compartido ────────────────────────────────────────
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=cambia_esto_ya_fuerte

# ── pgAdmin ──────────────────────────────────────────────────────
PGADMIN_EMAIL=admin@local.dev
PGADMIN_PASSWORD=cambia_esto_ya

# ── Dominio pgAdmin ──────────────────────────────────────────────
# Desarrollo:
PGADMIN_DOMAIN=pgadmin.localhost
# Producción (cambiar):
# PGADMIN_DOMAIN=pgadmin.tudominio.com

# ── Traefik entrypoint ───────────────────────────────────────────
TRAEFIK_ENTRYPOINT=web          # dev: web | prod: websecure

# ── Auth básica para producción (generar con: htpasswd -nb user pass) ──
# PGADMIN_BASIC_AUTH=admin:$$apr1$$xxxxx
```

---

## ~/db-shared/scripts/init-host-dirs.sh

```bash
#!/bin/bash
set -e
sudo mkdir -p /srv/shared/{postgres,pgadmin}
sudo chown -R 1000:1000 /srv/shared/postgres
sudo chown -R 5050:5050 /srv/shared/pgadmin   # pgAdmin corre como uid 5050
echo "✓ Directorios creados en /srv/shared/"
```

---

## ~/db-shared/README.md

```markdown
# db-shared — PostgreSQL + pgAdmin global

Un único PostgreSQL y pgAdmin para todos los proyectos de esta máquina.

## Prerequisitos
- Docker Engine >= 24
- Traefik corriendo (~/traefik/)

## Instalación (una sola vez)

cd ~/db-shared
cp .env.example .env         # completar passwords
bash scripts/init-host-dirs.sh
docker compose up -d

pgAdmin disponible en: http://pgadmin.localhost

## Conectar un proyecto nuevo a este PostgreSQL

1. En el compose.yaml del proyecto, declarar la red como externa:

   networks:
     db-shared:
       external: true     # ← conecta al PostgreSQL compartido

2. En el servicio app del proyecto, agregar la red:

   services:
     app:
       networks:
         - internal
         - db-shared      # ← puede ver el postgres compartido

3. En el .env del proyecto usar como host:

   POSTGRES_HOST=postgres   # nombre del servicio en ~/db-shared/
   POSTGRES_PORT=5432

4. Crear la base de datos del proyecto (primera vez):

   docker compose -f ~/db-shared/compose.yaml exec postgres \
     psql -U dbadmin -c "CREATE DATABASE nombre_proyecto;"

## Acceso con pgAdmin

Dev:  http://pgadmin.localhost
Prod: SSH tunnel recomendado →  ssh -L 5050:localhost:5050 user@vps
      luego abrir: http://localhost:5050

Para agregar una conexión en pgAdmin:
  Host:     postgres
  Port:     5432
  Username: dbadmin (o el user del .env)
  Password: la del .env

## Dónde viven los datos

/srv/shared/postgres/    ← datos de PostgreSQL (todos los proyectos)
/srv/shared/pgadmin/     ← configuración y conexiones guardadas de pgAdmin

## Comandos

docker compose up -d          # arrancar
docker compose down           # apagar (datos intactos en /srv/shared/)
docker compose logs -f        # logs
docker compose restart        # reiniciar sin perder datos

## Backups

# Backup de una base de datos específica
docker compose exec -T postgres pg_dump -U dbadmin nombre_proyecto \
  > /srv/shared/backups/nombre_proyecto_$(date +%Y%m%d).sql

# Restaurar
docker compose exec -T postgres psql -U dbadmin nombre_proyecto \
  < /srv/shared/backups/nombre_proyecto_20240101.sql
```

---

## SCRIPT DE INSTALACIÓN COMPLETA

Generar `~/db-shared/install.sh`:

```bash
#!/bin/bash
set -e

echo "── Instalando db-shared (PostgreSQL + pgAdmin) ────────────"

# Verificar que Traefik esté corriendo
if ! docker network inspect proxy &>/dev/null; then
  echo "✗ La red 'proxy' no existe. Ejecuta /docker-traefik-setup primero."
  exit 1
fi

# Directorios en host
bash scripts/init-host-dirs.sh

# Arrancar
docker compose up -d

echo ""
echo "✓ PostgreSQL + pgAdmin corriendo"
echo "pgAdmin: http://${PGADMIN_DOMAIN:-pgadmin.localhost}"
echo ""
echo "Para crear una DB para un proyecto:"
echo "  docker compose exec postgres psql -U dbadmin -c 'CREATE DATABASE nombre;'"
```

---

## VALIDACIÓN

1. `docker compose config` — cero errores
2. `bash ~/db-shared/install.sh` — servicios healthy
3. `http://pgadmin.localhost` — pgAdmin responde vía Traefik
4. Agregar conexión en pgAdmin → host `postgres`, port `5432` → conecta
5. `docker network inspect db-shared` — confirmar que la red existe
6. Levantar un proyecto con `/docker-project-setup` y verificar conexión a la DB compartida
