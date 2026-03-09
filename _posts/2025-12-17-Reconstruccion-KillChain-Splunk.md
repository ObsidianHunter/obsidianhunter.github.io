---
layout: post
title: "Reconstrucción Forense de Ataque con Splunk: Reconstrucción de un Ataque Living off the Lands [Nivel: Fácil]"
date: 2024-11-25
categories: [dfir]
permalink: /reconstruccion-ataque-completo/
---

Hoy, voy a reconstruir la <span class="solo-color-neon">ciber kill chain</span> de un ataque llevado a cabo mayoritariamente por tácticas <span class="solo-color-neon">living off the lands</span>. Voy a reconstruir todo el ataque identificando los puntos claves del ataque. Para ello, he simulado nuevamente un entorno de trabajo de oficina normal. He ordenado con el parámetro "sort" los registros de splunk, lo que me permite ir recorriendo el informe desde el principio hasta el final. Vemos que los primeros procesos parecen <span class="solo-color-neon">benignos</span> y no hay nada malo en ello (parecen tareas típicas de oficina).

Empecemos por el principio, por dónde ha entrado el atacante. Si nos fijamos, el usuario ha ejecutado un programa en temp (Ya eso debería ser tremendamente sospechoso por sí solo) y parece un documento de Word benigno... Bueno, para sorpresa de nadie, no lo es. La extensión es un docm, es decir, <span class="solo-color-neon">su arquitectura permite macros</span> (Y las macros han sido si siempre uno de los vectores de ataque más usados). Por lo tanto, el atacante se coló mediante un phishing y con una macro. Podemos decir entonces que:

- La táctica que ha usado es <span class="solo-color-neon">Phishing </span> (TA0001) que corresponde a Initial Access. 
- Además, si el usuario ejecutó sin saberlo una macro maliciosa, podemos también concluir que ha usado la técnica T1204, o sea, <span class="solo-color-neon">user Execution.</span>


![Imagen1Forense](https://github.com/user-attachments/assets/2a054693-c680-496f-b904-c3f849f7fe94)

Esa macro contiene un comando en powershell, ofuscado en base 64, como se puede ver aquí:

![Imagen2Forense](https://github.com/user-attachments/assets/a396dee4-c66c-4480-821c-1b243f2ae6a0)

Esto apunta a una dirección para bajar un script, la ip: 192.168.200.88, con un script en ps1 llamado update. 

- La táctica usada aquí es <span class="solo-color-neon">Execution </span> (TA0002) con la técnica T1059 <span class="solo-color-neon">(Command and Scripting Interpreter)</span>.
- También se ha usado la Táctica <span class="solo-color-neon">Defense Evasion </span> (TA0005) con la técnica T1140 <span class="solo-color-neon">(Deobfuscate/Decode Files or Information) </span>

7 segundos después, aparece en el ordenador un archivo llamado <span class="solo-color-neon">update.dll</span> en la ruta de programa C:\ProgramData\Microsoft\Windows\Caches\update.dll (muy probablemente lo ha bajado el script anterior) y el atacante está usando <span class="solo-color-neon">rundll.exe</span> para poder mandar órdenes al ordenador a través de esa dll. Recordemos que <span class="solo-color-neon">rundll.exe</span> es un programa de Windows, es tremendamente silencioso a ojos de los <span class="solo-color-neon">IDS</span>, el atacante está usando <span class="solo-color-neon">rundll.exe </span>para ejecutar comandos en el sistema a través de su dll infectada, la dll está sirviendo como puente para el <span class="solo-color-neon">command and control</span>. A partir de ahora, el atacante puede mandarle las órdenes a esa dll para que las ejecute, permitiendo el control total del sistema. Lo primero que va a hacer esa la enumeración tanto del propio ordenador, como los dominios y la red. 

![Imagen3Forense](https://github.com/user-attachments/assets/28666f59-889b-4c23-bb58-a84e2c33e7cd)

![Imagen4Forense](https://github.com/user-attachments/assets/3b2c4442-1649-4b91-98b4-abdcef66941a)

![Imagen5Forense](https://github.com/user-attachments/assets/e556775f-dd3e-490e-b1c9-26e7d6466a7b)

- Con <span class="solo-color-neon">whoami /groups </span> está viendo si ese usuario que está usando tiene privilegios.

- Con <span class="solo-color-neon">net group "Domain Admins" /domain </span> está viendo quiénes son los administradores de la red.

- Con <span class="solo-color-neon">nltest.exe /domain_trusts </span> aquí está mirando los dominios de la red, viendo si hay más conectados a los que poder saltar.

Aquí podemos concluir varias tácticas y técnicas:

- Táctica usada <span class="solo-color-neon">Command and Control </span> (TA0011)
- Táctica <span class="solo-color-neon">Defense Evasion </span> (TA0005) con la técnica T1218.011  <span class="solo-color-neon">(System Binary Proxy Execution: Rundll32) </span>
- Táctica usada <span class="solo-color-neon">Discovery </span> (TA0007) con las técnicas T1033 <span class="solo-color-neon">(System Owner/User Discovery) </span>, T1069 <span class="solo-color-neon">(Permission Groups Discovery) </span> y T1482 <span class="solo-color-neon">(Domain Trust Discovery)</span>.

Antes de descargar el programa que filtra todas las credenciales, lo cual podría activar el antivirus o el IDS, quiere asegurarse de que ha entrado por una razón y puede llevarse algo de información.

En la siguiente fase, el atacante usa <span class="solo-color-neon">bitadmins</span>, una herramienta legítima de descarga en segundo plano de Windows, para descargar el programa para robar las credenciales (muy probablemente, le ha gustado lo que hay en el ordenador actual y ha decidido darle la orden al ordenador a través de su dll para continuar), el cual lo descarga en una carpeta temporal con el nombre <span class="solo-color-neon">calc_updater.exe</span> (principalmente para hacer creer que es una actualización de una calculadora), muy probablemente se trate de <span class="solo-color-neon">Mimikatz o Procdump </span> de forma renombrada, aunque prácticamente cualquier antivirus detectaría a Mimikatz. También vemos en esa línea que va a usar ese mismo ejecutable para extraer las <span class="solo-color-neon">credenciales del sistema</span>, a un archivo llamado <span class="solo-color-neon">lsass.dmp</span>, el atacante ya está robando las credenciales del sistema. Lejos de quedarse tranquilo, el atacante lo empaqueta en un zip y le pone la contraseña <span class="solo-color-neon">password123</span> (Hasta los atacantes les preocupa la privacidad). Con toda probabilidad, está preparándose para <span class="solo-color-neon">exfiltrar los datos.</span>

- La táctica usada es <span class="solo-color-neon">Credential Access</span> (TA0006) con la técnicas T1003 <span class="solo-color-neon"(OS Credential Dumping, en especial con LSSAS (001)</span>

-También usa la tácita <span class="solo-color-neon">Command & Control</span> (TA0011) con la técnica  T1105 <span class="solo-color-neon">(Ingress tool Transfer)</span>.

![Imagen6Forense](https://github.com/user-attachments/assets/f293cd82-3c88-43e9-a8fa-975721bdd5d6)

En esta etapa vuelve a actuar desde su dll para hacer una subida del zip con las <span class="solo-color-neon">credenciales robadas</span>, lo que le permite enviar a su dominio <span class="solo-color-neon">https://192.168.200.88</span> (No sólo usa contraseña, también está usando encriptación TLS con https, qué majo). Con esto, ha terminado la exfiltración de datos y ya ha obtenido toda la información necesaria.

- La táctica que usa es <span class="solo-color-neon">Exfiltration</span> (TA0010) con la técnica T1041 <span class="solo-color-neon">(Exfiltration over C2 channel)</span>

![Imagen7Forense](https://github.com/user-attachments/assets/3e9ffaf0-ed8c-4040-9793-27e6c6889820)

Esto parece un ataque de coger y huir, no parece que quiera ser un <span class="solo-color-neon">APT</span>, no quiere quedarse en el sistema porque no hay ningún tipo de <span class="solo-color-neon">persistencia</span>, muy probablemente, al reiniciar el equipo no habrá nada que se ejecute automáticamente. (No hay logs de las carpetas startup, ni tareas programadas, ni servicios, ni claves de arranque, ni en winlogon...).

El último paso que queda es conectarse a una máquina administrador con un <span class="solo-color-neon">movimiento lateral</span>... En la siguiente imagen se puede ver

- La táctica usada es <span class="solo-color-neon">Lateral Movement</span> (TA0008)

![Imagen9Forense](https://github.com/user-attachments/assets/55044978-6798-4bae-a2b7-7a8a6ce43e4d)

Va a usar las credenciales robadas (las robó el calc_updater de lsass) para autenticarse como administrador (El atacante consiguió las llaves del administrador de la forma más tonta posible: <span class="solo-color-neon">aprovechó que el técnico de soporte había entrado antes en ese PC para ayudar al usuario a través del escritorio remoto.</span> Al hacerlo, sus credenciales se quedaron en la memoria RAM. Esto en un entorno laboral serio no debería pasar, porque ha sido una autenticación estándar.). Si nos fijamos bien, está usando el puerto 135, es decir, el <span class="solo-color-neon">RPC</span>. Básicamente lo que permite esto es que se ejecute un proceso en otra máquina desde otra máquina. Un administrativo u oficinista que use ese puerto es una bandera roja y una alarma urgente de resolver. Usará el archivo lateral.ps1 para iniciar la infección en ese ordenador.

Después de todo esto, <span class="solo-color-neon">va a limpiar sus huellas</span>, usando la utilidad de <span class="solo-color-neon">wevtutil</span> borrará los registros de seguridad, el archivo zip (ya lo ha enviado) y si nos fijamos más abajo también está borrando el archivo lsass.dmp, el archivo calc_updater… Es decir, no va a dejar nada en el sistema.

- La táctica usada es <span class="solo-color-neon">Defense Evasion</span> con la técnica T1070 <span class="solo-color-neon">(Indicator Removal, especialmente 001 que es clean Windows Event logs y la 004 que es File Deletion)</span> 

![Imagen10Forense](https://github.com/user-attachments/assets/0ae609f8-c0f7-4993-ac5a-39016d70dec6)

![Imagen11Forense](https://github.com/user-attachments/assets/30e74a5d-b14f-4235-8e02-fb207b9b4c11)

La cadena parece muy simple -> <span class="solo-color-neon">Entrada en el sistema a través de un phishing -> Ejecución de una macro con comandos powershell -> Descarga del primer script con la dll -> Hace una enumeración del sistema, de los dominios, de la red... -> A través de comandos enviados a través de su librería infectada va a descargar el programa para filtrar contraseñas de lsass -> El programa roba todas las credenciales, las empaqueta y las envía al atacante -> El atacante inicia un movimiento lateral hacia una cuenta de administrador, iniciando de nuevo todo el proceso de infección -> El atacante borra todos los registros y archivos usados.</span>

- Laboratorio creado: 17/12/2025
- Laboratorio publicado: 19/12/2025





