# ============================================================
# LAB 1 - GRUPO DOCKER (Dificultad: Fácil)
# ============================================================
# Escenario: El usuario "devuser" pertenece al grupo "docker".
# Un socket de Docker está montado, permitiendo escalar
# privilegios creando un contenedor privilegiado.
# ============================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Instalar dependencias básicas
RUN apt-get update && apt-get install -y \
    sudo \
    curl \
    nano \
    vim \
    bash \
    net-tools \
    openssh-server \
    && rm -rf /var/lib/apt/lists/*

# Instalar Docker CLI (solo el cliente, no el daemon)
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg && \
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu jammy stable" > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# Crear grupo docker y usuario vulnerable
RUN groupadd -f docker && \
    useradd -m -s /bin/bash -G docker devuser && \
    echo "devuser:devuser123" | chpasswd

# Crear la flag de root
RUN echo "FLAG{lab1_docker_group_privesc_success}" > /root/flag.txt && \
    chmod 600 /root/flag.txt

# Crear pista para el usuario
RUN mkdir -p /home/devuser/notas && \
    cat <<'EOF' > /home/devuser/notas/recordatorio.txt
Nota para mí mismo:
- Me agregaron al grupo "docker" para poder gestionar los contenedores de desarrollo.
- El socket de Docker del host está montado en /var/run/docker.sock
- Tengo que recordar pedirle al admin que revise los permisos...
EOF

RUN chown -R devuser:devuser /home/devuser/notas

# Banner de bienvenida
RUN cat <<'MOTD' > /etc/motd

╔══════════════════════════════════════════════════════════╗
║           LAB 1 - ABUSO DEL GRUPO DOCKER                ║
║                  Dificultad: FÁCIL                       ║
║                                                          ║
║  Objetivo: Leer /root/flag.txt                           ║
║  Usuario:  devuser / devuser123                          ║
║                                                          ║
║  Pista: Revisa a qué grupos perteneces con "id"          ║
╚══════════════════════════════════════════════════════════╝

MOTD

USER devuser
WORKDIR /home/devuser

CMD ["/bin/bash", "-l"]