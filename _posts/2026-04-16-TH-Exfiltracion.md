---
layout: post
title: "Caza Proactiva: Detección, Contención y Hardening de una Exfiltración DNS mediante Tunneling."
date: 2026-04-29
categories: [threat-hunting]
permalink: /TH-DNSExfiltracion/
---



Caza Proactiva: Detección, Contención y Hardening de una Exfiltración DNS mediante Tunneling

- <span class="solo-color-neon">Este contenido se ha creado exclusivamente con fines educativos y de concienciación en ciberseguridad.</span>
- <span class="solo-color-neon">Este laboratorio se ha hecho en un entorno controlado, con máquinas virtuales propias y con consentimiento.</span>
- <span class="solo-color-neon">Los datos presentados en el fichero txt de las capturas de pantalla son totalmente ficticios y han sido generados exclusivamente para fines de demostración en este laboratorio.</span>
- <span class="solo-color-neon">El uso de las herramientas y técnicas descritas en este laboratorio sobre sistemas o redes sin autorización previa y explícita es ilegal y éticamente reprobable.</span>
- <span class="solo-color-neon">El autor no se hace responsable del mal uso que terceros puedan dar a esta información.</span>
- <span class="solo-color-neon">Recuerda: el objetivo es aprender a defender, no a destruir.</span>

En este laboratorio voy a simular y detectar una exfiltración por <span class="solo-color-neon">DNS tunneling</span> (<span class="solo-color-neon">T1048.003 — Exfiltration Over Alternative Protocol: DNS</span>). Esto es tremendamente importante, porque el puerto 53 es un puerto de confianza, es decir, los antivirus, firewalls e IDS confían más o menos en él (aunque en entornos corporativos, los firewall e IDS son implacables y no se fían de nadie, por lo que inspeccionan este puerto también). ¿Por qué? Porque es <span class="solo-color-neon">tremendamente usado</span>, no hay compañía que no use algún tipo de servidor DNS para estar conectada al mundo. Sin él, no podrías por ejemplo poner Google.com o apple.com, tendrías que memorizar su <span class="solo-color-neon">dirección IP</span> para poder entrar en sus respectivas webs. Por eso, los hackers intentan exfiltrar datos a partir de él, se van a encontrar las puertas casi abiertas de par en par.


Para esta tarea voy a usar <span class="solo-color-neon">dnscat2</span> (Puedes usar otras herramientas de ciberseguridad como Iodine) . He descargado las dependencias para poder utilizarla. <span class="solo-color-neon">Dnscat2</span> me va a permitir crear un <span class="solo-color-neon">túnel DNS cifrado</span>, es decir, no vamos a poder saber lo que hay dentro de esos mensajes. La mayoría de las personas piensan que el protocolo DNS es sólo para resolver dominios o zonas inversas, pero hay también otro tipo, llamado <span class="solo-color-neon">TXT</span>. <span class="solo-color-neon">TXT</span> permite enviar texto, tremendamente importante porque no es algo común. De hecho, ver un <span class="solo-color-neon">TXT</span> o un <span class="solo-color-neon">NULL</span> debería ser una investigación más profunda (sobretodo el <span class="solo-color-neon">NULL</span>, ya que es muy muy poco frecuente).

Primero ejecuto <span class="solo-color-neon">dnscat2</span> en modo escucha directa, con el dominio que se ve en pantalla:

<img width="728" height="393" alt="Imagen1" src="https://github.com/user-attachments/assets/01f1fd2a-b706-43c5-8e85-9f1ad79fbb3f" />

El servidor genera un <span class="solo-color-neon">secret</span> único de sesión que usaré para autenticar al cliente Windows. El <span class="solo-color-neon">secret</span> evita que cualquier persona se conecte al servidor, lo cual es tremendamente útil para el atacante para que no venga alguien de su mismo gremio a quitarle el trozo de pastel. En este caso, en Windows he simulado un <span class="solo-color-neon">fileless</span>, simplemente ha ejecutado de la web un ps1, que es la extensión powershell y no deja ningún rastro de archivos... Es tremendamente útil para todo esto. ¿Cómo te puedes infectar de un fileless? Hay miles de formas, pero sobretodo el método principal sería por unos <span class="solo-color-neon">macros</span> en algún documento, un inicio de sesión maligno, un <span class="solo-color-neon">phishing</span>... Cualquier cosa de esas.

La <span class="solo-color-neon">exfiltración</span> va a ocurrir de un txt que me acabo de inventar que se llama <span class="solo-color-neon">confidencial</span>, ubicado en la raíz del sistema, paso su contenido (Obviamente no contiene credenciales verdaderas ni mucho menos):

<img width="753" height="517" alt="Imagen2" src="https://github.com/user-attachments/assets/82206460-fd5d-4486-9fd8-2b50d2a0cb57" />

Al crear el <span class="solo-color-neon">túnel DNS</span>, se me va abrir una sesión en Kali del pc de Windows:

<img width="709" height="571" alt="Imagen3" src="https://github.com/user-attachments/assets/11e54860-7731-45c5-a768-4a87b34565c8" />


Ya tenemos el túnel creado (y cifrado) y ahora vamos a lo que nos interesa, vamos a ver cómo se comporta una exfiltración de datos. Para ello, lo que va a ocurrir es lo que habíamos hablado antes del TXT. Esto lo vamos a pillar fácilmente con <span class="solo-color-neon">wireshark</span> con el label <span class="solo-color-neon">16</span>, que es el que identifica a <span class="solo-color-neon">TXT</span> (Y 10 a <span class="solo-color-neon">NULL</span>, por si algún día lo ves). En teoría, esto se va a poner como loco a soltar registros dns, así que voy a filtrar por <span class="solo-color-neon">dns.qry.type == 16 </span>

Procedo en Kali a la <span class="solo-color-neon">exfiltración</span>, voy a exfiltrar DatosSensibles.txt. Primero, escribo <span class="solo-color-neon">session -i 1</span>, para entrar en la de Windows, ahí ya tengo el prompt que me señala que está listo para empezar, vamos a proceder a la <span class="solo-color-neon">exfiltración</span>:

<img width="1003" height="308" alt="Imagen4" src="https://github.com/user-attachments/assets/b36e38c0-1715-4134-bea2-0f0bb93e7798" />

La <span class="solo-color-neon">exfiltración</span> ha sido un éxito, la tengo justo en el escritorio de Kali:

<img width="896" height="328" alt="Imagen5-Exfiltracion" src="https://github.com/user-attachments/assets/daff1d9f-0f58-4a16-9b6f-50ca40b89e69" />

Ahora viene lo que creo que es lo mejor del laboratorio. Voy a pegar la captura de <span class="solo-color-neon">wireshark</span>. Está llenísimo de eventos <span class="solo-color-neon">DNS</span> a la dirección de dominio que puse en Kali. Además, si te fijas en el campo Info, vas a ver los <span class="solo-color-neon">TXT</span>. Esos <span class="solo-color-neon">TXT</span> son los fragmentos que ha ido enviando <span class="solo-color-neon">dnscat2</span> del fichero DatosSensibles.txt. El fichero no se envía directamente, los paquetes DNS tienen un tamaño máximo, se van enviando en fragmentos y reconstruyéndose en el servidor DNS. Tener tantas consultas a un dominio DNS, con el label <span class="solo-color-neon">TXT</span> y a un dominio con una <span class="solo-color-neon">entropía</span> tan elevada, es una <span class="solo-color-neon">exfiltración</span>, no cabe mucha duda aquí.

<img width="1338" height="636" alt="Imagen5" src="https://github.com/user-attachments/assets/8d105f14-ea6a-4547-bd78-c13c564eea1c" />

Pero ahora vamos a mirar a <span class="solo-color-neon">Sysmon</span>, mucha gente cree que a estas alturas se generará un <span class="solo-color-neon">evento 22</span> (DNS query) en sysmon, sin embargo, en mi ordenador no ha salido nada de eso, pero sí se han creado numerosos <span class="solo-color-neon">eventos 3</span> (Conexión de red). El evento 22 de <span class="solo-color-neon">Sysmo</span>n solo se genera cuando las consultas DNS pasan por el reslvedor del sistema . Si una herramienta como dnscat2 construye y envía consultas DNS manualmente usando <span class="solo-color-neon">sockets</span>, sin usar esa API, <span class="solo-color-neon">Sysmon</span> no lo registra como <span class="solo-color-neon">DNS query</span>, sino únicamente como tráfico de red.  Hay que estar muy atento a esto, porque si no llega a ser por el <span class="solo-color-neon">evento 3</span> y <span class="solo-color-neon">wireshark</span> se nos podría haber colado la <span class="solo-color-neon">exfiltración</span>. En resumen, el tráfico sigue siendo DNS a nivel de red, pero al no pasar por el resolver de Windows, no genera el <span class="solo-color-neon">evento 22</span>.

<img width="669" height="554" alt="Imagen6" src="https://github.com/user-attachments/assets/e6a128de-0e4e-4ca3-9072-3d463df85383" />

He creado una pequeña regla <span class="solo-color-neon">Sigm</span>a, en este caso, si detecta que <span class="solo-color-neon">powershell</span> está involucrado en una conexión de red, que además usa el puerto 53 para la comunicación, activa una alarma de nivel medio. Esto no se da en una empresa normal. Quizá miles de peticiones DNS sí, pero un <span class="solo-color-neon">powershell</span> que esté usando el puerto 53 no es un comportamiento normal, quizá algún administrador para hacer alguna auditoría muy esporádica, y aún así, sospecharía.

```yaml
title: Detección de Tráfico DNS desde PowerShell
status: activo
description: Detecta cuando PowerShell abre una conexión de red por el puerto 53.
author: ObsidianHunter
logsource:
    product: windows
    service: sysmon
detection:
    selection:
        EventID: 3
        Image|endswith: '\powershell.exe'
        DestinationPort: 53
    condition: selection
level: medium
```
En este caso, la contención sería la de siempre, intentando <span class="solo-color-neon">aislar la máquina de internet</span> y la red privada (siempre que aislar ese ordenador no sea esencial para la salud de la empresa, no puedes permitirte una parada de 1 hora con ese ordenador fuera sabiendo que te podría costar millones de euros en ese periodo). El canal que crea <span class="solo-color-neon">dnscat2</span> es volátil, en este caso yo elegí usar el script de ps1 para cargarlo en memoria y simular un fileless, pero normalmente, sólo el canal de dnscat2 con reiniciar el ordenador suele ser suficiente. Ahora, si este <span class="solo-color-neon">DNS tunneling</span> hubiera sido por un malware y no una herramienta destinada exclusivamente a eso, habría que mirar la <span class="solo-color-neon">persistencia</span> que ha dejado, ya que reiniciar un ordenador con una persistencia de algún tipo (claves run, carpetas startups, servicios, winlogon…) sólo haría que el malware regresara tras cada reinicio.

Aquí, la <span class="solo-color-neon">erradicación</span> coincide un poco con la contención. En mi caso, al ser un fileless sin ningún archivo asociado, simplemente un script que puedes ejecutar por un phishing o macro, una vez contenido también lo eliminas. Si hubiera otro tipo de malware, como he dicho, la <span class="solo-color-neon">erradicación</span> se vuelve algo más tediosa.

Para evitar las exfiltraciones, hay que estar muy atento del DNS interno forzado, un equipo no debería poder hacer queries DNS directamente a internet, solo al DNS corporativo. Una consulta DNS a internet y no al dominio de la corporación, es una bandera naranja que tienes que vigilar muy de cerca. También hay que inspeccionar el DNS en el firewall ya que los firewall de nueva generación pueden analizar el contenido DNS (siempre y cuando no esté cifrado) y detectar entropía alta o volumen anómalo, como el que acabamos de ver en <span class="solo-color-neon">Wireshark</span> y la multitud de eventos 3 en <span class="solo-color-neon">sysmon</span>. Una táctica que me encanta para el hardening es el DNS sinkhole ya que redirige las consultas DNS de dominios maliciosos a una IP interna del SOC, que está fuertemente vigilada. Esto es fabuloso porque el propio atacante, sin saberlo, está llamando directamente a tu puerta (como un truco o trato).

Aquí viene un problema real, los atacantes podrían usar el protocolo <span class="solo-color-neon">https</span> sobre <span class="solo-color-neon">dns</span>, conocido como <span class="solo-color-neon">DoH</span>, lo que cambiaría el lab completamente. Ya no verías tráfico DNS en <span class="solo-color-neon">wireshark</span>, tampoco se vería ningún tipo de conexión al puerto <span class="solo-color-neon">53 udp</span>… Ahora todo se haría al puerto <span class="solo-color-neon">443</span>, pero igualmente estaría ocurriendo una <span class="solo-color-neon">exfiltración</span>, ya no podrías partir y deducir tan fácilmente que es una <span class="solo-color-neon">exfiltración</span> de datos. Pero sigue teniendo puntos débiles si lo hace de esa manera. Para empezar, ¿qué hace <span class="solo-color-neon">powershell</span> mandando miles de consultas cada x segundos al puerto <span class="solo-color-neon">443</span>? Esto no es un comportamiento normal, <span class="solo-color-neon">powershell</span> no debería salir a internet salvo en muy muy contadas ocasiones, pero menos haciendo miles de consultas. Si además la ip es desconocida o sospechosa, podría partir de la premisa de una potencial <span class="solo-color-neon">exfiltración</span> de datos mediante <span class="solo-color-neon">dns tunneling</span> sobre el protocolo <span class="solo-color-neon">https</span>, cuyo contenido está siendo protegido por el <span class="solo-color-neon">cifrado TLS</span>. Además, en wireshark puedes ver si el certificado que está usando https es válido o es autofirmado. Normalmente, los hackers usan uno autofirmado, les vale para proteger su contenido y que no vean realmente lo que están haciendo. 

Recuerda que tus datos pueden ser exfiltrados fácilmente. Vigila sysmon, vigila las salidas de tu red de TXT y NULL y, sobretodo, desconfío de todo en tu ordenador.

- <span class="solo-color-neon">Laboratorio creado:</span> 16/04/2026
- <span class="solo-color-neon">Laboratorio subido:</span> 29/04/2026
