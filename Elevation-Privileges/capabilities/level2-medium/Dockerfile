# ============================================================
# Capabilities Lab - Nivel 2: Medio
# Tecnicas: cap_dac_read_search + cap_net_raw (multiples vectores)
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano python3 gcc libcap2-bin \
    coreutils findutils tar gzip \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# --- VULNERABILIDAD 1: cap_dac_read_search en tar ---
# Permite leer CUALQUIER archivo del sistema ignorando permisos
# (DAC = Discretionary Access Control)
RUN cp /usr/bin/tar /usr/bin/tar-backup && \
    setcap cap_dac_read_search+ep /usr/bin/tar-backup

# --- RABBIT HOLE: cap_net_raw en python3 ---
# Parece peligroso pero cap_net_raw solo permite raw sockets
# (sniffing, ping), NO escalar privilegios
RUN cp /usr/bin/python3 /usr/bin/python3-net && \
    setcap cap_net_raw+ep /usr/bin/python3-net

# --- VULNERABILIDAD 2 (via indirecta): cap_dac_override en vim ---
# Permite escribir en CUALQUIER archivo (ignorar permisos de escritura)
# Se puede editar /etc/passwd o /etc/shadow
RUN cp /usr/bin/vim /usr/bin/vim-admin && \
    setcap cap_dac_override+ep /usr/bin/vim-admin

# Archivos sensibles con permisos estrictos (normalmente inaccesibles)
RUN echo "database_password=S3cur3P4ss!" > /etc/app-secrets.conf && \
    chmod 600 /etc/app-secrets.conf

# Flag
RUN echo "FLAG{nivel2_dac_read_search_override}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " Capabilities Lab - Nivel 2: Medio"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Hay varias capabilities asignadas en este sistema."\n\
echo "No todas son utiles para escalar privilegios."\n\
echo ""\n\
echo "Pistas:"\n\
echo "  1. Enumera: getcap -r / 2>/dev/null"\n\
echo "  2. Investiga que hace cada capability"\n\
echo "  3. Hay al menos DOS formas de leer la flag"\n\
echo "  4. Una de las capabilities es un rabbit hole"\n\
echo ""\n\
echo "man capabilities -> busca cada CAP_* que encuentres"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]