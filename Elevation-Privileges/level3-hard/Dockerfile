# ============================================================
# SUID Lab - Nivel 3: Dificil
# Tecnicas: Shared Object Injection + SUID binary RE
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano gcc python3 strace ltrace file binutils coreutils \
    && rm -rf /var/lib/apt/lists/*

# --- VULNERABILIDAD: Shared Object Injection ---

# Crear la libreria legitima
RUN printf '#include <stdio.h>\n\
\n\
void check_status() {\n\
    printf("System check: All services running.\\n");\n\
}\n' > /tmp/libutils.c && \
    gcc -shared -fPIC -o /usr/lib/libcustomutils.so /tmp/libutils.c && \
    rm /tmp/libutils.c

# Crear el binario SUID que usa la libreria
RUN printf '#include <stdio.h>\n\
#include <stdlib.h>\n\
\n\
extern void check_status();\n\
\n\
int main() {\n\
    printf("=== Security Monitor v3.1 ===\\n");\n\
    printf("Running diagnostics...\\n");\n\
    check_status();\n\
    printf("Diagnostics complete.\\n");\n\
    return 0;\n\
}\n' > /tmp/monitor.c && \
    gcc /tmp/monitor.c -o /usr/local/bin/security-monitor \
    -L/usr/lib -lcustomutils \
    -Wl,-rpath,/home/hacker/.config/lib:/usr/lib && \
    chmod u+s /usr/local/bin/security-monitor && \
    rm /tmp/monitor.c

# Directorio escribible por hacker (el rpath lo busca primero)
RUN mkdir -p /home/hacker/.config/lib

# --- RABBIT HOLE: backup-tool parece vulnerable pero no lo es ---
RUN printf '#!/bin/bash\n\
if [ "$#" -ne 1 ]; then\n\
    echo "Usage: backup-tool <directory>"\n\
    exit 1\n\
fi\n\
CLEAN_DIR=$(realpath -m -- "$1" 2>/dev/null)\n\
if [[ ! "$CLEAN_DIR" =~ ^/home/hacker/ ]]; then\n\
    echo "Error: Solo puedes hacer backup de tu home."\n\
    exit 1\n\
fi\n\
if [ ! -d "$CLEAN_DIR" ]; then\n\
    echo "Error: Directorio no existe."\n\
    exit 1\n\
fi\n\
tar czf /tmp/backup_$(date +%%s).tar.gz -C "$CLEAN_DIR" . 2>/dev/null\n\
echo "Backup completado."\n' > /usr/local/bin/backup-tool && \
    chmod u+s /usr/local/bin/backup-tool

# Pista oculta
RUN mkdir -p /opt/security-monitor && \
    printf "Security Monitor v3.1\n=====================\nCompiled with custom library support.\nFor debugging, use: ltrace or strace\nReport issues to: admin@internal\n" > /opt/security-monitor/README && \
    chmod 644 /opt/security-monitor/README

# Crear usuario
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

RUN chown -R hacker:hacker /home/hacker/.config

# Flags
RUN echo "FLAG{nivel3_shared_object_injection_master}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones minimas
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " SUID Lab - Nivel 3: Dificil"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "No todas las puertas llevan a algun lado."\n\
echo "Entiende que hace cada binario ANTES de intentar explotarlo."\n\
echo ""\n\
echo "Herramientas utiles: strings, ltrace, strace, file"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]