# ============================================================
# LAB 3 - "Service Chain" (Avanzado)
# ============================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    apache2 \
    libapache2-mod-php \
    redis-server \
    openssh-server \
    sudo \
    curl \
    nano \
    vim \
    net-tools \
    procps \
    socat \
    passwd \
    php-redis \
    nmap \
    && rm -rf /var/lib/apt/lists/*

# ========== SERVICIO 1: Redis sin auth ==========
RUN sed -i 's/^bind .*/bind 127.0.0.1/' /etc/redis/redis.conf && \
    sed -i 's/^protected-mode yes/protected-mode no/' /etc/redis/redis.conf

# ========== SERVICIO 2: Apache y Preparación para Webshell ==========
# Damos permisos al grupo redis para que pueda escribir la webshell en el webroot
RUN chown -R www-data:redis /var/www/html && \
    chmod 775 /var/www/html

# Quitamos el index.html por defecto
RUN rm -f /var/www/html/index.html

RUN echo '<?php\n\
echo "<html><head><title>Corp Portal</title></head><body>";\n\
echo "<h1>Corporate Internal Portal</h1>";\n\
echo "<p>Authorized personnel only.</p>";\n\
echo "";\n\
echo "</body></html>";\n\
?>' > /var/www/html/index.php

# ---------- EL PIVOTE: Credenciales ocultas ----------
# Solo www-data puede leer esto, obligando al atacante a ganar RCE web primero
RUN mkdir -p /var/www/html/internal && \
    echo '<?php\n\
// Credenciales de la base de datos interna y servicios\n\
define("DB_USER", "svc_monitor");\n\
define("DB_PASS", "M0n1t0r_S3rv1c3!");\n\
?>' > /var/www/html/internal/db_config.php && \
    chown www-data:www-data /var/www/html/internal/db_config.php && \
    chmod 400 /var/www/html/internal/db_config.php

# ========== SERVICIO 3: SSH + Usuario de servicio ==========
RUN mkdir -p /var/run/sshd && \
    sed -i 's/#PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config

RUN useradd -m -s /bin/bash svc_monitor && \
    echo 'svc_monitor:M0n1t0r_S3rv1c3!' | chpasswd

# Sudo misconfiguration: SETENV permite pasar variables.
RUN echo 'svc_monitor ALL=(root) SETENV: NOPASSWD: /opt/monitoring/check_services.sh' >> /etc/sudoers

# Script de monitoreo vulnerable a inyección por variable de entorno
RUN mkdir -p /opt/monitoring && \
    echo '#!/bin/bash\n\
echo "=== Service Check $(date) ==="\n\
\n\
# Ejecución sin sanitizar de una variable de entorno\n\
SERVICE_CMD=${SERVICE_CMD:-"/usr/bin/systemctl is-active apache2"}\n\
echo "Apache status: $($SERVICE_CMD 2>/dev/null)"\n\
\n\
echo "Check completed."' > /opt/monitoring/check_services.sh && \
    chmod 755 /opt/monitoring/check_services.sh

# ========== Usuario de práctica ==========
RUN useradd -m -s /bin/bash student && \
    echo 'student:student123' | chpasswd

RUN echo "Infraestructura interna:\n\
- Portal web activo.\n\
- Servicios de cacheado internos activados.\n\
- Objetivo: Encadenar vulnerabilidades para escalar a root." \
    > /home/student/welcome.txt && \
    chown student:student /home/student/welcome.txt

# ========== Flags ==========
RUN echo "FLAG{lab3_service_chain_user_pivot}" > /home/svc_monitor/user_flag.txt && \
    chmod 600 /home/svc_monitor/user_flag.txt && \
    chown svc_monitor:svc_monitor /home/svc_monitor/user_flag.txt

RUN echo "FLAG{lab3_multi_service_full_chain_root}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# ========== Entrypoint ==========
# Iniciamos los servicios correctamente con sus respectivos usuarios
RUN echo '#!/bin/bash\n\
service apache2 start\n\
service redis-server start\n\
service ssh start\n\
\n\
echo ""\n\
echo "================================================"\n\
echo "  LAB 3 - Service Chain (Avanzado)"\n\
echo "================================================"\n\
echo "  Usuario: student / student123"\n\
echo "  Objetivo 1: /home/svc_monitor/user_flag.txt"\n\
echo "  Objetivo 2: /root/flag.txt"\n\
echo "================================================"\n\
echo ""\n\
exec su - student' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

EXPOSE 80 22

ENTRYPOINT ["/entrypoint.sh"]