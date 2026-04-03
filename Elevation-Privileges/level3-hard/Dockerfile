# ============================================================
# SUID Lab - Nivel 3: Difícil
# Técnicas: Shared Object Injection + SUID binary RE
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
    strace \
    ltrace \
    file \
    strings \
    coreutils \
    && rm -rf /var/lib/apt/lists/*

# --- VULNERABILIDAD: Shared Object Injection ---
# Binario SUID que carga una librería desde un path escribible

# Crear la librería legítima
RUN cat <<'EOF' > /tmp/libutils.c
#include <stdio.h>

void check_status() {
    printf("System check: All services running.\n");
}
EOF

RUN gcc -shared -fPIC -o /usr/lib/libcustomutils.so /tmp/libutils.c && \
    rm /tmp/libutils.c

# Crear el binario SUID que usa la librería
RUN cat <<'EOF' > /tmp/monitor.c
#include <stdio.h>
#include <stdlib.h>

// Declaramos la función externa
extern void check_status();

int main() {
    printf("=== Security Monitor v3.1 ===\n");
    printf("Running diagnostics...\n");
    check_status();
    printf("Diagnostics complete.\n");
    return 0;
}
EOF

RUN gcc /tmp/monitor.c -o /usr/local/bin/security-monitor \
    -L/usr/lib -lcustomutils \
    -Wl,-rpath,/home/hacker/.config/lib:/usr/lib && \
    chmod u+s /usr/local/bin/security-monitor && \
    rm /tmp/monitor.c

# Crear el directorio que el binario busca (escribible por hacker)
# El rpath incluye /home/hacker/.config/lib ANTES que /usr/lib
RUN mkdir -p /home/hacker/.config/lib

# --- RABBIT HOLE: SUID en un script que parece vulnerable pero no lo es ---
# Esto es para despistar y que no vaya directo al vector real
RUN cat <<'EOF' > /usr/local/bin/backup-tool
#!/bin/bash
# Backup tool - runs as root
if [ "$#" -ne 1 ]; then
    echo "Usage: backup-tool <directory>"
    exit 1
fi

# Sanitiza input - NO es vulnerable (rabbit hole)
CLEAN_DIR=$(realpath -m -- "$1" 2>/dev/null)

if [[ ! "$CLEAN_DIR" =~ ^/home/hacker/ ]]; then
    echo "Error: Solo puedes hacer backup de tu home."
    exit 1
fi

if [ ! -d "$CLEAN_DIR" ]; then
    echo "Error: Directorio no existe."
    exit 1
fi

tar czf /tmp/backup_$(date +%s).tar.gz -C "$CLEAN_DIR" . 2>/dev/null
echo "Backup completado."
EOF

RUN chmod u+s /usr/local/bin/backup-tool

# --- PISTA OCULTA en un archivo de configuración ---
RUN mkdir -p /opt/security-monitor && \
    cat <<'EOF' > /opt/security-monitor/README
Security Monitor v3.1
=====================
Compiled with custom library support.
For debugging, use: ltrace or strace
Report issues to: admin@internal
EOF

RUN chmod 644 /opt/security-monitor/README

# Crear usuario
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# Asegurar que hacker es dueño de su .config
RUN chown -R hacker:hacker /home/hacker/.config

# Flags
RUN echo "FLAG{nivel3_shared_object_injection_master}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones mínimas
RUN cat <<'INST' > /home/hacker/instrucciones.sh
#!/bin/bash
echo "========================================="
echo " SUID Lab - Nivel 3: Difícil"
echo "========================================="
echo ""
echo "Objetivo: Leer /root/flag.txt"
echo ""
echo "No todas las puertas llevan a algún lado."
echo "Entiende qué hace cada binario ANTES de intentar explotarlo."
echo ""
echo "Herramientas útiles: strings, ltrace, strace, file"
echo ""
INST

RUN chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker

CMD ["/bin/bash"]