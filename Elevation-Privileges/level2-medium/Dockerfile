# ============================================================
# SUID Lab - Nivel 2: Medio
# Tecnica: Abuso de SUID en binarios menos obvios + PATH
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano gcc python3 coreutils \
    && rm -rf /var/lib/apt/lists/*

# --- VULNERABILIDAD 1: SUID en env ---
RUN chmod u+s /usr/bin/env

# --- VULNERABILIDAD 2: Binario custom con PATH hijack ---
RUN printf '#include <stdio.h>\n\
#include <stdlib.h>\n\
\n\
int main() {\n\
    printf("=== System Status Check ===\\n");\n\
    printf("Current time: ");\n\
    fflush(stdout);\n\
    system("date");\n\
    printf("Status: OK\\n");\n\
    return 0;\n\
}\n' > /tmp/statuscheck.c && \
    gcc /tmp/statuscheck.c -o /usr/local/bin/statuscheck && \
    chmod u+s /usr/local/bin/statuscheck && \
    rm /tmp/statuscheck.c

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# Flags
RUN echo "FLAG{nivel2_suid_env_path_hijack}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Flag intermedio
RUN mkdir -p /opt/secret && \
    echo "Bien hecho, pero la flag real esta en /root/flag.txt" > /opt/secret/nota.txt && \
    chmod 700 /opt/secret

# Instrucciones
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " SUID Lab - Nivel 2: Medio"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Hay MAS DE UNA via de escalada."\n\
echo "Intenta encontrar al menos dos formas diferentes."\n\
echo ""\n\
echo "Pistas:"\n\
echo "  1. No todos los SUID interesantes son binarios del sistema."\n\
echo "  2. Que pasa cuando un programa confia en el PATH?"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]