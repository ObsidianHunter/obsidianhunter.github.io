---
layout: post
title: "Reconstrucción Forense de Ataque con Splunk: Del Acceso Inicial a la Exfiltración [Nivel: Muy Fácil]"
date: 2024-11-25
categories: [dfir]
permalink: /reconstruccion-ataque-completo2/
---

- Laboratorio 100 % educativo realizado en entorno completamente aislado.  
- Todos los comandos, payloads y dominios son simulados y no funcionales fuera del laboratorio.  
- Este repositorio NO contiene binarios ejecutables ni enlaces a malware real.  
- Queda expresamente prohibida la ejecución de cualquier comando o técnica aquí mostrada en sistemas de producción o redes ajenas.  
- Las capturas de Splunk® se utilizan exclusivamente con fines educativos y de investigación bajo la doctrina de fair use.  
- Este contenido no está afiliado, patrocinado, aprobado ni certificado por Splunk Inc.  
- Splunk® y Splunk> son marcas registradas de Splunk Inc. en Estados Unidos y otros países.  
- Todos los derechos reservados por sus respectivos propietarios.

Hoy voy a reconstruir la Kill Chain de un ataque que se ha producido. Para ello, he cogido un csv con un ataque simulado e ingerido por Splunk. En ese csv también se usa la herramienta "Sysmon" de Microsoft. Voy a intentar documentar lo mejor que pueda todo el proceso, desde el acceso inicial hasta el final (Paradójicamente, nunca hay un final, sólo cuando se elimina completamente, porque estamos hablando de una amenaza que se va a quedar en modo residente y no va a dejar de mandar datos o monitorizar el ordenador de la víctima.)

Lo primero que voy a plantearme es... ¿Cuál fue el vector inicial de compromiso?

Bueno, esto es lo primordial, no podemos ir más allá si no sabemos cómo la amenaza consiguió entrar en el sistema. En este ejemplo, como es fácil, el primer acceso lo encontraremos en el último post post, por lo que haremos un sort para ordenar las fechas de manera creciente:

![CapturaPantallaSplunk1](https://github.com/user-attachments/assets/6458f819-6426-43df-b46f-2f3d728a2376)

Aquí vemos la técnica "Ingress Tool Transfer" (T1105) en su máximo explendor. Si estás fameliarizado con mitre att&ack, sabrás que esto permite al atacante transferir herramientas remotamente al equipo infectado. Eso significa que el usuario ha ejecutado un comando para descargar algo (nada bueno) de esa dirección URL. Como no nos lo dicen, quizá haya sido por phishing, un correo electrónico falso que ha instado al usuario a ejecutar ese comando de alguna manera (habría que revisar los emails). Con esto, el atacante no necesita nada más, su malware está siendo descargado en el pc de la víctima.

Por lo tanto, concluimos que:

- Acceso Inicial: Muy probablmenete a través de un phishing.
- Tácticas usadas: Phishing  -> Initital accesss con técnica T1566, Descarga del malware -> Command and control con técnica T1105

Esta ha sido muy fácil, vamos por la siguiente. ¿Qué acciones ejecutó el adversario después de obtener ejecución?. Vamos a mirar el siguiente evento:


![CapturaPantallaSplunk2](https://github.com/user-attachments/assets/25f57817-f731-4605-a246-e4a777bcbf80)

Aquí vemos que usa -Enc con el comando powershell, eso le permite evadir defensas. Su táctica está clara, es Defense Evasion, junto a la técnica bfuscated Files or Information (T1027). Si decodificamos eso, nos queda exactamente la misma cadena que en el primer ejemplo (¿Error de la simulación del malware?). Por lo tanto, nada nuevo, está haciendo exactamente lo mismo que en el anterior paso sólo que está usando codificación en base 64.

- Lo mismo que el anterior post.
- Táactica usada -> Defense evasion con técnica T1027.

Con calma pero sin prisa, pasamos a lo siguiente que debemos preguntarnos: ¿Hubo descarga de payload adicional? ¿Desde dónde?.

Normalmente, para no llamar mucho la atención de los antivirus, IDS, firewalls... Los malware no se descargan directamente, suelen hacerlo a través de droppers o downloaders, que son mucho más pequeños en tamaño y causan menos visibilidad que otros, en este caso, debemos pensar que queremos pillar una url y algo que se ha descargado, por lo tanto, en la query de spl vamos a ver url, timestamp y los bytes recibidos:


![CapturaPantallaSplunk3](https://github.com/user-attachments/assets/177d84b0-6651-45dc-aa69-746eb20d8151)

Aquí, en el recuadro rojo, veo una descarga de un archivo tremendamente grande comparado con el resto de bytes recibidos, justamente lo que está haciendo ahora el dropper es descargando el malware principal.

Aquí concluimos que:

- El dropper ha descargado el malware principal.
- De momento no está usando ninguna táctica o técnica que no hayamos visto.

Vamos a por lo siguiente que nos interesa, ¿Se estableció comunicación C2? ¿Qué dominio/IP? 

En este caso, lo que se va a ver es si se ha establecido un Command and control (la máquina víctima y el servidor del atacante están comunicándose, muy probablemente enviando datos de la víctima al atacante y recibiendo comandos remotos del atacante):

![CapturaPantallaSplunk4](https://github.com/user-attachments/assets/53f821ad-bd31-4d14-8287-ee9e19525818)

En este caso, vemos byts siendo enviados y siendo recibidos, en un periodo de cada 30 segundos, es obvio que hay una comunicación con el dominio del atacante ubicado en: c2.malwarelab.local/ping. Aquí se está produciendo la comunicación entre ellos. Podemos sacar varias conclusiones aquí:

- Táctica -> Command and Control (TA0011). No sabemos muy bien la técnica, pero yo me apuesto lo que sea a que es la de remote access tool (T1219). Con esto ya sabemos muchas cosas y sabemos que hay una comunicación.

- Vale, ya tenemos mucha información, ahora vamos a preguntarnos, ¿Hubo movimiento lateral o escalación de privilegios?.

- Esto es muy importante, queremos saber si el malware intenta conseguir privilegios de administrador (o por lo menos, mayores que el usuario en el que está resguardado y calentito) o intenta diseminarse a otras máquinas de la red local. En este caso, vamos a buscar los puertos principales usados para el movimiento lateral y herramientas del sistema usadas para ello:
- 
![CapturaPantallaSplunk5](https://github.com/user-attachments/assets/f6e13c5a-ed65-43ce-8340-a180d3bb8cd1)

En este caso, no hubo ningún intento de movimiento lateral.

Ahora algo muy importante, ¿Se produjo exfiltración de datos? ¿Cuántos bytes y a qué URL?.

Aquí el malware va a intentar mandar al atacante información de la máquina víctima. Si bien suelen mandar en pequeñas cantidades, para este ejemplo el ataccante está enviando una cantidad grande datos en de una sola sentada.

![CapturaPantallaSplunk6](https://github.com/user-attachments/assets/b4c8d36c-bf17-48f1-8170-05c0913a8234)

En este caso, se ve que ha enviado una cantidad brutal de datos (12457896 bytes) al dominio del atacante: https://c2.malawarelab.local/upload. Probablmente haya robado información, credenciales... Aquí lo está mandando todo al atacante.

Ahora, el atacante necesita asegurarse un sitio para volver a iniciarse cada vez que el sistema se reinicia, por lo tanto necesitamos preguntarnos, ¿Intentó el adversario persistencia en el sistema?

En este caso es fácil saberlo, vamos a buscar en el id 13 de sysmon, que es para cuando se asigna/modifica datos de un valor del registro (También se puede usar el 12, que es la creación de ese valor, para mi ejemplo usaré el 13):

![CapturaPantallaSplunk7](https://github.com/user-attachments/assets/bd141228-84c2-4d11-99d7-d3b709b2cc60)

En este caso, he buscado en el registro (Hay muchos más como tareas programadas, carpeta startup... Incluso en otras llaves del registro como puede ser winlogon (para que inicie al cargar el sistema), pero en este caso, al ser un lab muy fácil, nos quedamos sólo con la clave Run) y vemos que efectivamente en la clave Run de la rama HKLM se ha asignado al valor "updater" con datos llamado "svchost.exe". EL malware intenta usar el nombre legítimo de un programa de windows para pasar inadvertido.

- Táctica -> Persistencia (TA0003) con la técnica Boot or Logon Autostart Execution: Registry Run Keys / Startup Folder (ID: T1547). EL malware quiere iniciarse cada vez que el sistema es reiniciado/encendido, lo que le otorga persistencia en el sistema.

Y ahora, vamos a encontrar al susodicho, ¿Dónde se esconde el malware?

Aquí podemos usar dos cosas, la primera es buscar en splunk toda cadena que contenga svchost.exe. La segunda es usar el id 11 de sysmon para encontrar los archivos creados recientemente, yo voy a usar la primera:

![CapturaPantallaSplunk8](https://github.com/user-attachments/assets/88b3c0e3-7bbb-45b6-a7cd-08dd9c89e2c5)

En este caso, vemos que está ubicado en C:\Users\usuario1_oficina2\AppData\Local\Temp\svchost.exe. Llama mucho la atención, sin hacer el kill chain completo, que se cargue al inicio del sistema un archivo ubicado en los archivos temporales. Suele ser una ubicación muy poco común para archivos en memoria pero muchísimo menos para que cargue al inicio. Sin hacer nada profundo, ese malware ha sido demasiado difícil de detectar sólo si nos hubiéramos enfocado en el registro.

Ahora, ¿Qué pasa si el malware quiere tener su casa limpita? ¿Hubo acciones de borrado de evidencias?

En muchísimos casos, los malwares intentan cubrir sus huellas para que no les pillen. En este caso es muy notorio por el tipo de inicio y, sobretodo, por la ubicación, pero en malwares más sofisticados donde se les puede encontrar mejor es en los logs. Muchos de ellos, borran estos logs para hacerlos mucho más indetectables y puedAn estar como amenazas prolongadas (APT) sin ser vistos. Aunque, visto lo visto, este malware es muy poco sofisticado para ser considerado un APT. Como le gusta jugar mucho con powershell, vamos a buscar registros de su limpieza. En este caso, viene bien tener un programa ajeno al eventviewer de Microsoft, que vaya guardando en texto plano todos los comandos powershell ejecutados, si no, mucha evidencia se habría borrado (pero no en los logs de esa aplicación). En mi caso personal, Ghost-Sentinel la codifiqué para obtener comandos powershell ejecutados básicos de infecciones, lo que permite una mayor retención de logs después del borrado de huellas (en este lab no está siendo ejecutado):

![CapturaPantallaSplunk9](https://github.com/user-attachments/assets/0263840a-85d0-409e-bbd9-078100bb0e95)

Aquí vemos que ha usado el comando: Clear-EventLog -LogName "Windows PowerShell", el cual borra todos los eventos del log de PowerShell. Aquí podemos extraer otra táctica de evasión:

- Táctica -> Defense Evasion (TA0005) con técnica Indicator Removal: Clear Windows Event Logs (T1070)


La cadena que ha seguido el malware es la siguiente:

- Muy probablemente (no me apostaría un millón de euros pero casi al 99% seguro) el usuario recibió un email con algún programa, instrucciones o directamente el comando powershell para descargar el dropper. Acto seguido, el dropper entró en el sistema y empezó a robar información y a comunicarse con el atacante. Creó persistencia, exfiltró datos e intentó borrar sus huellas.

- Es una amenaza muy básica, con poca sofisticación y muy predecible. No intenta técnicas muy sofisticadas ni alternativas, lo centra todo en intentar sacar la máxima cantidad posible de información sin ser detectado, pero con rutas y técnicas muy predecibles.

 












