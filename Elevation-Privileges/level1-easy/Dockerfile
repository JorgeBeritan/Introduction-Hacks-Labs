# ============================================================
# SUID Lab - Nivel 1: Fácil
# Técnica: Abuso de binario conocido con bit SUID
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04
 
# Evitar prompts interactivos
ENV DEBIAN_FRONTEND=noninteractive
 
RUN apt-get update && apt-get install -y \
    sudo \
    vim \
    nano \
    findutils \
    coreutils \
    && rm -rf /var/lib/apt/lists/*
 
# --- VULNERABILIDAD INTENCIONADA ---
# Se configura el bit SUID en /usr/bin/find
# GTFOBins: find puede ejecutar comandos con -exec
RUN chmod u+s /usr/bin/find
 
# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd
 
# Crear flag de root para verificar escalada
RUN echo "FLAG{nivel1_suid_find_completado}" > /root/flag.txt && \
    chmod 600 /root/flag.txt
 
# Pista para el jugador
RUN echo '#!/bin/bash' > /home/hacker/instrucciones.sh && \
    echo 'echo "========================================="' >> /home/hacker/instrucciones.sh && \
    echo 'echo " SUID Lab - Nivel 1: Fácil"' >> /home/hacker/instrucciones.sh && \
    echo 'echo "========================================="' >> /home/hacker/instrucciones.sh && \
    echo 'echo ""' >> /home/hacker/instrucciones.sh && \
    echo 'echo "Objetivo: Leer /root/flag.txt"' >> /home/hacker/instrucciones.sh && \
    echo 'echo ""' >> /home/hacker/instrucciones.sh && \
    echo 'echo "Pista: Busca binarios con permisos especiales..."' >> /home/hacker/instrucciones.sh && \
    echo 'echo "       find / -perm -4000 2>/dev/null"' >> /home/hacker/instrucciones.sh && \
    echo 'echo ""' >> /home/hacker/instrucciones.sh && \
    echo 'echo "Recurso: https://gtfobins.github.io/"' >> /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh
 
USER hacker
WORKDIR /home/hacker
 
CMD ["/bin/bash"]