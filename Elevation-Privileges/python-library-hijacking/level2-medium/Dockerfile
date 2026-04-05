# ============================================================
# Python Library Hijacking Lab - Nivel 2: Medio
# Tecnica: PYTHONPATH injection + cronjob de root
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano python3 cron coreutils findutils procps \
    && rm -rf /var/lib/apt/lists/*

RUN sed -i 's\session   required    pam_loginuid.so\session optional    pam_loginuid.so\g' /etc/pam.d/cron

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# --- ESCENARIO: Cronjob de root ejecuta un script Python ---
RUN mkdir -p /opt/monitoring

# Script de monitoreo (NO escribible por hacker)
RUN printf '#!/usr/bin/env python3\n\
import os\n\
import logging\n\
\n\
logging.basicConfig(filename="/var/log/monitor.log", level=logging.INFO)\n\
\n\
disk = os.popen("df -h /").read()\n\
logging.info("Disk check completed")\n\
\n\
mem = os.popen("free -m").read()\n\
logging.info("Memory check completed")\n\
\n\
print("Monitoring complete")\n' > /opt/monitoring/monitor.py && \
    chmod 755 /opt/monitoring/monitor.py && \
    chown root:root /opt/monitoring/monitor.py && \
    chown root:root /opt/monitoring && \
    chmod 755 /opt/monitoring

# VULNERABILIDAD: El cronjob exporta PYTHONPATH con un directorio escribible
# Simula un admin que quiso agregar modulos custom y lo hizo mal
RUN mkdir -p /opt/pylibs && \
    chown root:root /opt/pylibs && \
    chmod 777 /opt/pylibs

# Cronjob que corre cada minuto como root
RUN printf '* * * * * root PYTHONPATH=/opt/pylibs /usr/bin/python3 /opt/monitoring/monitor.py >> /var/log/monitor.log 2>&1\n' > /etc/cron.d/monitoring && \
    chmod 644 /etc/cron.d/monitoring

# Crear el log para que hacker pueda ver que el cron esta corriendo
RUN touch /var/log/monitor.log && \
    chmod 644 /var/log/monitor.log

# Flag
RUN echo "FLAG{nivel2_pythonpath_cronjob_hijack}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " Python Library Hijacking - Nivel 2: Medio"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Algo se ejecuta periodicamente en este sistema..."\n\
echo ""\n\
echo "Pistas:"\n\
echo "  1. Revisa /etc/cron.d/ y los logs del sistema"\n\
echo "  2. Que modulos importa el script? Son de la stdlib?"\n\
echo "  3. PYTHONPATH puede cambiar donde Python busca modulos"\n\
echo ""\n\
echo "Util: cat /var/log/monitor.log"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

# Script de inicio: arrancar cron y dar shell
RUN printf '#!/bin/bash\n\
cron\n\
exec su - hacker\n' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

USER root
CMD ["/entrypoint.sh"]