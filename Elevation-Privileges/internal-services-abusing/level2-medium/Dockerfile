# ============================================================
# LAB 2 - "Database Nightmare" (Intermedio)
# ============================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y \
    apache2 \
    libapache2-mod-php \
    php-mysql \
    mysql-server \
    sudo \
    curl \
    nano \
    vim \
    net-tools \
    procps \
    gcc \
    libmysqlclient-dev \
    passwd \
    && rm -rf /var/lib/apt/lists/*

# Fix para problemas de MySQL en Docker con Ubuntu 22.04
RUN usermod -d /var/lib/mysql/ mysql

RUN mkdir -p /var/run/mysqld && \
    chown root:root /var/run/mysqld

# Configuramos secure-file-priv vacío (permite UDF y permite el vector de Cron)
RUN echo '[mysqld]\n\
user=root\n\
datadir=/var/lib/mysql\n\
socket=/var/run/mysqld/mysqld.sock\n\
pid-file=/var/run/mysqld/mysqld.pid\n\
secure-file-priv=""\n\
local-infile=1\n\
\n\
[client]\n\
socket=/var/run/mysqld/mysqld.sock' > /etc/mysql/my.cnf

# Inicializar MySQL y crear entorno
RUN service mysql start && \
    sleep 2 && \
    mysql -e "CREATE DATABASE webapp;" && \
    mysql -e "CREATE USER 'webapp_user'@'localhost' IDENTIFIED BY 'S3cur3_But_N0t_3n0ugh!';" && \
    mysql -e "GRANT ALL PRIVILEGES ON *.* TO 'webapp_user'@'localhost' WITH GRANT OPTION;" && \
    mysql -e "FLUSH PRIVILEGES;" && \
    mysql -e "USE webapp; CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, username VARCHAR(50), password VARCHAR(100), role VARCHAR(20));" && \
    mysql -e "USE webapp; INSERT INTO users VALUES (1, 'admin', 'admin_p4ssw0rd_2024', 'admin'), (2, 'guest', 'guest123', 'user');" && \
    mysql -e "USE webapp; CREATE TABLE secrets (id INT, data TEXT);" && \
    mysql -e "USE webapp; INSERT INTO secrets VALUES (1, 'Pista: MySQL corre como root. Investiga el directorio de plugins y User Defined Functions (UDF)');"

# ---------- Aplicación web ----------
RUN echo '<?php\n\
$db_host = "localhost";\n\
$db_user = "webapp_user";\n\
$db_pass = "S3cur3_But_N0t_3n0ugh!";\n\
$db_name = "webapp";\n\
?>' > /var/www/html/config.php && \
    chmod 644 /var/www/html/config.php

RUN echo '<?php\n\
include("config.php");\n\
$conn = new mysqli($db_host, $db_user, $db_pass, $db_name);\n\
?>\n\
<!DOCTYPE html>\n\
<html><head><title>Internal App</title></head>\n\
<body>\n\
<h1>Employee Management System</h1>\n\
<p>Internal use only. Version 3.2.1</p>\n\
</body></html>' > /var/www/html/index.php

RUN rm -f /var/www/html/index.html

# ---------- Crear usuario de práctica y proporcionar el Exploit en C ----------
RUN useradd -m -s /bin/bash student && \
    echo 'student:student123' | chpasswd && \
    usermod -aG www-data student

# Dejamos el código C de la UDF listo para ser compilado
RUN echo '#include <stdlib.h>\n\
#include <string.h>\n\
\n\
enum Item_result {STRING_RESULT, REAL_RESULT, INT_RESULT, ROW_RESULT, DECIMAL_RESULT};\n\
typedef struct st_udf_args {\n\
    unsigned int arg_count;\n\
    enum Item_result *arg_type;\n\
    char **args;\n\
    unsigned long *lengths;\n\
    char *maybe_null;\n\
} UDF_ARGS;\n\
\n\
typedef struct st_udf_init {\n\
    char maybe_null;\n\
    unsigned int decimals;\n\
    unsigned long max_length;\n\
    char *ptr;\n\
    char const_item;\n\
} UDF_INIT;\n\
\n\
char sys_exec_init(UDF_INIT *initid, UDF_ARGS *args, char *message) {\n\
    return 0;\n\
}\n\
\n\
int sys_exec(UDF_INIT *initid, UDF_ARGS *args, char *is_null, char *error) {\n\
    if (args->arg_count == 1) system(args->args[0]);\n\
    return 0;\n\
}' > /home/student/sys_exec.c && \
    chown student:student /home/student/sys_exec.c

RUN chmod 755 /var/www/html

RUN echo "Nota: Revisar permisos de config.php..." > /var/www/html/.maintenance_notes

# ---------- ANTI-BYPASS: Flag compilada en C ----------
# Compilamos la flag en un ejecutable para evitar que la lean con SELECT LOAD_FILE()
RUN echo '#include <stdio.h>\n\
int main() {\n\
    printf("FLAG{lab2_mysql_udf_root_pwned}\\n");\n\
    return 0;\n\
}' > /tmp/flag.c && \
    gcc /tmp/flag.c -o /root/get_flag && \
    rm /tmp/flag.c && \
    chmod 700 /root/get_flag

# ---------- Entrypoint ----------
RUN echo '#!/bin/bash\n\
service mysql start\n\
service apache2 start\n\
echo ""\n\
echo "================================================"\n\
echo "  LAB 2 - Database Nightmare (Intermedio)"\n\
echo "================================================"\n\
echo "  Usuario: student / student123"\n\
echo "  Objetivo: Ejecutar /root/get_flag"\n\
echo "================================================"\n\
echo ""\n\
exec su - student' > /entrypoint.sh && \
    chmod +x /entrypoint.sh

EXPOSE 80 3306

ENTRYPOINT ["/entrypoint.sh"]