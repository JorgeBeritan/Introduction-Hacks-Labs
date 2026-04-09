FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# ============================================================
# PRIVESC LAB - 4 Técnicas de Escalada de Privilegios
# ============================================================
# Cadena: user1 -> user2 -> user3 -> user4 -> root
#
# Técnica 1: Abuso del grupo "adm" (lectura de logs)
# Técnica 2: Python Library Hijacking
# Técnica 3: Shared Library Hijacking (.so injection)
# Técnica 4: Abuso de Linux Capabilities (cap_setuid)
# ============================================================

RUN apt-get update && apt-get install -y \
    sudo \
    python3 \
    python3-pip \
    gcc \
    libcap2-bin \
    debianutils \
    openssh-server \
    vim \
    nano \
    strace \
    ltrace \
    && rm -rf /var/lib/apt/lists/*

# ============================================================
# CREACIÓN DE USUARIOS
# ============================================================

# user1: punto de entrada (contraseña: starthere)
RUN useradd -m -s /bin/bash user1 && \
    echo "user1:starthere" | chpasswd

# user2: pivote desde user1 (sin login directo)
RUN useradd -m -s /bin/bash user2

# user3: pivote desde user2 (sin login directo)
RUN useradd -m -s /bin/bash user3

# user4: pivote desde user3 (sin login directo)
RUN useradd -m -s /bin/bash user4

# ============================================================
# TÉCNICA 1: Abuso del grupo "adm"
# ============================================================

RUN usermod -aG adm user1

# Crear clave SSH de user2
RUN mkdir -p /home/user2/.ssh && \
    ssh-keygen -t rsa -f /home/user2/.ssh/id_rsa -N "" -q && \
    cp /home/user2/.ssh/id_rsa.pub /home/user2/.ssh/authorized_keys && \
    chown -R user2:user2 /home/user2/.ssh && \
    chmod 700 /home/user2/.ssh && \
    chmod 600 /home/user2/.ssh/id_rsa && \
    chmod 644 /home/user2/.ssh/authorized_keys

# Simular la filtración de la clave en un log del sistema
RUN mkdir -p /var/log/custom_app && \
    echo "ERROR: Fallo al sincronizar credenciales del user2. Dumping private key for debug:" > /var/log/custom_app/sync.log && \
    cat /home/user2/.ssh/id_rsa >> /var/log/custom_app/sync.log && \
    chown root:adm /var/log/custom_app/sync.log && \
    chmod 640 /var/log/custom_app/sync.log

# Pista para user1
RUN echo "Hint: You belong to an interesting group... try 'id'." > /home/user1/README.txt && \
    echo "Administrators often leave sensitive data in log files..." >> /home/user1/README.txt && \
    chown user1:user1 /home/user1/README.txt

# ============================================================
# TÉCNICA 2: Python Library Hijacking
# ============================================================

# Script legítimo que importa un módulo custom (usamos comillas dobles en printf para no chocar con el código)
RUN mkdir -p /opt/scripts && \
    printf "#!/usr/bin/env python3\n\
\"\"\"\n\
Backup utility - runs system health checks\n\
\"\"\"\n\
import sys\n\
# Añadimos la ruta legítima al final. PYTHONPATH se evaluará antes que esto.\n\
sys.path.append('/opt/scripts/lib')\n\
import utils  # <-- Este módulo es secuestrable\n\
import os\n\
\n\
def main():\n\
    print(\"[*] Running backup health check...\")\n\
    result = utils.check_system()\n\
    print(f\"[*] System status: {result}\")\n\
    print(\"[*] Backup completed successfully.\")\n\
\n\
if __name__ == \"__main__\":\n\
    main()\n" > /opt/scripts/backup.py

# Crear el módulo utils legítimo en un directorio del sistema
RUN mkdir -p /opt/scripts/lib && \
    printf "def check_system():\n\
    return \"All systems nominal\"\n" > /opt/scripts/lib/utils.py

# Directorio donde user2 puede escribir (para el PYTHONPATH)
RUN mkdir -p /opt/scripts/modules && \
    chown user2:user2 /opt/scripts/modules

# Hacer el script propiedad de root pero ejecutable
RUN chown root:root /opt/scripts/backup.py && \
    chmod 755 /opt/scripts/backup.py

# Configurar sudoers para permitir SETENV
RUN echo "user2 ALL=(user3) NOPASSWD: SETENV: /usr/bin/python3 /opt/scripts/backup.py" > /etc/sudoers.d/user2 && \
    chmod 440 /etc/sudoers.d/user2

# Pista para user2
RUN echo "Hint: There's a backup script you can run as another user." > /home/user2/README.txt && \
    echo "Check sudo -l and think about how Python finds its modules..." >> /home/user2/README.txt && \
    echo "The PYTHONPATH environment variable might be your friend." >> /home/user2/README.txt && \
    chown user2:user2 /home/user2/README.txt

# ============================================================
# TÉCNICA 3: Shared Library Hijacking (.so injection)
# ============================================================

# Crear el programa vulnerable en C (usamos comillas simples en printf y escapamos los \n internos del C a \\n)
RUN echo '#include <stdio.h>' > /tmp/health_check.c && \
    echo '#include <stdlib.h>' >> /tmp/health_check.c && \
    echo '#include <dlfcn.h>' >> /tmp/health_check.c && \
    echo '' >> /tmp/health_check.c && \
    echo 'int main() {' >> /tmp/health_check.c && \
    echo '    void *handle;' >> /tmp/health_check.c && \
    echo '    void (*run_check)();' >> /tmp/health_check.c && \
    echo '    ' >> /tmp/health_check.c && \
    echo '    printf("[*] Health Check Utility v1.0\\n");' >> /tmp/health_check.c && \
    echo '    printf("[*] Loading monitoring library...\\n");' >> /tmp/health_check.c && \
    echo '    ' >> /tmp/health_check.c && \
    echo '    // Intenta cargar desde /opt/lib/ primero (path hijackeable)' >> /tmp/health_check.c && \
    echo '    handle = dlopen("/opt/lib/libmonitor.so", RTLD_LAZY);' >> /tmp/health_check.c && \
    echo '    ' >> /tmp/health_check.c && \
    echo '    if (!handle) {' >> /tmp/health_check.c && \
    echo '        printf("[-] Custom monitor not found, using defaults.\\n");' >> /tmp/health_check.c && \
    echo '        printf("[*] System OK (default check)\\n");' >> /tmp/health_check.c && \
    echo '        return 0;' >> /tmp/health_check.c && \
    echo '    }' >> /tmp/health_check.c && \
    echo '    ' >> /tmp/health_check.c && \
    echo '    run_check = dlsym(handle, "run_check");' >> /tmp/health_check.c && \
    echo '    if (run_check) {' >> /tmp/health_check.c && \
    echo '        run_check();' >> /tmp/health_check.c && \
    echo '    }' >> /tmp/health_check.c && \
    echo '    ' >> /tmp/health_check.c && \
    echo '    dlclose(handle);' >> /tmp/health_check.c && \
    echo '    return 0;' >> /tmp/health_check.c && \
    echo '}' >> /tmp/health_check.c

# Compilar el binario y configurar SUID para user4
RUN gcc -o /usr/local/bin/health_check /tmp/health_check.c -ldl && \
    chown user4:user4 /usr/local/bin/health_check && \
    chmod 4755 /usr/local/bin/health_check && \
    rm /tmp/health_check.c

# Crear el directorio de libraries escribible por user3
RUN mkdir -p /opt/lib && \
    chown user3:user3 /opt/lib

# Pista para user3
RUN echo "Hint: There's a health_check binary with interesting permissions." > /home/user3/README.txt && \
    echo "Try: ls -la /usr/local/bin/health_check" >> /home/user3/README.txt && \
    echo "What libraries does it try to load? Can you influence that?" >> /home/user3/README.txt && \
    echo "Tools: ldd, strace, ltrace might help you understand what it does." >> /home/user3/README.txt && \
    chown user3:user3 /home/user3/README.txt

# ============================================================
# TÉCNICA 4: Abuso de Linux Capabilities (cap_setuid)
# ============================================================

# Copiar python3 a un path custom y darle cap_setuid
RUN cp /usr/bin/python3 /usr/local/bin/python3-special && \
    setcap cap_setuid+ep /usr/local/bin/python3-special && \
    chown root:user4 /usr/local/bin/python3-special && \
    chmod 750 /usr/local/bin/python3-special

# Pista para user4
RUN echo "Hint: Sometimes binaries have special... capabilities." > /home/user4/README.txt && \
    echo "Try: getcap -r / 2>/dev/null" >> /home/user4/README.txt && \
    echo "What can you do with cap_setuid?" >> /home/user4/README.txt && \
    chown user4:user4 /home/user4/README.txt

# ============================================================
# FLAG DE ROOT
# ============================================================
RUN echo "CONGRATULATIONS! You've completed the full privesc chain!" > /root/flag.txt && \
    echo "" >> /root/flag.txt && \
    echo "Chain completed:" >> /root/flag.txt && \
    echo "  user1 -> [adm group abuse] -> user2" >> /root/flag.txt && \
    echo "  user2 -> [python library hijacking] -> user3" >> /root/flag.txt && \
    echo "  user3 -> [shared library hijacking] -> user4" >> /root/flag.txt && \
    echo "  user4 -> [capabilities abuse] -> root" >> /root/flag.txt && \
    echo "" >> /root/flag.txt && \
    echo "Don't forget to subscribe! :)" >> /root/flag.txt && \
    chmod 400 /root/flag.txt

# ============================================================
# CONFIGURACIÓN SSH Y ENTRYPOINT
# ============================================================
RUN mkdir -p /run/sshd

# Script de inicio
RUN printf '#!/bin/bash\n\
echo "==========================================="\n\
echo "  PRIVESC LAB - Escalada de Privilegios"\n\
echo "==========================================="\n\
echo ""\n\
echo "  Objetivo: user1 -> user2 -> user3 -> user4 -> root"\n\
echo ""\n\
echo "  Credenciales iniciales:"\n\
echo "    Usuario: user1"\n\
echo "    Password: starthere"\n\
echo ""\n\
echo "  Técnicas involucradas:"\n\
echo "    1. Abuso de grupos especiales (adm)"\n\
echo "    2. Python Library Hijacking"\n\
echo "    3. Shared Library Hijacking"\n\
echo "    4. Linux Capabilities Abuse"\n\
echo ""\n\
echo "  Cada usuario tiene un README.txt con una pista."\n\
echo "  ¡Buena suerte!"\n\
echo "==========================================="\n\
echo ""\n\
\n\
# Generar las llaves del host para que SSHD no crashee\n\
ssh-keygen -A\n\
\n\
# Iniciar SSH en background\n\
/usr/sbin/sshd\n\
\n\
# Entrar como user1 usando exec para reemplazar el shell\n\
exec su - user1\n' > /entrypoint.sh

RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]