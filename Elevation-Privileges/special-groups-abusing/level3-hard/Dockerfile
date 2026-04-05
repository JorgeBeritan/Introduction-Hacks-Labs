# ============================================================
# LAB 3 - CADENA DE GRUPOS: staff + shadow + ssl-cert
# Dificultad: DIFÍCIL
# ============================================================
# Escenario: El usuario "webdev" pertenece a los grupos
# "staff", "shadow" y "ssl-cert". Debe encadenar múltiples
# técnicas: leer hashes de /etc/shadow (grupo shadow),
# descubrir certificados/claves privadas (grupo ssl-cert),
# y escribir en directorios del sistema (grupo staff) para
# plantar un script que se ejecute como root vía cron.
# ============================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo \
    nano \
    vim \
    bash \
    coreutils \
    openssl \
    john \
    hashcat \
    net-tools \
    cron \
    python3 \
    python3-pip \
    openssh-server \
    findutils \
    wget \
    curl \
    gcc \
    make \
    && rm -rf /var/lib/apt/lists/*

# ──────────────────────────────────────────────
# FASE 1 - Crear usuarios del escenario
# ──────────────────────────────────────────────

# Usuario atacante: pertenece a staff, shadow y ssl-cert
RUN useradd -m -s /bin/bash -G staff,shadow,ssl-cert webdev && \
    echo "webdev:webdev789" | chpasswd

# Usuario intermedio: dbadmin (su hash será crackeable)
RUN useradd -m -s /bin/bash dbadmin && \
    echo "dbadmin:database1" | chpasswd

# Usuario intermedio: deployer (tiene permisos sudo limitados)
RUN useradd -m -s /bin/bash deployer && \
    echo "deployer:D3pl0y_Pr0d_2024!" | chpasswd

# ──────────────────────────────────────────────
# FASE 2 - Configurar sudo para deployer (limitado)
# ──────────────────────────────────────────────

RUN echo "deployer ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl restart apache2, /usr/local/bin/deploy.sh" >> /etc/sudoers

# ──────────────────────────────────────────────
# FASE 3 - Certificados SSL y claves
# ──────────────────────────────────────────────

# Crear estructura de certificados
RUN mkdir -p /etc/ssl/private /etc/ssl/certs && \
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/server.key \
    -out /etc/ssl/certs/server.crt \
    -subj "/C=US/ST=State/L=City/O=Corp/CN=internal.corp.local" 2>/dev/null

# Esconder una clave SSH privada de deployer dentro del directorio ssl
RUN mkdir -p /etc/ssl/private/.keys && \
    ssh-keygen -t rsa -b 2048 -f /etc/ssl/private/.keys/deployer_id_rsa -N "" -q && \
    cat <<'EOF' > /etc/ssl/private/.keys/README.internal
=================================================
Clave SSH de emergencia para el usuario deployer
Generada por el equipo de infraestructura
Fecha: 2024-01-15
=================================================
IMPORTANTE: Esta clave tiene acceso SSH como
el usuario "deployer" en este servidor.
=================================================
EOF

# Configurar permisos del directorio ssl-cert
RUN chown -R root:ssl-cert /etc/ssl/private && \
    chmod 750 /etc/ssl/private && \
    chmod 750 /etc/ssl/private/.keys && \
    chmod 640 /etc/ssl/private/.keys/deployer_id_rsa && \
    chgrp ssl-cert /etc/ssl/private/.keys/deployer_id_rsa

# Configurar SSH para deployer con la clave
RUN mkdir -p /home/deployer/.ssh && \
    cat /etc/ssl/private/.keys/deployer_id_rsa.pub > /home/deployer/.ssh/authorized_keys && \
    chown -R deployer:deployer /home/deployer/.ssh && \
    chmod 700 /home/deployer/.ssh && \
    chmod 600 /home/deployer/.ssh/authorized_keys

# ──────────────────────────────────────────────
# FASE 4 - Script de deploy vulnerable (grupo staff)
# ──────────────────────────────────────────────

# El grupo "staff" puede escribir en /usr/local/bin
RUN chown root:staff /usr/local/bin && \
    chmod 775 /usr/local/bin

# Crear script de deploy que se ejecuta como root via sudo
RUN cat <<'SCRIPT' > /usr/local/bin/deploy.sh
#!/bin/bash
# Deploy script - v2.3
# Ejecuta las tareas de deployment

LOG="/var/log/deploy.log"
echo "[$(date)] Deploy iniciado por $(whoami)" >> $LOG

# Cargar configuración de deploy
if [ -f /usr/local/etc/deploy.conf ]; then
    source /usr/local/etc/deploy.conf
fi

# Ejecutar hooks pre-deploy
if [ -d /usr/local/lib/deploy-hooks/ ]; then
    for hook in /usr/local/lib/deploy-hooks/*.sh; do
        [ -f "$hook" ] && bash "$hook"
    done
fi

echo "[$(date)] Deploy completado" >> $LOG
SCRIPT
RUN chmod 755 /usr/local/bin/deploy.sh

# Crear directorio de hooks (escribible por staff)
RUN mkdir -p /usr/local/lib/deploy-hooks && \
    chown root:staff /usr/local/lib/deploy-hooks && \
    chmod 775 /usr/local/lib/deploy-hooks

# Crear directorio de config (escribible por staff)
RUN mkdir -p /usr/local/etc && \
    chown root:staff /usr/local/etc && \
    chmod 775 /usr/local/etc

# ──────────────────────────────────────────────
# FASE 5 - Cron job que ejecuta deploy como root
# ──────────────────────────────────────────────

RUN cat <<'EOF' > /etc/cron.d/auto-deploy
# Auto-deploy cada 2 minutos
*/2 * * * * root /usr/local/bin/deploy.sh
EOF
RUN chmod 644 /etc/cron.d/auto-deploy

# ──────────────────────────────────────────────
# FASE 6 - Flag y evidencias
# ──────────────────────────────────────────────

RUN echo "FLAG{lab3_multi_group_chain_master_privesc}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Notas de dbadmin con pistas
RUN mkdir -p /home/dbadmin/docs && \
    cat <<'EOF' > /home/dbadmin/docs/server-notes.txt
Notas del servidor:
- El deploy automático corre cada 2 minutos via cron como root
- Los hooks de deploy están en /usr/local/lib/deploy-hooks/
- deployer tiene sudo para ejecutar deploy.sh
- Las claves SSL están en /etc/ssl/private/
- El equipo de staff puede modificar scripts en /usr/local/
EOF
RUN chown -R dbadmin:dbadmin /home/dbadmin/docs && \
    chmod 644 /home/dbadmin/docs/server-notes.txt

# Pistas para el atacante
RUN mkdir -p /home/webdev/workspace && \
    cat <<'EOF' > /home/webdev/workspace/tareas.txt
=== Tareas pendientes ===

[ ] Revisar los certificados SSL que expiran este mes
[ ] Coordinar con dbadmin el acceso a la base de datos
[ ] Actualizar los hooks de deployment en /usr/local/lib/
[ ] Verificar que el cron de auto-deploy sigue funcionando

Nota: Mis grupos actuales me dan acceso a bastantes cosas
en el sistema. Debería revisar si todo esto es necesario...
EOF

RUN cat <<'EOF' > /home/webdev/workspace/grupos-info.txt
Información de grupos del sistema:

- staff: Permite modificar programas en /usr/local sin ser root.
         Los miembros pueden escribir en /usr/local/bin, /usr/local/lib, etc.

- shadow: Permite leer /etc/shadow (hashes de contraseñas).
          Normalmente solo root y el grupo shadow tienen acceso.

- ssl-cert: Permite acceder a claves privadas SSL/TLS en /etc/ssl/private.
            Usado por servicios web para HTTPS.
EOF

RUN chown -R webdev:webdev /home/webdev/workspace

# ──────────────────────────────────────────────
# FASE 7 - Configuración SSH
# ──────────────────────────────────────────────

RUN mkdir -p /var/run/sshd && \
    echo "PermitRootLogin no" >> /etc/ssh/sshd_config && \
    echo "PubkeyAuthentication yes" >> /etc/ssh/sshd_config && \
    echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config

# Banner
RUN cat <<'MOTD' > /etc/motd

╔══════════════════════════════════════════════════════════════╗
║     LAB 3 - CADENA DE GRUPOS: staff + shadow + ssl-cert      ║
║                   Dificultad: DIFÍCIL                        ║
║                                                              ║
║  Objetivo: Leer /root/flag.txt                               ║
║  Usuario:  webdev / webdev789                                ║
║                                                              ║
║  Este lab requiere encadenar múltiples vectores.             ║
║  Hay al menos 3 caminos distintos hacia root.                ║
║                                                              ║
║  Pistas:                                                     ║
║  - Enumera tus grupos y qué acceso te da cada uno            ║
║  - Busca archivos accesibles por tus grupos                  ║
║  - Piensa en cómo encadenar accesos para escalar             ║
║  - /usr/local/ es tu mejor amigo                             ║
╚══════════════════════════════════════════════════════════════╝

MOTD

# Script de inicio que arranca cron y SSH
RUN cat <<'ENTRY' > /entrypoint.sh
#!/bin/bash
service cron start 2>/dev/null
service ssh start 2>/dev/null
su - webdev
ENTRY
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]