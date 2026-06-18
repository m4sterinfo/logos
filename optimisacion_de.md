# XUI Database Optimization & Monitoring Implementation

## Fecha

18/06/2026

## Servidor

-   CPU: AMD EPYC 7413
-   Threads: 96
-   RAM: 125 GB
-   MariaDB: 10.3.39
-   Sistema: Ubuntu Server
-   Aplicación: XUI

------------------------------------------------------------------------

# 1. Diagnóstico inicial

Se detectó durante eventos de alta demanda:

-   Problemas de login de usuarios.
-   MariaDB consumiendo CPU.
-   Muchas conexiones abiertas.
-   Poca ejecución paralela real.

Estado inicial:

    max_connections = 8192
    Threads_connected = 1423
    Threads_running = 1
    Sleep = 1425
    Query = 3

Conclusión:

El problema no era falta de CPU. La mayoría de conexiones estaban
inactivas durante horas.

------------------------------------------------------------------------

# 2. Configuración inicial detectada

MariaDB cargaba configuración desde:

    /etc/mysql/my.cnf
    /etc/mysql/mariadb.cnf
    /etc/mysql/mariadb.conf.d/

Verificación:

    mysqld --print-defaults

Valores iniciales:

    max_connections=8192
    innodb_buffer_pool_size=10G
    thread_pool_size=12
    thread_handling=pool-of-threads

------------------------------------------------------------------------

# 3. Backups realizados

Archivos encontrados:

    /etc/mysql/mariadb.conf.d/50-server.cnf.save
    /etc/mysql/mariadb.conf.d/50-server.cnf.backup_antes_optimizacion

Propósito:

Conservar configuración anterior antes de optimizar.

------------------------------------------------------------------------

# 4. Archivo modificado

Archivo activo:

    /etc/mysql/my.cnf

------------------------------------------------------------------------

# 5. Optimizaciones aplicadas

## Conexiones

Antes:

    max_connections=8192

Después:

    max_connections=15000

Motivo:

Mayor capacidad durante picos de usuarios.

------------------------------------------------------------------------

## Backlog

Antes:

    back_log=4096

Después:

    back_log=5000

Motivo:

Aceptar más conexiones durante ráfagas.

------------------------------------------------------------------------

## Buffer InnoDB

Antes:

    innodb_buffer_pool_size=10G

Después:

    innodb_buffer_pool_size=64G
    innodb_buffer_pool_instances=16

Motivo:

Aprovechar la RAM disponible.

------------------------------------------------------------------------

## Threads InnoDB

Configurado:

    innodb_read_io_threads=64
    innodb_write_io_threads=64

------------------------------------------------------------------------

## Thread Pool MariaDB

Antes:

    thread_pool_size=12

Después:

    thread_pool_size=96
    thread_handling=pool-of-threads
    thread_pool_max_threads=1024
    thread_pool_idle_timeout=20

------------------------------------------------------------------------

## Cache de threads

Configurado:

    thread_cache_size=2048

------------------------------------------------------------------------

## Timeouts

Configurado:

    wait_timeout=300
    interactive_timeout=300

Motivo:

Evitar miles de conexiones dormidas durante horas.

------------------------------------------------------------------------

# 6. Reinicio aplicado

    systemctl restart mariadb

Validación:

    max_connections = 15000
    innodb_buffer_pool_size = 64G
    thread_pool_size = 96
    back_log = 5000

------------------------------------------------------------------------

# 7. Estado posterior

RAM:

    Total: 125GB
    Usada: 18GB
    Disponible: 103GB

MariaDB:

    Threads_connected: 180
    Threads_running: 1
    Max_used_connections: 239

------------------------------------------------------------------------

# 8. XUI Database Watcher

Ruta:

    /opt/xui_db_watcher/

Estructura:

    /opt/xui_db_watcher/

    watcher.py
    config.py
    venv/

------------------------------------------------------------------------

# 9. watcher.py

Archivo:

    /opt/xui_db_watcher/watcher.py

Funciones:

-   Monitor CPU total.
-   Monitor CPU de MariaDB.
-   Monitor RAM.
-   Monitor Load Average.
-   Monitor conexiones MariaDB.
-   Detecta cuellos de botella.
-   Envía alertas Telegram.
-   Guarda logs.

------------------------------------------------------------------------

# 10. config.py

Archivo:

    /opt/xui_db_watcher/config.py

Contiene:

-   Token Telegram.
-   Chat ID.
-   Intervalos.
-   Límites de alerta.

------------------------------------------------------------------------

# 11. Bot Telegram

Comandos:

Activar modo evento:

    evento

Finalizar evento:

    fin_evento

Estado manual:

    status

------------------------------------------------------------------------

# 12. Servicio Systemd

Archivo creado:

    /etc/systemd/system/xui-db-watcher.service

Ejecuta:

    /opt/xui_db_watcher/venv/bin/python
    /opt/xui_db_watcher/watcher.py

Características:

-   Inicio automático.
-   Reinicio automático.
-   No depende de SSH.

------------------------------------------------------------------------

# 13. Logs

Archivos:

    /var/log/xui-db-watcher.log
    /var/log/xui-db-watcher-error.log

------------------------------------------------------------------------

# 14. Administración

Estado:

    systemctl status xui-db-watcher

Reiniciar:

    systemctl restart xui-db-watcher

Logs:

    journalctl -u xui-db-watcher -f

------------------------------------------------------------------------

# Objetivo final

Recolectar datos reales durante eventos masivos para determinar:

-   Si MariaDB llega al límite.
-   Si max_connections es suficiente.
-   Si Threads_running aumenta.
-   Si CPU es cuello de botella.
-   Si XUI/PHP necesita optimización adicional.

La próxima optimización se realizará basada en métricas reales.
