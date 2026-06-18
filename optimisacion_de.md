XUI Database Optimization & Monitoring Implementation
Fecha

18/06/2026

Servidor

Sistema optimizado:

CPU: AMD EPYC 7413
Sockets: 2
Cores físicos: 48
Threads disponibles: 96
RAM: 125 GB
Sistema: Ubuntu Server
Base de datos: MariaDB 10.3.39
Aplicación: XUI
1. Diagnóstico inicial
Problema detectado

Durante eventos de alta demanda se observaba:

Usuarios con problemas para iniciar sesión.
MariaDB consumiendo CPU.
Procesos mysqld utilizando varios cores pero sin aprovechar completamente el servidor.
Gran cantidad de conexiones abiertas.

Análisis inicial:

CPU:
96 threads

MariaDB:

max_connections:
8192

Threads_connected:
1423

Threads_running:
1

Processlist:

Sleep:
1425

Query:
3
Conclusión inicial

El problema no era falta de CPU.

El servidor tenía capacidad disponible pero MariaDB mantenía demasiadas conexiones inactivas.

La mayoría de conexiones estaban:

Sleep

debido a:

wait_timeout = 28800
interactive_timeout = 28800

equivalente a 8 horas.

2. Verificación de configuración real usada por MariaDB

Se verificó que XUI no estaba usando únicamente:

/etc/mysql/mariadb.conf.d/50-server.cnf

sino también:

/etc/mysql/my.cnf
/etc/mysql/mariadb.cnf

Comando utilizado:

mysqld --print-defaults

Resultado inicial:

max_connections=8192

innodb_buffer_pool_size=10G

thread_pool_size=12

thread_handling=pool-of-threads
3. Backup realizado

Antes de modificaciones se conservó:

Archivo original
/etc/mysql/mariadb.conf.d/50-server.cnf

Backup:

/etc/mysql/mariadb.conf.d/50-server.cnf.save

También existía:

/etc/mysql/mariadb.conf.d/50-server.cnf.backup_antes_optimizacion

Este contenía los ajustes previos de timeout:

wait_timeout = 300
interactive_timeout = 300
4. Archivo modificado principal

Archivo activo:

/etc/mysql/my.cnf

Este fue el archivo que realmente cargaba MariaDB.

Confirmación:

grep -n "max_connections\|innodb_buffer_pool_size\|thread_pool_size" /etc/mysql/my.cnf

Resultado:

max_connections = 15000

innodb_buffer_pool_size = 64G

thread_pool_size = 96
5. Optimizaciones MariaDB aplicadas
5.1 Aumento de conexiones

Antes:

max_connections = 8192

Después:

max_connections = 15000

Motivo:

Permitir mayor cantidad de usuarios concurrentes durante eventos masivos.

5.2 Backlog aumentado

Antes:

back_log = 4096

Después:

back_log = 5000

Motivo:

Permitir más conexiones esperando durante picos repentinos.

5.3 Buffer Pool InnoDB

Antes:

innodb_buffer_pool_size = 10G

Después:

innodb_buffer_pool_size = 64G

Motivo:

El servidor dispone de 125 GB RAM.

Se aumentó memoria disponible para cachear:

tablas
índices
páginas InnoDB
5.4 Instancias del buffer pool

Nuevo:

innodb_buffer_pool_instances = 16

Motivo:

Reducir contención interna con múltiples hilos.

5.5 Threads de lectura/escritura InnoDB

Configurado:

innodb_read_io_threads = 64

innodb_write_io_threads = 64

Motivo:

Aprovechar almacenamiento rápido y mayor capacidad paralela.

5.6 Thread Pool MariaDB

Antes:

thread_pool_size = 12

Después:

thread_pool_size = 96

Manteniendo:

thread_handling = pool-of-threads

Motivo:

El servidor dispone de 96 threads.

MariaDB puede manejar mejor miles de conexiones.

5.7 Cache de threads

Nuevo:

thread_cache_size = 2048

Motivo:

Reducir creación/destrucción constante de threads.

5.8 Timeout de conexiones

Configurado:

wait_timeout = 300

interactive_timeout = 300

Motivo:

Evitar miles de conexiones abandonadas durante horas.

Equivale a:

5 minutos

6. Parámetros mantenidos

Se mantuvieron:

innodb_flush_log_at_trx_commit = 0

innodb_flush_method = O_DIRECT

innodb_io_capacity = 20000

performance_schema = 0

Motivo:

Configuración orientada a alto rendimiento.

7. Reinicio aplicado

Servicio reiniciado:

systemctl restart mariadb
8. Validación posterior
Conexiones máximas

Comando:

mysql -e "SHOW VARIABLES LIKE 'max_connections';"

Resultado:

15000
Buffer Pool

Comando:

mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"

Resultado:

68719476736 bytes

64GB
Thread Pool

Resultado:

thread_pool_size = 96
Backlog

Resultado:

back_log = 5000
Estado actual

Memoria:

RAM total:
125GB

Usada:
18GB

Disponible:
103GB

MariaDB:

Threads_connected:
180

Threads_running:
1

Max_used_connections:
239

Conclusión:

La base de datos tiene capacidad suficiente y todavía mucho margen.

9. Implementación XUI Database Watcher

Se creó un sistema propio de monitoreo.

Ruta:

/opt/xui_db_watcher/

Estructura:

/opt/xui_db_watcher/

├── watcher.py
├── config.py
└── venv/
10. Python Virtual Environment

Motivo:

No modificar Python del sistema.

Versiones:

Sistema:

Python 2.7.18

Disponible:

Python 3.8.10

Se utilizó:

/opt/xui_db_watcher/venv

para aislar dependencias.

11. Archivo watcher.py

Ruta:

/opt/xui_db_watcher/watcher.py

Función:

Monitoriza:

CPU total
CPU de MariaDB
RAM
Load average
conexiones MariaDB
queries activas
máximo histórico de conexiones

También:

Guarda logs locales.
Envía reportes Telegram.
Tiene modo evento.
12. Archivo config.py

Ruta:

/opt/xui_db_watcher/config.py

Contiene:

Token Telegram
Chat ID
Intervalos normales
Intervalos evento
Límites de alerta
13. Funciones Telegram

Comandos soportados:

Activar evento

Enviar:

evento

Resultado:

Activa modo monitoreo intensivo.

Finalizar evento

Enviar:

fin_evento

Resultado:

Vuelve a modo normal.

Estado manual

Enviar:

status

Respuesta:

Reporte inmediato:

CPU
MySQL CPU
RAM
LOAD
Connections
Running
Max Used
Evento
14. Servicio Systemd creado

Archivo:

/etc/systemd/system/xui-db-watcher.service

Contenido:

Ejecuta:

/opt/xui_db_watcher/venv/bin/python

/opt/xui_db_watcher/watcher.py

Características:

Inicio automático.
Reinicio automático.
Independiente del SSH.
15. Logs del watcher

Archivos:

/var/log/xui-db-watcher.log

y

/var/log/xui-db-watcher-error.log

Ver logs:

tail -f /var/log/xui-db-watcher.log

Errores:

tail -f /var/log/xui-db-watcher-error.log
16. Comandos administración
Estado
systemctl status xui-db-watcher
Reiniciar
systemctl restart xui-db-watcher
Parar
systemctl stop xui-db-watcher
Inicio automático
systemctl enable xui-db-watcher
17. Objetivo final

Con esta implementación ahora se puede medir durante eventos:

si realmente MariaDB llega al límite
si max_connections es suficiente
si aumenta Threads_running
si la CPU es problema real
si el cuello de botella está en PHP/XUI
si hace falta una nueva optimización

La próxima optimización estará basada en datos reales y no en estimaciones.
