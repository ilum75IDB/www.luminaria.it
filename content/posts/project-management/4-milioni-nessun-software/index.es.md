---
title: "4 millones de euros, dos multinacionales, cero software: historia real de un fracaso anunciado"
description: "Un cliente del sector asegurador gastó más de 4 millones de euros confiando en dos gigantes de la consultoría IT para un software a medida. Resultado: nada. En comparación, un data warehouse construido por dos personas en 3 años con 60.000 líneas de código que funciona cada día."
date: "2025-12-30T10:00:00+01:00"
draft: false
translationKey: "consulenza_it_milioni_sprecati"
tags: ["consulting", "data-warehouse", "insurance", "oracle", "outsourcing"]
categories: ["Project Management"]
image: "4-milioni-nessun-software.cover.jpg"
---

La historia que voy a contar es real. No daré nombres — no por diplomacia, sino porque los nombres no importan. Lo que importa es entender el mecanismo. Porque este mecanismo se repite, idéntico, en decenas de empresas. Y cuesta millones.

------------------------------------------------------------------------

## 🏢 El cliente: un grupo asegurador con una ambición legítima

Una empresa sólida del sector asegurador. Operaciones en Italia, Francia, países del Norte de Europa, España. Miles de empleados, millones de pólizas gestionadas, un negocio en crecimiento.

En un momento dado, la dirección toma una decisión razonable: **necesitamos un software de gestión a medida**. Un sistema que refleje nuestros procesos, nuestras reglas de negocio, las especificidades normativas de cada país en el que operamos.

Decisión legítima. Sensata. Incluso estratégica.

El problema no es la decisión.\
El problema es **a quién se la confían**.

------------------------------------------------------------------------

## 💰 Primer acto: la gran multinacional (2013–2018)

Se contrata — en pleno {{< glossary term="outsourcing" >}}outsourcing{{< /glossary >}} — a una de las Big de la consultoría IT mundial. Un nombre que todos conocen. Miles de consultores, oficinas en cada continente, presentaciones en PowerPoint que harían llorar de emoción.

El proyecto arranca. Se definen los requisitos. Se estima el presupuesto. Se firman los contratos.

Pasan los meses. Luego los años.

Los entregables llegan — sobre el papel. Pero el software no funciona como debería. Las especificaciones cambian. Los costes se disparan. Los consultores rotan: el que había entendido el dominio se va, llega otro que empieza de cero. El clásico esquema de consultoría a cuerpo que se convierte, en la práctica, en una consultoría a tiempo indefinido.

**De 2013 a 2018: más de 2,5 millones de euros gastados.**\
Resultado: un software incompleto, inestable, que nadie internamente sabía mantener.

Porque el código lo habían escrito *ellos*. Con *sus* convenciones. Con *su* arquitectura. Y cuando se fueron, se llevaron también el conocimiento.

------------------------------------------------------------------------

## 🔄 Segundo acto: cambiamos de proveedor (2018–2022)

La dirección, escarmentada pero no resignada, decide cambiar. "El problema era el proveedor", piensan. "Busquemos uno mejor."

Entra en escena otra multinacional. Igual de famosa. Igual de grande. Igual de cara.

Nuevo kickoff. Nuevo análisis de requisitos — porque obviamente no pueden partir del trabajo del proveedor anterior. Nuevas diapositivas. Nuevas promesas.

Y la historia se repite.

Mismos problemas, actores diferentes. Rotación de consultores. Pérdida de know-how. Plazos que se alargan. Presupuestos que explotan. Reuniones interminables en las que se discuten hitos que nunca llegan.

**De 2018 a 2022: otros 1,5 millones de euros.**\
Resultado: otro software que no satisface las necesidades del negocio.

Total invertido en casi una década: **más de 4 millones de euros**.\
Software funcionando: **cero**.

------------------------------------------------------------------------

## 📊 Hagamos las cuentas del desastre

| Período | Proveedor | Inversión | Resultado |
|---|---|---|---|
| 2013 – 2018 | Multinacional A | ~2.500.000 € | Software incompleto, abandonado |
| 2018 – 2022 | Multinacional B | ~1.500.000 € | Software inadecuado, abandonado |
| **Total** | | **~4.000.000+ €** | **Ningún software en producción** |

Cuatro millones de euros. Casi diez años de proyecto. Dos de los nombres más prestigiosos de la consultoría IT global.

Y al final, la empresa se encuentra exactamente en el punto de partida.

No es mala suerte. Es un patrón.\
Y quien lleva treinta años trabajando en este sector, como yo, lo reconoce a primera vista.

------------------------------------------------------------------------

## 🧠 Por qué sucede: la anatomía del fracaso

Este tipo de fracaso no es un accidente. Es el resultado previsible de un modelo de negocio con un defecto estructural.

**1. El incentivo está equivocado.**\
Una gran consultora gana dinero vendiendo jornadas-hombre. Cuanto más dura el proyecto, más factura. No hay ningún incentivo real para cerrar el proyecto rápido y bien. Hay un incentivo para mantenerlo vivo el mayor tiempo posible.

**2. La rotación es endémica.**\
Las multinacionales de consultoría tienen tasas de rotación del 15-25% anual. En un proyecto que dura cinco años, el equipo se renueva completamente al menos dos veces. Cada vez se empieza de nuevo: nueva curva de aprendizaje, nueva interpretación de los requisitos, nuevos errores.

**3. El know-how se va por la puerta ({{< glossary term="vendor-lock-in" >}}vendor lock-in{{< /glossary >}}).**\
Cuando el proveedor termina (o es despedido), el conocimiento del sistema se va con él. El cliente se queda con un software que no entiende, no sabe mantener y no puede evolucionar.

**4. Las especificaciones se convierten en un arma ({{< glossary term="scope-creep" >}}scope creep{{< /glossary >}}).**\
En un proyecto custom de esta envergadura, las especificaciones siempre están incompletas — porque el negocio es complejo y está en evolución. Esto se convierte en la coartada perfecta: "el software no funciona porque las especificaciones han cambiado". Y siempre es culpa de otro.

------------------------------------------------------------------------

## ✅ El punto de inflexión: comprar, no construir

Al final, tras casi una década y más de 4 millones quemados, la empresa toma la decisión que debería haber tomado desde el principio:

**Comprar un software de mercado ya funcional y adaptarlo internamente a sus necesidades.**

Un producto asegurador comercial, probado, con una base de código estable y una comunidad de soporte. Y un equipo interno — personas que conocen el negocio, que permanecen en la empresa, que acumulan conocimiento en lugar de dispersarlo — encargado de personalizarlo y evolucionarlo.

¿Coste? Una fracción de lo gastado en los diez años anteriores.\
¿Resultado? Un sistema que funciona. Que evoluciona. Que la empresa realmente posee.

La lección es brutal en su sencillez:

> No todo hay que construirlo desde cero. Y sobre todo, no todo hay que delegarlo en quien no tiene interés en terminar.

------------------------------------------------------------------------

## 🏗️ La comparación que duele: nuestro Data Warehouse

Y aquí entra la parte de la historia que conozco por dentro. Porque para la misma empresa, en el mismo período, un colega y yo construimos algo que funciona. Cada día.

Un {{< glossary term="data-warehouse" >}}**Data Warehouse**{{< /glossary >}} completo. Diseñado, desarrollado, puesto en producción y mantenido **por dos personas**.

No una demo. No un prototipo. Un sistema de producción que:

- **Carga datos cada día** — el ciclo {{< glossary term="etl" >}}ETL{{< /glossary >}} completo se ejecuta en **una hora y media**
- **Integra 4 sistemas fuente diferentes** — cada uno con su formato, su protocolo, sus particularidades
- **Recoge datos de 4 áreas geográficas**: Italia, Francia, países del Norte de Europa, España
- **Comprende aproximadamente 60.000 líneas de código** escritas a cuatro manos
- **La arquitectura fue diseñada por mí** — desde el modelo de datos a la estrategia de carga, desde la gestión de errores a la historización

| | Software de gestión custom | Data Warehouse |
|---|---|---|
| Equipo | Dos multinacionales (decenas de consultores) | **2 personas** |
| Duración del proyecto | ~10 años (y contando) | **3 años** |
| Presupuesto | 4.000.000+ € | Una fracción |
| Líneas de código | Desconocidas (y abandonadas) | **~60.000** (documentadas, mantenidas) |
| Resultado | Ningún software en producción | **Sistema en producción diaria** |
| Tiempo de procesamiento | — | **1h 30min / día** |
| Cobertura geográfica | — | 4 países, 4 sistemas fuente |
| Know-how | Perdido con cada cambio de proveedor | **Interno, estable, documentado** |

Dos personas. Tres años. Un sistema que cada mañana se despierta, recoge datos de cuatro rincones de Europa, los transforma, los carga y los pone a disposición para las decisiones empresariales. En una hora y media.

Sesenta mil líneas de código. Cada una pensada, probada, mantenida por quien la escribió.

Ningún PowerPoint. Ningún kickoff. Ningún consultor que se va llevándose el conocimiento.\
Solo competencia, arquitectura sólida y trabajo bien hecho.

------------------------------------------------------------------------

## 🎯 La lección

Cuando hablo con empresas que están a punto de emprender un gran proyecto IT, siempre les digo lo mismo:

**No paguéis por una marca. Pagad por las personas.**

Un equipo pequeño de profesionales que conocen el dominio, que permanecen en el proyecto, que son responsables del resultado — vale más que cien consultores rotando y facturando jornadas.

El software no se construye con diapositivas. Se construye con las manos en el código, con la arquitectura en la cabeza y con la responsabilidad encima.

Cuatro millones de euros en humo enseñan una sola cosa:

> El coste más alto no es el del proveedor equivocado que eliges.\
> Es el del tiempo que pierdes antes de entender que la solución era más sencilla de lo que te habían vendido.

------------------------------------------------------------------------

## 💬 A quien está a punto de firmar ese contrato

Si tu empresa está a punto de confiar un proyecto crítico a una gran consultora, detente un momento.

Pregúntate:

- ¿Quién escribirá el código? ¿Seguirá en la empresa dentro de dos años?
- Si el proveedor se va mañana, ¿sabríamos mantener el sistema?
- ¿Existe un producto de mercado que cubra el 80% de nuestras necesidades?
- ¿Podemos construir un equipo interno pequeño, competente y estable?

Las respuestas a estas preguntas valen más que cualquier propuesta comercial.\
Porque la diferencia entre un proyecto que funciona y uno que quema millones no está en la tecnología.

Está en las personas. En la continuidad. En la responsabilidad.

Y en la capacidad de decir "no" a quien te vende complejidad cuando la solución es simple.

------------------------------------------------------------------------

## Glosario

**[Data Warehouse](/es/glossary/data-warehouse/)** — Sistema centralizado de recopilación e historización de datos de fuentes diversas, diseñado para análisis y soporte a decisiones empresariales. En el caso descrito, construido por dos personas con 60.000 líneas de código.

**[ETL](/es/glossary/etl/)** — Extract, Transform, Load: proceso de extracción de datos de los sistemas fuente, transformación y carga en el data warehouse. El ciclo ETL del DWH descrito se ejecuta en una hora y media.

**[Vendor Lock-in](/es/glossary/vendor-lock-in/)** — Dependencia estructural de un proveedor externo que hace difícil cambiar de provider. Se instaura cuando el know-how y el código quedan en manos del proveedor.

**[Scope Creep](/es/glossary/scope-creep/)** — Expansión incontrolada de los requisitos del proyecto más allá del alcance inicial. Las especificaciones incompletas se convierten en la coartada para retrasos y costes adicionales.

**[Outsourcing](/es/glossary/outsourcing/)** — Externalización de actividades IT a proveedores externos. Arriesgado para proyectos estratégicos a largo plazo, donde la rotación de consultores y la pérdida de know-how pueden quemar millones.
