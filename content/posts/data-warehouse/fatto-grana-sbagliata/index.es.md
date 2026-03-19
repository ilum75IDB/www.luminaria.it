---
title: "Granularidad equivocada: cuando la fact table no responde las preguntas correctas"
description: "Una fact table construida sobre el total mensual de facturación parecía perfecta. Luego el negocio pidió detalle por producto, por línea, por cliente. Y el data warehouse enmudeció."
date: "2025-10-21T10:00:00+01:00"
draft: false
translationKey: "fatto_grana_sbagliata"
tags: ["data-warehouse", "fact-table", "granularity", "grain", "dimensional-modeling", "kimball"]
categories: ["Data Warehouse"]
image: "fatto-grana-sbagliata.cover.jpg"
---

La reunión había empezado bien. El director comercial de una empresa de distribución industrial — unos sesenta millones de facturación, tres mil clientes activos, un catálogo con doce mil referencias — había abierto la presentación del nuevo data warehouse con una sonrisa. Los números cuadraban, los dashboards lucían bien, los totales mensuales por agente y por zona coincidían con contabilidad.

Luego alguien hizo la pregunta equivocada. O mejor dicho, la correcta.

*"¿Puedo ver cuánto compró el cliente Bianchi en marzo, línea por línea, producto por producto?"*

Silencio.

El responsable de BI me miró. Yo miré la pantalla. La pantalla mostraba una {{< glossary term="fact-table" >}}fact table{{< /glossary >}} con una fila por cliente por mes: importe total facturado, cantidad total, número de facturas. Ningún detalle. Ninguna línea de factura. Ningún producto.

Esa fact table respondía a una sola pregunta: *¿cuánto facturó cada cliente en un mes determinado?* Todo lo demás — por producto, por familia de productos, por factura individual — quedaba fuera de alcance.

## 🔍 El grain: la decisión que determina todo

En el {{< glossary term="star-schema" >}}modelado dimensional{{< /glossary >}}, el **{{< glossary term="grain" >}}grain{{< /glossary >}}** (la granularidad) de la fact table es la primera decisión que se toma. No la segunda, no una entre tantas: la primera. Kimball lo repite en cada capítulo, y tiene razón.

El grain responde a la pregunta: *¿qué representa una fila individual de la fact table?*

En el proyecto que describí, quien diseñó el modelo había elegido un grain mensual-cliente: una fila = un cliente en un mes. Las razones parecían razonables: el sistema fuente exportaba un resumen mensual, la carga era rápida, las tablas eran pequeñas, las consultas simples.

Pero el grain determina las preguntas que el data warehouse puede responder. Si la granularidad es el resumen mensual por cliente, no puedes bajar de ese nivel. No puedes hacer {{< glossary term="drill-down" >}}drill-down{{< /glossary >}} por producto. No puedes saber si el cliente Bianchi compró diez veces el mismo artículo o diez artículos diferentes. No puedes comparar márgenes por familia de productos.

Tienes un total. Punto.

## 📊 Los números del problema

La fact table original tenía esta estructura:

```sql
CREATE TABLE fact_facturacion_mensual (
    sk_cliente        INT NOT NULL,
    sk_tiempo         INT NOT NULL,  -- mes (YYYYMM)
    sk_agente         INT NOT NULL,
    sk_zona           INT NOT NULL,
    importe_total     DECIMAL(15,2),
    cantidad_total    INT,
    num_facturas      INT,
    num_lineas        INT,
    FOREIGN KEY (sk_cliente) REFERENCES dim_cliente(sk_cliente),
    FOREIGN KEY (sk_tiempo)  REFERENCES dim_tiempo(sk_tiempo)
);
```

Filas por año: unas 180.000 (3.000 clientes × 12 meses × alguna variación). Pequeña, rápida, fácil de cargar. El {{< glossary term="etl" >}}ETL{{< /glossary >}} corría en menos de cinco minutos.

¿El problema? Las {{< glossary term="additive-measure" >}}medidas aditivas{{< /glossary >}} ya estaban agregadas. `importe_total` era la suma de todas las líneas de factura del mes. Imposible rastrear la composición. Como tener el total de un ticket sin saber qué compraste.

## 🏗️ La reestructuración: bajar a la línea de factura

La solución era una sola: cambiar el grain. Llevar la fact table al nivel más bajo disponible en el sistema fuente — la línea individual de factura.

```sql
CREATE TABLE fact_facturacion_linea (
    sk_linea_factura  INT PRIMARY KEY,
    sk_factura        INT NOT NULL,
    sk_cliente        INT NOT NULL,
    sk_producto       INT NOT NULL,
    sk_tiempo         INT NOT NULL,  -- día (YYYYMMDD)
    sk_agente         INT NOT NULL,
    sk_zona           INT NOT NULL,
    sk_familia        INT NOT NULL,
    cantidad          INT,
    precio_unitario   DECIMAL(12,4),
    importe_linea     DECIMAL(15,2),
    descuento_pct     DECIMAL(5,2),
    importe_neto      DECIMAL(15,2),
    coste_producto    DECIMAL(15,2),
    margen            DECIMAL(15,2),
    FOREIGN KEY (sk_cliente)  REFERENCES dim_cliente(sk_cliente),
    FOREIGN KEY (sk_producto) REFERENCES dim_producto(sk_producto),
    FOREIGN KEY (sk_tiempo)   REFERENCES dim_tiempo(sk_tiempo)
);
```

Filas por año: unos 2,4 millones (3.000 clientes × ~800 líneas/año en promedio). Un orden de magnitud más. Pero cada fila llevaba consigo el detalle completo: qué producto, qué factura, qué precio, qué descuento, qué margen.

## ⚡ El impacto en el ETL

El cambio de grain tuvo un efecto en cascada sobre el ETL que nadie había previsto — o mejor dicho, que quien eligió el grain agregado había evitado enfrentar.

**Nuevas dimensiones necesarias:**

| Dimensión          | Cardinalidad | Notas                                |
|---------------------|-------------|--------------------------------------|
| `dim_producto`      | ~12.000     | No existía: antes no era necesaria   |
| `dim_familia`       | ~180        | Jerarquía de productos a 3 niveles   |
| `dim_factura`       | ~45.000/año | Cabecera de factura con datos maestros|

**Nueva ventana de carga:**

| Fase                | Antes   | Después   |
|---------------------|---------|-----------|
| Extracción          | 40 seg  | 3 min     |
| Transformación      | 1 min   | 8 min     |
| Carga fact          | 30 seg  | 4 min     |
| **Total**           | **~2 min** | **~15 min** |

Quince minutos contra dos. Un precio aceptable para un data warehouse que ahora respondía preguntas reales.

## 🔬 Las consultas que antes eran imposibles

Con el nuevo grain, las consultas que el negocio quería se volvieron triviales:

**Detalle de compras de un cliente por producto:**

```sql
SELECT
    c.razon_social,
    p.codigo_producto,
    p.descripcion,
    SUM(f.cantidad)       AS piezas,
    SUM(f.importe_neto)   AS facturado_neto,
    SUM(f.margen)         AS margen_total
FROM fact_facturacion_linea f
JOIN dim_cliente  c ON f.sk_cliente  = c.sk_cliente
JOIN dim_producto p ON f.sk_producto = p.sk_producto
JOIN dim_tiempo   t ON f.sk_tiempo   = t.sk_tiempo
WHERE c.razon_social = 'Bianchi Srl'
  AND t.anio = 2024
  AND t.mes = 3
GROUP BY c.razon_social, p.codigo_producto, p.descripcion
ORDER BY facturado_neto DESC;
```

**Top 10 productos por margen en un trimestre:**

```sql
SELECT
    p.codigo_producto,
    p.descripcion,
    fm.desc_familia,
    SUM(f.importe_neto)  AS facturado,
    SUM(f.margen)        AS margen,
    ROUND(SUM(f.margen) / NULLIF(SUM(f.importe_neto), 0) * 100, 1) AS margen_pct
FROM fact_facturacion_linea f
JOIN dim_producto p  ON f.sk_producto = p.sk_producto
JOIN dim_familia  fm ON f.sk_familia  = fm.sk_familia
JOIN dim_tiempo   t  ON f.sk_tiempo   = t.sk_tiempo
WHERE t.anio = 2024
  AND t.trimestre = 1
GROUP BY p.codigo_producto, p.descripcion, fm.desc_familia
ORDER BY margen DESC
LIMIT 10;
```

**Comparación entre agentes: facturación media por línea:**

```sql
SELECT
    a.nombre_agente,
    COUNT(*)                        AS num_lineas,
    SUM(f.importe_neto)             AS facturado_total,
    ROUND(AVG(f.importe_neto), 2)   AS media_por_linea
FROM fact_facturacion_linea f
JOIN dim_agente a ON f.sk_agente = a.sk_agente
JOIN dim_tiempo t ON f.sk_tiempo = t.sk_tiempo
WHERE t.anio = 2024
GROUP BY a.nombre_agente
ORDER BY facturado_total DESC;
```

Ninguna de estas consultas era posible con el grain mensual-cliente. Ninguna. No era un problema de tuning o de índices — era un problema estructural, escrito en el ADN del modelo.

## 📋 La regla de Kimball que habíamos ignorado

Ralph Kimball lo dice claramente: *"siempre modelar al nivel de detalle más fino disponible en el sistema fuente."*

No es una sugerencia. No es una opción entre varias. Es el principio fundacional del modelado dimensional. Y la razón es simple: siempre se puede agregar del detalle al total, pero nunca se puede desagregar un total en su detalle.

La agregación es una operación irreversible. Como mezclar colores: del rojo y el amarillo puedes obtener el naranja, pero del naranja nunca vuelves a los colores originales.

En nuestro proyecto, la elección del grain agregado fue dictada por pereza de diseño, no por una restricción técnica. El sistema fuente tenía el detalle por línea de factura — simplemente nadie había querido enfrentar la complejidad de modelarlo, gestionar las dimensiones adicionales, extender la ventana del ETL.

¿El resultado? Un data warehouse que tuvo que reconstruirse desde cero seis meses después del go-live.

## 🎯 Cuándo el grain agregado tiene sentido

La granularidad fina no siempre es la única respuesta. Hay casos legítimos para fact tables agregadas:

- **Tablas de agregación** (aggregate fact table) junto a la tabla de detalle, para acelerar las consultas más frecuentes
- **Snapshots periódicos** donde el negocio razona por período (saldo mensual de una cuenta, inventario a fin de semana)
- **Restricciones de origen** cuando el sistema fuente no expone el detalle y no hay forma de obtenerlo

Pero la regla es: parte del detalle, luego agrega. Nunca al revés. Las aggregate fact tables son una optimización, no un sustituto de la granularidad fina.

En nuestro caso, después de la reestructuración, también creamos una vista materializada con el resumen mensual por cliente — la misma estructura de antes — para los dashboards ejecutivos que no necesitaban el detalle. Lo mejor de ambos mundos, sin sacrificar nada.

## Lo que aprendí

Ese proyecto me enseñó algo que llevo conmigo en cada encargo posterior: la primera media hora de diseño de un data warehouse, aquella en la que se decide el grain, vale más que todas las optimizaciones que vendrán después. Un ETL perfecto, índices calibrados, hardware potente — nada de esto compensa un grain equivocado.

Si tu fact table no responde las preguntas del negocio, no es culpa de las consultas. Es culpa del modelo. Y el modelo se decide en el grain.

------------------------------------------------------------------------

## Glosario

**[Grain](/es/glossary/grain/)** — El nivel de detalle (granularidad) de una fact table en el data warehouse. Determina qué representa cada fila y qué preguntas puede responder el modelo. Es la primera decisión en el diseño dimensional.

**[Fact table](/es/glossary/fact-table/)** — Tabla central del star schema que contiene las medidas numéricas (importes, cantidades, márgenes) y las claves foráneas hacia las dimensiones. Su granularidad determina el nivel de análisis posible.

**[Additive Measure](/es/glossary/additive-measure/)** — Medida numérica que puede sumarse a lo largo de todas las dimensiones (ej. importe, cantidad). Una vez agregada a nivel superior, el detalle original se pierde irreversiblemente.

**[Drill-down](/es/glossary/drill-down/)** — Navegación en reportes desde el nivel agregado al detalle, a lo largo de una jerarquía. Solo posible si la fact table contiene datos a un nivel de granularidad suficiente.

**[Star Schema](/es/glossary/star-schema/)** — Modelo de datos con una fact table central y tablas dimensionales enlazadas. La estructura más usada en data warehouses por la simplicidad de las consultas analíticas.

**[ETL](/es/glossary/etl/)** — Extract, Transform, Load: el proceso de extracción, transformación y carga de datos en el data warehouse. Un cambio de grain impacta directamente tiempos y complejidad del ETL.
