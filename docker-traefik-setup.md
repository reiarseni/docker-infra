# /docker-traefik-setup
# Slash command para Claude Code
# Se ejecuta UNA SOLA VEZ por máquina (dev local o VPS de producción)
# Nunca se repite por proyecto — es infraestructura global
# Parte del trio de infraestructura: /docker-traefik-setup + /docker-db-shared-setup + /docker-project-setup
# ══════════════════════════════════════════════════════════════════

Crea el setup global de Traefik en `~/traefik/` en esta máquina.
Este directorio NO es un repo de proyecto — es infraestructura permanente de la máquina.

---

## QUÉ ES ESTO Y POR QUÉ

Traefik es el único proceso que escucha en los puertos 80 y 443 de la máquina.
Todos los proyectos Docker se conectan a él a través de una red compartida llamada `proxy`.
Cada proyecto declara su dominio en 3 líneas de labels — Traefik lo detecta solo.

```
Internet / Browser
       ↓
   Traefik :80/:443   ← único punto de entrada
       ↓
  red Docker "proxy"
       ↓
  ┌────────────┬────────────┬────────────┐
  │ proyecto-a │ proyecto-b │ proyecto-c │
  │ .localhost │ .localhost │ .localhost │
  └────────────┴────────────┴────────────┘
```

**En desarrollo:** dominios `*.localhost` — funcionan sin editar `/etc/hosts`
**En producción:** dominios reales con SSL automático via Let's Encrypt

---

## ARCHIVOS A GENERAR

```
~/traefik/
├── compose.yaml          ← Traefik + red proxy
├── .env                  ← configuración mínima
└── README.md             ← instrucciones de uso
```

---

## ~/traefik/compose.yaml

```yaml
services:
  traefik:
    image: traefik:v3-alpine
    restart: unless-stopped
    command:
      # ── Provider Docker ────────────────────────────────
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false   # opt-in por proyecto
      - --providers.docker.network=proxy

      # ── Entrypoints ────────────────────────────────────
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443

      # ── Redirect HTTP → HTTPS (descomentar solo en producción) ──
      # - --entrypoints.web.http.redirections.entrypoint.to=websecure
      # - --entrypoints.web.http.redirections.entrypoint.scheme=https
      # - --entrypoints.web.http.redirections.entrypoint.permanent=true

      # ── Let's Encrypt (solo producción) ────────────────
      - --certificatesresolvers.le.acme.httpchallenge=true
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.le.acme.email=${ACME_EMAIL:-dev@localhost}
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json

      # ── Dashboard (solo dev, desactivar en prod) ───────
      - --api.dashboard=${TRAEFIK_DASHBOARD:-false}
      - --api.insecure=${TRAEFIK_DASHBOARD:-false}

      # ── Logs ───────────────────────────────────────────
      - --log.level=${LOG_LEVEL:-ERROR}
      - --accesslog=false

    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"    # dashboard (solo si TRAEFIK_DASHBOARD=true)

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - type: bind
        source: /srv/traefik/letsencrypt
        target: /letsencrypt

    networks:
      - proxy

    deploy:
      resources:
        limits:
          memory: 64M

networks:
  proxy:
    external: true
```

---

## ~/traefik/.env

```dotenv
# ── Desarrollo local ─────────────────────────────────
TRAEFIK_DASHBOARD=true          # activa dashboard en http://localhost:8080
LOG_LEVEL=ERROR

# ── Producción (descomentar y ajustar) ───────────────
# TRAEFIK_DASHBOARD=false
# ACME_EMAIL=tu@email.com       # email para Let's Encrypt
# LOG_LEVEL=ERROR
# Además: descomentar las 3 líneas de redirect HTTP→HTTPS en compose.yaml
```

---

## ~/traefik/README.md

```markdown
# Traefik — Proxy global

Corre una vez. Enruta todos los proyectos Docker de esta máquina.

## Instalación (una sola vez)

# 1. Crear la red compartida
docker network create proxy

# 2. Crear directorio de certificados
sudo mkdir -p /srv/traefik/letsencrypt
sudo chmod 700 /srv/traefik/letsencrypt

# 3. Arrancar
cd ~/traefik
docker compose up -d

# En desarrollo: dashboard en http://localhost:8080
# En producción: editar .env antes de arrancar

## Comandos

docker compose up -d      # arrancar
docker compose down       # apagar
docker compose logs -f    # ver logs
docker compose pull       # actualizar imagen

## Agregar un proyecto nuevo

Solo necesita estas 3 líneas en el servicio nginx de su compose.yaml.
Ver `/docker-project-setup` para el setup completo de cada proyecto.

## Infraestructura global de la máquina

Esta máquina usa 3 capas de infraestructura compartida:

  ~/traefik/      → proxy reverso global     (/docker-traefik-setup)
  ~/db-shared/    → PostgreSQL + pgAdmin      (/docker-db-shared-setup)
  proyectos/      → cada proyecto             (/docker-project-setup)

    labels:
      - traefik.enable=true
      - traefik.http.routers.PROYECTO.rule=Host(`PROYECTO.localhost`)
      - traefik.http.routers.PROYECTO.entrypoints=web

En producción (con SSL automático):

    labels:
      - traefik.enable=true
      - traefik.http.routers.PROYECTO.rule=Host(`dominio.com`)
      - traefik.http.routers.PROYECTO.entrypoints=websecure
      - traefik.http.routers.PROYECTO.tls.certresolver=le

## Dominios en desarrollo

*.localhost funciona nativamente en Chrome y Firefox sin editar /etc/hosts.

proyecto-a → http://proyecto-a.localhost
proyecto-b → http://proyecto-b.localhost
api        → http://api.localhost
```

---

## SCRIPT DE INSTALACIÓN COMPLETA

Generar `~/traefik/install.sh`:

```bash
#!/bin/bash
set -e

echo "── Instalando Traefik global ──────────────────────────"

# Red proxy compartida
if ! docker network inspect proxy &>/dev/null; then
  docker network create proxy
  echo "✓ Red 'proxy' creada"
else
  echo "✓ Red 'proxy' ya existe"
fi

# Directorio de certificados Let's Encrypt
sudo mkdir -p /srv/traefik/letsencrypt
sudo chmod 700 /srv/traefik/letsencrypt
echo "✓ Directorio de certificados listo"

# Arrancar Traefik
docker compose up -d
echo "✓ Traefik corriendo"
echo ""
echo "Dashboard: http://localhost:8080"
echo "Listo para conectar proyectos con 3 líneas de labels."
```

---

## VALIDACIÓN

Después de generar los archivos, ejecutar en orden:

1. `bash ~/traefik/install.sh` — instala todo
2. `curl -s http://localhost:8080/api/version` — confirma que Traefik responde
3. Levantar cualquier proyecto con `/docker-setup` y verificar que su dominio
   `*.localhost` resuelve correctamente
