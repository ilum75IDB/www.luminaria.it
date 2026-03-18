---
title: "La técnica del Sí-Y: cómo evité una discusión que estaba a punto de estallar"
description: "La técnica del Yes-And, nacida en el teatro de improvisación, aplicada a la gestión de conflictos en equipos IT. Un caso real de una reunión que estaba degenerando — y cómo tres palabras lo cambiaron todo."
date: "2026-01-13T10:00:00+01:00"
draft: false
translationKey: "tecnica_si_e_yes_and"
tags: ["conflict-management", "team-leadership", "communication", "meeting"]
categories: ["Project Management"]
image: "tecnica-si-e-yes-and.cover.jpg"
---

Era un jueves por la tarde, una de esas reuniones que sobre el papel debía durar una hora. Éramos siete, conectados en una llamada. El orden del día era sencillo: decidir la estrategia de migración de una base de datos Oracle de on-premise a cloud.

Sencillo, claro. Sobre el papel.

A los veinte minutos, la reunión se había convertido en un duelo.

------------------------------------------------------------------------

## 🔥 La chispa

Por un lado estaba el responsable de infraestructura. Hombre de experiencia, veinte años de datacenters a sus espaldas. Su posición era granítica: **migración lift-and-shift, cero cambios en la arquitectura, se lleva todo tal como está**.

Por el otro, el lead developer. Joven, brillante, con las ideas claras. Quería reescribir la capa aplicativa, adoptar servicios cloud-native, containerizar todo. **Lo rehacemos desde cero, total el código es viejo.**

Dos posiciones legítimas. Dos perspectivas reales. Dos personas inteligentes.

Pero la conversación había tomado un rumbo familiar — y peligroso.

"No, no tiene sentido llevar todo a cloud sin repensar la arquitectura."\
"No, reescribir todo es un riesgo enorme y no tenemos presupuesto."

No. No. No.

Cada frase empezaba con "no". Cada respuesta era una negación de la anterior. Brazos cruzados, el tono subiendo, las frases acortándose. Conozco ese patrón. Lo he visto cientos de veces. Y sé cómo termina: no termina. La reunión se cierra sin decisión, se pospone a la semana siguiente, y mientras tanto nadie hace nada porque "todavía no hemos decidido".

El proyecto se detiene. No por motivos técnicos. Por orgullo.

------------------------------------------------------------------------

## 🎭 Tres palabras que lo cambian todo

En ese momento hice algo muy sencillo. Esperé una pausa — porque en las discusiones acaloradas siempre hay un momento en que todos toman aire — y dije:

> "Marco, tienes razón: llevar todo a cloud sin cambiar nada es la vía más rápida para llegar a producción. **Y** podríamos identificar dos o tres componentes que, durante la migración, tiene sentido repensar en clave cloud-native. Luca, ¿cuáles elegirías tú?"

Nadie dijo "no". Nadie fue contradicho.

Marco vio validada su posición — su enfoque conservador era el punto de partida. A Luca se le ofreció un papel concreto — elegir qué modernizar, con un mandato preciso.

En treinta segundos, dos personas que estaban discutiendo se encontraron colaborando sobre la misma pizarra.

La reunión terminó antes de lo previsto. Con una decisión. Una de verdad.

------------------------------------------------------------------------

## 🧠 Qué es la técnica del "Sí-Y"

Lo que hice tiene un nombre. Se llama **"Yes-And"** — en español, **"Sí-Y"**. Viene del teatro de improvisación, donde hay una regla fundamental: **nunca negar la propuesta de tu compañero de escena**.

Si alguien dice "Estamos en un barco en medio del océano", no respondes "No, estamos en una oficina". Respondes "Sí, y parece que se acerca una tormenta". Construyes. Añades. Avanzas.

En la gestión de proyectos funciona del mismo modo.

Cuando alguien propone algo y respondes "No, pero...", esto es lo que pasa a nivel psicológico:

- el interlocutor se pone a la defensiva
- deja de escuchar lo que viene después del "pero"
- se concentra en cómo rebatir, no en cómo resolver
- la conversación se convierte en un ping-pong de negaciones

Cuando respondes "Sí, y...", sucede lo contrario:

- el interlocutor se siente reconocido
- baja las defensas
- se vuelve receptivo a tu aportación
- la conversación se vuelve constructiva

No es manipulación. No es diplomacia vacía. Es una técnica precisa para hacer avanzar las decisiones sin quemar las relaciones.

------------------------------------------------------------------------

## 🛠️ Cómo funciona en la práctica diaria

En treinta años de proyectos, he aplicado el "Sí-Y" en decenas de situaciones. Funciona dondequiera que haya una decisión que tomar y varias personas con opiniones diferentes.

### En las reuniones de proyecto

**En lugar de:** "No, la timeline de tres meses es irreal."\
**Prueba con:** "Sí, tres meses es el objetivo. Y para llegar, deberíamos reducir el alcance del primer lanzamiento a estas tres funcionalidades — las demás las ponemos en la fase dos."

¿Notas la diferencia? En la primera versión tienes un muro. En la segunda tienes un plan.

### En las code reviews

**En lugar de:** "No, este enfoque está mal, lo has escrito de forma demasiado complicada."\
**Prueba con:** "Sí, funciona. Y podríamos simplificarlo extrayendo esta lógica a un método separado — se vuelve más testeable."

El desarrollador no se siente atacado. Se siente ayudado. Y la próxima vez viene a pedirte opinión *antes* de escribir el código, no después.

### En las negociaciones con stakeholders

**En lugar de:** "No, no podemos añadir esa feature ahora, ya vamos retrasados."\
**Prueba con:** "Sí, esa feature tiene sentido. Y para incluirla sin comprometer la fecha de entrega, deberíamos sustituirla por esta otra que es menos prioritaria. ¿Cuál de las dos preferís?"

El stakeholder no oye un "no". Oye un "sí, y ahora decidimos juntos cómo hacerlo".

------------------------------------------------------------------------

## ⚠️ Cuándo el "Sí-Y" no funciona

Sería bonito decir que funciona siempre. No es así. Hay situaciones en las que el "Sí-Y" es la herramienta equivocada.

**Problemas de seguridad.** Si alguien propone quitar la autenticación de la base de datos de producción porque "ralentiza las queries", la respuesta no es "Sí, y...". La respuesta es "No. Punto."

**Violaciones de proceso.** Si un desarrollador quiere hacer deploy en producción el viernes por la noche sin tests, no hay "Sí-Y" que valga. Hay un proceso, y hay que respetarlo.

**Deadlines no negociables.** Cuando el go-live es el lunes y estamos a jueves, no es el momento de construir sobre las ideas de todos. Es el momento de decidir, ejecutar y cerrar.

**Comportamientos tóxicos.** El "Sí-Y" funciona con personas de buena fe que tienen opiniones diferentes. No funciona con quien solo quiere tener razón, con quien boicotea, con quien no escucha por principio. En esos casos hace falta otro tipo de conversación — privada, directa y muy franca.

La técnica no es una fórmula mágica. Es una herramienta. Y como todas las herramientas, hay que saber cuándo usarla y cuándo dejarla.

------------------------------------------------------------------------

## 📊 El coste oculto del "No, pero..."

Intenté hacer un cálculo aproximado en un proyecto que gestioné hace dos años. Un equipo de ocho personas, reuniones tres veces por semana.

| Situación | Duración media de la reunión | Decisiones tomadas |
|---|---|---|
| Antes (cultura del "No, pero...") | 1h 20min | 0.5 por reunión |
| Después (cultura del "Sí, y...") | 45min | 1.8 por reunión |

El equipo tomaba decisiones tres veces más rápido y las reuniones duraban casi la mitad.

No tengo datos científicos. Son datos empíricos, recogidos en un proyecto específico. Pero el patrón es coherente con lo que he visto en veinte años: los equipos que discuten de forma constructiva van más rápido que los que pelean. No porque eviten el conflicto — porque lo atraviesan mejor.

------------------------------------------------------------------------

## 🎯 Lo que he aprendido

El "Sí-Y" no es diplomacia. No es evitar el enfrentamiento. No es decir sí a todo.

Es reconocer que **la mayoría de las discusiones en los proyectos IT no tratan de quién tiene razón**. Tratan de cómo hacer que las cosas avancen. Y las cosas avanzan cuando las personas se sienten escuchadas, no cuando son derrotadas.

He visto proyectos bloquearse durante semanas porque dos personas brillantes no conseguían dejar de decirse "no" mutuamente. Y he visto esos mismos proyectos desbloquearse en quince minutos cuando alguien tuvo el buen sentido de decir "sí, y...".

No hace falta un curso de comunicación. No hace falta un coach. Hace falta probar, la próxima vez que alguien diga algo con lo que no estés de acuerdo, a responder "Sí, y..." en lugar de "No, pero...".

Un ejercicio sencillo. Que cambia la forma en que se toman las decisiones.\
Que cambia la forma en que las personas trabajan juntas.\
Y que, a veces, salva una reunión que estaba a punto de estallar.

------------------------------------------------------------------------

## 💬 Para quien le haya pasado al menos una vez

Si alguna vez has estado en una reunión donde dos personas hablaban a la vez y nadie escuchaba a nadie. Si alguna vez has visto un proyecto bloquearse no por un problema técnico, sino por un problema de comunicación. Si alguna vez has pensado "¿por qué no podemos simplemente decidir?"

Prueba el "Sí-Y". La próxima reunión. Una sola vez.

No cuesta nada. No necesita aprobación. No requiere presupuesto.\
Solo requiere la capacidad de contenerte un segundo antes de decir "no" — y sustituirlo por "sí, y...".

El resultado podría sorprenderte.

------------------------------------------------------------------------

## Glosario

**[Yes-And](/es/glossary/yes-and/)** — Técnica de comunicación nacida en el teatro de improvisación que sustituye el "No, pero..." por "Sí, y...", transformando las discusiones en construcción colaborativa.

**[Stakeholder](/es/glossary/stakeholder/)** — Persona o grupo con un interés directo en el resultado de un proyecto: cliente, usuario final, sponsor, equipo técnico o cualquier parte afectada por las decisiones del proyecto.

**[Scope](/es/glossary/scope/)** — Perímetro de un proyecto que define qué está incluido y qué excluido: funcionalidades, entregables, restricciones y límites acordados con los stakeholders.

**[Lift-and-Shift](/es/glossary/lift-and-shift/)** — Estrategia de migración que traslada un sistema de un entorno a otro sin modificar su arquitectura, código o configuración.

**[Timeboxing](/es/glossary/timeboxing/)** — Técnica de gestión del tiempo que asigna un intervalo fijo y no negociable a una actividad, forzando la conclusión dentro del límite establecido.

El resultado podría sorprenderte.
