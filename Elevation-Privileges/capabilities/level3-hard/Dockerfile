# ============================================================
# Capabilities Lab - Nivel 3: Dificil
# Tecnicas: cap_sys_admin (mount) + cap_fowner + cadena de explotacion
# Objetivo: Escalar de usuario "hacker" a root
# ============================================================
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    sudo vim nano python3 gcc libcap2-bin \
    coreutils findutils strace ltrace file \
    cron e2fsprogs mount \
    && rm -rf /var/lib/apt/lists/*

# Crear usuario sin privilegios
RUN useradd -m -s /bin/bash hacker && \
    echo "hacker:hacker123" | chpasswd

# ===============================================================
# VECTOR REAL: Cadena cap_fowner -> modificar /etc/passwd
# ===============================================================
# cap_fowner permite cambiar el dueño/permisos de CUALQUIER archivo
# Cadena: cambiar permisos de /etc/passwd -> agregar usuario root

# Binario custom que usa cap_fowner
RUN printf '#include <stdio.h>\n\
#include <sys/stat.h>\n\
#include <errno.h>\n\
#include <string.h>\n\
\n\
// Herramienta interna para fijar permisos de logs\n\
// Solo deberia usarse para /var/log/*\n\
int main(int argc, char *argv[]) {\n\
    if (argc != 3) {\n\
        printf("logfixer - Fix log file permissions\\n");\n\
        printf("Usage: %%s <file> <mode_octal>\\n", argv[0]);\n\
        return 1;\n\
    }\n\
\n\
    unsigned int mode;\n\
    sscanf(argv[2], "%%o", &mode);\n\
\n\
    // NOTA: No valida que el archivo sea de /var/log/\n\
    // El dev confio en que "solo admins lo usarian"\n\
    if (chmod(argv[1], mode) != 0) {\n\
        printf("Error: %%s\\n", strerror(errno));\n\
        return 1;\n\
    }\n\
\n\
    printf("Permissions of %%s set to %%04o\\n", argv[1], mode);\n\
    return 0;\n\
}\n' > /tmp/logfixer.c && \
    gcc /tmp/logfixer.c -o /usr/local/bin/logfixer && \
    setcap cap_fowner+ep /usr/local/bin/logfixer && \
    rm /tmp/logfixer.c

# ===============================================================
# RABBIT HOLE 1: cap_sys_ptrace en strace (parece peligroso)
# ===============================================================
# cap_sys_ptrace permite trazar procesos pero no escala directamente
# sin un proceso root con datos sensibles en memoria (no hay ninguno)
RUN setcap cap_sys_ptrace+ep /usr/bin/strace

# ===============================================================
# RABBIT HOLE 2: cap_net_bind_service en un binario custom
# ===============================================================
# Permite bindear puertos <1024 - suena peligroso pero no escala
RUN printf '#include <stdio.h>\n\
#include <string.h>\n\
#include <sys/socket.h>\n\
#include <arpa/inet.h>\n\
#include <unistd.h>\n\
\n\
int main(int argc, char *argv[]) {\n\
    if (argc != 2) {\n\
        printf("webserv - Simple HTTP listener\\n");\n\
        printf("Usage: %%s <port>\\n", argv[0]);\n\
        return 1;\n\
    }\n\
    int port = atoi(argv[1]);\n\
    int sock = socket(AF_INET, SOCK_STREAM, 0);\n\
    struct sockaddr_in addr;\n\
    addr.sin_family = AF_INET;\n\
    addr.sin_port = htons(port);\n\
    addr.sin_addr.s_addr = INADDR_ANY;\n\
    if (bind(sock, (struct sockaddr*)&addr, sizeof(addr)) < 0) {\n\
        perror("bind failed");\n\
        return 1;\n\
    }\n\
    printf("Listening on port %%d...\\n", port);\n\
    listen(sock, 1);\n\
    printf("Use Ctrl+C to stop\\n");\n\
    while(1) {\n\
        int client = accept(sock, NULL, NULL);\n\
        char *resp = "HTTP/1.1 200 OK\\r\\nContent-Length: 2\\r\\n\\r\\nOK";\n\
        write(client, resp, strlen(resp));\n\
        close(client);\n\
    }\n\
    return 0;\n\
}\n' > /tmp/webserv.c && \
    gcc /tmp/webserv.c -o /usr/local/bin/webserv && \
    setcap cap_net_bind_service+ep /usr/local/bin/webserv && \
    rm /tmp/webserv.c

# ===============================================================
# PISTAS
# ===============================================================
# Nota del dev que creo logfixer
RUN mkdir -p /opt/tools && \
    printf "logfixer v1.0\n\
-----------\n\
Tool to fix broken log permissions after rotation.\n\
Uses CAP_FOWNER to chmod files regardless of ownership.\n\
\n\
NOTE: Should probably add path validation at some point.\n\
For now, only sysadmins have access so it is fine.\n" > /opt/tools/README.txt && \
    chmod 644 /opt/tools/README.txt

# Algunos logs para que logfixer parezca tener uso real
RUN touch /var/log/app.log /var/log/app-error.log && \
    chmod 600 /var/log/app.log /var/log/app-error.log

# Flag
RUN echo "FLAG{nivel3_cap_fowner_chain_privesc}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Instrucciones
RUN printf '#!/bin/bash\n\
echo "========================================="\n\
echo " Capabilities Lab - Nivel 3: Dificil"\n\
echo "========================================="\n\
echo ""\n\
echo "Objetivo: Leer /root/flag.txt"\n\
echo ""\n\
echo "Este sistema tiene varias capabilities asignadas."\n\
echo "La mayoria son distracciones."\n\
echo ""\n\
echo "Preguntas clave:"\n\
echo "  - Que capability permite MODIFICAR archivos del sistema?"\n\
echo "  - Que archivos criticos podrias alterar para escalar?"\n\
echo "  - Que pasa si cambias los permisos de /etc/passwd?"\n\
echo ""\n\
echo "Enumeracion:"\n\
echo "  getcap -r / 2>/dev/null"\n\
echo "  ls -la /etc/passwd /etc/shadow"\n\
echo "  cat /opt/tools/README.txt"\n\
echo ""\n' > /home/hacker/instrucciones.sh && \
    chmod +x /home/hacker/instrucciones.sh && \
    chown hacker:hacker /home/hacker/instrucciones.sh

USER hacker
WORKDIR /home/hacker
CMD ["/bin/bash"]