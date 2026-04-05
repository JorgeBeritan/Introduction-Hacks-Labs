# ============================================================
# SUID Lab - Nivel 3: Avanzado (Multi-Stage Pivoting)
# Tecnicas: SUID + Pivoting entre usuarios + Sudoers + Capabilities
# Objetivo: Escalar de "hacker" -> "sysadmin" -> "devops" -> root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano gcc python3 coreutils \
    findutils net-tools libcap2-bin openssl\
    && rm -rf /var/lib/apt/lists/*

# ============================================================
# USUARIOS
# ============================================================
# Usuario inicial (punto de entrada)
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# Usuario intermedio 1
RUN useradd -m -s /bin/bash sysadmin && \
    echo "sysadmin:$(openssl rand -hex 16)" | chpasswd

# Usuario intermedio 2
RUN useradd -m -s /bin/bash devops && \
    echo "devops:$(openssl rand -hex 16)" | chpasswd

# ============================================================
# ETAPA 1: hacker -> sysadmin
# Tecnica: SUID en binario custom que ejecuta un script
#          usando ruta relativa. El binario usa setreuid()
#          para que bash no pueda droppear privilegios.
#
# Vulnerabilidad: El binario busca "./scripts/netcheck.sh"
#   relativo al CWD. Si el atacante crea su propio
#   ./scripts/netcheck.sh y ejecuta healthcheck desde ahi,
#   obtiene shell como sysadmin.
#
# Solucion del lab:
#   cd /tmp
#   mkdir -p scripts
#   echo '#!/bin/bash' > scripts/netcheck.sh
#   echo 'bash -p' >> scripts/netcheck.sh   (o /bin/bash -p)
#   chmod +x scripts/netcheck.sh
#   /usr/local/bin/healthcheck
#   -> shell como sysadmin
# ============================================================

# Binario SUID owned by sysadmin que ejecuta un "health check"
# FIX: Usa setreuid(1001,1001) y setregid(1001,1001) hardcodeados
#      para fijar uid/gid real y efectivo a sysadmin ANTES del exec.
#      Esto garantiza que bash no droppee privilegios.
RUN printf '#include <stdio.h>\n\
#include <stdlib.h>\n\
#include <unistd.h>\n\
\n\
int main() {\n\
    printf("[*] Running network health check...\\n");\n\
    fflush(stdout);\n\
    \n\
    setregid(1001, 1001);\n\
    setreuid(1001, 1001);\n\
    \n\
    system("./scripts/netcheck.sh");\n\
    return 0;\n\
}\n' > /tmp/healthcheck.c && \
    gcc /tmp/healthcheck.c -o /usr/local/bin/healthcheck && \
    chown sysadmin:sysadmin /usr/local/bin/healthcheck && \
    chmod u+s /usr/local/bin/healthcheck && \
    rm /tmp/healthcheck.c

# Script legitimo del health check (pista falsa)
RUN mkdir -p /opt/monitoring/scripts && \
    printf '#!/bin/bash\necho "Network: OK"\necho "DNS: OK"\n' \
    > /opt/monitoring/scripts/netcheck.sh && \
    chmod 755 /opt/monitoring/scripts/netcheck.sh && \
    chown sysadmin:sysadmin /opt/monitoring/scripts/netcheck.sh

# Pista para el jugador en el home de hacker
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " SUID Lab - Nivel 3: Avanzado"\n\
echo " Multi-Stage Privilege Escalation"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Este lab requiere MULTIPLES saltos entre usuarios."\n\
echo "No vas a llegar a root directamente."\n\
echo ""\n\
echo "Cadena esperada:"\n\
echo "  hacker -> ??? -> ??? -> root"\n\
echo ""\n\
echo "Pistas:"\n\
echo "  1. Empieza buscando binarios SUID inusuales."\n\
echo "  2. Presta atencion a QUIEN es el owner del SUID."\n\
echo "  3. Cada usuario tiene sus propias debilidades."\n\
echo "  4. No todo es SUID... mira sudoers, capabilities..."\n\
echo "  5. A veces un programa busca archivos donde no deberia..."\n\
echo ""\n\
echo "Buena suerte."\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

# ============================================================
# ETAPA 2: sysadmin -> devops
# Tecnica: sudoers mal configurado - sysadmin puede ejecutar
#          /usr/bin/find como devops sin password
# Explotacion:
#   sudo -u devops /usr/bin/find /tmp -name "x" -exec /bin/bash \;
# ============================================================
RUN echo "sysadmin ALL=(devops) NOPASSWD: /usr/bin/find" >> /etc/sudoers

# ============================================================
# ETAPA 3: devops -> root
# Tecnica: Capability cap_setuid en python3
# Explotacion:
#   /usr/local/bin/python3-tools -c \
#     'import os; os.setuid(0); os.system("/bin/bash")'
# ============================================================
RUN cp /usr/bin/python3 /usr/local/bin/python3-tools && \
    chown devops:devops /usr/local/bin/python3-tools && \
    setcap cap_setuid+ep /usr/local/bin/python3-tools

# Breadcrumb para devops
RUN mkdir -p /home/devops/tools && \
    printf '#!/usr/local/bin/python3-tools\n\
# Script interno del equipo devops\n\
# Usamos python3-tools para tareas de mantenimiento\n\
# TODO: revisar por que tiene capabilities raras\n\
import os\n\
import subprocess\n\
\n\
def check_services():\n\
    print("[*] Checking services...")\n\
    # Solo un check basico\n\
    subprocess.run(["ps", "aux"])\n\
\n\
if __name__ == "__main__":\n\
    check_services()\n' \
    > /home/devops/tools/maintenance.py && \
    chown -R devops:devops /home/devops/tools && \
    chmod 750 /home/devops/tools/maintenance.py

# ============================================================
# FLAGS Y BREADCRUMBS
# ============================================================
# Flag final
RUN echo "FLAG{nivel3_suid_pivot_sudoers_capabilities_master}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Flag intermedia etapa 1 (en home de sysadmin)
RUN echo "STAGE1{pivoted_to_sysadmin_via_suid_path}" > /home/sysadmin/stage1.txt && \
    chown sysadmin:sysadmin /home/sysadmin/stage1.txt && \
    chmod 600 /home/sysadmin/stage1.txt

# Flag intermedia etapa 2 (en home de devops)
RUN echo "STAGE2{pivoted_to_devops_via_sudo_find}" > /home/devops/stage2.txt && \
    chown devops:devops /home/devops/stage2.txt && \
    chmod 600 /home/devops/stage2.txt

# ============================================================
# HARDENING (para que no haya atajos)
# ============================================================
# Quitar SUID de binarios que podrian dar atajos
RUN chmod u-s /usr/bin/su && \
    chmod u-s /usr/bin/chsh && \
    chmod u-s /usr/bin/newgrp 2>/dev/null; \
    chmod u-s /usr/bin/gpasswd 2>/dev/null; \
    chmod u-s /usr/bin/passwd

# Asegurar que /tmp sea escribible por todos (necesario para etapa 1)
RUN chmod 1777 /tmp

# Limpiar historial
RUN rm -f /root/.bash_history /home/*/.bash_history

# ============================================================
# PUNTO DE ENTRADA
# ============================================================
USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]