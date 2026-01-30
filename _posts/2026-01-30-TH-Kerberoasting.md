---
layout: post
title: "Caza Proactiva en Directorio Activo: Detección y contención de Kerberoasting."
date: 2026-01-30
categories: [threat-hunting]
permalink: /TH-Kerberoasting/
---

# Caza Proactiva en Directorio Activo: Detección, contención y mitigación de Kerberoasting.

Hoy vengo a hablar de algo que, por lo menos a mí, me encanta y es sobre el protocolo <span class="solo-color-neon">Kerberos</span>. Creo que entender todo lo que pasa en <span class="solo-color-neon">Active Directory</span> es esencial, porque muchos de los <span class="solo-color-neon">movimientos laterales</span> y <span class="solo-color-neon">escalada de privilegios</span> ocurren allí. En esta ocasión, quiero hacer una pequeña introducción a <span class="solo-color-neon">Kerberos</span>, uno de los sistemas de autenticación más usados y seguros en entornos <span class="solo-color-neon">Active DIrectory</span> (Windows). Para que quede un poco claro, viene siendo algo parecido a esto:

- Si tú eres un usuario autenticado en el dominio, el <span class="solo-color-neon">KDC</span> (que es el centro de distribución de llaves) te da un <span class="solo-color-neon">TGT</span>, que es un ticket de larga duración. Te permite estar e interactuar con los servicios del dominio sin tener que estar metiendo la contraseña cada X tiempo.

- Si quieres acceder a un servicio, enseñas tu <span class="solo-color-neon">TGT</span> al <span class="solo-color-neon">KDC</span> (más exactamente al <span class="solo-color-neon">Ticket Granting Service</span>, que vive dentro del <span class="solo-color-neon">KDC</span>, aquí hay que hilar muy fino) y este te dará un <span class="solo-color-neon">TGS</span>, que es un ticket específico para ese servicio.

- Al introducir el <span class="solo-color-neon">TGS</span>, si tu ticket es correcto, se validará en el servicio, permitiéndote entrar. (Cualquier usuario, incluso sin privilegios, puede pedir tickets <span class="solo-color-neon">TGS</span>)

Y ahora vamos con <span class="solo-color-neon">Kerberoasting</span> (el nombre suena bien, la finalidad es pura maldad). ¿Recuerdas que antes dije que cualquier usuario del dominio, incluso sin privilegios, puede pedir tickets <span class="solo-color-neon">TGS</span>? Pues aquí reside realmente la potencia de ataque del <span class="solo-color-neon">Kerberoasting</span>. Un hacker tiene que estar dentro del dominio para poder obtener los tickets, o está él con su ordenador dentro o ha infectado un ordenador del dominio (por eso es una técnica de <span class="solo-color-neon">post-explotación</span>). Si mapeamos <span class="solo-color-neon">Mitre Att&ck</span>, vemos que la táctica es <span class="solo-color-neon">Credentials Access</span> (TA0006) con la técnica <span class="solo-color-neon">T1558.003</span> (. Al estar autenticado, ya tiene un ticket <span class="solo-color-neon">TGT</span> (normalmente con privilegios bajos, tiene que esforzarse). Ahora el hacker puede pedir <span class="solo-color-neon">TGS</span> para cualquier servicio que tenga un <span class="solo-color-neon">SPN</span> registrado, asociado a una cuenta que generalmente se creó con privilegios muy altos y que suelen tener acceso a datos <span class="solo-color-neon">muy sensibles</span> (por ejemplo, una base de datos, archivos confidenciales... Cosas que le gustan a los hackers). Normalmente hay que ir cambiando las contraseñas, pero hay administraciones que no las tocan en meses y eso es un error fatal.

Aquí viene lo interesante: los datos del ticket <span class="solo-color-neon">TGS</span> están cifrados y la clave para descifrarlo es el hash <span class="solo-color-neon">NTLM</span> (o la clave derivada en <span class="solo-color-neon">AES</span>, normalmente 256, pero no está de más mirarlo en la configuración) de la contraseña de la cuenta asociada al <span class="solo-color-neon">SPN</span>.  Si el atacante crackea el ticket offline y obtiene la contraseña o el <span class="solo-color-neon">hash</span>, ya puede autenticarse como esa cuenta de servicio y hacer acciones como loguearse, pedir tickets nuevos, acceder a los recursos que esa cuenta controla (lo que dijimos antes de que le gustan a los hackers) y <span class="solo-color-neon">escalar privilegios</span>. Por eso, <span class="solo-color-neon">kerberoasting</span> es tanto<span class="solo-color-neon"> movimiento lateral</span> como <span class="solo-color-neon">escalada de privilegios</span>, y en un <span class="solo-color-neon">directorio activo</span>, no te interesa para nada ninguna de esas dos cosas.

Y te estarás preguntando, ¿cómo se detecta un <span class="solo-color-neon">kerberoasting</span>? La respuesta es depende. Depende de si tu <span class="solo-color-neon">IDS</span> tiene logs para <span class="solo-color-neon">kerberos</span>, si tienes habilitado el visor de eventos, si estás usando <span class="solo-color-neon">wireshark</span>… Yo en este caso, tengo un dataset que ingeriré con <span class="solo-color-neon">splunk</span> (bien ordenadito con sus columnas bien separadas) para detectarlo, hay 3 maneras principales de pillar un <span class="solo-color-neon">kerberoasting</span>:

- Ves en los logs que el cifrado no es <span class="solo-color-neon">0x12</span> que corresponde a <span class="solo-color-neon">AES-256</span>, si no que cambia a <span class="solo-color-neon">0x17</span> (RC4) (o <span class="solo-color-neon">0x18</span>). ¿Por qué? Porque <span class="solo-color-neon">RC4</span> es mucho más fácil de crackear. Sí, sé lo que piensas, que ambos tienen la misma contraseña que los protege y entonces daría igual, pero lanzar un fuerza bruta con <span class="solo-color-neon">AES-256</span> es, computacionalmente hablando, mucho más costoso que <span class="solo-color-neon">RC4</span>. Puedes tener una contraseña débil con <span class="solo-color-neon">RC4</span> en segundos, pero en <span class="solo-color-neon">AES-256</span> te puede llevar meses. Que alguien esté forzando a una encriptación <span class="solo-color-neon">RC4</span> es una bandera muy roja. Puede haber algún falso positivo con dispositivos antiguos que no soporten <span class="solo-color-neon">AES-256</span> y pidan <span class="solo-color-neon">RC4</span>, pero no es para nada lo normal.

- Ves en los logs peticiones muy rítmicas o muy rápidas a varios <span class="solo-color-neon">SPN</pan>. Normalmente puedes ver que en 1 minuto se han pedido <span class="solo-color-neon">TGS</span> para 50 <span class="solo-color-neon">SPN</span>, cosa que indica un ataque automatizado, ningún humano es tan rápido (sería el Usain Bolt de la mecanografía). Para ello usan herramientas como <span class="solo-color-neon">Rubeus</span> o <span class="solo-color-neon">Impacket</span>). Consultas a varios <span class="solo-color-neon">SPN</span> está indicando claramente que está haciendo un reconocimiento de los <span class="solo-color-neon">SPN</span> disponibles, aquí 100% debes mirar porque con toda garantía es un ataque.

- Ves eventos en el visor con el número <span class="solo-color-neon">4769</span>: Esto es una forma de cazar, el evento <span class="solo-color-neon">4769</span> salta cuando alguien pide un <span class="solo-color-neon">TGS</span>. Puede ser un administrador legítimo que intente acceder al servicio o, por el contrario, puede ser un oficinista que está intentando entrar en un <span class="solo-color-neon">SPN</span> que no corresponde con su rol... Sólo uno es el <span class="solo-color-neon">falso positivo</span> y es el primero.

Y es así como hoy, con splunk (lamento el fondo blanco, desde hace unas semanas ya no me deja cambiarlo), voy a cazar un <span class="solo-color-neon">kerberoasting</span>. Recuerda las premisas y vamos a zambullirnos. Voy a ver mi log y ordenar los eventos de manera ascendente:

![Imagen1](https://github.com/user-attachments/assets/f4d2326a-60e8-4faf-a8e9-08bcf99442fe)


Vamos a aplicar un filtro muy sencillo, en base a lo que hemos visto: muchos dominios consultados + cifrado <span class="solo-color-neon">0x17</span>= <span class="solo-color-neon">Kerberoasting</span>. Primero aplico el filtro a las ips que han consultado 3 o más dominios:

![Imagen2](https://github.com/user-attachments/assets/76ae7c15-0797-48a1-a57c-cf48d629b31e)


Me salen unas cuantas, vamos a filtrar por el tipo de cifrado:

![Imagen3](https://github.com/user-attachments/assets/c8917e75-fd85-447d-9188-6847c1cf7928)


Me sale una sóla ip, voy a verla de manera más profunda:

![Imagen4](https://github.com/user-attachments/assets/60e4967d-85ce-49d2-acb9-efaf3dba2eeb)


Fíjate en la columna <span class="solo-color-neon">Time</span>, hay ráfagas de consulta cada 15 segundos, eso no lo hace un humano (el humano es más aleatorio), eso lo hace un <span class="solo-color-neon">script automatizado</span>. Ha hecho ráfagas perfectas de tiempo, está intentando evitar la detección, seguramente si hubiera visto esas 3 en un segundo hubieran saltado las alarmas, pero al hacer un ataque <span class="solo-color-neon">Low and Slow</span> intenta pasar desapercibido. No es un dispositivo antiguo, es un hacker pidiendo <span class="solo-color-neon">TGS</span> de los <span class="solo-color-neon">SPN</span> del equipo, en ráfagas de 15 segundos y pidiéndolo con un cifrado <span class="solo-color-neon">RC4</span>, para que sea más crackeable:

Estas consultas pueden generar <span class="solo-color-neon">falsos positivos</span>. ¿Y si un hacker sólo ha pedido un <span class="solo-color-neon">TGS</span>? Evadiría nuestra búsqueda. Yo, en mi caso, lo que haría sería primero filtrar por el tipo de evento (recuerda, <span class="solo-color-neon">4769</span>) y por el cifrado. Ahí deberían salir sólo los eventos kerberoasting o los falsos positivos de los dispositivos antiguos. Si tienes un poco de, como le dicen en el mundillo, adversarial mindset, si yo hiciera ese ataque y no quisiera ser detectado, no haría una enumeración tan ruidosa, me quedaría con uno para evadir el conteo, sin embargo, me seguiría pillando el tipo de cifrado. En el caso de que el cifrado no me delatara, la única manera de cazarlo sería por intuición (es decir, qué hace alguien de marketing intentando acceder a SPN de administrador como NóminasAdmin, cuando realmente ese SPN no corresponde a su actividad? Eso, sin el cifrado y sin el conteo, me harían ver que hay un hacker en la red ya que el comportamiento del usuario no es el esperado. Podría ser un falso positivo, que algún administrador esté ejecutando el ordenador de forma remota y necesite acceder a ese SPN, pero no es lo normal ni lo habitual, es algo que debes investigar.

Ahora, ¿cómo se contiene un ataque Kerberoasting?

Cada compañía es diferente, pero al ser una máquina de Marketing, es muy probable que se pueda aislar. La primera medida es justamente esa, necesitamos apartarla de nuestro dominio, no queremos que vaya saltando de un lado para otro o haciendo escaladas verticales. También es muy importante rotar las contraseñas de las cuentas de servicio asociadas a esos <span class="solo-color-neon">SPN</span>. Una vez hecho eso, deshabilita la cuenta de marketing2 (siempre que <span class="solo-color-neon">no sea un riesgo para la empresa</span>). Y sobre todo, pero especialmente y muy importante para que no te pase, fuerza a que todos los servicios sólo den el cifrado <span class="solo-color-neon">AES-256</span> (Creo que Microsoft ya está implementando que sólo se pueda ese ccifrado), pero por si acaso, verifícalo.

Recolecta la memoria del equipo infectado, es muy importante para el análisis posterior, ya que podríamos identificar el acceso inicial al ordenador infectado. También mirar los evento <span class="solo-color-neon">4624</span>, muy importante para saber si el usuario a entrado en el ordenador vía física, <span class="solo-color-neon">RDP</span>, <span class="solo-color-neon">ssh</span>… 

Recuerda que si en la empresa o en tu dominio hay dispositivos que aún necesitan el RC4 tendrás un dilema, el de la seguridad vs disponibilidad. ¿Estás dispuesto a invertir en equipos nuevos? ¿Hasta qué punto esos dispositivos te convienen que estén ahí sabiendo que son una brecha de seguridad importante?

- <span class="solo-color-neon">Laboratorio creado:</span> 30/01/2026
- <span class="solo-color-neon">Laboratorio subido:</span> 30/01/2026
