# ============================================================
# LAB 2 - GRUPO DISK + GRUPO ADM (Dificultad: Media)
# ============================================================
# Escenario: El usuario "sysop" pertenece a los grupos
# "disk" y "adm". Debe descubrir cómo usar el acceso
# raw a disco para leer archivos protegidos, y luego
# encadenar con información de logs para obtener
# credenciales de root.
# ============================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo \
    nano \
    vim \
    bash \
    coreutils \
    util-linux \
    grep \
    findutils \
    debianutils \
    openssh-server \
    rsyslog \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario vulnerable en los grupos disk y adm
RUN useradd -m -s /bin/bash -G disk,adm sysop && \
    echo "sysop:sysop456" | chpasswd

# Crear un usuario intermedio "backupadmin" con sudo
RUN useradd -m -s /bin/bash backupadmin && \
    echo "backupadmin:B4ckup_S3cur3_2024!" | chpasswd && \
    usermod -aG sudo backupadmin

# Configurar sudo para backupadmin (sin restricciones)
RUN echo "backupadmin ALL=(ALL:ALL) ALL" >> /etc/sudoers

# Crear la flag de root
RUN echo "FLAG{lab2_disk_adm_group_chain_privesc}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Simular un log donde backupadmin dejó su contraseña expuesta
# (accesible porque sysop está en el grupo adm -> puede leer /var/log/)
RUN mkdir -p /var/log && \
    cat <<'EOF' > /var/log/auth.log
Jan 15 08:23:11 server sshd[1234]: Accepted password for root from 10.0.0.1 port 22
Jan 15 08:45:02 server sudo: backupadmin : TTY=pts/0 ; PWD=/home/backupadmin ; USER=root ; COMMAND=/bin/bash
Jan 15 09:12:33 server sshd[1567]: Failed password for invalid user admin from 10.0.0.50 port 443
Jan 15 09:30:01 server CRON[1890]: (root) CMD (/usr/local/bin/backup.sh)
Jan 15 10:01:15 server sudo: backupadmin : TTY=pts/1 ; PWD=/tmp ; USER=root ; COMMAND=/usr/bin/apt update
Jan 15 10:15:44 server sshd[2001]: Accepted password for backupadmin from 10.0.0.5 port 22
Jan 15 11:00:00 server CRON[2100]: (root) CMD (/usr/local/bin/cleanup.sh)
EOF

# Crear un script de backup que contiene la contraseña hardcodeada
RUN mkdir -p /usr/local/bin && \
    cat <<'SCRIPT' > /usr/local/bin/backup.sh
#!/bin/bash
# Script de backup automático
# Autor: backupadmin
# NOTA: cambiar estas credenciales hardcodeadas eventualmente
BACKUP_USER="backupadmin"
BACKUP_PASS="B4ckup_S3cur3_2024!"
tar czf /tmp/backup_$(date +%Y%m%d).tar.gz /etc /home 2>/dev/null
SCRIPT
RUN chmod 750 /usr/local/bin/backup.sh && \
    chown root:root /usr/local/bin/backup.sh

# Crear archivo shadow legible via grupo disk (debugfs)
# y un archivo adicional con pista en /etc
RUN cat <<'EOF' > /etc/cron.d/backup-job
# Backup diario ejecutado por backupadmin
0 3 * * * backupadmin /usr/local/bin/backup.sh
EOF

# Configurar permisos de logs para grupo adm
RUN chown root:adm /var/log/auth.log && \
    chmod 640 /var/log/auth.log

# Pista inicial para el usuario
RUN mkdir -p /home/sysop/notas && \
    cat <<'EOF' > /home/sysop/notas/onboarding.txt
¡Bienvenido al equipo de operaciones!

Tu cuenta está configurada con acceso a:
- Logs del sistema (para monitoreo)
- Dispositivos de disco (para mantenimiento de emergencia)

Recuerda:
- Los scripts de backup están en /usr/local/bin/
- Si necesitas acceso root, contacta a backupadmin
- Los logs de autenticación están en /var/log/

Cualquier duda, pregunta al equipo de infraestructura.
EOF

RUN chown -R sysop:sysop /home/sysop/notas

# Crear un archivo "secreto" accesible solo via debugfs/raw disk
RUN mkdir -p /root/.secrets && \
    cat <<'EOF' > /root/.secrets/credentials.txt
=== Credenciales de Emergencia ===
Usuario: backupadmin
Password: B4ckup_S3cur3_2024!
Acceso: sudo completo
=== NO COMPARTIR ===
EOF
RUN chmod 600 /root/.secrets/credentials.txt

# Banner
RUN cat <<'MOTD' > /etc/motd

╔══════════════════════════════════════════════════════════╗
║        LAB 2 - ABUSO DE GRUPOS DISK + ADM                ║
║                Dificultad: MEDIA                         ║
║                                                          ║
║  Objetivo: Leer /root/flag.txt                           ║
║  Usuario:  sysop / sysop456                              ║
║                                                          ║
║  Pistas:                                                 ║
║  - ¿A qué grupos perteneces?                             ║
║  - ¿Qué puedes leer con el grupo "adm"?                 ║
║  - ¿Qué puedes hacer con el grupo "disk"?                ║
║  - Hay más de un camino para resolver esto               ║
╚══════════════════════════════════════════════════════════╝

MOTD

USER sysop
WORKDIR /home/sysop

CMD ["/bin/bash", "-l"]