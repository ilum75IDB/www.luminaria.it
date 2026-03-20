---
title: "Exchange Partition"
description: "Operación DDL de Oracle que intercambia instantáneamente los segmentos de datos entre una tabla no particionada y una partición, sin mover físicamente ningún dato."
translationKey: "glossary_exchange-partition"
aka: "Intercambio de partición"
articles:
  - "/posts/data-warehouse/partitioning-dwh"
---

El **Exchange Partition** es una operación DDL de Oracle que permite intercambiar instantáneamente el contenido de una partición con el de una tabla no particionada. No se mueve ni un solo byte de datos — la operación solo modifica los punteros en el data dictionary.

## Cómo funciona

El comando `ALTER TABLE ... EXCHANGE PARTITION ... WITH TABLE ...` modifica los metadatos en el data dictionary de modo que los segmentos físicos de la partición y de la tabla de staging intercambien su propiedad. La tabla de staging se convierte en la partición y viceversa. La operación dura menos de un segundo independientemente del volumen de datos, porque no implica ningún movimiento físico.

## Para qué sirve

En los data warehouses, el exchange partition es la herramienta principal para la carga masiva de datos. El proceso típico es: el ETL carga los datos en una tabla de staging, construye los índices, valida los datos, y luego ejecuta el exchange con la partición objetivo. Durante el exchange, las queries sobre las demás particiones siguen funcionando sin interrupción.

## Qué puede salir mal

La cláusula `WITHOUT VALIDATION` omite la verificación de que los datos de la staging table caigan realmente dentro del rango de la partición — acelera la operación pero requiere que el ETL garantice la corrección de los datos. Si los datos en la staging contienen fechas fuera de rango, acaban en la partición equivocada sin generar ningún error. La cláusula `INCLUDING INDEXES` requiere que la staging table tenga índices con la misma estructura que los índices locales de la tabla particionada.
