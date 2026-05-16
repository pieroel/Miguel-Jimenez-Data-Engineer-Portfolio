versión más profesional y completa del sistema de backups en Bash, incluyendo:

Backups automáticos
Compresión
Logs avanzados
Eliminación automática
Alertas por correo
Uso de archivo .env
Backups incrementales con rsync
Backups MySQL/PostgreSQL
Backups remotos por SSH
Integración con systemd
Subida opcional a AWS S3
Estructura enterprise


Estructura Profesional del Proyecto
backup-system/
│
├── backup.sh
├── .env
├── functions.sh
├── logs/
├── backups/
├── mysql/
├── postgres/
└── systemd/
    └── backup.service
1. Archivo .env

Guarda la configuración separada del script.

# =====================================
# DIRECTORIOS
# =====================================

SOURCE_DIR="/home/usuario/documentos"
BACKUP_DIR="/home/usuario/backups"
LOG_DIR="/home/usuario/backups/logs"

# =====================================
# RETENCIÓN
# =====================================

RETENTION_DAYS=7

# =====================================
# EMAIL
# =====================================

EMAIL="admin@correo.com"

# =====================================
# MYSQL
# =====================================

MYSQL_USER="root"
MYSQL_PASSWORD="password"
MYSQL_DATABASE="empresa"

# =====================================
# POSTGRESQL
# =====================================

POSTGRES_USER="postgres"
POSTGRES_DATABASE="empresa"

# =====================================
# REMOTO SSH
# =====================================

REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.100"
REMOTE_DIR="/backups"

# =====================================
# AWS S3
# =====================================

S3_BUCKET="s3://mis-backups"
2. Archivo functions.sh

Funciones reutilizables.

#!/bin/bash

log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

send_error_email() {
    SUBJECT="ERROR Backup $(hostname)"
    MESSAGE="Error detectado.\n\nRevisar log:\n$LOG_FILE"

    echo -e "$MESSAGE" | mail -s "$SUBJECT" "$EMAIL"
}
3. Script Principal backup.sh
#!/bin/bash

# ============================================
# CARGAR VARIABLES
# ============================================

source .env
source functions.sh

DATE=$(date +"%Y-%m-%d_%H-%M-%S")

BACKUP_FILE="backup_$DATE.tar.gz"

LOG_FILE="$LOG_DIR/backup_$DATE.log"

mkdir -p "$BACKUP_DIR"
mkdir -p "$LOG_DIR"

log_info "INICIANDO BACKUP"

# ============================================
# BACKUP DIRECTORIO
# ============================================

tar -czf "$BACKUP_DIR/$BACKUP_FILE" "$SOURCE_DIR" >> "$LOG_FILE" 2>&1

if [ $? -ne 0 ]; then
    log_error "Fallo backup principal"
    send_error_email
    exit 1
fi

log_info "Backup comprimido generado"

# ============================================
# BACKUP INCREMENTAL RSYNC
# ============================================

rsync -av --delete "$SOURCE_DIR/" "$BACKUP_DIR/incremental/" >> "$LOG_FILE" 2>&1

if [ $? -ne 0 ]; then
    log_error "Fallo backup incremental"
    send_error_email
fi

log_info "Backup incremental completado"

# ============================================
# MYSQL BACKUP
# ============================================

mysqldump -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" > "$BACKUP_DIR/mysql_$DATE.sql"

if [ $? -ne 0 ]; then
    log_error "Fallo backup MySQL"
    send_error_email
fi

# ============================================
# POSTGRESQL BACKUP
# ============================================

PGPASSWORD=$POSTGRES_PASSWORD pg_dump -U "$POSTGRES_USER" "$POSTGRES_DATABASE" > "$BACKUP_DIR/postgres_$DATE.sql"

if [ $? -ne 0 ]; then
    log_error "Fallo backup PostgreSQL"
    send_error_email
fi

# ============================================
# BACKUP REMOTO SSH
# ============================================

scp "$BACKUP_DIR/$BACKUP_FILE" "$REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR"

if [ $? -ne 0 ]; then
    log_error "Fallo transferencia remota"
    send_error_email
fi

log_info "Transferencia remota completada"

# ============================================
# SUBIDA A AWS S3
# ============================================

aws s3 cp "$BACKUP_DIR/$BACKUP_FILE" "$S3_BUCKET"

if [ $? -ne 0 ]; then
    log_error "Fallo subida S3"
    send_error_email
fi

log_info "Subida S3 completada"

# ============================================
# ELIMINAR BACKUPS ANTIGUOS
# ============================================

find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

log_info "Limpieza completada"

log_info "BACKUP FINALIZADO"

exit 0
4. Instalar Dependencias
Ubuntu/Debian
sudo apt update

sudo apt install \
    rsync \
    mailutils \
    mysql-client \
    postgresql-client \
    awscli -y
5. Configurar AWS CLI
aws configure

Te pedirá:

AWS Access Key
AWS Secret Key
Region
Output format
6. Configurar SSH sin contraseña

Generar llave:

ssh-keygen

Copiar llave:

ssh-copy-id backupuser@192.168.1.100
7. Automatizar con Cron
crontab -e

Agregar:

0 2 * * * /ruta/backup-system/backup.sh
8. Integración con systemd
backup.service
[Unit]
Description=Sistema de Backups Automáticos

[Service]
Type=oneshot
ExecStart=/ruta/backup-system/backup.sh
backup.timer
[Unit]
Description=Ejecutar Backup Diario

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
Activar servicio
sudo cp backup.service /etc/systemd/system/
sudo cp backup.timer /etc/systemd/system/

sudo systemctl daemon-reload

sudo systemctl enable backup.timer

sudo systemctl start backup.timer
9. Verificar Estado
systemctl status backup.timer
10. Logs del Sistema
journalctl -u backup.service
11. Seguridad Recomendada
Proteger .env
chmod 600 .env
12. Mejoras Enterprise

