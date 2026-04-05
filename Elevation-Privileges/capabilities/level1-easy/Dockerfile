# ============================================================
# Capabilities Lab - Nivel 1: Facil
# Tecnica: cap_setuid en Python -> cambiar UID a 0
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano python3 libcap2-bin coreutils findutils \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# --- VULNERABILIDAD: cap_setuid en python3 ---
# Permite a cualquier usuario cambiar su UID (a 0 = root)
RUN cp /usr/bin/python3 /usr/bin/python3-custom && \
    setcap cap_setuid+ep /usr/bin/python3-custom

# Flag
RUN echo "FLAG{nivel1_cap_setuid_python}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " Capabilities Lab - Nivel 1: Facil"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "En Linux, las capabilities dividen los poderes de root"\n\
echo "en permisos granulares. Algunas son peligrosas..."\n\
echo ""\n\
echo "Pista: Enumera capabilities en el sistema"\n\
echo "       getcap -r / 2>/dev/null"\n\
echo ""\n\
echo "Recurso: https://gtfobins.github.io/#+capabilities"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]