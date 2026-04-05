# ============================================================
# Python Library Hijacking Lab - Nivel 1: Facil
# Tecnica: Script root importa modulo desde directorio escribible
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano python3 coreutils findutils \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# Directorio de scripts
RUN mkdir -p /opt/scripts

# Script "de mantenimiento" que corre como root via sudo
RUN printf '#!/usr/bin/env python3\n\
import base64\n\
\n\
print("=== System Health Check ===")\n\
# Simulamos que hace algo con base64\n\
msg = b"T0sgLSA0NSUgdXNlZA=="\n\
print(f"Disk status: {base64.b64decode(msg).decode()}")\n\
print("Check complete.")\n' > /opt/scripts/healthcheck.py && \
    chmod 755 /opt/scripts/healthcheck.py && \
    chown root:root /opt/scripts/healthcheck.py


# Permitir a hacker ejecutar el script como root sin password
RUN printf 'hacker ALL=(root) NOPASSWD: /usr/bin/python3 /opt/scripts/healthcheck.py\n' > /etc/sudoers.d/healthcheck && \
    chmod 440 /etc/sudoers.d/healthcheck

# VULNERABILIDAD: /opt/scripts es escribible por todos (mala config)
RUN chmod 1777 /opt/scripts

# Flag
RUN echo "FLAG{nivel1_python_library_hijack_basico}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " Python Library Hijacking - Nivel 1: Facil"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Hay un script de mantenimiento que se ejecuta como root."\n\
echo "Revisa que puedes ejecutar con: sudo -l"\n\
echo ""\n\
echo "Pista: Como resuelve Python los imports?"\n\
echo "       python3 -c \"import sys; print(sys.path)\""\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]