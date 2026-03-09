---
layout: post
title: "Análisis Forense de memoria RAM: Detección de Ramnit [Volatility3]"
date: 2026-02-17
categories: [dfir]
permalink: /trafico-ramnit/
---

# Análisis forense de Memoria RAM: Infección por Ramnit [Volatility3]

- <span class="solo-color-neon">Material de blue team que busca instruir sobre la detección en memoria RAM con Volatility.</span>
- <span class="solo-color-neon">No me responsabilizo del mal uso de este laboratorio. Queda prohibido usar parte o la totalidad del tutorial para usarlo en ordenadores de terceros.</span>
- <span class="solo-color-neon">Aunque estén protegidas por corchetes, por favor, nada de clicar en las webs de este lab.</span>
- <span class="solo-color-neon">Gracias por respetar estas condiciones.</span>

Hoy os traigo un nuevo laboratorio de [CyberDefenders](https://cyberdefenders.org/blueteam-ctf-challenges/ramnit/). En este caso, vamos a analizar el volcado de la memoria RAM de un ordenador infectado con Ramnit, un malware tipo gusano que es todo un clásico por su capacidad de replicación y persistencia. Para este análisis, volveremos a usar Volatility como nuestra herramienta principal.

Entender Volatility no es muy difícil (te lo prometo), pero para sacarle el jugo a este lab hay que tener un poco más de ojo clínico. Aquí entran en juego conceptos como la reflective code injection, saber distinguir procesos legítimos de los que parecen serlo y, sobre todo, no fiarse de nada que haga conexiones de red extrañas (Pero nunca nunca).

- <span class="solo-color-neon">Pregunta 1:</span> ¿Cómo se llama el proceso responsable de la actividad sospechosa? 

Este malware ha sido realmente escurridizo. He buscado en la memoria <span class="solo-color-neon">los procesos padres e hijos</span>, procesos ocultos, he buscado con <span class="solo-color-neon">malfind</span> permisos <span class="solo-color-neon">RWX</span> y no daba con él. Todos parecían procesos legítimos. Hasta que me ha tocado filtrarlos uno por uno y hay uno que me llama la atención:

![Encontrararchivo](https://github.com/user-attachments/assets/f3befcc9-b4a8-4ec5-983e-e365a6524b32)

Ese <span class="solo-color-neon">Chrome setup</span> parece legítimo, pero es el único que me chirría. Investigando un poco, he visto que este malware suele usar <span class="solo-color-neon">reflective code injection</span>. Básicamente, en lugar de crear un proceso raro con un nombre sospechoso, inyecta su código directamente en la memoria de un proceso que parece inofensivo, como un instalador. Así engaña a herramientas básicas porque el proceso padre parece legíitimo, pero por dentro está ejecutando otra cosa (dios sabe qué).

Así que voy a investigarlo. Primero voy a ver qué conexiones tiene para ver dónde está yendo:

![Busarporip](https://github.com/user-attachments/assets/0eb82183-da0f-4e9b-a1b4-5bb19a803bd7)

La lógica me dice que, si un instalador de Google Chrome necesita conectarse a algo, debería ser a alguna <span class="solo-color-neon">IP de Google</span>, pero esa IP no parece de ellos. Voy a buscar de dónde es:

![LocalizandoUbicacion](https://github.com/user-attachments/assets/f4aeb5e6-287f-46d5-b523-22479800258e)

Pertenece a una infraestructura asiática que nada tiene que ver con <span class="solo-color-neon">Google</span>. No hay dudas, el malware es ese: chromesetup.exe. Esto me parece muy interesante, porque <span class="solo-color-neon">Ramnit</span> es del tipo gusano, así que lo mismo el <span class="solo-color-neon">chromesetup</span> era legítimo pero <span class="solo-color-neon">Ramnit</span> lo ha infectado con una inyección de archivo. También me parece interesante que en la captura de pantalla de volatility aparezca el <span class="solo-color-neon">syn_sent</span>. No se ve <span class="solo-color-neon">established</span> o <span class="solo-color-neon">listening</span>. Eso significa que, en el momento de la captura, el malware no estaba conectado al <span class="solo-color-neon">C2C</span>, estaba intentándolo pero no lo lograba. Eso no significa que no hubiera habido una conexión anterior, si no que en el momento que se tomó la captura, el malware no estaba siendo capaz de mandar las instrucciones ni recibirlas. ¿El hacker cerró el proceso al haber completado sus operaciones? ¿El analista cortó la conexión? ¿El firewall bloqueó la conexión? Bueno, creo que nunca lo sabremos.

- <span class="solo-color-neon">Pregunta 2:</span> ¿Cuál es la ruta exacta del ejecutable para el proceso malicioso? 

La ruta es: <span class="solo-color-neon">C:\Users\alex\Downloads\ChromeSetup.exe</span>

- <span class="solo-color-neon">Pregunta 3:</span> Identificar las conexiones de red es crucial para comprender la estrategia de comunicación del malware. ¿A qué dirección IP intentó conectarse el malware?

Ahora tenemos que ver a qué se intenta conectar. Ya con la lógica que he sacado antes, me sé la IP donde intenta conectarse: <span class="solo-color-neon">58[.]64[.]204[.]181</span>

- <span class="solo-color-neon">Pregunta 4:</span> Para determinar el origen geográfico específico del ataque, ¿qué ciudad está asociada con la dirección IP con la que se comunicó el malware?

No hay mucho más que añadir, la ciudad es <span class="solo-color-neon">Hong Kong</span>.

- <span class="solo-color-neon">Pregunta 5:</span> Los hashes sirven como identificadores únicos para los archivos y ayudan a detectar amenazas similares en diferentes máquinas. ¿Cuál es el hash SHA1 del ejecutable del malware?

Para sacar el <span class="solo-color-neon">hash</span>, primero debo descargar los archivos que había en la memoria:

![Volcado](https://github.com/user-attachments/assets/4db0516e-d131-4948-b880-87834bb8a1b9)

Como tengo muchos archivos, me voy a quedar con los <span class="solo-color-neon">.exe</span>:

![Volcado2](https://github.com/user-attachments/assets/9250fdfe-0b6c-4862-b925-2fb6011c2550)

Voy a sacar los <span class="solo-color-neon">hashes</span> de ambos, sólo uno será el que pide el ejercicio, pero el otro también está bien tenerlo para reforzar las defensas:

![Hashes](https://github.com/user-attachments/assets/003fdc33-25cc-4c76-91fb-c5c5d8a35f73)

La respuesta es el de img: <span class="solo-color-neon">280c9d36039f9432433893dee6126d72b9112ad2</span>

- <span class="solo-color-neon">Pregunta 6:</span> Examinar el cronograma de desarrollo del malware puede proporcionar información sobre su implementación. ¿Cuál es la marca de tiempo de compilación del malware?

Ahora nos toca ir a <span class="solo-color-neon">VirusTotal</span> con este hash para saber cuándo fue compilado. Es bastante famoso:

![Virustotal1](https://github.com/user-attachments/assets/4c5cfe6f-c360-442b-bfb5-d8fbc83ec404)

En la pestaña <span class="solo-color-neon">details</span> de <span class="solo-color-neon">VirusTotal</span> lo tenemos:

![Virustotal2](https://github.com/user-attachments/assets/67ae7a10-eabf-4ee9-a352-41ae56f7efd3)

- <span class="solo-color-neon">Pregunta 7:</span> Identificar los dominios asociados con este malware es crucial para bloquear futuras comunicaciones maliciosas y detectar cualquier interacción en curso con esos dominios dentro de nuestra red. ¿Puedes proporcionar el dominio conectado al malware?

Ahora nos pregunta por el dominio al que se conectó el malware. Si vamos a<span class="solo-color-neon"> VirusTotal</span>, el dominio al que se conectó fue: <span class="solo-color-neon">dnsnb8[.]net</span>:

![Virusotal3](https://github.com/user-attachments/assets/4df2ef78-f958-450d-bc38-46947a6f680d)

Esto trae una moraleja considerable... <span class="solo-color-neon">Nunca hay que fiarse de los ejecutables</span>, aplica <span class="solo-color-neon">zerotrust</span> siempre que puedas, porque puedes estar viendo un proceso legítimo con código inyectado, eso haría que te estén exfiltrando los datos sin que tú te des cuenta hasta que sea demasiado tarde. En ciberseguridad, creo yo, lo mejor es la <span class="solo-color-neon">desconfianza.</span>

- Laboratorio creado: <span class="solo-color-neon">17/02/2026</span>

- Laboratorio subido: <span class="solo-color-neon">03/03/2026</span>
