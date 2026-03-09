---
layout: post
title: "Análisis Forense de Red: Intrusión y Acceso Inicial de DanaBot en Tráfico PCAP"
date: 2026-01-23
categories: [dfir]
permalink: /pcap-danabot/
---

# Análisis Forense de Red: Intrusión y Acceso Inicial de DanaBot en Tráfico PCAP

Hoy voy a documentar el laboratorio del malware DanaBot de la página <span class="solo-color-neon">[CyberDefenders](https://cyberdefenders.org/blueteam-ctf-challenges/danabot/)</span>. La misión es realizar un análisis forense de la red con <span class="solo-color-neon">Wireshark</span> a partir de un archivo de tráfico capturado para reconstruir la fase inicial de la infección.

<span class="solo-color-neon">DanaBot</span>, hoy en día, es un troyano bancario de tipo <span class="solo-color-neon">Malware as a Service</span> (Lo verás mucho como MaaS), conocido por sus capacidades de robo de credenciales, espionaje y envío de spam. En este análisis me voy a enfocar en los <span class="solo-color-neon">IOCs </span> del pcap y en el análisis de la intrusión.


<span class="solo-color-neon">Pregunta 1) Me pide identificar la ip del atacante en el acceso inicial. </span>

Voy a iniciar la investigación, lo primero que quiero lograr ver es el <span class="solo-color-neon">ordenador infectado de la red local</span> (Por algo hay que empezar). Siempre suelo usar un método bastante seguro y es que el host comprometido suele presentar el mayor volumen de tráfico saliente o actividad anómala, voy a ver las estadísticas en Wireshark con <span class="solo-color-neon">Statistics -> Endpoints -> IPv4</span>, ordenando por el volumen de bytes de forma decreciente. Aquí ya veo realmente cuál es la ip del ordenador infectado.

![Wireshark1](https://github.com/user-attachments/assets/63281099-145e-4516-a672-22d0044e736a)


Aislando el ruido, identifiqué dos direcciones IP externas que me huelen a chamusquina. Mientras que una pertenecía a Google, la IP <span class="solo-color-neon">62[.]173[.]142[.]148</span> era la única sospechosa. Apliqué el filtro  <span class="solo-color-neon">http.request</span> y añadí la columna <span class="solo-color-neon">URI</span>.

![Wireshark1-2](https://github.com/user-attachments/assets/b8548d30-93d6-4e2e-b8a2-5989e5698941)


Aquí veo por ejemplo <span class="solo-color-neon">serveirc</span>, que es un servicio <span class="solo-color-neon">DNS </span> dinámico que los malwares adoran, porque se pueden reconectar muy rápido si bloquean su IP. También veo un <span class="solo-color-neon">login.php</span>. No es un login cualquiera, porque en el mundillo de los atacantes es un método de autenticación del malware para hacer el <span class="solo-color-neon">C2</span>. En la anterior captura vi que el atacante envía muchísimos bytes a la víctima, muy probablemente le está pasando un regalo de cumpleaños atrasado o el malware que quiere poner (¿Quién sabe cuál será?). Por lo tanto, confirmo que la IP es -> <span class="solo-color-neon">62[.]173[.]142[.]148 </span>

![Wireshark1-3](https://github.com/user-attachments/assets/ee6b8b9b-be7a-43ec-a756-5d986c732a36)


<span class="solo-color-neon">Pregunta 2) Aquí me pide el archivo malicioso usado para el acceso inicial.</span>

Ahora voy a analizar el flujo del tráfico con <span class="solo-color-neon">stream follow http</span>, voy a reconstruir la relación cliente-malhechor. Detecté la descarga de un archivo denominado <span class="solo-color-neon">allegato_708.js</span> (Suena muy italiano, ¿no?). La extensión de este archivo es de <span class="solo-color-neon">JavaScript</span> lo cual es un vector de infección común utilizado para ejecutar código malicioso en el sistema operativo host. El archivo es -> <span class="solo-color-neon">allegato_708.js.</span>

![Wireshark2](https://github.com/user-attachments/assets/e38838c5-422d-46fc-b2f8-291e7954e521)


Pregunta 3) Ahora nos toca obtener la firma digita con el <span class="solo-color-neon">hash sha-256</span>, utilicé técnicas de <span class="solo-color-neon">OSINT</span> consultando la plataforma <span class="solo-color-neon">Any.run</span>, el cual me confirma que es un archivo que pertenece a <span class="solo-color-neon">DanaBot</span>. El hash es -> <span class="solo-color-neon">847B4AD90B1DABA2D9117A8E05776F3F902DDA593FB1252289538ACF476C4268</span>

![Wireshark3](https://github.com/user-attachments/assets/28d75054-9201-4898-a6c6-a6265797be94)


Pregunta 4) La pregunta 4 es más fácil, porque nos piden que qué proceso fue usado para ejecutar el archivo malicioso. Normalmente, la gente piensa que los archivos Javascript sólo se pueden ejecutar en el navegador, pero eso sería muy ruidoso para un malware. Para que se vea más legítimo, usa un archivo llamado <span class="solo-color-neon">wscript.exe</span>, que viene con Windows (<span class="solo-color-neon">Living off the Lands</span>, querido). Con eso, está usando un programa totalmente legítimo que no levanta sospechas, es mucho más útil. Así que el la respuesta es -> <span class="solo-color-neon">wscript.exe</span>

Pregunta 5) Sigo con el análisis, nos piden la etensión del segundo archivo malicioso usado por el atacante, ahora me voy al menú <span class="solo-color-neon">File -> Export Objects -> HTTP</span>. Entre todo lo extraído pude ver un archivo con extensión .dll. Parece ser la segunda fase de la infección, muy probablemente usará la dll para una inyección de dll (valga la redundancia) o inyección de comandos. La respuesta es -> <span class="solo-color-neon">.dll </span>

![Wireshark5](https://github.com/user-attachments/assets/1e2e2189-65cc-4dee-92f9-ada3cf4167bb)
![Wireshark5-1](https://github.com/user-attachments/assets/91b7ad8d-bfb3-4a8b-ac9e-dc8ec930242c)


Pregunta 6) Nos piden ahora calcular el hash <span class="solo-color-neon">MD5</span> del archivo de la anterior respuesta. Para calcular el hash del archivo, primero debo descargar el archivo. Voy a la misma sección de antes y le doy a <span class="solo-color-neon">save</span> para guardar el archivo. Luego, sólo me queda calcular el hash MD5 y ponerlo -> <span class="solo-color-neon">e758e07113016aca55d9eda2b0ffeebe</span>

![Wireshark6](https://github.com/user-attachments/assets/c4ccc566-b5f3-4da3-b751-3f7e90aadb6e)

- <span class="solo-color-neon">Laboratorio creado:</span> 15/01/2026
- <span class="solo-color-neon">Laboratorio publicado:</span> 23/01/2026
