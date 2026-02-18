---
layout: post
title: "Detección y contención de un escaneo de puertos [ELK + Wireshark + Zeek]"
date: 2025-08-07
categories: [blue-team]
permalink: /deteccion-escaneo/
---

# Detección y contención de un escaneo de puertos [ELK + Wireshark + Zeek]

Hoy quiero detenerme en un laboratorio que hice hace unos meses atrás y que hoy he podido ya editar y publicar. Se trata de un <span class="solo-color-neon">escaneo de puertos</span>. A esta altura, nadie que trabaje en ciberseguridad debería desconocer ese término. La mayoría de la gente cree que los escaneos de puertos suele venir de IP externas o que son bots, pero en este lab, voy a tratar de hacerte ver que hay muchas más técnicas en las que lo pueden usar los hackers. Yo simulé en mi entorno de máquinas virtuales un escaneo con <span class="solo-color-neon">NMap</span> desde mi <span class="solo-color-neon">Kali</span> hasta una de mis máquinas <span class="solo-color-neon">Ubuntu</span> víctima. Voy a ir enseñándote, paso a paso, cómo lo se detecta y se contiene (abróchate el cinturón).

En este caso, el primer paso para detectar un escaneo de puertos, y muchos otros ataques, es el <span class="solo-color-neon">IDS</span>. En mi caso, me encanta <span class="solo-color-neon">Zeek</span> y mirando el archivo <span class="solo-color-neon">conn.log</span> me encontré con esta maravilla:

![Escaneo de puertos zeek](https://github.com/user-attachments/assets/38c855cf-07e4-404d-86e1-90ec08140406)

Veo que una <span class="solo-color-neon">ip fija</span>, desde un <span class="solo-color-neon">puerto fijo</span> está haciendo una enumeración de mis puertos. Esto no es un bot que circula por la red intentando conectarse a puertos típicos con contraseñas fáciles, esto tenía pinta de un <span class="solo-color-neon">escaneo tcp completo</span>, es decir, desde el primer al último puerto. Si te fijas, <span class="solo-color-neon">Zeek</span> me da el flag <span class="solo-color-neon">REJ</span>, eso significa que ese puerto específico estaba cerrado. Normalmente los hackers intentan hacer escaneos más silenciosos (posiblemente, con retrasos entre escaneo y escaneo, probando puertos específicos, etc...) pero en este caso, ha hecho un escaneo muy ruidoso y muy fácil de detectar.

Como analista, quedarte con un sólo log sería como ver la mitad de un cuadro, vamos a correlacionarlo en <span class="solo-color-neon">Wireshark</span>. Yo he buscado por dos tipos:

- <span class="solo-color-neon">tcp.flags.syn==1 && tcp.flags.ack==1</span> : En este caso, voy a ver qué puertos han mandado un paquete <span class="solo-color-neon">SYN + ACK </span> al ordenador del atacante. Recuerda el saludo a tres vías, <span class="solo-color-neon">SYN -> SYN + ACK -> ACK</span>. En este caso, la máquina víctima ha respondido (es muy sociable) con que tiene abiertos los puertos <span class="solo-color-neon">22, 80</span> (Puertos bien conocidos) y <span class="solo-color-neon">3306</span> (puerto registrado), puertos super típicos que seguramente estén en todos los ordenadores. El atacante ya sabe que el servicio <span class="solo-color-neon">ssh, http y MySQL</span> está corriendo en el sistema.

- ![puertos abiertos wireshark](https://github.com/user-attachments/assets/e054143d-8843-484e-af14-d6438b8bd046)

- <span class="solo-color-neon">tcp.flags.reset==1</span> : Aquí voy a ver qué puertos estaban cerradas en el momento del escaneo. Incluso si todo estuviera cerrado, un paquete <span class="solo-color-neon">RST</span> da mucha información, como por ejemplo, que el host está encendido.

- ![Wireshark puertos cerrados](https://github.com/user-attachments/assets/634bbd2d-33e1-4c10-bd04-f6d99c35f931)

Ya tenemos 2 logs confirmando que estamos sufriendo un escaneo de puertos. Con <span class="solo-color-neon">ELK</span>, en la sección messages, busqué la ip del atacante y me encontré esto:

![Elk escaneo de puertos](https://github.com/user-attachments/assets/ff5afe3e-2957-4415-a0a6-bb3ef961d7eb)

<span class="solo-color-neon">65536</span> nos hace pensar que el barrido ha sido completo. Ha escaneado todos los puertos que había en la red. El ordenador no está siendo víctima de un bot robacredenciales, está siendo el objetivo de alguien que quiere entrar. (Y además, alguien que no le importa ser descubierto, porque es muy ruidoso).

¿Deberías preocuparte si ves esto? La respuesta es, depende. Si no tienes contraseñas fáciles y sólo un par de puertos abiertos, normalmente no mucho. Ahora, aquí hay dos grandes connotaciones que quiero hacer:

- <span class="solo-color-neon">IP Externa:</span> Si la ip es externa, puedes estar algo más tranquilo, normalmente el escaneo de puertos en una red con buena seguridad sólo podría ser explotada con un exploit de día cero, es algo que puede pasar, pero no es lo más común. Además, protocolos como ssh o http son terriblemente difíciles de explotar.

- <span class="solo-color-neon">IP Interna:</span> Aquí sí que tendrías que tener mucho ojo. Si te están haciendo un escaneo desde una red interna, no sólo se ha colado el hacker en tu red, si no que muy probablemente esté intentando hacer <span class="solo-color-neon">movimientos laterales</span> en tu sistema. Piénsalo así, imagina que la ip de un oficinista te está haciendo un escaneo completo de tus puertos, eso no tiene ningún tipo de sentido, es un anomalía clara de un ataque. Está buscando en tus puertos si tienes alguna <span class="solo-color-neon">vulnerabilidad</span> o alguna <span class="solo-color-neon">configuración mal hecha</span> para poder entrar. Parece una tontería, pero una mala configuración o una contraseña débil puede hacer que toda tu red se venga abajo muy fácilmente.

La <span class="solo-color-neon">contención</span> debería ser inmediata. Si es una red externa, bloquear con el Firewall y poner un IPS es una estrategia muy efectiva. Si está en la red interna, hay que ver si ese ordenador es el único infectado. De ser así, y según las <span class="solo-color-neon">políticas de la empresa</span> (recuerda, no siempre vas a poder desconectar un ordenador de internet a la ligera, porque puedes hacer caer a todo el mundo) deberías <span class="solo-color-neon">aislar ese ordenador</span>, <span class="solo-color-neon">renovar credenciales</span> y buscar <span class="solo-color-neon">malware</span> (puede que no se haya colado nadie en el sentido de que un hacker ha entrado, ha podido ser un <span class="solo-color-neon">malware</span> que ha entrado por <span class="solo-color-neon">phishing</span> u otras causas y esté automatizado para ir saltando lateralmente). Si quieres un consejo para el <span class="solo-color-neon">malware</span>, mira <span class="solo-color-neon">procesos</span> (tanto hijos como padres), <span class="solo-color-neon">archivos conectados a internet</span> y <span class="solo-color-neon">persistencia</span>, si hay algún archivo sospechoso en todo ello, activa la bandera roja.

Aquí también hay que tener ojo avizor, podría ser un <span class="solo-color-neon">falso positivo</span>. Imagina que tu empresa ha contratado a un equipo de hackers para poner a prueba sus defensas. Un escaneo legítimo con herramientas como <span class="solo-color-neon">OpenVas</span> podría hacer saltar la alarma de que es alguien escaneando para un ataque o también podría ser que alguien esté mirando qué tan fuertes son las defensas y mejorar los puntos débiles. Es imprescindible la comunicación entre equipos y estar atento a los <span class="solo-color-neon">tickets</span> o <span class="solo-color-neon">emails de la empresa</span>.

Una de las mejores formas de evitar las vulnerabilidades que podría mostrar un escaneo de puertos, es <span class="solo-color-neon">limitar la exposición de ataque</span>. Dejar única y exclusivamente los puertos abiertos que se van a usar, si alguno no se va a usar, mejor ciérralo, es una medida de seguridad muy simple pero terriblemente efectiva. Si lo quieres llevar más allá, <span class="solo-color-neon">segmenta</span>. Segmenta tu red local para que, en el caso de que alguien encuentre una vulnerabilidad y la explote, el departamento afectado sea mínimo, eso contiene muchísimo la propagación. Y lo más importante, siempre (SIEMPRE) <span class="solo-color-neon">mantén tus aplicaciones actualizadas.</span> Entrar por un puerto con un servicio bien configurado es tremendamente difícil, pero si tiene una vulnerabilidad por no actualizar regularmente, eso se convierte en un muro de papel, caerá muy pronto.

- <span class="solo-color-neon">Laboratorio creado:</span> 07/08/2025
- <span class="solo-color-neon">Laboratorio subido:</span> 18/02/2025
