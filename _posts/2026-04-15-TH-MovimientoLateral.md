---
layout: post
title: "Caza Proactiva: Detección, contención, erradicación y hardening de un envenenamiento LLMNR y Movmiento Lateral."
date: 2026-04-14
categories: [threat-hunting]
permalink: /TH-MovimientoLateral/
---

# Caza Proactiva: Detección, contención, erradicación y hardening de un envenenamiento LLMNR y Movmiento Lateral.

<span class="solo-color-neon">- Este contenido se ha creado exclusivamente con fines educativos y de concienciación en ciberseguridad.</span>
<span class="solo-color-neon">- Este laboratorio se ha hecho en un entorno controlado, con máquinas virtuales propias y con consentimiento.</span> 
<span class="solo-color-neon">- El uso de las herramientas y técnicas descritas en este laboratorio sobre sistemas o redes sin autorización previa y explícita es ilegal y éticamente reprobable.</span>
<span class="solo-color-neon">- El autor no se hace responsable del mal uso que terceros puedan dar a esta información.</span> 
<span class="solo-color-neon">- Recuerda: el objetivo es aprender a defender, no a destruir.</span>


Hoy presento un ejercicio de <span class="solo-color-neon">purple team</span>. El objetivo es entender la mecánica de un ataque común para fortalecer nuestras capacidades de detección y respuesta en el SOC. No sólo voy a hacer el laboratorio desde el ala <span class="solo-color-neon">blue team</span>, sino también del <span class="solo-color-neon">red team</span>, para que podáis aprender cómo funciona un poco Windows en ese aspecto (Detectarlo está muy bien, pero sabiendo dónde flaquea te da una perspectiva mucho más amplia).

Lo primero que hice fue ponerme en la perspectiva de un hacker que ya tiene acceso a la red local. Para entrar en algún ordenador, en este caso, usé un <span class="solo-color-neon">envenenamiento de LLMNR</span> (<span class="solo-color-neon">T1557.001</span> de la táctica Credential Accesss). Pero, ¿para qué sirve esto?. Bueno, cuando un ordenador busca un recurso y no lo encuentra, suelta una petición en <span class="solo-color-neon">broadcast</span> diciendo a toda la red que está buscando ese recurso. Un hacker con una herramienta como <span class="solo-color-neon">Responder</span>, puede hacerse pasar por ese recurso, pedir las <span class="solo-color-neon">credenciales</span> para entrar y obtenerlas.

En este caso, yo he ejecutado <span class="solo-color-neon">responder</span>, como veis, ya está escuchando:

<img width="737" height="666" alt="Imagen1" src="https://github.com/user-attachments/assets/a216805d-d952-4f0f-b514-f54c16cdea15" />

<img width="529" height="412" alt="Imagen2" src="https://github.com/user-attachments/assets/a12c52e6-f473-4fd8-8e61-df5eb304006d" />

En mi Windows, quiero buscar un recurso compartido llamado <span class="solo-color-neon">Contabilidad</span>, pero por error, pongo <span class="solo-color-neon">Contablidad</span>. Ese recurso no existe, por lo tanto, Windows va a lanzar una <span class="solo-color-neon">petición broadcast</span> en toda la red para encontrarlo. Como estoy con <span class="solo-color-neon">Responder</span> activo, Responder se va a hacer pasar por el servicio, la cuenta de Windows le va a dar el <span class="solo-color-neon">hash NTLM</span> y <span class="solo-color-neon">Responder</span> lo va a guardar:

<img width="802" height="535" alt="Imagen3" src="https://github.com/user-attachments/assets/896d09dd-c769-46a6-adf3-6e268429c234" />

<img width="1303" height="380" alt="Imagen4" src="https://github.com/user-attachments/assets/2a530fd5-8b3f-46e6-ac0c-f09abae11b2e" />

Ahora ya tengo el hash, ese hash lo podría reenviar a la máquina con <span class="solo-color-neon">SMB Relay</span> para logearme, pero en este caso, voy a crackearlo con <span class="solo-color-neon">John the Ripper</span> y obtener la contraseña:

<img width="932" height="296" alt="Imagen5" src="https://github.com/user-attachments/assets/bd3bd74b-cc48-4f38-a8d1-8d850762e021" />

Ahí obtengo la contraseña, que es 1234 (Sí, es una contraseña muy tonta, pero si la pongo muy difícil john no la podrá resolver). Ya tengo todo lo que necesito, tengo la ip, el nombre de la cuenta y su contraseña. 

Ahora lo que necesito es una herramienta tipo <span class="solo-color-neon">impacket-psexec</span>, gracias a ella, podré conectarme vía <span class="solo-color-neon">SMB</span> (Recuerda, puerto 445) para poder acceder a ese ordenador (Se puede hacer también por ssh pero normalmente no está operativo por defecto). Esto es muy útil para hacer <span class="solo-color-neon">movimientos laterales</span> (<span class="solo-color-neon">T1021.002</span> de la táctica Lateral Movement). Lanzo <span class="solo-color-neon">impacket</span> y me crea una <span class="solo-color-neon">Shell interactiva</span>, ya estoy dentro del equipo:

<img width="746" height="307" alt="Imagen6" src="https://github.com/user-attachments/assets/3a8feb84-aaa1-460f-a2fc-ba46e73143b0" />

Para un hacker es un gran paso, sin embargo, una <span class="solo-color-neon">Shell interactiva</span> es muy volátil, si ese equipo se aísla da la red o se apaga, el hacker pierde el acceso al ordenador. Un hacker más sofisticado descargaría algún regalo personalizado para hacer un <span class="solo-color-neon">command and control</span>, algún tipo de <span class="solo-color-neon">ransomware</span>, alguna <span class="solo-color-neon">exfiltración</span>... No creo que se quedaran sólo en la Shell.

Y ahora, lo que realmente nos interesa, ¿Cómo lo detectamos nosotros?. Si bien no hay una manera única, tenemos varias formas de ver un movimiento lateral de estas características. Lo primero que tenía era en Kali corriendo <span class="solo-color-neon">Wireshark</span>. En él se ve cómo <span class="solo-color-neon">impacket</span> intenta hacer la conexión mediante autenticación <span class="solo-color-neon">NTLM</span>. Básicamente, lo que ocurre en esos paquetes, es que <span class="solo-color-neon">impacket</span> está estableciendo qué protocolo van a usar ambas máquinas y luego va a autenticarse con las credenciales robadas. Básicamente, la máquina de Windows va a darle un número llamado <span class="solo-color-neon">nonce</span> a Kali, Kali usando <span class="solo-color-neon">impacket</span> va a cifrar ese número con la contraseña filtrada y se lo va a devolver. El cifrado será correcto y le dejará pasar. El atacante ya ha logrado entrar:

<img width="1642" height="885" alt="Imagen6 1" src="https://github.com/user-attachments/assets/5fc36742-34b3-4f50-8be3-bae3857172b0" />

Se puede ver un montón de paquetes con la información de <span class="solo-color-neon">EncryptedSMB3</span>. Ese tráfico está cifrado, ahí me hubiera gustado enseñarte cómo el atacante toca los recursos compartidos de <span class="solo-color-neon">IPC$</span> y <span class="solo-color-neon">ADMIN$</span> pero más adelante, con la auditoría, también te lo mostraré. Y por cierto, como dato curioso, el protocolo SMB al estar cifrado ya incopora la firma SMB, por lo tanto, con esto se verifica que un ataque <span class="solo-color-neon">SMB Relay</span>, en este ordenador, no podría haberse llevado a cabo.

Una de las cosas más importantes para detectar los <span class="solo-color-neon">movimientos laterales</span> son los eventos de Windows. Al tener las auditorías disponibles, nos dan una información muy preciada para nuestros análisis. Lo primero que vamos a hacer es detectar intromisiones, el evento <span class="solo-color-neon">4624</span> nos viene de lujo aquí. En este caso, el evento <span class="solo-color-neon">4624</span> tiene un inicio de sesión <span class="solo-color-neon">tipo 3</span>, es decir, <span class="solo-color-neon">inicio de sesión de tipo red</span>, común en accesos <span class="solo-color-neon">SMB</span> remotos. Aunque es algo común en entornos corporativos, es una señal que hay que prestar atención. Lo único que no te tendría que cuadrar es la dirección ip desde donde ha iniciado sesión. No hay nadie sentado iniciando sesión, hay alguien desde un ordenador remoto iniciando sesión:

<img width="408" height="297" alt="Imagen8" src="https://github.com/user-attachments/assets/6339f5da-7981-4f0c-b956-16716426954b" />

<img width="316" height="289" alt="Imagen9" src="https://github.com/user-attachments/assets/e5b6974b-2424-42b5-a201-915bcfd91707" />

Otro de los eventos más importantes es <span class="solo-color-neon">7045</span>, es decir, <span class="solo-color-neon">creación de servicios</span>. Para lograr la <span class="solo-color-neon">Shell inversa</span>, <span class="solo-color-neon">impacket</span> debe mandar paquetes para construir un ejecutable en una carpeta con acceso a escritura (Casi siempre será <span class="solo-color-neon">ADMIN$</span>, o lo que es lo mismo, la ruta C:\Windows). Luego, se conectará al recurso <span class="solo-color-neon">IPC$</span> (aquí no puedes escribir nada, pero sirve para inicializar el ejecutable creado en <span class="solo-color-neon">ADMIN$</span> a través de svcctl). Primero, escribe un binario en <span class="solo-color-neon">ADMIN$</span>, luego, lo ejecuta a través de tuberías desde <span class="solo-color-neon">IPC$</span>. Eso genera la inicialización de un servicio, un servicio que corre con privilegios de <span class="solo-color-neon">SYSTEM</span>, es decir, tiene aún más nivel que un <span class="solo-color-neon">administrador</span> (acceso directo al kernel, saltarse el UAC...):

<img width="499" height="464" alt="Imagen7" src="https://github.com/user-attachments/assets/d3e3c92c-7b55-4b14-a9f5-25546e8c15fa" />

¿Cómo vemos que ha tocado esos recursos? Fácil, el evento <span class="solo-color-neon">5140</span> de Windows nos lo está diciendo, se ha tocado los recursos de <span class="solo-color-neon">ADMIN$</span> y <span class="solo-color-neon">IPC$</span>:

<img width="459" height="256" alt="Imagen10" src="https://github.com/user-attachments/assets/f68a86e7-e162-4c7a-82df-bba5a23d6588" />

<img width="459" height="280" alt="Imagen11" src="https://github.com/user-attachments/assets/3fb008a4-659b-4566-9032-2a21a5cbba56" />

Otro evento que me gusta mirar es el evento <span class="solo-color-neon">4672</span>. Aunque para mucha gente no es tan conocido como los típicos <span class="solo-color-neon">4624</span>, <span class="solo-color-neon">4688</span> y demás, es tremendamente importante, porque nos indica que ha habido una <span class="solo-color-neon">escalada de privilegios</span>. Si te fijas en la captura que pondré a continuación, se le concede al usuario muchos privilegios como alterar la memoria de otro proceso, permitir cargar drivers... Es decir, le está dando privilegios muy importantes, prácticamente el hacker puede hacer lo que quiera con ese ordenador.

<img width="455" height="240" alt="Imagen11-4672" src="https://github.com/user-attachments/assets/b7a6cedf-6759-466c-8b5f-b5132a4e45b4" />

Ahora que lo hemos detectado, vamos a <span class="solo-color-neon">contenerlo</span>. Si el hacker ya ha entrado, probablemente ya ha hecho mucho daño. La mejor manera de contenerlo es <span class="solo-color-neon">aislando ese ordenador</span>, ya sea sacándolo de la red privada o desconectándolo (Esto varía según el protocolo de la empresa, si es un ordenador esencial para el funcionamiento no puedes simplemente aislarlo). Lo bueno de este movimiento es que, aunque apagues y vuelvas a iniciar el ordenador, <span class="solo-color-neon">la evidencia no se pierde</span> (Tienes la ip, tienes el programa creado en C:\windows, el servicio...). Siempre hay que sacar evidencias, tanto de los logs como del propio archivo (con algún tipo de análisis estático).

Llega el momento de la <span class="solo-color-neon">fase de remediación</span>. Esto variará enormemente dependiendo del daño que haya causado el hacker, no es lo mismo eliminar las tuberías por donde ha hecho el <span class="solo-color-neon">movimiento lateral</span> que eliminar si ha descargado algún regalo más. En mi caso, sólo usé <span class="solo-color-neon">el movimiento lateral</span>, sin nada más, por lo tanto, si el hacker hubiera desplegado un <span class="solo-color-neon">ransomware</span> no sólo hay que eliminar los rastros del <span class="solo-color-neon">movimiento lateral</span>, si no el propio <span class="solo-color-neon">ransomware</span>. Yo voy a usar mi herramienta <span class="solo-color-neon">Génesis</span>, ya la usé en otro post, me permitirá detectar los servicios creados y el archivo causante:

<img width="782" height="563" alt="Imagen12" src="https://github.com/user-attachments/assets/21b0f28a-9309-49c2-92e0-14673c850b8f" />

<img width="701" height="517" alt="Imagen13" src="https://github.com/user-attachments/assets/16924001-117b-483d-ac1d-eae23cad29bd" />

Ya los he detectado, ahora sólo me queda usar su <span class="solo-color-neon">función de script</span>, copio las líneas afectadas y Génesis se encargará de analizar qué tipo de eliminación usará al leer el código numérico y detectará una ruta sola que le indicará que es una carpeta/archivo (Aquí, si le paso una ruta aislada, tiene dos opciones, o es una carpeta o es un archivo, al analizar si es "A" o "D", sabrá si tiene que eliminar un archivo o una carpeta):

<img width="820" height="329" alt="Imagen14" src="https://github.com/user-attachments/assets/b2e4a4f6-9bd5-4448-a648-e02a965214ee" />

<img width="545" height="161" alt="Imagen15" src="https://github.com/user-attachments/assets/9672d001-6f42-4325-9a3d-091e5063c4be" />

<img width="557" height="337" alt="Imagen16" src="https://github.com/user-attachments/assets/379e676d-f602-44bc-8016-42b5e8ab16a1" />

<img width="816" height="335" alt="Imagen17" src="https://github.com/user-attachments/assets/5a8f6989-fcf3-4204-bd20-b009f76ef50c" />

En el log de eliminado, sale que todo ha sido resuelto correctamente. Se ha eliminado el servicio, se ha eliminado la ruta del registro asociada al servicio y el archivo.

<img width="820" height="260" alt="Imagen18" src="https://github.com/user-attachments/assets/239da556-5dc6-4ff2-a11c-ea0836e7c535" />

Antes de esto, es imperativo <span class="solo-color-neon">cambiar la contraseña de la cuenta afectada</span>. Si sigue teniendo las mismas credenciales, entrar de nuevo es tremendamente fácil. En este caso, se elige una contraseña larga (Más de 12 caracteres), combinando letras mayúsculas, mínusculas, símbolos y números. De esta forma, incluso si el <span class="solo-color-neon">hash NTLM</span> le llega al hacker con el <span class="solo-color-neon">responder</span> de nuevo, no podrá ser crackeada, es demasiado compleja para ello.

La manera más eficaz para evitar esto, es mediante una <span class="solo-color-neon">GPO</span> desactivar los protocolos <span class="solo-color-neon">NetBIOS y LLMNR</span>. Son protocolos antiguos que no suelen traer nada bueno y son muy usados por los hackers en explotaciones. Otra medida muy efectiva es lo que se conoce como el <span class="solo-color-neon">mínimo privilegio</span>. ObisidanHunter lo mismo era un oficinista, ¿qué hacía con privilegios de administración en esa máquina? Siemrpe hay que otorgar los permisos de administración con cuentagotas, jamás a la ligera. Si ObsidianHunter era algún tipo de administrador de sistemas, está más justificado. Y, lo más importante para mí, sería establecer una <span class="solo-color-neon">firma SMB</span>, porque te evita por ejemplo ataques Relay. Al tener una firma válida, aseguramos que el paquete no ha sido vulnerado por el camino, que es de quien dice ser. También muy importante es <span class="solo-color-neon">separar contraseñas</span>, que un administrador no use las mismas de un usuario o similar, esto se puede implementar con <span class="solo-color-neon">LAPS</span>, ya que si el hacker obtiene la contraseña de un ordenador, al tener los demás otra distinta, no podrá ir saltando de un lado a otro, es bastante potente.

A lo mejor, te interesa también la típica carta de <span class="solo-color-neon">el cazador cazado</span>. Puedes implementar un recurso compartido inexistente y que toda la empresa sepa que ese recurso no se puede tocar. Le pones un nombre atractivo como <span class="solo-color-neon">BDEmpresa</span>. Nadie en tu oficina lo va a tocar, pero si un hacker ha entrado y ve eso, se pensará que ahí tienes todo lo importante con respecto a la base de datos de la empresa y querrá entrar ahí. EL IDS, la auditoría e incluso reglas del SIEM deberían detectar cualquier usuario que intenta acceder a ese recurso, porque 99,99% será un hacker. Con esto, creas un objeto muy atractivo en tu red local (Sería algo así como un <span class="solo-color-neon">honeyobject</span> o algo así) y los propios hackers se descubren ellos solos.

He creado una regla <span class="solo-color-neon">SIGMA</span> simple para implantar en el <span class="solo-color-neon">SIEM</span>. Si hay un inicio de sesión de tipo 3 y luego se toca <span class="solo-color-neon">IPC$</span> o <span class="solo-color-neon">ADMIN$</span> salta una alarma con severidad alta. En teoría, esto es muy raro, se ve en <span class="solo-color-neon">movimientos laterales</span> pero también se puede ver en auditorías de los administradores. Por eso, ojo clínico con lo que se está viendo y ver si realmente es una alarma o un <span class="solo-color-neon">falso positivo</span> (porque va a generar <span class="solo-color-neon">falsos positivos</span>, eso seguro, pero va a hacer que no pases ningún <span class="solo-color-neon">falso negativo</span>).

```yaml
title: Correlación de Movimiento Lateral 
id: 15042026
description: Correlaciona un inicio de sesión con un acceso a objeto compartido.
status: activo
author: ObsidianHunter

events:
    - name: login3
      logsource:
          product: windows
          service: security
      detection:
          selection:
              EventID: 4624
              LogonType: 3
    - name: accesocompartido
      logsource:
          product: windows
          service: security
      detection:
          selection:
              EventID: 5140
              ShareName: 
                  - '*\ADMIN$'
                  - '*\IPC$'
correlation:
    type: temporal
    rules:
        - login3
        - accesocompartido
    group-by:
        - ComputerName
        - SubjectUserName
    timespan: 2m
    condition: login3 then accesocompartido
level: critical
```


Recuerda, lo más importante aquí es la <span class="solo-color-neon">prevención</span>. Nadie quiere llegar y ver que todo está roto y exfiltrado. Usa siempre <span class="solo-color-neon">contraseñas robustas</span> como dije antes, usa el <span class="solo-color-neon">mínimo privilegio</span> siempre y ten en cuenta las protecciones para los protocolos <span class="solo-color-neon">SMB, NetBIOS y LLMNR.</span>

- <span class="solo-color-neon">Laboratorio creado:</span> 14/04/2026
- <span class="solo-color-neon">Laboratorio subido:</span> 15/04/2026
