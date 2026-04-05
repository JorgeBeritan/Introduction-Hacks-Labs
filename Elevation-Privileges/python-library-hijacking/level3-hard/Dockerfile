# ============================================================
# Python Library Hijacking Lab - Nivel 3: Dificil
# Tecnicas: .pth injection + site-packages writeable + rabbit holes
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano python3 python3-pip cron coreutils \
    findutils procps strace ltrace file \
    && rm -rf /var/lib/apt/lists/*

# Fix CRÍTICO para que cron funcione en Docker
RUN sed -i 's/session    required     pam_loginuid.so/session    optional     pam_loginuid.so/g' /etc/pam.d/cron

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# ===============================================================
# VECTOR REAL: .pth file injection en site-packages escribible
# ===============================================================
RUN SITE_PKG=$(python3 -c "import site; print(site.getsitepackages()[0])") && \
    chmod 777 "$SITE_PKG" && \
    echo "$SITE_PKG" > /tmp/site_pkg_path

# Script de backup que corre como root via cron
RUN mkdir -p /opt/backup && \
    printf '#!/usr/bin/env python3\n\
import json\n\
import datetime\n\
import shutil\n\
import os\n\
\n\
BACKUP_DIR = "/var/backups/system"\n\
os.makedirs(BACKUP_DIR, exist_ok=True)\n\
\n\
timestamp = datetime.datetime.now().isoformat()\n\
\n\
status = {\n\
    "timestamp": timestamp,\n\
    "disk_free": shutil.disk_usage("/").free,\n\
    "backup": "completed"\n\
}\n\
\n\
with open(os.path.join(BACKUP_DIR, "status.json"), "w") as f:\n\
    json.dump(status, f, indent=2)\n\
\n\
print(f"Backup completed at {timestamp}")\n' > /opt/backup/backup.py && \
    chmod 755 /opt/backup/backup.py && \
    chown root:root /opt/backup/backup.py && \
    chmod 755 /opt/backup

# Cronjob de root (cada minuto)
RUN printf '* * * * * root /usr/bin/python3 /opt/backup/backup.py >> /var/log/backup.log 2>&1\n' > /etc/cron.d/backup && \
    chmod 644 /etc/cron.d/backup

RUN touch /var/log/backup.log && chmod 644 /var/log/backup.log
RUN mkdir -p /var/backups/system && chmod 755 /var/backups/system

# ===============================================================
# RABBIT HOLE 1 CORREGIDO: Script en BASH (no activa el .pth)
# ===============================================================
RUN mkdir -p /opt/tools && \
    printf '#!/bin/bash\n\
# Solo permite listar procesos\n\
if [[ "$1" != "ps" && "$1" != "top" && "$1" != "uptime" ]]; then\n\
    echo "Usage: $0 [ps|top|uptime]"\n\
    exit 1\n\
fi\n\
\n\
if [ "$1" == "ps" ]; then /bin/ps; fi\n\
if [ "$1" == "top" ]; then /usr/bin/top; fi\n\
if [ "$1" == "uptime" ]; then /usr/bin/uptime; fi\n' > /opt/tools/sysinfo.sh && \
    chmod 755 /opt/tools/sysinfo.sh && \
    chown root:root /opt/tools && \
    chmod 755 /opt/tools

RUN printf 'hacker ALL=(root) NOPASSWD: /opt/tools/sysinfo.sh *\n' > /etc/sudoers.d/sysinfo && \
    chmod 440 /etc/sudoers.d/sysinfo

# ===============================================================
# RABBIT HOLE 2: Archivo .pyc sospechoso (no lleva a nada)
# ===============================================================
RUN mkdir -p /opt/.hidden && \
    printf 'SECRET_KEY = "this_is_not_the_flag_keep_looking"\nDB_PASS = "not_useful_either"\n' > /opt/.hidden/config.py && \
    python3 -c "import py_compile; py_compile.compile('/opt/.hidden/config.py', '/opt/.hidden/config.pyc')" && \
    rm /opt/.hidden/config.py && \
    chmod 644 /opt/.hidden/config.pyc

# ===============================================================
# PISTAS SUTILES
# ===============================================================
RUN printf '[2024-03-15 02:14] admin: Installed custom packages\n\
[2024-03-15 02:15] admin: TODO - fix permissions on python dirs\n\
[2024-03-15 02:15] admin: chmod done... I think\n\
[2024-03-22 09:00] admin: backup cron is running fine\n' > /var/log/admin_notes.log && \
    chmod 644 /var/log/admin_notes.log

# Flag
RUN echo "FLAG{nivel3_pth_injection_site_packages}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones minimas
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " Python Library Hijacking - Nivel 3: Dificil"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Este sistema tiene varios scripts y servicios."\n\
echo "No todo lo que brilla es oro."\n\
echo ""\n\
echo "Herramientas utiles:"\n\
echo "  sudo -l"\n\
echo "  cat /etc/cron.d/*"\n\
echo "  python3 -c \"import sys; print(sys.path)\""\n\
echo "  find / -writable -type d 2>/dev/null"\n\
echo "  ls -la /var/log/"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

# Entrypoint: arrancar cron y dar shell a hacker
RUN printf '#!/bin/bash\n\
cron\n\
exec su - hacker\n' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

USER root
CMD ["/entrypoint.sh"]