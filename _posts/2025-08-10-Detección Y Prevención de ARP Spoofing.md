---
layout: post
title: "Detección y Prevención de ARP Spoofing [Wireshark]"
date: 2025-08-10
categories: [blue-team]
permalink: /deteccion-arp/
---
En este laboratorio, voy a detectar fácilmente un <span class="solo-color-neon">ARP poisoning</span>. Para el que no sepa lo que es, es una técnica en la que el atacante envía <span class="solo-color-neon">paquetes arp falsificados para asociar su mac a una ip dentro de la red</span>. ¿Por qué hace esto?, bueno, el protocolo ARP funciona en la capa 2 del <span class="solo-color-neon">modelo OSI</span> y ahí no hay IP, hay MAC. Para comunicarse entre los que están en la misma capa, un dispositivo debe conocer la MAC del dispositivo al que quiere enviar el paquete. De este modo, si el atacante suplanta su MAC, todo el tráfico que vaya destino a la MAC destino también irá para el atacante. Este ataque es muy difícil de detectar, ya que no lo detecta ni el <span class="solo-color-neon">firewall ni los antivirus</span>, quizá si tienes la suerte de tener un sistema de detección de intrusos tu día estará salvado (en entornos empresariales se usa mucho <span class="solo-color-neon">DAI</span>), de lo contrario, es muy fácil caer, ya que el atacante sólo necesita estar conectado a la misma red local que las víctimas. En este caso, usé un programa llamado <span class="solo-color-neon">Ettercap</span> tanto en la máquina virtual que tengo para algunos laboratorios como mi windows. He logrado con unos simples pasos un ataque <span class="solo-color-neon">Man in the middle</span>, ya que ahora puedo ver todo lo que están enviando entre máquinas. En este caso, decidí envenenar ambas, lo que me permite registrar TODO el tráfico de red que pasará entre ellas. En esta dos imágenes podemos ver las consecuencias del ataque:


![ip neighbour](https://github.com/user-attachments/assets/21e818fb-a0da-4cee-bf83-d75005332d11)

![windows](https://github.com/user-attachments/assets/4ed3ad05-e0d6-4aab-b9a8-0d453ef596d5)


No ha habido alarma, ni señales ni nada, pero si nos fijamos muy bien (pero bastante bien) vemos que hay <span class="solo-color-neon">3 ip con la misma MAC.</span> Primero, la tabla ARP de linux está envenenada, cuando quiera enviar un paquete a windows, primero se la está mandando al atacante, que con un sniffer podría ver toda la conversación y luego reenviarla a windows. Luego, windows también tiene su tabla <span class="solo-color-neon">ARP envenenada</span>, cuando quiera enviar un paquete a Linux, lo enviará primero al atacante y luego este lo reenviará a Linux. Es un ataque perfecto porque con un sniffer entre medias no haces ruido, simplemente te sientas a recolectar datos mientras ves tu serie favorita. En este caso, voy a profundizar un poco más y enseñaré cómo poder detectarlo con wireshark.

En este sentido, wireshark tiene un comando muy útil, llamado <span class="solo-color-neon">arp.isgratuitous</span>, este comando permite detectar qué ordenador ha lanzado un paquete ARP que no ha sido solicitado por nadie, es algo como presentándose y diciéndole a todos cuál es su ip y MAC. Esto es terriblemente peligroso, ya que si ves en wireshark esto y no tienes una red empresarial, <span class="solo-color-neon">podría ser un ataque de envenamiento de las tablas ARP</span>. Es muy muy raro ver ese paquete suelto sin más. En este sentido, si pongo en wireshark este comando, veo la IP que está envenenando: 192.168.0.200. Nadie ha pedido su ip, nadie ha pedido su mac, pero aún así él envía un mensaje diciéndole a todo el mundo su ip y la mac.

![wiresharkisgratoitous](https://github.com/user-attachments/assets/b6191e0d-8037-4aaa-8a40-e0b6337257c9)


De todas maneras, wireshark permite detectar estos ataques directamente filtrando por el protocolo ARP, como vemos en esta imagen:

Wireshark nos dice que la MAC está duplicada, que alguien ha envenenado las tablas ARP de los dispositivos. El protocolo ARP es <span class="solo-color-neon">inseguro por naturaleza</span>, si no tienes ninguna defensa por defecto, va a creer lo que le diga cualquier ordenador. En este caso, Ettercap ha envenenado ambas tablas ARP para decirle a windows, oye que Linux tiene mi misma MAC. Y a Linux ha envenenado para decirle, oye que windows tiene mi misma MAC. Con esto, va a estar en medio de cualquier conversación.

Con un IDS normalito debería ser suficiente para detectarlo (debería). Esto también se evita con <span class="solo-color-neon">tablas estáticas</span>:

![Wireshark](https://github.com/user-attachments/assets/47bdb189-6d1b-4257-b376-cb8d65eea15b)


- Windows -> <span class="solo-color-neon">netsh interface ipv4 add neighbors "Ethernet2" 192.168.0.191 LAMACDELINUX </span>
- Linux -> <span class="solo-color-neon">ip neighbor add 192.168.0.19 lladdr LAMACDEWINDOWS dev eth0 nud permanent </span>

Normalmente un ataque ARP poisoning envenena la puerta de enlace, para leer todo lo que vaya a salir a internet... En mi caso, preferí envenenar ambas tablas para controlar la conversación porque puede haber información buena de FTP, telnet o demás (aunque son protocolos inseguros que no deberían usarse nunca). Incluso podría usar el protocolo HTTPS (que va cifrado) pero el atacante podría hacer un <span class="solo-color-neon">SSL Stripping</span>, lo que obligaría al usuario a pasar de HTTPS a HTTP (que va sin cifrar). Para evitar esto está <span class="solo-color-neon">el protocolo HSTS</span> en el navegador, que para resumir, si alguien intenta derivar tu petición de HTTPS a HTTP, ignora esa orden.

Otra forma de eliminar este peligro usar el <span class="solo-color-neon">DAI</span> que cité antes (prefiero repetirme que no quede todo claro), que es un <span class="solo-color-neon">Dynamic ARP Inspection </span>. El DAI compararía la MAC (e IP) del atacante con una base de datos de MAC ya establecida. Si no coincide, <span class="solo-color-neon">descartaría cualquier paquete proviniendo del atacante</span>. Para que DAI esté activado, que no se te olvide que tiene que estar activado el <span class="solo-color-neon">DHCP Snooping</span> (A menos que te guste un ARP poisoning, que oye, yo ahí no me meto).

EN este caso concreto, una respuesta a incidentes no sólo se basa en la prevención (todas las anteriores son prevención). Ya no sólo vale también bloquear en el firewall la IP del atacante, hay que encontrarlo dentro de la red local, <span class="solo-color-neon">expulsarlo de la red y bloquear su MAC.</span>

- <span class="solo-color-neon">Laboratorio Creado: 10-08-2025</span>
- <span class="solo-color-neon">Laboratorio Editado y Subido: 31-12-2025</span>
