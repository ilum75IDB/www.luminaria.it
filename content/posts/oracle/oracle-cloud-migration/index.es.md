---
title: "Oracle de on-premises a cloud: estrategia, planificación y cutover"
description: "Un Oracle 19c Enterprise con RAC, Data Guard y 2 TB de datos. Tres meses para migrar todo a OCI sin perder una transacción. Desde el assessment de licencias hasta el cutover nocturno, la crónica de una migración real."
date: "2026-04-28T10:00:00+01:00"
draft: false
translationKey: "oracle_cloud_migration"
tags: ["migration", "cloud", "oci", "data-guard", "architecture", "licensing"]
categories: ["oracle"]
image: "oracle-cloud-migration.cover.jpg"
---

La semana pasada un colega me escribió: "Tengo que llevar Oracle a la nube, ¿cuánto se tarda?" Le respondí con una pregunta: "¿Sabes cuántas features de la Enterprise Edition estás usando?" Silencio.

Es la misma escena cada vez. Alguien arriba decide que hay que ir a la nube — porque el contrato del hosting vence, porque el CFO leyó un informe de Gartner, porque el CTO nuevo quiere modernizar. Y lo primero que sale es: hagamos un lift-and-shift, cogemos lo que hay y lo movemos. Tres meses, presupuesto aprobado, adelante.

El problema es que Oracle no es una aplicación que empaquetas en un container y envías. Es un ecosistema de licencias, dependencias, configuraciones de kernel, conexiones de red que atraviesan firewalls y VPNs. Moverlo sin entenderlo antes significa acabar en la nube con los mismos problemas de siempre — más algunos nuevos.

## El cliente y el contexto

El proyecto era para una empresa manufacturera del norte de Italia, de esas que facturan bastante pero tienen un IT reducido — cuatro personas para todo, desde el gestional hasta los ERP. Su Oracle 19c Enterprise Edition corría sobre dos nodos RAC con Data Guard hacia un sitio secundario a veinte kilómetros. Dos terabytes de datos, unos doscientos usuarios concurrentes en horas punta, y un batch nocturno que alimentaba el data warehouse.

El proveedor de hosting había comunicado que el contrato vencía en seis meses y la renovación incluía un aumento del 40%. La dirección había decidido: migramos a la nube. Tres meses de plazo, el proyecto debía cerrarse antes del vencimiento contractual.

Cuando llegué, el plan ya estaba escrito: lift-and-shift a AWS. El integrador de turno había propuesto un par de instancias EC2, algo de almacenamiento EBS y listo. En la hoja Excel todo parecía sencillo. Dos filas: coste actual, coste futuro. El coste futuro era más bajo. Todos contentos.

## Por qué dije no a AWS

Lo primero que hice fue pedir el informe de licencias. Oracle 19c Enterprise Edition con las opciones RAC, Data Guard, Partitioning y Advanced Compression. Sobre hardware on-premises con un contrato de soporte directo con Oracle.

Aquí es donde la cosa se complica. Oracle tiene una política de licenciamiento para la nube que no es exactamente intuitiva. En AWS, cada vCPU cuenta como medio procesador a efectos de licenciamiento. Dos nodos RAC en EC2 con, digamos, 8 vCPU cada uno significan 8 processor licenses. Con Enterprise Edition más las opciones activas, la factura de licencias se dispara. Y Oracle, cuando audita — y lo hace — no mira lo que dice el contrato del proveedor cloud. Mira lo que está corriendo en los servidores.

En OCI — Oracle Cloud Infrastructure — la situación es diferente. Oracle reconoce sus propias OCPU con una relación 1:1, y sobre todo ofrece el programa BYOL (Bring Your Own License) que permite reutilizar las licencias on-premises existentes. El cliente ya había pagado esas licencias. Moverlas a OCI no costaba nada extra. Moverlas a AWS significaba recomprarlas o arriesgarse a una auditoría.

Preparé una hoja de comparación con tres escenarios: AWS con licencias nuevas, AWS con el riesgo de auditoría, OCI con BYOL. Los números hablaban solos. La dirección cambió de opinión en media hora.

## El assessment: dos semanas que valen seis

Antes de tocar nada, pedí dos semanas para un assessment completo. No es una fase que se pueda saltar. Lo aprendí por las malas en un proyecto anterior donde descubrimos a mitad de la migración que la base de datos usaba Advanced Queuing con procedimientos PL/SQL que dependían de una dirección IP hardcoded. Dos días de parada por algo que se descubría en cinco minutos con un grep.

El assessment cubrió cuatro áreas.

**Features en uso.** Ejecuté el Database Feature Usage Report de Oracle (`DBMS_FEATURE_USAGE_INTERNAL`) para entender qué opciones de la Enterprise Edition estaban realmente activas. El RAC era obvio, el Data Guard también. Pero el Partitioning solo se usaba en tres tablas, y el Advanced Compression lo había activado un consultor años antes y nadie sabía si aún servía. Verifiqué: las tablas comprimidas estaban todas en el archivo histórico, material que se leía una vez al año para los auditores. El Advanced Compression se podía desactivar sin impacto.

**Dependencias externas.** La base de datos recibía datos de cuatro sistemas fuente mediante DB links, dos de los cuales apuntaban a bases de datos MySQL en servidores del mismo data center. También había llamadas HTTP salientes desde procedimientos PL/SQL hacia una API REST interna. Todo esto tenía que seguir funcionando después de la migración, lo que significaba VPN site-to-site o FastConnect entre OCI y el data center on-premises.

**Red y latencia.** Medí la latencia entre el data center y la región OCI más cercana (Frankfurt). Con un test sostenido de tnsping, el round-trip era de 12 milisegundos. Aceptable para las queries interactivas, pero el batch nocturno hacía un join masivo vía DB link con un MySQL remoto — y ahí 12 milisegundos multiplicados por millones de filas significaban horas extra. La solución fue sencilla: replicar los datos del MySQL en una staging table en Oracle antes de lanzar el batch. Un paso más, pero el batch pasó de seis horas a dos.

**Sizing.** Analicé los informes AWR de las últimas cuatro semanas para entender el perfil de carga real. El pico de CPU era del 35% en los dos nodos RAC, la memoria utilizada nunca superaba los 48 GB. En OCI dimensioné dos VM.Standard.E4.Flex con 16 OCPU y 256 GB de RAM cada una para el RAC, más una tercera para el standby de Data Guard. Almacenamiento en Block Volumes con tier de rendimiento balanceado — 60 IOPS por GB, suficiente para el perfil de I/O medido.

## La estrategia de migración: Data Guard, no Data Pump

Cuando se habla de migración Oracle, las opciones principales son tres: Data Pump (export/import lógico), Zero Downtime Migration (ZDM), y Data Guard.

Data Pump estaba descartado. Dos terabytes de datos con export lógico significan horas de export, horas de transferencia, horas de import. Y durante todo ese tiempo la base de datos origen tiene que estar parada, o acabas con datos inconsistentes. Para una empresa manufacturera que trabaja en tres turnos, parar la base de datos un día entero no era una opción.

ZDM es la herramienta que Oracle propone para las migraciones a OCI. Funciona, pero añade una capa de automatización sobre Data Guard y Data Pump. En una infraestructura con RAC y configuraciones no estándar — como los DB links cross-engine — prefiero tener el control directo.

La estrategia fue: configurar Data Guard entre el RAC on-premises y una instancia standby en OCI, dejar que se sincronice, y luego hacer el cutover con un switchover controlado. Downtime previsto: menos de una hora. Downtime real: cuarenta y dos minutos.

### La configuración de Data Guard cross-site

La parte complicada no fue el Data Guard en sí — eso lo configuramos cada semana. La parte complicada fue hacerlo funcionar a través de la red. Data Guard necesita un canal de redo transport entre primary y standby, y ese canal tiene que ser fiable y con latencia predecible.

Configuré un túnel VPN site-to-site entre el data center y OCI, con un ancho de banda dedicado de 500 Mbps. El redo generate rate medio era de 15 MB por minuto — ampliamente dentro del presupuesto de banda. Pero quise probar el peor caso: durante el batch nocturno el redo llegaba a 180 MB por minuto. También pasaba, pero con un transport lag que subía a 45 segundos. Aceptable para un Data Guard en modo Maximum Performance.

La configuración del broker fue estándar:

    DGMGRL> CREATE CONFIGURATION dg_migration AS
             PRIMARY DATABASE IS prod_rac
             CONNECT IDENTIFIER IS prod_rac;

    DGMGRL> ADD DATABASE oci_standby AS
             CONNECT IDENTIFIER IS oci_standby
             MAINTAINED AS PHYSICAL;

    DGMGRL> ENABLE CONFIGURATION;

La primera sincronización completa tardó 14 horas — dos terabytes por una VPN a 500 Mbps dan exactamente ese número. Tras la sincronización inicial, Data Guard mantuvo el standby alineado con un apply lag medio de 3 segundos.

## El cutover: una noche, un plan, cero sorpresas

El cutover estaba planificado para un sábado por la noche. Preparé un runbook de 47 pasos — sí, cuarenta y siete. Cada paso con el tiempo estimado, el comando exacto, el criterio de éxito y el rollback. Porque si algo sale mal a las tres de la mañana, no quieres estar improvisando.

La secuencia crítica:

1. **22:00** — Parada del aplicativo. Verificado que todas las sesiones activas hubieran terminado.
2. **22:15** — Último check del transport lag: 2 segundos. Apply lag: 0.
3. **22:20** — Switchover vía Data Guard Broker:

        DGMGRL> SWITCHOVER TO oci_standby;

4. **22:22** — El switchover se completó en 98 segundos. El nuevo primary estaba en OCI.
5. **22:25** — Actualización de los connection strings en el connection pool del aplicativo. El SCAN listener en OCI ya estaba configurado.
6. **22:30** — Test de conectividad: login, queries sobre tablas críticas, insert de prueba.
7. **22:45** — Test del batch job: ejecución de un mini-batch sobre una muestra de datos.
8. **23:00** — Apertura gradual a los usuarios del turno de noche.
9. **23:30** — Monitorización: snapshots AWR cada 15 minutos en vez del default de 60.

A medianoche y media todo funcionaba. El downtime real — desde el momento en que el último usuario se desconectó hasta el momento en que el primero se reconectó — fue de 42 minutos.

## Después de la migración: las cosas que ningún plan prevé

La primera semana después del cutover es la que separa una migración exitosa de una migración "técnicamente exitosa pero todos se quejan". Tres cosas surgieron.

**Los timezone.** Las VMs en OCI usaban UTC, la base de datos on-premises usaba Europe/Rome. Los procedimientos PL/SQL que calculaban fechas con `SYSDATE` devolvían horarios incorrectos. El `ALTER DATABASE SET TIME_ZONE` requiere reiniciar la base de datos y reconstruir las columnas `TIMESTAMP WITH LOCAL TIME ZONE`. Lo descubrí el lunes por la mañana cuando el responsable de logística me llamó diciendo que los pedidos tenían fechas "del futuro". Arreglado en dos horas, pero se podía haber evitado si hubiera incluido el timezone en el runbook.

**El TLS.** Las llamadas HTTP salientes desde procedimientos PL/SQL usaban `UTL_HTTP` con wallet Oracle para los certificados. El wallet estaba configurado con los certificados del data center. En OCI, los certificados CA eran diferentes. Los procedimientos fallaban con `ORA-29024: Certificate validation failure`. Tuve que recrear el wallet importando los nuevos certificados CA y redistribuirlo.

**El scheduler.** Los jobs de Oracle Scheduler (`DBMS_SCHEDULER`) tenían windows y schedules basados en el timezone de la base de datos. Después del fix del timezone, las ventanas de mantenimiento se realinearon, pero tres jobs que usaban `SYSTIMESTAMP` directamente en el código PL/SQL siguieron arrancando una hora antes durante una semana — hasta que los encontré y corregí uno por uno.

## Los costes reales: más allá de la hoja Excel

Tres meses después de la migración, preparé un informe de costes reales para la dirección. La comparación con el viejo hosting fue instructiva.

| Concepto | On-premises | OCI |
|----------|-------------|-----|
| Compute (RAC 2 nodos + standby) | incluido en contrato hosting | €4.200/mes |
| Storage (2 TB + backup) | incluido | €680/mes |
| Networking (VPN + egress) | €200/mes | €350/mes |
| Licencias Oracle | €18.000/año (soporte) | €0 (BYOL) |
| Hosting/housing | €8.500/mes | €0 |
| Total anual | ~€120.000 | ~€63.000 |

El ahorro existía, y era significativo. Pero el número que más impactó al CFO no fue el total: fue el coste del networking. La VPN site-to-site y el tráfico egress de OCI costaban casi el doble que antes. Es una partida que en los presupuestos cloud siempre se subestima.

Y luego estaba el coste oculto: mi tiempo. Dos meses de consultoría para el assessment, la planificación, la migración y el tuning post-migración. Ese coste no aparecía en la comparación mensual, pero era real.

## Lo que aprendí (otra vez)

Cada migración enseña algo, incluso cuando crees que lo has visto todo.

El licenciamiento Oracle en la nube es un campo de minas. No basta con leer la documentación: hay que hablar con Oracle, obtener confirmaciones por escrito, y llevar un registro de todo. Una auditoría post-migración puede convertir un ahorro en una catástrofe.

El assessment no es opcional. Esas dos semanas iniciales evitaron al menos tres problemas que habrían requerido semanas de arreglos una vez completada la migración. El informe de features en uso, el mapa de dependencias externas, los tests de latencia — son trabajos aburridos, pero son la diferencia entre un cutover de 42 minutos y uno de 42 horas.

Data Guard cross-site es la estrategia de migración más limpia para Oracle. Te da una red de seguridad permanente: si algo sale mal, haces switchback y estás en el punto de partida. Con Data Pump, si algo sale mal a mitad del import, empiezas de cero.

Y el timezone. Dios, el timezone. Ponedlo al principio de la checklist.

------------------------------------------------------------------------

## Glosario

**[OCI](/es/glossary/oci/)** — Oracle Cloud Infrastructure, la plataforma cloud de Oracle. Para las bases de datos Oracle ofrece ventajas significativas en licenciamiento gracias al programa BYOL y a la relación 1:1 de las OCPU.

**[BYOL](/es/glossary/byol/)** — Bring Your Own License, programa que permite reutilizar las licencias Oracle on-premises en la nube OCI sin costes adicionales de licenciamiento.

**[RAC](/es/glossary/rac/)** — Real Application Clusters, tecnología Oracle que permite a múltiples instancias acceder simultáneamente a la misma base de datos, garantizando alta disponibilidad y escalabilidad horizontal.

**[Data Guard](/es/glossary/data-guard/)** — Tecnología Oracle para la replicación en tiempo real de una base de datos a uno o más servidores standby, garantizando alta disponibilidad y recuperación ante desastres.

**[ZDM](/es/glossary/zdm/)** — Zero Downtime Migration, herramienta Oracle para automatizar las migraciones a OCI combinando Data Guard y Data Pump bajo una capa de orquestación.

**[Switchover](/es/glossary/switchover/)** — Operación planificada de Data Guard que invierte los roles entre primary y standby sin pérdida de datos. A diferencia del failover, es reversible y controlada.

**[AWR](/es/glossary/awr/)** — Automatic Workload Repository, herramienta de diagnóstico integrada en Oracle Database para la recopilación y análisis de estadísticas de rendimiento.

**[Transport Lag](/es/glossary/transport-lag/)** — Retardo en la transmisión de los redo logs desde la base de datos primary al standby en una configuración Data Guard. Un indicador crítico de la salud de la replicación.

**[SCAN Listener](/es/glossary/scan-listener/)** — Single Client Access Name, componente de Oracle RAC que proporciona un único punto de acceso al cluster, distribuyendo automáticamente las conexiones entre los nodos disponibles.

**[Cutover](/es/glossary/cutover/)** — Momento crítico de una migración en el que el sistema de producción se traslada definitivamente de la vieja a la nueva infraestructura.
