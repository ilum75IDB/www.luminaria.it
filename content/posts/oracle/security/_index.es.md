---
title: "Access & Control"
date: "2026-03-10T10:00:00+01:00"
description: "Seguridad Oracle en la práctica: usuarios, roles, privilegios y auditoría. El modelo de seguridad más granular del mercado — y el más fácil de configurar mal."
layout: "list"
image: "security.cover.jpg"
---
En Oracle la seguridad no es una opción.<br>
Es una estructura portante de la base de datos, presente desde el primer día.<br>

El modelo de seguridad de Oracle es probablemente el más completo entre las bases de datos relacionales: usuarios separados de los esquemas (pero vinculados), roles predefinidos y personalizados, privilegios de sistema y de objeto, perfiles de recursos, auditoría unificada.<br>

¿Potente? Enormemente.<br>
¿Mal configurado en el 90% de las instalaciones que he visto? También.<br>

El problema no es la falta de herramientas. Es el exceso. Oracle ofrece tantas opciones de seguridad que la tentación es simplificar con un `GRANT DBA` y pasar al siguiente problema. Lo he visto hacer a desarrolladores, administradores de sistemas, incluso DBAs con años de experiencia.<br>

En esta sección cuento cómo se diseña la seguridad Oracle en entornos reales: separación de roles, principio del privilegio mínimo, auditoría y las trampas en las que caen incluso los profesionales experimentados.<br>

Porque en Oracle la seguridad no se añade después.<br>
Se diseña antes. O se paga después.
