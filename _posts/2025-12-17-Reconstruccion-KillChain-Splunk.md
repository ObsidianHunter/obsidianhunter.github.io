---
layout: post
title: "Reconstrucción Forense de Ataque con Splunk: Reconstrucción de un Ataque Living off the Lands [Nivel: Fácil]"
date: 2024-11-25
categories: [dfir]
permalink: /reconstruccion-ataque-completo/
---

Hoy, voy a reconstruir la kill chain de un ataque llevado a cabo mayoritariamente por tácticas living off the lands. Voy a reconstruir todo el ataque identificando los puntos claves del ataque. Para ello, he simulado nuevamente un entorno de trabajo de oficina normal. He ordenado con el parámetro "sort" los registros de splunk, lo que me permite ir recorriendo el informe desde el principio hasta el final. Vemos que los primeros procesos parecen benignos y no hay nada malo en ello (parecen tareas típicas de oficina).

Empecemos por el principio, por dónde ha entrado el atacante. Si nos fijamos, el usuario ha ejecutado un programa en temp (Ya eso debería ser tremendamente sospechoso por sí solo) y parece un documento de Word benigno... Bueno, para sorpresa de nadie, no lo es. La extensión es un docm, es decir, su arquitectura permite macros (Y las macros han sido si siempre uno de los vectores de ataque más usados). Por lo tanto, el atacante se coló mediante un phishing y con una macro. Podemos decir entonces que:

- La táctica que ha usado es Phishing (TA0001) que corresponde a Initial Access. 
- Además, si el usuario ejecutó sin saberlo una macro maliciosa, podemos también concluir que ha usado la técnica T1204, o sea, user Execution.


![Imagen1Forense](https://github.com/user-attachments/assets/2a054693-c680-496f-b904-c3f849f7fe94)

Esa macro contiene un comando en powershell, ofuscado en base 64, como se puede ver aquí:

![Imagen2Forense](https://github.com/user-attachments/assets/a396dee4-c66c-4480-821c-1b243f2ae6a0)

Esto apunta a una dirección para bajar un script, la ip: 192.168.200.88, con un script en ps1 llamado update. 

- La táctica usada aquí es Execution (TA0002) con la técnica T1059 (Command and Scripting Interpreter).
- También se ha usado la Táctica Defense Evasion (TA0005) con la técnica T1140 (Deobfuscate/Decode Files or Information)

7 segundos después, aparece en el ordenador un archivo llamado update.dll en la ruta de programa C:\ProgramData\Microsoft\Windows\Caches\update.dll (muy probablemente lo ha bajado el script anterior) y el atacante está usando rundll.exe para poder mandar órdenes al ordenador a través de esa dll. Recordemos que rundll.exe es un programa de Windows, es tremendamente silencioso a ojos de los IDS, el atacante está usando rundll.exe para ejecutar comandos en el sistema a través de su dll infectada, la dll está sirviendo como puente para el command and control. A partir de ahora, el atacante puede mandarle las órdenes a esa dll para que las ejecute, permitiendo el control total del sistema. Lo primero que va a hacer esa la enumeración tanto del propio ordenador, como los dominios y la red. 

![Imagen3Forense](https://github.com/user-attachments/assets/28666f59-889b-4c23-bb58-a84e2c33e7cd)

![Imagen4Forense](https://github.com/user-attachments/assets/3b2c4442-1649-4b91-98b4-abdcef66941a)

![Imagen5Forense](https://github.com/user-attachments/assets/e556775f-dd3e-490e-b1c9-26e7d6466a7b)

- Con whoami /groups está viendo si ese usuario que está usando tiene privilegios.

- Con net group "Domain Admins" /domain está viendo quiénes son los administradores de la red.

- nltest.exe /domain_trusts aquí está mirando los dominios de la red, viendo si hay más conectados a los que poder saltar.

Aquí podemos concluir varias tácticas y técnicas:

- Táctica usada Command and Control (TA0011)
- Táctica Defense Evasion (TA0005) con la técnica T1218.011  (System Binary Proxy Execution: Rundll32)
- Táctica usada Discovery (TA0007) con las técnicas T1033 (System Owner/User Discovery), T1069 (Permission Groups Discovery) y T1482 (Domain Trust Discovery).

Antes de descargar el programa que filtra todas las credenciales, lo cual podría activar el antivirus o el IDS, quiere asegurarse de que ha entrado por una razón y puede llevarse algo de información.

En la siguiente fase, el atacante usa bitadmins, una herramienta legítima de descarga en segundo plano de Windows, para descargar el programa para robar las credenciales (muy probablemente, le ha gustado lo que hay en el ordenador actual y ha decidido darle la orden al ordenador a través de su dll para continuar), el cual lo descarga en una carpeta temporal con el nombre calc_updater.exe (principalmente para hacer creer que es una actualización de una calculadora), muy probablemente se trate de Mimikatz o Procdump de forma renombrada, aunque prácticamente cualquier antivirus detectaría a Mimikatz. También vemos en esa línea que va a usar ese mismo ejecutable para extraer las credenciales del sistema, a un archivo llamado lsass.dmp, el atacante ya está robando las credenciales del sistema. Lejos de quedarse tranquilo, el atacante lo empaqueta en un zip y le pone la contraseña password123 (Hasta los atacantes les preocupa la privacidad). Con toda probabilidad, está preparándose para exfiltrar los datos.

![Imagen6Forense](https://github.com/user-attachments/assets/f293cd82-3c88-43e9-a8fa-975721bdd5d6)

En esta etapa vuelve a actuar desde su dll para hacer una subida del zip con las credenciales robadas, lo que le permite enviar a su dominio https://192.168.200.88 (No sólo usa contraseña, también está usando encriptación TLS con https, qué majo). Con esto, ha terminado la exfiltración de datos y ya ha obtenido toda la información necesaria.

![Imagen7Forense](https://github.com/user-attachments/assets/3e9ffaf0-ed8c-4040-9793-27e6c6889820)

Esto parece un ataque de coger y huir, no parece que quiera ser un APT, no quiere quedarse en el sistema porque no hay ningún tipo de persistencia, muy probablemente, al reiniciar el equipo no habrá nada que se ejecute automáticamente. (No hay logs de las carpetas startup, ni tareas programadas, ni servicios, ni claves de arranque, ni en winlogon...).

El último paso que queda es conectarse a una máquina administrador con un movimiento lateral... En la siguiente imagen se puede ver

![Imagen9Forense](https://github.com/user-attachments/assets/55044978-6798-4bae-a2b7-7a8a6ce43e4d)

Va a usar las credenciales robadas (las robó el calc_updater de lsass) para autenticarse como administrador (El atacante consiguió las llaves del administrador de la forma más tonta posible: aprovechó que el técnico de soporte había entrado antes en ese PC para ayudar al usuario a través del escritorio remoto. Al hacerlo, sus credenciales se quedaron en la memoria RAM. Esto en un entorno laboral serio no debería pasar, porque ha sido una autenticación estándar.). Si nos fijamos bien, está usando el puerto 135, es decir, el RPC. Básicamente lo que permite esto es que se ejecute un proceso en otra máquina desde otra máquina. Un administrativo u oficinista que use ese puerto es una bandera roja y una alarma urgente de resolver. Usará el archivo lateral.ps1 para iniciar la infección en ese ordenador.

Después de todo esto, va a limpiar sus huellas, usando la utilidad de wevtutil borrará los registros de seguridad, el archivo zip (ya lo ha enviado) y si nos fijamos más abajo también está borrando el archivo lsass.dmp, el archivo calc_updater… Es decir, no va a dejar nada en el sistema.

![Imagen10Forense](https://github.com/user-attachments/assets/0ae609f8-c0f7-4993-ac5a-39016d70dec6)

![Imagen11Forense](https://github.com/user-attachments/assets/30e74a5d-b14f-4235-8e02-fb207b9b4c11)

La cadena parece muy simple -> Entrada en el sistema a través de un phishing -> Ejecución de una macro con comandos powershell -> Descarga del primer script con la dll -> Hace una enumeración del sistema, de los dominios, de la red... -> A través de comandos enviados a través de su librería infectada va a descargar el programa para filtrar contraseñas de lsass -> El programa roba todas las credenciales, las empaqueta y las envía al atacante -> El atacante inicia un movimiento lateral hacia una cuenta de administrador, iniciando de nuevo todo el proceso de infección -> El atacante borra todos los registros y archivos usados.

Pendiente de editar, mejorar y corregir.




