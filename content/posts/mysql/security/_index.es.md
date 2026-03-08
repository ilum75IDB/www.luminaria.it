---
title: "Access & Control"
description: "Seguridad en MySQL y MariaDB en la práctica: el modelo usuario@host, privilegios granulares y las trampas de un sistema de autenticación que vincula la identidad al origen de la conexión."
layout: "list"
image: "security.cover.jpg"
---
En MySQL la seguridad empieza antes de la contraseña.<br>
Empieza con la pregunta: ¿desde dónde te estás conectando?<br>

Porque MySQL no solo te pregunta quién eres.<br>
Te pregunta quién eres **y de dónde vienes**.<br>
Y la respuesta lo cambia todo: privilegios, acceso, incluso tu existencia como usuario.<br>

Es un modelo que otros motores de bases de datos no tienen.<br>
Es potente cuando lo dominas.<br>
Es una trampa cuando lo ignoras.<br>

Aquí encontrarás análisis sobre el modelo de autenticación de MySQL y MariaDB, los errores más comunes en la gestión de accesos y las diferencias operativas entre los dos motores que quien trabaja en producción necesita conocer.<br>

Porque la mayoría de los problemas de seguridad en MySQL no vienen de ataques externos.<br>
Vienen de usuarios creados sin entender cómo el motor los interpreta.
