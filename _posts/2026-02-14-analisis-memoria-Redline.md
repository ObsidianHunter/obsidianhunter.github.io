---
layout: post
title: "Análisis Forense de memoria RAM: Detección de Redline [Volatility3]"
date: 2026-02-14
categories: [dfir]
permalink: /trafico-redline/
---

# Análisis forense de Memoria RAM: Infección por Redline [Volatility3]

Hoy, os traigo un nuevo laboratorio de [CyberDefenders](https://cyberdefenders.org/blueteam-ctf-challenges/redline/). En este caso, vamos a analizar el volcado de la memoria RAM de un ordenador infectado con <span class="solo-color-neon">RedLine</span>. En este caso, vamos a usar <span class="solo-color-neon">volatility</span> para tal contenido. Entender <span class="solo-color-neon">volatility</span> no es muy difícil (te lo prometo) pero para hacer laboratorios de este tipo tienes que estar un poco fameliarizado con los permisos RWX, procesos padre/hijos sospechosos, conexiones de red anómalas... Son conceptos que se van cogiendo con la experiencia. Sin más dilación, vamos a empezar:

- <span class="solo-color-neon">Pregunta 1:</span> ¿Cómo se llama el proceso sospechoso?. He ejecutado el comando <span class="solo-color-neon">Windows.malware.malfind</span>, esto nos permite detectar inyección de procesos, de librerías dinámicas, process hollowing... me salen dos (uno es un <span class="solo-color-neon">falso positivo</span>, pertenece a Microsoft) y el otro que me sale es <span class="solo-color-neon">oneetx.exe</span>. Este último no sólo tiene permisos de lectura, ejecución y escritura, si no que también tiene un <span class="solo-color-neon">MZ</span>, <span class="solo-color-neon">MZ</span> es el indicativo de que no ha cargado comandos, ha cargado un ejecutable o una librería dinámica. Por lo tanto tenemos que no se ha cargado una pieza de código, <span class="solo-color-neon">ha inyectado el ejecutable directamente en la memoria de otro proceso.</span>

![Pregunta1](https://github.com/user-attachments/assets/6dd00744-6f04-47bf-a970-14b4bb926348)

- <span class="solo-color-neon">Pregunta 2:</span> ¿Cuál es el nombre del proceso hijo del proceso sospechoso?, como puede verse en el número del proceso padre, es <span class="solo-color-neon">rundll32.exe</span>. Que esté accediendo a <span class="solo-color-neon">rundll32</span> lo más probable es que esté intentando inyectar una dll suya y cargarla a través de este programa. Esto confirmaría, que es el malware que buscamos, está intentando usar herramientas <span class="solo-color-neon">living of the lands</span> para pasar desapercibido.

![Pregunta2](https://github.com/user-attachments/assets/92c6190f-62aa-4d94-ba74-ba69896deada)

- <span class="solo-color-neon">Pregunta 3:</span> ¿Cuál es la protección de memoria aplicada a la región de memoria del proceso sospechoso? Los permisos son de lectura, ejecución y escritura. Si ves algo de ejecución y escritura en la misma línea, bandera roja del tamaño de Madagascar. Normalmente, hay pocos programas que requieren los permisos de <span class="solo-color-neon">RWE</span> a la vez, como navegadores o instancias de Java, pero si no es ninguna de esas, deberías marcarlo como muy sospechoso siendo generosos. Normalmente, los procesos usan o <span class="solo-color-neon">lectura/escritura</span> o <span class="solo-color-neon">lectura/ejecución</span>, pero no las 3 al mismo tiempo.

![Pregunta3](https://github.com/user-attachments/assets/d0b7c01c-c1c3-4fd3-946c-7f5aa6047142)

- <span class="solo-color-neon">Pregunta 4:</span> ¿Cómo se llama el proceso responsable de la conexión VPN? Para esto voy a usar el comando <span class="solo-color-neon">netscan</span>, me va a permitir saber qué archivos han establecido conexiones de red. En este caso, detecto el nombre del proceso, <span class="solo-color-neon">tun2socks</span> (ya sólo por el socks lo sabía, pero el tun 2 parece algún tipo de túnel, justo lo que es una VPN). La cosa es que hubo un proceso que llamó a este archivo primero, para ello, quiero buscar el proceso padre del que inició el archivo del túnel. Si miro el número del proceso y luego examino el <span class="solo-color-neon">proceso padre</span>, me queda que el que ha llamado al proceso es un tal <span class="solo-color-neon">Outline.exe</span>.

![Pregunta4](https://github.com/user-attachments/assets/ab076180-a898-4428-a630-80235ac530a9)

![Pregunta4-1](https://github.com/user-attachments/assets/65df863a-29ae-4bcd-9947-040b9a39e40b)

- <span class="solo-color-neon">Pregunta 5:</span> ¿Cuál es la dirección IP del atacante? Ahora necesitamos encontrar la ip del atacante. Para ello, lo que hago es coger el proceso del bicho y sacar todas las conexiones de red que tiene con <span class="solo-color-neon">Windows.netscan</span>. Ahí puedo ver la ip del atacante.

![Pregunta5](https://github.com/user-attachments/assets/beef675e-0bab-428b-865e-9be691963e57)


- <span class="solo-color-neon">Pregunta 6:</span> ¿Cuál es la URL completa del archivo PHP que visitó el atacante? Para conocer esto, primero necesito descargar en mi ordenador todo lo que tenía la memoria del bicho. Para ello hago un <span class="solo-color-neon">python3 vol.py -f RedLine.mem -o dumpeo windows.memmap --pid 5896 --dump </span>

Creé una carpeta llamada dumpeo para volcarlo. Y ahora voy a analizar las cadenas que hay dentro. Uso el comandos <span class="solo-color-neon">strings</span> para analizar la memoria y busco las que contenga php. Van a salir unas cuantas, pero yo me enfoco en las que tengan la ip del atacante. Aquí descubrimos la respuesta: <span class="solo-color-neon">http://77[.]91[.]124[.]20/store/games/index[.]php</span>

![Pregunta6](https://github.com/user-attachments/assets/312a9c88-29ea-4c75-b794-a20db33c3a85)

Pregunta 7: ¿Cuál es la ruta completa del ejecutable malicioso? En este caso, es bastante fácil, porque con hacer el comando <span class="solo-color-neon">Windows.pstree</span> me va a salir la ruta completa del archivo: <span class="solo-color-neon">C:\Users\Tammam\AppData\Local\Temp\c3912af058\oneetx.exe</span> es sospechoso (ojo, hay algunos legítimos, como instaladores o archivos temporales) pero si además llama a <span class="solo-color-neon">rundll32</span> y se conecta a internet, es una bandera roja muy grande. Yo siempre suelo decirlo en otros posts, pero te lo vuelvo a decir: Para mí una ruta anormal + salida a internet + Persistencia = <span class="solo-color-neon">Archivo que hay que mirar detenidamente</span> (y no sólo con<span class="solo-color-neon"> virustotal</span>, la entropía, firma digital, fecha de creación, nombre original...)

Con esto termino el lab, la verdad es que todo lo que tiene que ver con Volatility es muy entretenido y útil.

![Pregunta7](https://github.com/user-attachments/assets/357f88eb-b753-47f9-9f5d-029809cd2ebc)

- Laboratorio creado: <span class="solo-color-neon">14/02/2026</span>

- Laboratorio subido: <span class="solo-color-neon">01/03/2026</span>
