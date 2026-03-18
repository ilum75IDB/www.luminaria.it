---
title: "Jerarquías desbalanceadas: cuando el cliente no tiene padre y el grupo no tiene abuelo"
description: "Un cliente con una jerarquía de tres niveles — Top Group, Group, Client — donde no todas las ramas están completas. Cómo balanceé una ragged hierarchy con self-parenting: quien no tiene padre se convierte en padre de sí mismo."
date: "2026-01-20T10:00:00+01:00"
draft: false
translationKey: "ragged_hierarchies"
tags: ["hierarchies", "dimensional-modeling", "etl", "oracle", "olap", "reporting"]
categories: ["data-warehouse"]
image: "ragged-hierarchies.cover.jpg"
---

Tres niveles. Top Group, Group, Client. Parece una estructura trivial — el tipo de jerarquía que dibujas en una pizarra en cinco minutos y que cualquier herramienta de BI debería manejar sin problemas.

Luego descubres que no todos los clientes pertenecen a un grupo. Y que no todos los grupos pertenecen a un top group. Y que los reportes de agregación que el negocio pide — facturación por top group, número de clientes por grupo, {{< glossary term="drill-down" >}}drill-down{{< /glossary >}} desde la cima hasta la hoja — producen resultados erróneos o incompletos porque la jerarquía tiene huecos.

En jerga técnica se llama **{{< glossary term="ragged-hierarchy" >}}ragged hierarchy{{< /glossary >}}**: una jerarquía en la que no todas las ramas alcanzan la misma profundidad. En el mundo real se llama "el problema que nadie ve hasta que abre el reporte y los números no cuadran."

---

## El cliente y el modelo original

El proyecto era un data warehouse para una empresa del sector energético — distribución de gas y servicios relacionados. El sistema fuente gestionaba un maestro de clientes con una estructura jerárquica: los clientes podían agruparse bajo una entidad comercial (el **Group**), y los grupos podían a su vez pertenecer a una entidad superior (el **Top Group**).

El modelo en la fuente era una tabla única con las referencias jerárquicas:

``` sql
CREATE TABLE stg_clienti (
    client_id       NUMBER(10)    NOT NULL,
    client_name     VARCHAR2(100) NOT NULL,
    group_id        NUMBER(10),
    group_name      VARCHAR2(100),
    top_group_id    NUMBER(10),
    top_group_name  VARCHAR2(100),
    revenue         NUMBER(15,2),
    region          VARCHAR2(50),
    CONSTRAINT pk_stg_clienti PRIMARY KEY (client_id)
);
```

Aquí un ejemplo de los datos:

``` sql
INSERT INTO stg_clienti VALUES (1001, 'Rossi Energia Srl',     10, 'Consorzio Nord',      100, 'Holding Nazionale',  125000.00, 'Lombardia');
INSERT INTO stg_clienti VALUES (1002, 'Bianchi Gas SpA',       10, 'Consorzio Nord',      100, 'Holding Nazionale',   89000.00, 'Piemonte');
INSERT INTO stg_clienti VALUES (1003, 'Verdi Distribuzione',   20, 'Gruppo Centro',       100, 'Holding Nazionale',   67000.00, 'Toscana');
INSERT INTO stg_clienti VALUES (1004, 'Neri Servizi',          20, 'Gruppo Centro',       NULL, NULL,                  45000.00, 'Lazio');
INSERT INTO stg_clienti VALUES (1005, 'Gialli Utilities',      NULL, NULL,                NULL, NULL,                  38000.00, 'Sicilia');
INSERT INTO stg_clienti VALUES (1006, 'Blu Energia',           NULL, NULL,                NULL, NULL,                  52000.00, 'Sardegna');
INSERT INTO stg_clienti VALUES (1007, 'Viola Gas Srl',         30, 'Rete Sud',            NULL, NULL,                  71000.00, 'Campania');
INSERT INTO stg_clienti VALUES (1008, 'Arancio Distribuzione', 30, 'Rete Sud',            NULL, NULL,                  33000.00, 'Calabria');
```

Miren los datos con atención. Hay cuatro situaciones diferentes:

- **Client 1001, 1002, 1003**: jerarquía completa — Client → Group → Top Group
- **Client 1004**: tiene un Group pero el Group no tiene Top Group
- **Client 1005, 1006**: sin Group, sin Top Group — clientes directos
- **Client 1007, 1008**: tienen un Group (Rete Sud) pero el Group no tiene Top Group

Esto es una ragged hierarchy. Tres niveles sobre el papel, pero en la realidad las ramas tienen profundidades diferentes.

---

## El problema: los reportes no cuadran

El negocio pedía un reporte sencillo: facturación agregada por Top Group, con posibilidad de drill-down por Group y luego por Client. Una petición razonable — el tipo de cosa que esperas de cualquier DWH.

La consulta más natural:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clientes,
       SUM(revenue)    AS facturacion_total
FROM   stg_clienti
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

El resultado:

``` text
TOP_GROUP_NAME      GROUP_NAME        NUM_CLIENTES  FACTURACION_TOTAL
------------------  ----------------  ------------  -----------------
Holding Nazionale   Consorzio Nord               2         214000.00
Holding Nazionale   Gruppo Centro                1          67000.00
(null)              Gruppo Centro                1          45000.00
(null)              Rete Sud                     2         104000.00
(null)              (null)                       2          90000.00
```

Cinco filas. Y al menos tres problemas.

Gruppo Centro aparece dos veces: una bajo "Holding Nazionale" (el cliente 1003 que tiene top group) y una bajo NULL (el cliente 1004 cuyo top group es NULL). El mismo grupo, partido en dos filas, con totales separados. Cualquiera que mire este reporte pensará que Gruppo Centro tiene 67K de facturación bajo la holding y 45K en algún otro sitio. En realidad es un único grupo con 112K totales.

Los clientes directos (Gialli Utilities y Blu Energia) terminan en una fila con dos NULL. La dirección no sabe qué hacer con una fila sin nombre.

El total por Top Group está mal porque faltan las filas con NULL. Si sumas solo las filas con top group, pierdes 239K de facturación — el 30% del total.

---

## El enfoque clásico: COALESCE y rezos

La primera reacción, la que veo en el 90% de los casos, es añadir {{< glossary term="coalesce" >}}`COALESCE`{{< /glossary >}} en la consulta:

``` sql
SELECT COALESCE(top_group_name, group_name, client_name) AS top_group_name,
       COALESCE(group_name, client_name)                 AS group_name,
       client_name,
       revenue
FROM   stg_clienti;
```

¿Funciona? En cierto sentido sí — rellena los huecos. Pero introduce problemas nuevos.

El cliente "Gialli Utilities" ahora aparece como Top Group, Group y Client simultáneamente. Si el negocio quiere contar cuántos Top Groups hay, el número está inflado. Si quiere filtrar por "verdaderos" top groups, no hay forma de distinguirlos de los clientes promovidos por la COALESCE.

Y este es el caso sencillo, con tres niveles. He visto jerarquías de cinco niveles gestionadas con cadenas de COALESCE anidados, múltiples CASE WHEN, y una lógica de reportes tan enrevesada que nadie se atrevía a tocarla. Cada nueva petición del negocio requería un cambio en cascada en todas las consultas.

El problema de fondo es que la COALESCE es un parche aplicado en la capa de presentación. No resuelve el problema estructural: la jerarquía está incompleta y el modelo dimensional no lo sabe.

---

## La solución: self-parenting

El principio es simple: **quien no tiene padre se convierte en padre de sí mismo**. Esta técnica se llama {{< glossary term="self-parenting" >}}self-parenting{{< /glossary >}}.

¿Un Client sin Group? Ese cliente se convierte en su propio Group. ¿Un Group sin Top Group? Ese grupo se convierte en su propio Top Group. De esta forma la jerarquía siempre está completa a tres niveles, sin huecos, sin NULL.

No es un truco. Es una técnica estándar en modelado dimensional, descrita por {{< glossary term="kimball" >}}Kimball{{< /glossary >}} y usada en producción desde hace décadas. La idea es que la dimensión jerárquica en el DWH debe ser **balanceada**: cada registro debe tener un valor válido para cada nivel de la jerarquía. Si la fuente no lo garantiza, lo garantiza el {{< glossary term="etl" >}}ETL{{< /glossary >}}.

### La tabla dimensional

``` sql
CREATE TABLE dim_client_hierarchy (
    client_key      NUMBER(10)    NOT NULL,
    client_id       NUMBER(10)    NOT NULL,
    client_name     VARCHAR2(100) NOT NULL,
    group_id        NUMBER(10)    NOT NULL,
    group_name      VARCHAR2(100) NOT NULL,
    top_group_id    NUMBER(10)    NOT NULL,
    top_group_name  VARCHAR2(100) NOT NULL,
    region          VARCHAR2(50),
    is_direct_client  CHAR(1)     DEFAULT 'N',
    is_standalone_group CHAR(1)   DEFAULT 'N',
    CONSTRAINT pk_dim_client_hier PRIMARY KEY (client_key)
);
```

Noten dos cosas. Primero: ninguna columna es nullable. Group y Top Group son `NOT NULL`. Segundo: añadí dos flags — `is_direct_client` e `is_standalone_group` — que permiten distinguir los registros balanceados artificialmente de los que tienen una jerarquía natural. Esto es importante: el negocio debe poder filtrar los "verdaderos" top groups de los clientes promovidos.

### La lógica ETL

``` sql
INSERT INTO dim_client_hierarchy (
    client_key, client_id, client_name,
    group_id, group_name,
    top_group_id, top_group_name,
    region, is_direct_client, is_standalone_group
)
SELECT
    client_id AS client_key,
    client_id,
    client_name,
    -- Si no tiene group, el cliente se convierte en group de sí mismo
    COALESCE(group_id, client_id)          AS group_id,
    COALESCE(group_name, client_name)      AS group_name,
    -- Si no tiene top group, el group (o cliente) se convierte en su propio top group
    COALESCE(top_group_id, group_id, client_id)       AS top_group_id,
    COALESCE(top_group_name, group_name, client_name)  AS top_group_name,
    region,
    CASE WHEN group_id IS NULL THEN 'Y' ELSE 'N' END  AS is_direct_client,
    CASE WHEN group_id IS NOT NULL AND top_group_id IS NULL
         THEN 'Y' ELSE 'N' END                        AS is_standalone_group
FROM stg_clienti;
```

Miren la cascada de COALESCE en la transformación. La lógica es:

- `group_id`: si el cliente tiene grupo, usa ese; si no, usa el propio cliente
- `top_group_id`: si hay top group, usa ese; si no pero hay group, usa el group; si tampoco hay group, usa el cliente

Cada nivel "faltante" se rellena con el nivel inmediatamente inferior. El resultado es una jerarquía siempre completa.

### El resultado después del balanceo

``` sql
SELECT client_key, client_name, group_name, top_group_name,
       is_direct_client, is_standalone_group
FROM   dim_client_hierarchy
ORDER BY top_group_id, group_id, client_id;
```

``` text
KEY   CLIENT_NAME           GROUP_NAME        TOP_GROUP_NAME      DIRECT  STANDALONE
----  --------------------  ----------------  ------------------  ------  ----------
1001  Rossi Energia Srl     Consorzio Nord    Holding Nazionale   N       N
1002  Bianchi Gas SpA       Consorzio Nord    Holding Nazionale   N       N
1003  Verdi Distribuzione   Gruppo Centro     Holding Nazionale   N       N
1004  Neri Servizi          Gruppo Centro     Gruppo Centro       N       Y
1007  Viola Gas Srl         Rete Sud          Rete Sud            N       Y
1008  Arancio Distribuzione Rete Sud          Rete Sud            N       Y
1005  Gialli Utilities      Gialli Utilities  Gialli Utilities    Y       N
1006  Blu Energia           Blu Energia       Blu Energia         Y       N
```

Ocho filas, cero NULL. Cada cliente tiene un group y un top group. Los flags dicen la verdad: Gialli y Blu son clientes directos (self-parented a todos los niveles), Gruppo Centro y Rete Sud son grupos standalone (self-parented al nivel top group).

---

## Los reportes después del balanceo

La misma consulta de agregación que antes producía resultados rotos:

``` sql
SELECT top_group_name,
       group_name,
       COUNT(*)        AS num_clientes,
       SUM(f.revenue)  AS facturacion_total
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name, group_name
ORDER BY top_group_name, group_name;
```

``` text
TOP_GROUP_NAME      GROUP_NAME          NUM_CLIENTES  FACTURACION_TOTAL
------------------  ------------------  ------------  -----------------
Blu Energia         Blu Energia                    1          52000.00
Gialli Utilities    Gialli Utilities               1          38000.00
Gruppo Centro       Gruppo Centro                  1          45000.00
Holding Nazionale   Consorzio Nord                 2         214000.00
Holding Nazionale   Gruppo Centro                  1          67000.00
Rete Sud            Rete Sud                       2         104000.00
```

Ningún NULL. Cada fila tiene un top group y un group identificables. Los totales cuadran.

Y si el negocio quiere solo los "verdaderos" top groups, excluyendo los clientes promovidos:

``` sql
SELECT top_group_name,
       COUNT(*)        AS num_clientes,
       SUM(f.revenue)  AS facturacion_total
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.is_direct_client = 'N'
AND    d.is_standalone_group = 'N'
GROUP BY top_group_name
ORDER BY facturacion_total DESC;
```

``` text
TOP_GROUP_NAME      NUM_CLIENTES  FACTURACION_TOTAL
------------------  ------------  -----------------
Holding Nazionale              3         281000.00
```

Los flags hacen todo filtrable. Sin lógica condicional en el reporte, sin CASE WHEN, sin COALESCE. El modelo dimensional ya contiene toda la información necesaria.

---

## El drill-down completo

La verdadera prueba de una jerarquía balanceada es el drill-down: desde el nivel más alto al más bajo, sin sorpresas.

``` sql
-- Nivel 1: total por Top Group
SELECT top_group_name,
       COUNT(DISTINCT group_id) AS num_grupos,
       COUNT(*)                 AS num_clientes,
       SUM(f.revenue)           AS facturacion
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
GROUP BY top_group_name
ORDER BY facturacion DESC;
```

``` text
TOP_GROUP_NAME      NUM_GRUPOS  NUM_CLIENTES  FACTURACION
------------------  ----------  ------------  -----------
Holding Nazionale            2             3    281000.00
Rete Sud                     1             2    104000.00
Blu Energia                  1             1     52000.00
Gruppo Centro                1             1     45000.00
Gialli Utilities             1             1     38000.00
```

``` sql
-- Nivel 2: drill-down dentro de "Holding Nazionale"
SELECT group_name,
       COUNT(*)       AS num_clientes,
       SUM(f.revenue) AS facturacion
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.top_group_name = 'Holding Nazionale'
GROUP BY group_name
ORDER BY facturacion DESC;
```

``` text
GROUP_NAME        NUM_CLIENTES  FACTURACION
----------------  ------------  -----------
Consorzio Nord               2    214000.00
Gruppo Centro                1     67000.00
```

``` sql
-- Nivel 3: drill-down dentro de "Consorzio Nord"
SELECT client_name, f.revenue
FROM   dim_client_hierarchy d
JOIN   stg_clienti f ON d.client_id = f.client_id
WHERE  d.group_name = 'Consorzio Nord'
ORDER BY f.revenue DESC;
```

``` text
CLIENT_NAME          REVENUE
-------------------  ----------
Rossi Energia Srl     125000.00
Bianchi Gas SpA        89000.00
```

Tres niveles de drill-down, cero NULL, cero lógica condicional. La jerarquía está balanceada y los números cuadran en cada nivel.

---

## Por qué la COALESCE en los reportes no basta

Alguien podría objetar: "Pero la COALESCE en el reporte hace lo mismo, sin necesidad de modificar el modelo."

No. Hace algo parecido, pero con tres diferencias fundamentales.

**Primero: la COALESCE hay que repetirla en todas partes.** Cada consulta, cada reporte, cada dashboard, cada extracción. Si tienes veinte reportes que usan la jerarquía, debes acordarte de aplicar la COALESCE en los veinte. Y cuando llega el veintiuno, debes acordarte de nuevo. El self-parenting en el modelo dimensional se hace una vez en el ETL y punto.

**Segundo: la COALESCE no distingue.** No sabes si "Gialli Utilities" en el campo top_group es un verdadero top group o un cliente promovido. Con los flags en el modelo dimensional tienes la información para filtrar. Sin flags, el negocio está ciego.

**Tercero: el rendimiento.** Un GROUP BY con COALESCE sobre columnas nullable es menos eficiente que un GROUP BY sobre columnas NOT NULL. El optimizador de Oracle gestiona mejor las columnas con restricción NOT NULL — puede eliminar verificaciones de NULL, usar índices más agresivamente y producir planes de ejecución más simples. Sobre una tabla dimensional con millones de filas, la diferencia se nota.

---

## Cuándo usar el self-parenting (y cuándo no)

El self-parenting funciona bien cuando:

- La jerarquía tiene un **número fijo de niveles** (típicamente 2-5)
- El caso de uso principal es la **agregación y el drill-down** en reportes
- El modelo es un **data warehouse** o un cubo {{< glossary term="olap" >}}OLAP{{< /glossary >}}
- Los niveles faltantes son la excepción, no la regla

No funciona bien cuando:

- La jerarquía es **recursiva** con profundidad variable (ej. organigramas con N niveles)
- Se necesita navegar el **grafo** de relaciones (ej. redes sociales, cadenas de suministro)
- El modelo es **OLTP** y el self-parenting crearía ambigüedad en las lógicas aplicativas
- Los niveles de la jerarquía cambian frecuentemente en el tiempo

Para jerarquías recursivas con profundidad variable, el enfoque correcto es diferente: tablas de bridge, closure tables o modelos parent-child con CTEs recursivas. Son herramientas potentes pero resuelven un problema distinto.

El self-parenting resuelve un problema específico — jerarquías de niveles fijos con ramas incompletas — y lo resuelve de la forma más simple posible: balanceando la estructura aguas arriba, en el modelo, en lugar de aguas abajo, en los reportes.

---

## La regla que me guía

He diseñado decenas de dimensiones jerárquicas en veinte años de data warehousing. La regla que llevo conmigo es siempre la misma:

**Si el reporte necesita lógica condicional para gestionar la jerarquía, el problema está en el modelo, no en el reporte.**

Un reporte debería hacer GROUP BY y JOIN. Si además tiene que decidir cómo gestionar los niveles faltantes, está haciendo el trabajo del ETL. Y un reporte que hace el trabajo del ETL es un reporte que tarde o temprano se rompe.

El self-parenting no es elegante. No es sofisticado. Es una solución que un informático recién graduado podría encontrar fea. Pero funciona, es mantenible, y transforma un problema que infesta cada reporte individual en un problema que se resuelve una vez, en un solo punto, y no vuelve más.

A veces la mejor solución es la más simple. Esta es una de esas veces.

---

## Glosario

**[COALESCE](/es/glossary/coalesce/)** — Función SQL que devuelve el primer valor no NULL de una lista de expresiones. A menudo usada como workaround para las jerarquías incompletas en los reportes, pero no resuelve el problema estructural en el modelo dimensional.

**[Drill-down](/es/glossary/drill-down/)** — Navegación en los reportes desde un nivel agregado hasta un nivel de detalle (ej. de Top Group a Group a Client). Requiere una jerarquía completa y balanceada para funcionar correctamente sin NULLs ni filas faltantes.

**[OLAP](/es/glossary/olap/)** — Online Analytical Processing — procesamiento orientado al análisis multidimensional de datos, típico de los data warehouses y cubos de análisis. Contrapuesto al OLTP (Online Transaction Processing) de los sistemas transaccionales.

**[Ragged hierarchy](/es/glossary/ragged-hierarchy/)** — Jerarquía en la que no todas las ramas alcanzan la misma profundidad: algunos niveles intermedios están ausentes. Común en datos maestros de clientes, productos y estructuras organizativas donde no todas las entidades comparten la misma estructura jerárquica.

**[Self-parenting](/es/glossary/self-parenting/)** — Técnica de balanceo de jerarquías desequilibradas: quien no tiene padre se convierte en padre de sí mismo. El nivel faltante se rellena con los datos del nivel inferior, eliminando los NULLs de la dimensión y garantizando un drill-down correcto.
