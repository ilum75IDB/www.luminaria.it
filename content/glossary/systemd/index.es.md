---
title: "systemd"
description: "Sistema de inicio y gestor de servicios en Linux, usado para gestionar múltiples instancias MySQL/MariaDB en el mismo servidor mediante unit files separados."
translationKey: "glossary_systemd"
articles:
  - "/posts/mysql/mysql-multi-istanza-secure-file-priv"
---

**systemd** es el sistema de inicio y gestor de servicios predeterminado en las distribuciones Linux modernas (CentOS/RHEL 7+, Ubuntu 16.04+, Debian 8+). En el ámbito de bases de datos, es el mecanismo que arranca, detiene y monitoriza las instancias MySQL o MariaDB.

## Cómo funciona

Cada servicio está definido por un archivo unit (ej. `mysqld.service`) que especifica el comando de arranque, el archivo de configuración, las dependencias y el comportamiento en caso de crash. En un setup multi-instancia, se crean unit files separados para cada instancia (ej. `mysqld-app2.service`, `mysqld-reporting.service`), cada uno con su propio `--defaults-file` apuntando a un `my.cnf` diferente.

## Para qué sirve

systemd permite gestionar las instancias MySQL como servicios independientes: arrancarlas, detenerlas, reiniciarlas y monitorizarlas por separado. El comando `systemctl cat <servicio>` es fundamental para rastrear desde el nombre del servicio hasta el archivo de configuración de la instancia, y de ahí al puerto, socket y datadir.

## Cuándo se usa

systemd está activo automáticamente en cualquier servidor Linux moderno. En el trabajo de DBA, se interactúa con él mediante `systemctl start/stop/status/restart <servicio>`. En entornos multi-instancia, `systemctl list-units --type=service | grep mysql` es el primer comando para identificar cuántas instancias están activas en un servidor.
