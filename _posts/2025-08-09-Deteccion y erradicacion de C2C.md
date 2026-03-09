---
layout: post
title: "Detección, Contención, Erradicación y Ticketing Soc de un Command and Control [Zeek + Wireshark]"
date: 2025-08-09
categories: [blue-team]
permalink: /deteccion-c2c/
---

En el incidente de hoy voy a hacer un laboratorio algo más especial. Voy a <span class="solo-color-neon">detectar, contener, erradicar y hacer un ticketing muy típico en SOC</span>. EL incidente comienza con una monitorización típica de los logs de <span class="solo-color-neon">Zeek</span> (post no apto para fans de suricata o snort), al no tener configuradas alertas, los logs se revisan manualmente (en un entorno empresarial esto sería una pesadilla, pero para un sólo equipo sin un SIEM, esto sería la norma a menos que lo automatices). Para este laboratorio, he usado mi máquina virtual de <span class="solo-color-neon">ubuntu</span> como víctima y mi máquina virtual de <span class="solo-color-neon">kali</span> para crear el payload (Que por razones obvias, no pondré la creación de la shell inversa).

El primer hallazgo en el log de zeek fue tremendamente sospechoso. Se descargó "algo" (no dice lo que se ha descargado), desde <span class="solo-color-neon">http</span> con una dirección ip sospechosa. Aquí vamos a comenzar la detección, porque <span class="solo-color-neon">no tenemos ni idea de lo que ha descargado el usuario</span>. Mirando el log de Zeek correspondiente con las conexiones de red (captura 2) nos damos cuenta de que esa misma ip está comunicándose por su puerto <span class="solo-color-neon">4444</span>. Por si no lo sabes, un puerto 4444 está mayoritariamente asociado a <span class="solo-color-neon">Metasploit</span>, es una bandera roja con un chorro de agua fría que te cae la cabeza, tiene que ser monitorizado con urgencia porque hay una amenaza potencial en el equipo.

![zeek3](https://github.com/user-attachments/assets/55d3d2b2-ca1e-45ae-8433-6a751d811f7e)

![zeek2](https://github.com/user-attachments/assets/c96964f4-ea00-4d7d-9feb-e473fefd28a1)


Para cerciorarme de que la conexión sigue <span class="solo-color-neon">activa</span>, voy a analizarlo todo con wireshark. Como me sé el puerto, voy a centrarme sólo en el puerto de destino, que es <span class="solo-color-neon">4444</span>. 

![wireshark3](https://github.com/user-attachments/assets/0f3017d1-ea60-4607-83e2-56d21ada1520)

Después de ver los paquetes capturados por <span class="solo-color-neon">wireshark</span>, no hay ninguna duda, <span class="solo-color-neon">la conexión sigue activa</span>. Aún no sé lo que está causando esa conexión.

Lo primero que voy a hacer es <span class="solo-color-neon">desconectar el ordenador de la red</span>, quiero romper esa conexión. También voy a<span class="solo-color-neon"> bloquear la dirección ip en iptables</span>. Con esto, aislo el ordenador de la red (se le está acabando la fiesta al atacante) y puedo investigar más tranquilamente.

Ya que los logs de <span class="solo-color-neon">Zeek</span> y <span class="solo-color-neon">wireshark</span> no me dicen mucho, voy a ponerme en la tarea de buscar cosas raras en los procesos.

![ps aux1](https://github.com/user-attachments/assets/71ecdd21-eb12-4b66-9153-8c034d95ec6e)

![ps aux2](https://github.com/user-attachments/assets/1c406316-1096-4f80-893f-e40e622d6f51)

Me encuentro con uno particularmente llamativo <span class="solo-color-neon">Shell.elf</span> (Está claro que el atacante no se ha estrujado mucho los sesos con el nombre). Para mí, tanto la extensión como el nombre me parecen sumamente sospechosos, por lo tanto, voy a coger su <span class="solo-color-neon">PID (3013)</span> y voy a buscar si está haciendo algún tipo de conexión saliente. El comando lsof en Linux me viene de lujo para esto. Quiero que sólo se enfoque en un PID, no quiero ningún tipo de dominio, quiero las ip, para eso sirve el comando "-n". También quiero ver el número del puerto (no quiero ver ldap, http, ftp...). Y con el -i indico que sólo quiero ver archivos/conexiones de red.

![lsof](https://github.com/user-attachments/assets/6f9204ab-0666-4c19-91b3-e904b98a67f9)

![lsof2](https://github.com/user-attachments/assets/43b62b9f-73af-40e7-a3dd-fd69bad87765)

Como suponía, este archivo <span class="solo-color-neon">(Shell.elf)</span> es el que está ocasionando el <span class="solo-color-neon">command and control</span>. Ya he detectado todo lo que necesitaba detectar, he contenido el <span class="solo-color-neon">command and control</span> y ahora me toca erradicarlo.

Lo primero que voy a hacer es buscar <span class="solo-color-neon">persistencia</span>, no quiero que haya nada que reinicie el malware:

![persistencia](https://github.com/user-attachments/assets/81765c47-7c74-440c-a6d6-efe33adf3c96)


Buscando en <span class="solo-color-neon">crontab</span> (las tareas programadas de Linux) y en el archivo <span class="solo-color-neon">bashrc</span> (que básicamente es una lista de comandos que se ejecutan automáticamente cada vez que abres una nueva terminal) no he encontrado nada del archivo en cuestión. Esto es muy bueno, significa que no hay una <span class="solo-color-neon">persistencia</span> en el sistema.

Ahora me toca dar con el archivo, lo voy a buscar y ver su ubicación: 

![localizar archivo](https://github.com/user-attachments/assets/692b45de-39e2-4970-b722-fd8c499dfc58)


Lo he encontrado en la carpeta root, si hago un ls ahí voy a ver que, efectivamente, está ahí (junto a otros logs). Esto me indica que el archivo ha sido descargado y ejecutado con <span class="solo-color-neon">privilegios de administrador</span>. Esto podría suponer que el ordenador ha sido comprometido para instalar el malware y no sólo hacer el <span class="solo-color-neon">command and control</span>, si no usarlo como backdoor después de irse.

![eliminar proceso archivo](https://github.com/user-attachments/assets/81b141c3-d001-4b9b-8866-e53b581369c8)

Y en una sóla línea <span class="solo-color-neon">mato su proceso y su archivo asociado</span>. No me ha supuesto ningún error ni nada y el archivo ya no aparece, por lo tanto, ya se ha eliminado cualquier rastro del malware.

En un SOC, de las cosas más comunes es el <span class="solo-color-neon">ticketing</span>. Por lo tanto, vamos a crear un <span class="solo-color-neon">ticketing </span> (tienes muchas herramientas para ello, pero para un sólo ticket lo voy a crear yo mismo):


- Título del incidente: <span class="solo-color-neon">Detección, contención y erradicación de un Command and control.</span>
- Estado actual: <span class="solo-color-neon">Resuelto</span>
- Prioridad: <span class="solo-color-neon">Crítica</span>
- Analista responsable: <span class="solo-color-neon">Obsidian Hunter</span>
- Vector de ataque: <span class="solo-color-neon">El usuario descargó, seguramente de un servidor infectado, un archivo llamado Shell.elf</span>
- Dirección IP Atacante: <span class="solo-color-neon">192.168.56.200</span>
- Dirección IP Víctima: <span class="solo-color-neon">192.168.56.101</span>
- Puerto de comunicación: <span class="solo-color-neon">TCP 4444</span>
- PID del proceso: <span class="solo-color-neon">3013</span>
- Nombre del artefacto: <span class="solo-color-neon">Shell.elf</span>
- Ruta del binario: <span class="solo-color-neon">/root/Shell.elf</span>
- Hash MD5: <span class="solo-color-neon">17122ea489399dd470314fa3157ad08e</span>
- Hash SHA-256: <span class="solo-color-neon">7d66e60858b293a24aa2ba55cd0857ccd9cea7c0f3c6f04672e443a96553d49c</span>
- Acciones de contención: <span class="solo-color-neon">Bloqueo de la ip atacante en iptables y desconexión de internet del ordenador.</span>
- Acciones de erradicación: <span class="solo-color-neon">Verificación de la conexión con lsof, erradicación del proceso y el archivo.</span>
- Conclusión: <span class="solo-color-neon">Incidente cerrado tras verificar que no existen más conexiones persistentes hacia la IP atacante.</span>

- <span class="solo-color-neon">Laboratorio Creado: 09-08-2025</span>
- <span class="solo-color-neon">Laboratorio Subido: 19-12-2025</span>
