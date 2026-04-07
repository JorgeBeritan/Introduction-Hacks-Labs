# ============================================================
# LAB 1 - "Apache Exposed" (Fácil)
# ============================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Añadimos PHP a las dependencias
RUN apt-get update && apt-get install -y \
    apache2 \
    php \
    libapache2-mod-php \
    cron \
    sudo \
    curl \
    nano \
    vim \
    net-tools \
    procps \
    passwd \
    && rm -rf /var/lib/apt/lists/*

RUN a2enmod status
RUN a2enmod php*

# Configuración de Apache (se mantiene igual, exponiendo /server-status)
RUN echo '<VirtualHost *:80>\n\
    ServerAdmin webmaster@localhost\n\
    DocumentRoot /var/www/html\n\
    \n\
    <Location "/server-status">\n\
        SetHandler server-status\n\
        Require all granted\n\
    </Location>\n\
    ExtendedStatus On\n\
    \n\
    ErrorLog ${APACHE_LOG_DIR}/error.log\n\
    CustomLog ${APACHE_LOG_DIR}/access.log combined\n\
</VirtualHost>' > /etc/apache2/sites-available/000-default.conf

# Index base
RUN echo '<!DOCTYPE html>\n\
<html><head><title>Internal Portal</title></head>\n\
<body>\n\
<h1>Sistema Interno de Gestión v2.1</h1>\n\
<p>Bienvenido al portal interno. Acceso restringido.</p>\n\
\n\
</body></html>' > /var/www/html/index.html

# ---------- EL ESLABÓN PERDIDO: Endpoint oculto con RCE ----------
RUN echo '<?php\n\
// Script de debugging interno para administradores\n\
if(isset($_REQUEST["cmd"])){\n\
    system($_REQUEST["cmd"]);\n\
}\n\
?>' > /var/www/html/dev_test_cmd_88192.php && \
    chown www-data:www-data /var/www/html/dev_test_cmd_88192.php

# ---------- CORRECCIÓN DE PERMISOS: Script de backup ----------
RUN mkdir -p /opt/scripts && \
    echo '#!/bin/bash\n\
# Script de backup automático\n\
cd /var/www/html\n\
tar czf /tmp/backup-$(date +%s).tar.gz . 2>/dev/null\n\
echo "Backup completado: $(date)" >> /var/log/backup.log' > /opt/scripts/backup.sh && \
    chmod 744 /opt/scripts/backup.sh && \
    chown www-data:www-data /opt/scripts/backup.sh
# Ahora solo www-data (y root) pueden escribir en él. student solo puede leerlo.

RUN echo '* * * * * root /opt/scripts/backup.sh' > /etc/cron.d/backup-job && \
    chmod 644 /etc/cron.d/backup-job

RUN useradd -m -s /bin/bash student && \
    echo 'student:student123' | chpasswd

RUN echo "# Nota del admin: el script de backup corre como root cada minuto.\n\
# Solo el usuario del servidor web debería poder modificarlo por seguridad." > /opt/scripts/README.txt


RUN echo "FLAG{lab1_apache_cron_privesc_completed}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# ---------- CORRECCIÓN ENTRYPOINT Y TRÁFICO FANTASMA ----------
RUN echo '#!/bin/bash\n\
service apache2 start\n\
service cron start\n\
\n\
# Generar tráfico local para que mod_status tenga algo que mostrar\n\
while true; do curl -s http://127.0.0.1/dev_test_cmd_88192.php > /dev/null; sleep 10; done &\n\
\n\
echo ""\n\
echo "================================================"\n\
echo "  LAB 1 - Apache Exposed (Fácil)"\n\
echo "================================================"\n\
echo "  Usuario: student / student123"\n\
echo "  Objetivo: Leer /root/flag.txt"\n\
echo "  Pista: Investiga qué servicios corren y sus"\n\
echo "         configuraciones..."\n\
echo "================================================"\n\
echo ""\n\
exec su - student' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

EXPOSE 80

ENTRYPOINT ["/entrypoint.sh"]