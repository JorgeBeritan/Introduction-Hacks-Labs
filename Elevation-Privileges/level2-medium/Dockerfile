# ============================================================
# SUID Lab - Nivel 2: Medio
# Técnica: Abuso de SUID en binarios menos obvios + PATH
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo \
    vim \
    nano \
    gcc \
    python3 \
    coreutils \
    && rm -rf /var/lib/apt/lists/*

# --- VULNERABILIDAD 1: SUID en env (menos conocido) ---
# env puede lanzar otros programas manteniendo el SUID
RUN chmod u+s /usr/bin/env

# --- VULNERABILIDAD 2: Script custom con SUID + abuso de PATH ---
# Binario C que llama a "date" sin ruta absoluta
COPY <<EOF /tmp/statuscheck.c
#include <stdio.h>
#include <stdlib.h>

int main() {
    printf("=== System Status Check ===\\n");
    printf("Current time: ");
    fflush(stdout);
    system("date");
    printf("Status: OK\\n");
    return 0;
}
EOF

RUN gcc /tmp/statuscheck.c -o /usr/local/bin/statuscheck && \
    chmod u+s /usr/local/bin/statuscheck && \
    rm /tmp/statuscheck.c

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# Flags
RUN echo "FLAG{nivel2_suid_env_path_hijack}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Crear un flag intermedio para dar feedback
RUN mkdir -p /opt/secret && \
    echo "Bien hecho, pero la flag real está en /root/flag.txt" > /opt/secret/nota.txt && \
    chmod 700 /opt/secret

# Instrucciones (menos pistas que nivel 1)
COPY <<EOF /home/hacker/instrucciones.sh
#!/bin/bash
echo "========================================="
echo " SUID Lab - Nivel 2: Medio"
echo "========================================="
echo ""
echo "Objetivo: Leer /root/flag.txt"
echo ""
echo "Hay MAS DE UNA via de escalada."
echo "Intenta encontrar al menos dos formas diferentes."
echo ""
echo "Pistas:"
echo "  1. No todos los SUID interesantes son binarios del sistema."
echo "  2. Que pasa cuando un programa confia en el PATH?"
echo ""
EOF

RUN chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker

CMD ["/bin/bash"]