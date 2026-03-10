# /docker-backup-setup
# Slash command para Claude Code
# Añade backup automatizado (cron → S3/rsync) + restore documentado a un proyecto existente
# Requiere: /docker-project-setup ejecutado previamente en el proyecto
# Herramienta: rclone (un solo binario — soporta S3 · Wasabi · Backblaze B2 · MinIO · SFTP/rsync)
# ══════════════════════════════════════════════════════════════════════════════

Agrega la capa de backup al proyecto actual. Detecta automáticamente el nombre
del proyecto desde `.env`. Si hay ambigüedad en la configuración de destino,
pregunta UNA SOLA VEZ antes de generar los archivos.

---

## QUÉ GENERA ESTE COMANDO

```
docker/scripts/
  backup.sh           ← script principal (dump SQL + uploads → rclone remote)
  restore.sh          ← restauración guiada con confirmación y pre-backup de seguridad
```

Variables nuevas en `.env` y `.env.example`.
Targets nuevos en `Makefile`: `backup`, `backup-dry`, `restore`, `restore-remote`, `backup-list`.
Sección "Backups & Restore" en `docker/README.docker.md`.
Instrucciones de cron en el host.

---

## DISEÑO — principios que NO se negocian

1. **Sin contenedor de backup** — el script corre directamente en el host como cron job.
   Los contenedores son efímeros; el cron y rclone viven en el host.

2. **rclone como abstracción única** — un solo binario cubre S3, Backblaze B2, Wasabi,
   MinIO, SFTP/rsync. El script no cambia según el destino.

3. **Pre-backup antes de restaurar** — el script `restore.sh` siempre hace un dump
   del estado actual antes de sobreescribir. Sin red de seguridad, no hay restore.

4. **GFG retention** — 7 diarios · 4 semanales · 12 mensuales, aplicada local y remotamente.

5. **Dry-run siempre disponible** — `make backup-dry` simula sin escribir nada.

6. **Datos locales primero** — el backup se escribe en `/srv/appdata/${PROJECT_NAME}/backups/`
   y luego se sincroniza al remote. La copia local sirve de cache y permite restore inmediato.

---

## VARIABLES A AGREGAR EN `.env` Y `.env.example`

Agregar al final de `.env` del proyecto (con valores reales):

```dotenv
# ── Backup ─────────────────────────────────────────────────────
BACKUP_RCLONE_REMOTE=s3-myapp:mi-bucket/backups   # remote configurado en rclone.conf + ruta base
BACKUP_RETENTION_DAILY=7                           # días a conservar en daily/
BACKUP_RETENTION_WEEKLY=30                         # días a conservar en weekly/
BACKUP_RETENTION_MONTHLY=365                       # días a conservar en monthly/
```

Agregar al `.env.example` (sin valores reales):

```dotenv
# ── Backup ─────────────────────────────────────────────────────
# Formato: nombre-del-remote-rclone:bucket/ruta-base
# Ejemplos:
#   s3-aws:mi-bucket/backups          ← AWS S3
#   s3-wasabi:mi-bucket/backups       ← Wasabi
#   b2-myapp:mi-bucket/backups        ← Backblaze B2
#   sftp-vps:/srv/backups             ← SFTP / rsync
BACKUP_RCLONE_REMOTE=CHANGE_ME
BACKUP_RETENTION_DAILY=7
BACKUP_RETENTION_WEEKLY=30
BACKUP_RETENTION_MONTHLY=365
```

---

## PASO 1 — Instalar y configurar rclone en el host

### Instalación (una vez por máquina)

```bash
# Linux/macOS — instalador oficial
curl https://rclone.org/install.sh | sudo bash

# Verificar
rclone version
```

### Configurar destino S3-compatible

```bash
rclone config
```

Seguir el asistente interactivo. Ejemplos de configuración para cada proveedor:

#### AWS S3

```
n) New remote → nombre: s3-aws
Storage: Amazon S3 Compliant Storage Providers → s3
Provider: AWS
env_auth: false
access_key_id: AKIAXXXXXXXXXXXXXXXX
secret_access_key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
region: us-east-1
endpoint: (vacío — AWS default)
```

#### Wasabi (S3-compatible, más barato)

```
n) New remote → nombre: s3-wasabi
Storage: s3
Provider: Wasabi
access_key_id: XXXXXXXXXXXXXXXXXXXXXXXX
secret_access_key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
endpoint: s3.wasabisys.com
region: (vacío)
```

#### Backblaze B2

```
n) New remote → nombre: b2-myapp
Storage: Backblaze B2
account: xxxxxxxxxxxxxxxxxxxxxxxxx
key: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

#### SFTP / rsync a otro servidor

```
n) New remote → nombre: sftp-vps
Storage: SSH/SFTP
host: tu-vps.com
user: backupuser
port: 22
key_file: ~/.ssh/id_ed25519
```

### Verificar remote configurado

```bash
# Listar contenido del remote (debe responder sin errores)
rclone ls s3-aws:mi-bucket/
rclone ls sftp-vps:/srv/backups/

# Test de escritura
rclone mkdir s3-aws:mi-bucket/backups/test
rclone rmdir s3-aws:mi-bucket/backups/test
```

---

## PASO 2 — `docker/scripts/backup.sh`

```bash
#!/bin/bash
# docker/scripts/backup.sh — Backup PostgreSQL + uploads → rclone remote
#
# Uso:
#   bash docker/scripts/backup.sh              # backup completo
#   bash docker/scripts/backup.sh --dry-run    # simula sin escribir
#
# Cron (producción — diario a las 2:00 AM):
#   0 2 * * * cd /ruta/al/proyecto && bash docker/scripts/backup.sh >> /srv/appdata/PROJECT_NAME/logs/backup.log 2>&1

set -euo pipefail

# ── Configuración ──────────────────────────────────────────────────────────
DRY_RUN="${1:-}"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_DIR="$(cd "${SCRIPT_DIR}/../.." && pwd)"

# Cargar .env
if [[ ! -f "${PROJECT_DIR}/.env" ]]; then
  echo "✗ No se encontró .env en ${PROJECT_DIR}" >&2
  exit 1
fi
set -a; source "${PROJECT_DIR}/.env"; set +a

# Variables requeridas — falla inmediatamente si falta alguna
: "${PROJECT_NAME:?'Falta PROJECT_NAME en .env'}"
: "${POSTGRES_USER:?'Falta POSTGRES_USER en .env'}"
: "${POSTGRES_DB:?'Falta POSTGRES_DB en .env'}"
: "${BACKUP_RCLONE_REMOTE:?'Falta BACKUP_RCLONE_REMOTE en .env'}"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DAY_OF_WEEK=$(date +%u)    # 1=Lun … 7=Dom
DAY_OF_MONTH=$(date +%d)
LOCAL_DIR="/srv/appdata/${PROJECT_NAME}/backups"
DB_COMPOSE="${HOME}/db-shared/compose.yaml"

# Colores
GREEN='\033[0;32m'; YELLOW='\033[1;33m'; RED='\033[0;31m'; NC='\033[0m'

log()  { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
ok()   { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] ${GREEN}✓${NC} $*"; }
warn() { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] ${YELLOW}⚠${NC} $*"; }
fail() { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] ${RED}✗${NC} $*" >&2; exit 1; }

# ── Inicio ─────────────────────────────────────────────────────────────────
echo ""
log "════════════════════════════════════════════════════"
log "  Backup ${PROJECT_NAME} — ${TIMESTAMP}"
[[ -n "${DRY_RUN}" ]] && warn "  MODO DRY-RUN — no se escribirán archivos"
log "════════════════════════════════════════════════════"

# Verificar dependencias
command -v rclone &>/dev/null || fail "rclone no está instalado. Ver: https://rclone.org/install/"
[[ -f "${DB_COMPOSE}" ]] || fail "No se encontró ${DB_COMPOSE}. ¿Está corriendo db-shared?"

# Crear estructura local de directorios
[[ -z "${DRY_RUN}" ]] && mkdir -p "${LOCAL_DIR}"/{daily,weekly,monthly}

# ── 1. Dump PostgreSQL ─────────────────────────────────────────────────────
log "→ [1/4] Dump PostgreSQL: ${POSTGRES_DB}"
DB_FILE="${LOCAL_DIR}/daily/db_${TIMESTAMP}.sql.gz"

if [[ -z "${DRY_RUN}" ]]; then
  docker compose -f "${DB_COMPOSE}" exec -T postgres \
    pg_dump -U "${POSTGRES_USER}" --no-password "${POSTGRES_DB}" \
    | gzip -9 > "${DB_FILE}" \
    || fail "pg_dump falló. Verifica que db-shared esté corriendo: docker compose -f ${DB_COMPOSE} ps"
  ok "DB dump: ${DB_FILE} ($(du -sh "${DB_FILE}" | cut -f1))"
else
  log "  [dry-run] → ${DB_FILE}"
fi

# ── 2. Comprimir uploads ───────────────────────────────────────────────────
UPLOADS_SRC="/srv/appdata/${PROJECT_NAME}/uploads"
UPLOADS_FILE="${LOCAL_DIR}/daily/uploads_${TIMESTAMP}.tar.gz"
UPLOADS_BACKED_UP=false

if [[ -d "${UPLOADS_SRC}" ]]; then
  log "→ [2/4] Comprimiendo uploads/"
  if [[ -z "${DRY_RUN}" ]]; then
    tar -czf "${UPLOADS_FILE}" -C "/srv/appdata/${PROJECT_NAME}" uploads/ 2>/dev/null \
      || warn "uploads/ vacío o error al comprimir — se omite"
    [[ -f "${UPLOADS_FILE}" ]] && {
      ok "Uploads: ${UPLOADS_FILE} ($(du -sh "${UPLOADS_FILE}" | cut -f1))"
      UPLOADS_BACKED_UP=true
    }
  else
    log "  [dry-run] → ${UPLOADS_FILE}"
  fi
else
  warn "[2/4] uploads/ no existe — se omite"
fi

# ── 3. Sincronizar al remote ───────────────────────────────────────────────
log "→ [3/4] Sync a ${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/"
RCLONE_FLAGS="--progress --stats-one-line --transfers 4 --checkers 8"
[[ -n "${DRY_RUN}" ]] && RCLONE_FLAGS="${RCLONE_FLAGS} --dry-run"

if [[ -z "${DRY_RUN}" ]]; then
  rclone copy "${LOCAL_DIR}/daily/" \
    "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/" \
    --include "*${TIMESTAMP}*" \
    ${RCLONE_FLAGS}
  ok "Archivos subidos al remote"
else
  rclone copy "${LOCAL_DIR}/daily/" \
    "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/" \
    --include "*${TIMESTAMP}*" \
    ${RCLONE_FLAGS}
fi

# ── 4. Copias semanales (domingo) y mensuales (día 1) ─────────────────────
log "→ [4/4] Retención GFG (7 diarios · 4 semanales · 12 mensuales)"

if [[ -z "${DRY_RUN}" ]]; then
  # Semanal — cada domingo
  if [[ "${DAY_OF_WEEK}" == "7" ]]; then
    WEEK_TAG=$(date +%Y-W%V)
    cp "${DB_FILE}" "${LOCAL_DIR}/weekly/db_${WEEK_TAG}.sql.gz"
    ${UPLOADS_BACKED_UP} && cp "${UPLOADS_FILE}" "${LOCAL_DIR}/weekly/uploads_${WEEK_TAG}.tar.gz"
    rclone sync "${LOCAL_DIR}/weekly/" "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/weekly/" ${RCLONE_FLAGS}
    ok "Copia semanal guardada: ${WEEK_TAG}"
  fi

  # Mensual — día 1 de cada mes
  if [[ "${DAY_OF_MONTH}" == "01" ]]; then
    MONTH_TAG=$(date +%Y-%m)
    cp "${DB_FILE}" "${LOCAL_DIR}/monthly/db_${MONTH_TAG}.sql.gz"
    ${UPLOADS_BACKED_UP} && cp "${UPLOADS_FILE}" "${LOCAL_DIR}/monthly/uploads_${MONTH_TAG}.tar.gz"
    rclone sync "${LOCAL_DIR}/monthly/" "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/monthly/" ${RCLONE_FLAGS}
    ok "Copia mensual guardada: ${MONTH_TAG}"
  fi

  # Limpiar locales según retención
  find "${LOCAL_DIR}/daily/"   -name "*.gz" -mtime +"${BACKUP_RETENTION_DAILY:-7}"   -delete 2>/dev/null || true
  find "${LOCAL_DIR}/weekly/"  -name "*.gz" -mtime +"${BACKUP_RETENTION_WEEKLY:-30}" -delete 2>/dev/null || true
  find "${LOCAL_DIR}/monthly/" -name "*.gz" -mtime +"${BACKUP_RETENTION_MONTHLY:-365}" -delete 2>/dev/null || true

  # Limpiar remote según retención (solo daily — weekly y monthly se manejan por nombre)
  rclone delete "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/" \
    --min-age "${BACKUP_RETENTION_DAILY:-7}d" --progress 2>/dev/null || true
fi

echo ""
ok "════ Backup completado — ${TIMESTAMP} ════"
echo ""
```

---

## PASO 3 — `docker/scripts/restore.sh`

```bash
#!/bin/bash
# docker/scripts/restore.sh — Restaurar backup de PostgreSQL + uploads
#
# Uso:
#   bash docker/scripts/restore.sh                # restaurar desde backup LOCAL
#   bash docker/scripts/restore.sh --from-remote  # descargar desde remote y restaurar

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_DIR="$(cd "${SCRIPT_DIR}/../.." && pwd)"
set -a; source "${PROJECT_DIR}/.env"; set +a

: "${PROJECT_NAME:?'Falta PROJECT_NAME en .env'}"
: "${POSTGRES_USER:?'Falta POSTGRES_USER en .env'}"
: "${POSTGRES_DB:?'Falta POSTGRES_DB en .env'}"
: "${BACKUP_RCLONE_REMOTE:?'Falta BACKUP_RCLONE_REMOTE en .env'}"

FROM_REMOTE="${1:-}"
LOCAL_DIR="/srv/appdata/${PROJECT_NAME}/backups"
DB_COMPOSE="${HOME}/db-shared/compose.yaml"
TMP_DIR=""

GREEN='\033[0;32m'; YELLOW='\033[1;33m'; RED='\033[0;31m'; BOLD='\033[1m'; NC='\033[0m'

log()  { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] $*"; }
ok()   { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] ${GREEN}✓${NC} $*"; }
warn() { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] ${YELLOW}⚠${NC} $*"; }
fail() { echo -e "[$(date '+%Y-%m-%d %H:%M:%S')] ${RED}✗${NC} $*" >&2; exit 1; }

cleanup() {
  [[ -n "${TMP_DIR}" && -d "${TMP_DIR}" ]] && rm -rf "${TMP_DIR}"
}
trap cleanup EXIT

# ── Cabecera ───────────────────────────────────────────────────────────────
echo ""
echo -e "${BOLD}╔══════════════════════════════════════════════════════════╗${NC}"
echo -e "${BOLD}║         RESTAURACIÓN DE BACKUP                          ║${NC}"
echo -e "${BOLD}║         Proyecto: ${PROJECT_NAME}$(printf '%*s' $((40 - ${#PROJECT_NAME})) '')║${NC}"
echo -e "${BOLD}╚══════════════════════════════════════════════════════════╝${NC}"
echo ""

command -v rclone &>/dev/null || fail "rclone no instalado"
[[ -f "${DB_COMPOSE}" ]] || fail "No se encontró ${DB_COMPOSE}"

# ── Seleccionar archivo de DB ──────────────────────────────────────────────
DB_RESTORE_FILE=""

if [[ "${FROM_REMOTE}" == "--from-remote" ]]; then
  log "Backups remotos disponibles (daily):"
  echo ""
  rclone ls "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/" 2>/dev/null \
    | grep "db_" | sort -r | head -20 \
    || fail "No se pudo listar el remote. Verificar configuración de rclone."
  echo ""
  read -rp "Nombre del archivo a restaurar (ej: db_20260310_020000.sql.gz): " BACKUP_FILENAME
  [[ -z "${BACKUP_FILENAME}" ]] && { warn "Cancelado."; exit 0; }

  TMP_DIR=$(mktemp -d)
  log "→ Descargando ${BACKUP_FILENAME}..."
  rclone copy \
    "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/${BACKUP_FILENAME}" \
    "${TMP_DIR}/" --progress \
    || fail "No se pudo descargar ${BACKUP_FILENAME}"
  DB_RESTORE_FILE="${TMP_DIR}/${BACKUP_FILENAME}"

  # Buscar uploads asociado (mismo timestamp)
  UPLOADS_FILENAME="${BACKUP_FILENAME/db_/uploads_}"
  UPLOADS_FILENAME="${UPLOADS_FILENAME/.sql.gz/.tar.gz}"
  if rclone ls "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/${UPLOADS_FILENAME}" &>/dev/null; then
    log "→ Descargando uploads asociados: ${UPLOADS_FILENAME}..."
    rclone copy \
      "${BACKUP_RCLONE_REMOTE}/${PROJECT_NAME}/daily/${UPLOADS_FILENAME}" \
      "${TMP_DIR}/" --progress
  fi

else
  log "Backups locales disponibles (daily):"
  echo ""
  ls -lt "${LOCAL_DIR}/daily/"db_*.sql.gz 2>/dev/null | head -15 \
    || { warn "Sin backups locales en ${LOCAL_DIR}/daily/"; exit 1; }
  echo ""
  read -rp "Ruta completa al archivo DB (Enter para cancelar): " DB_RESTORE_FILE
  [[ -z "${DB_RESTORE_FILE}" ]] && { warn "Cancelado."; exit 0; }
  [[ -f "${DB_RESTORE_FILE}" ]] || fail "Archivo no encontrado: ${DB_RESTORE_FILE}"
fi

# Deducir archivo de uploads del mismo backup
UPLOADS_RESTORE_FILE="${DB_RESTORE_FILE/db_/uploads_}"
UPLOADS_RESTORE_FILE="${UPLOADS_RESTORE_FILE/.sql.gz/.tar.gz}"

# ── Confirmación de seguridad ──────────────────────────────────────────────
echo ""
echo -e "${RED}${BOLD}⚠  ADVERTENCIA — Esta operación es DESTRUCTIVA${NC}"
echo ""
echo "  Base de datos : ${POSTGRES_DB}"
echo "  Archivo DB    : $(basename "${DB_RESTORE_FILE}")"
[[ -f "${UPLOADS_RESTORE_FILE}" ]] && echo "  Archivo uploads: $(basename "${UPLOADS_RESTORE_FILE}")"
echo ""
echo "  Se creará un pre-backup automático antes de proceder."
echo ""
read -rp "  Escribe 'SI RESTAURAR' para confirmar: " CONFIRM
[[ "${CONFIRM}" != "SI RESTAURAR" ]] && { warn "Cancelado por el usuario."; exit 0; }

# ── Pre-backup de seguridad ────────────────────────────────────────────────
echo ""
log "→ [1/4] Creando pre-backup de seguridad..."
PRE_FILE="${LOCAL_DIR}/daily/pre-restore_$(date +%Y%m%d_%H%M%S).sql.gz"
mkdir -p "${LOCAL_DIR}/daily"
docker compose -f "${DB_COMPOSE}" exec -T postgres \
  pg_dump -U "${POSTGRES_USER}" "${POSTGRES_DB}" \
  | gzip -9 > "${PRE_FILE}" \
  || fail "No se pudo crear pre-backup. Abortar restauración."
ok "Pre-backup: ${PRE_FILE}"

# ── Parar la app ───────────────────────────────────────────────────────────
log "→ [2/4] Parando la app durante la restauración..."
cd "${PROJECT_DIR}"
docker compose stop app 2>/dev/null || warn "No se pudo parar la app (puede no estar corriendo)"

# ── Restaurar PostgreSQL ───────────────────────────────────────────────────
log "→ [3/4] Restaurando base de datos ${POSTGRES_DB}..."
docker compose -f "${DB_COMPOSE}" exec -T postgres \
  psql -U "${POSTGRES_USER}" postgres \
  -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='${POSTGRES_DB}' AND pid <> pg_backend_pid();" \
  > /dev/null 2>&1 || true

docker compose -f "${DB_COMPOSE}" exec -T postgres \
  psql -U "${POSTGRES_USER}" postgres \
  -c "DROP DATABASE IF EXISTS ${POSTGRES_DB};"

docker compose -f "${DB_COMPOSE}" exec -T postgres \
  psql -U "${POSTGRES_USER}" postgres \
  -c "CREATE DATABASE ${POSTGRES_DB} OWNER ${POSTGRES_USER};"

gunzip -c "${DB_RESTORE_FILE}" \
  | docker compose -f "${DB_COMPOSE}" exec -T postgres \
    psql -U "${POSTGRES_USER}" "${POSTGRES_DB}" \
  || fail "Error al restaurar la DB. Estado pre-restore disponible en: ${PRE_FILE}"
ok "Base de datos restaurada"

# ── Restaurar uploads (opcional) ───────────────────────────────────────────
if [[ -f "${UPLOADS_RESTORE_FILE}" ]]; then
  echo ""
  read -rp "¿Restaurar también uploads/ ? Sobreescribirá archivos existentes (s/N): " RESTORE_UPLOADS
  if [[ "${RESTORE_UPLOADS}" =~ ^[sS]$ ]]; then
    log "→ Restaurando uploads/..."
    tar -xzf "${UPLOADS_RESTORE_FILE}" -C "/srv/appdata/${PROJECT_NAME}/"
    ok "uploads/ restaurado"
  else
    warn "uploads/ NO restaurado (mantiene versión actual)"
  fi
fi

# ── Reiniciar app ──────────────────────────────────────────────────────────
log "→ [4/4] Reiniciando la app..."
docker compose start app 2>/dev/null || docker compose up -d app

echo ""
ok "════════════════════════════════════════════════"
ok "  Restauración completada"
ok "  Pre-backup de seguridad: ${PRE_FILE}"
ok "════════════════════════════════════════════════"
echo ""
echo "  Próximos pasos:"
echo "  1. make logs          — verificar que la app arrancó sin errores"
echo "  2. Verificar datos en http://${APP_DOMAIN:-la-aplicacion}"
echo "  3. Si algo falló: gunzip -c ${PRE_FILE} | docker compose -f ~/db-shared/compose.yaml exec -T postgres psql -U ${POSTGRES_USER} ${POSTGRES_DB}"
echo ""
```

---

## PASO 4 — Configurar cron en el host

### Editar crontab del usuario

```bash
crontab -e
```

### Agregar la línea del backup (ajustar la ruta al proyecto)

```cron
# ── Backup diario a las 2:00 AM ───────────────────────────────────────────
# PROJECT_NAME — backup PostgreSQL + uploads → rclone remote
0 2 * * * cd /ruta/absoluta/al/proyecto && bash docker/scripts/backup.sh >> /srv/appdata/PROJECT_NAME/logs/backup.log 2>&1

# Si hay múltiples proyectos, escalonar las horas para evitar carga simultánea:
# 0 2 * * *  proyecto-a → backup.sh
# 0 3 * * *  proyecto-b → backup.sh
# 0 4 * * *  proyecto-c → backup.sh
```

### Verificar que cron corre

```bash
# Ver el crontab activo
crontab -l

# Ejecutar manualmente para probar (sin esperar las 2 AM)
cd /ruta/al/proyecto && bash docker/scripts/backup.sh

# Ver logs de la última ejecución
tail -50 /srv/appdata/PROJECT_NAME/logs/backup.log

# Ver logs del cron del sistema (si algo no ejecuta)
grep CRON /var/log/syslog | tail -20
```

### Alertas en caso de fallo (opcional)

```bash
# Opción 1: email via sendmail (requiere mailutils)
MAILTO=admin@tudominio.com
0 2 * * * cd /ruta/proyecto && bash docker/scripts/backup.sh || echo "BACKUP FALLÓ en $(hostname)" | mail -s "⚠ Backup ${PROJECT_NAME}" admin@tudominio.com

# Opción 2: webhook (Slack, Discord, ntfy.sh)
0 2 * * * cd /ruta/proyecto && bash docker/scripts/backup.sh \
  || curl -s -X POST https://ntfy.sh/mi-canal \
       -d "⚠ Backup ${PROJECT_NAME} FALLÓ en $(hostname) a las $(date)"
```

---

## PASO 5 — Targets a agregar en Makefile

Agregar después del target `db-backup` existente:

```makefile
backup:        ## Backup completo: dump SQL + uploads → remote rclone
	bash docker/scripts/backup.sh

backup-dry:    ## Simular backup sin escribir ni subir archivos
	bash docker/scripts/backup.sh --dry-run

backup-list:   ## Listar backups disponibles (local y remote)
	@echo "── Backups locales (daily) ──────────────────────────────"
	@ls -lht /srv/appdata/$$PROJECT_NAME/backups/daily/ 2>/dev/null | head -15 || echo "  (vacío)"
	@echo ""
	@echo "── Backups en remote ────────────────────────────────────"
	@set -a; . .env; set +a; \
	rclone ls $$BACKUP_RCLONE_REMOTE/$$PROJECT_NAME/daily/ 2>/dev/null | sort -r | head -20 || echo "  (sin remote configurado)"

restore:       ## Restaurar backup desde almacenamiento LOCAL (interactivo)
	bash docker/scripts/restore.sh

restore-remote: ## Descargar backup del remote y restaurar (interactivo)
	bash docker/scripts/restore.sh --from-remote
```

---

## PASO 6 — Sección a agregar en `docker/README.docker.md`

Insertar después de la sección "Comandos":

```markdown
## Backups & Restore

El backup corre automáticamente vía cron a las 2:00 AM y sincroniza al remote configurado.

### Destinos de backup

| Proveedor     | Remote configurado              | Tipo          |
|---------------|---------------------------------|---------------|
| AWS S3        | `s3-aws:mi-bucket/backups`      | Objeto (S3)   |
| Wasabi        | `s3-wasabi:mi-bucket/backups`   | Objeto (S3)   |
| Backblaze B2  | `b2-myapp:mi-bucket/backups`    | Objeto (B2)   |
| SFTP/rsync    | `sftp-vps:/srv/backups`         | Filesystem    |

### Estructura de retención (GFG)

```
/srv/appdata/PROJECT_NAME/backups/
  daily/     ← últimos 7 días  (también en remote)
  weekly/    ← últimas 4 semanas — copia de cada domingo
  monthly/   ← últimos 12 meses  — copia del día 1
```

### Comandos de backup

```bash
make backup           # backup inmediato (útil antes de un deploy arriesgado)
make backup-dry       # simular sin escribir nada — para verificar configuración
make backup-list      # listar todos los backups disponibles (local + remote)
```

### Procedimiento de restore completo

**Caso 1 — Restore desde backup LOCAL** (más rápido, sin descarga):

```bash
make restore
# El script lista los backups disponibles y pide confirmación.
# Siempre crea un pre-backup antes de sobreescribir.
```

**Caso 2 — Restore descargando desde REMOTE** (cuando el host local está comprometido):

```bash
make restore-remote
# Lista backups en el remote → seleccionar → descarga → restaura.
```

**Caso 3 — Restore manual de emergencia** (sin scripts, máximo control):

```bash
# 1. Descargar el backup del remote
rclone copy s3-aws:mi-bucket/backups/PROJECT_NAME/daily/db_20260310_020000.sql.gz /tmp/

# 2. Crear un pre-backup de seguridad
docker compose -f ~/db-shared/compose.yaml exec -T postgres \
  pg_dump -U dbadmin PROJECT_DB | gzip -9 > /tmp/pre-restore-$(date +%s).sql.gz

# 3. Terminar conexiones activas a la DB
docker compose -f ~/db-shared/compose.yaml exec postgres \
  psql -U dbadmin -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='PROJECT_DB';"

# 4. Drop y recrear la DB
docker compose -f ~/db-shared/compose.yaml exec postgres \
  psql -U dbadmin -c "DROP DATABASE IF EXISTS PROJECT_DB;"
docker compose -f ~/db-shared/compose.yaml exec postgres \
  psql -U dbadmin -c "CREATE DATABASE PROJECT_DB OWNER dbadmin;"

# 5. Restaurar
gunzip -c /tmp/db_20260310_020000.sql.gz | \
  docker compose -f ~/db-shared/compose.yaml exec -T postgres \
  psql -U dbadmin PROJECT_DB

# 6. Restaurar uploads (si aplica)
tar -xzf /tmp/uploads_20260310_020000.tar.gz -C /srv/appdata/PROJECT_NAME/

# 7. Reiniciar la app
docker compose restart app
```

### Ante un desastre total (migración de servidor)

```bash
# En el servidor NUEVO (con Traefik y db-shared ya corriendo):

# 1. Clonar el repo
git clone git@github.com:ORG/proyecto.git && cd proyecto
cp .env.example .env   # rellenar valores

# 2. Instalar rclone y configurar el mismo remote
curl https://rclone.org/install.sh | sudo bash
rclone config   # configurar mismo remote que tenía el servidor anterior

# 3. Crear estructura de directorios
make setup

# 4. Levantar la app vacía
make prod

# 5. Restaurar el último backup del remote
make restore-remote

# 6. Verificar
make logs
curl https://tudominio.com/health
```
```

---

## VALIDACIÓN

1. `rclone listremotes` — el remote del proyecto aparece listado
2. `make backup-dry` — simula sin errores (verifica rclone + acceso a db-shared)
3. `make backup` — backup real completado; `make backup-list` muestra el archivo
4. `rclone ls ${BACKUP_RCLONE_REMOTE}/daily/` — el archivo está en el remote
5. `make restore` — lista backups locales, pide confirmación, restaura
6. `crontab -l` — la línea del cron aparece con la ruta correcta al proyecto
7. `/srv/appdata/${PROJECT_NAME}/logs/backup.log` — log generado tras el primer cron
