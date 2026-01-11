---
layout: post
title: "Caza Proactiva de Persistencia: Técnicas más usadas + Génesis"
date: 2026-01-06
categories: [threat-hunting]
permalink: /TH-persistencia/
---

# Caza Proactiva de Persistencia: Técnicas más usadas + Génesis

- <span class="solo-color-neon">Este post es para fines educativos y de investigación. Su objetivo es ayudar a defensores a identificar técnicas de persistencia reales.</span>
- <span class="solo-color-neon">Los archivos y rutas mostradas son simulaciones inofensivas. No se proporciona ni se fomenta el uso de código malicioso real.</span>
- <span class="solo-color-neon">El autor no se hace responsable del uso indebido de esta información ni de posibles daños en sistemas donde le usuario intente replicar estas técnicas.</span>
- <span class="solo-color-neon">Este repositorio NO contiene binarios ejecutables ni enlaces a malware real.</span>

Una de las cosas que siempre hago, tanto con mi equipo con los de otros, es suponer que ya está infectado. Da igual cuántas herramientas antimalware he pasado, me gusta siempre suponer que el ordenador está infectado para hacer una exploración más exhaustiva, por si las herramientas no cazan el malware. Y con la <span class="solo-color-neon">persistencia</span> siempre, siempre lo hago. Para el que no lo sepa, la persistencia es la capacidad del malware para <span class="solo-color-neon">ejecutarse tras cada inicio del sistema</span>. Si un malware crea <span class="solo-color-neon">persistencia</span>, tras cada inicio del ordenador estará activo. Hoy voy a bucear en las persistencias más comunes, enseñarte dónde se puede esconder el malware en ellas y por qué. Partimos de la hipótesis de que el sistema está infectado y que las herramientas antimalwares no han lanzado ninguna alarma. Yo lo que siempre hago es mirar los procesos activos del sistema y la persistencia. Si no veo nada ahí, es complicado que un malware haya infectado el sistema (ojo, no es que no haya amenazas, pero de malwares es poco probable, buscan mucho la supervivencia).

La primera son dos claves del registro de Windows, las llaves Run:

![HKCU imagen](https://github.com/user-attachments/assets/efcbb34c-c0e3-4a35-9e8e-549fb82fa878)
![HKLM imagen](https://github.com/user-attachments/assets/bd536c8c-2464-4ddf-a590-f025138bb1e9)

Son las más típicas, se usan muchísimo. Es una persistencia muy fácil de detectar, sólo hay que bucear en esas llaves del registro valores raros (Lo que es una bandera roja son archivos en temporales, archivos en la carpeta <span class="solo-color-neon">%appdata%</span>, <span class="solo-color-neon">%localappdata%</span>, <span class="solo-color-neon">%programfiles%</span> o incluso en <span class="solo-color-neon">ProgramData</span> (Pero si el archivo está dentro de una carpeta de estos directorios, la gravedad baja, yo me refiero a los archivos justo en esos mismos directorios, porque están pensados para albergar directorios, no archivos).

Con esta táctica los hackers hacen uso de la técnica <span class="solo-color-neon">T1547.001 (Boot or Logon Autostart Execution: Registry Run Keys/Startup Folder)</span>, tremendamente importante darse cuenta, en mi máquina, por ejemplo, no hay rastro de malware en ninguna de ellos:

![HKCURUN](https://github.com/user-attachments/assets/cca799a3-a4c8-4dc4-b0a8-d2a251a9079e)
![HKLMRUN](https://github.com/user-attachments/assets/9960ecc5-104a-491a-876d-61dbd735495c)

Esta que viene me gusta mucho, porque es tremendamente eficiente y difícil de ver. En este caso, el malware puede cargarse junto a la rama de <span class="solo-color-neon">winlogon</span> en el valor <span class="solo-color-neon">userinit</span> o <span class="solo-color-neon">Shell</span>. En <span class="solo-color-neon">userinit</span> este valor siempre debe estar como el que tengo yo (coma incluido, ni se te ocurra borrar esa coma). Pero si ves una ruta justo después de la coma, muchísimo ojo, porque eso no debería estar ahí. Yo todavía no me he encontrado ningún archivo legítimo después de esa coma (puede haberlos, pero lo dudo muchísimo). Así que si ves algo ahí después de la coma, <span class="solo-color-neon">99% que no será benigno</span>. Igual pasa con el valor <span class="solo-color-neon">Shell</span>, normalmente está en <span class="solo-color-neon">explorer.exe</span>, si está apuntando a otro proceso es otra bandera roja. Con esto, el malware puede cargarse al iniciar sesión en tu Windows, no deja rastro y mucha gente no conoce esta rama del registro ni para qué sirve. 

![Winlogon clave](https://github.com/user-attachments/assets/8ac79deb-fe2f-48ed-8be9-74e5a5dd37f6)

Esta técnica es <span class="solo-color-neon">T1547.004 (Winlogon Helper DLL)</span> y es más complicada de detectar. En mi caso, todo está limpio:

![WINLOGON](https://github.com/user-attachments/assets/0c6d609a-ccf7-4a2c-b475-6877ad58a9ff)

Otra que sigue y esta vez no te voy a hablar del registro (te doy un respiro) es una carpeta en la que los archivos se cargan al inicio (en realidad son dos rutas, pero la carpeta en sí es la misma, aunque con una ligera variación entre ambas).

Aquí seguramente lo único que veas son archivos <span class="solo-color-neon">.lnk</span> (que son accesos directos) aunque también puedes ver otro tipo de archivos (he visto a veces ejecutables, pero no son muy eficientes en esa ruta). También puedes ver archivos <span class="solo-color-neon">.vbs</span> y <span class="solo-color-neon">.ps1</span> (ambos bandera roja). Los <span class="solo-color-neon">.lnk</span> van a apuntar directamente a archivos ejecutables, es decir, Windows ejecutará el acceso directo y el acceso directo ejecutará el archivo. Se crea una nueva <span class="solo-color-neon">persistencia</span>. Un dato curioso, aunque tu ordenador esté en español, esa ruta funciona en todos los Windows y también te funcionará en español, pero si la intentas copiar te saldrá en inglés (dije que era curioso, no divertido). En mi caso, no hay nada (ni benigno ni maligno):

![Startup](https://github.com/user-attachments/assets/1c3d8e2f-d33a-41c2-a923-5f0f876076b4)
![Startup2](https://github.com/user-attachments/assets/5d7bdd95-700a-4c22-953e-5d0e9fdd9fda)

Esta técnica que usan es la misma que la primera, <span class="solo-color-neon">T1547.001 (Boot or Logon Autostart Execution: Registry Run Keys/Startup Folder).</span>

Una entrada del registro que me gusta por la misma razón que la de<span class="solo-color-neon">winlogon</span>, es la de <span class="solo-color-neon">IFEO</span>. Aquí, el malware se ejecutará a la vez que la clave a la que apunta. Por ejemplo, si hay un archivo (ojo, el valor de esa clave debe ser debugger y en datos la ruta, el <span class="solo-color-neon">debugger</span> es una bandera amarilla, hay que investigarla siempre) en <span class="solo-color-neon">Excel.exe</span>, significa que cada vez que Excel.exe se ejecute, también ejecutará la ruta del <span class="solo-color-neon">debugger</span>. Muy, muy sigiloso.

![IFEO registro](https://github.com/user-attachments/assets/667823fe-7d4e-41af-a95c-6774c1f72780)

En este caso, yo en este caso sí he encontrado algo:

![Debugger](https://github.com/user-attachments/assets/d7901bd0-4d59-4710-8b8e-60eb0f1ff219)

Eso quiere decir que cada vez que ejecute <span class="solo-color-neon">Notepad.exe</span>, se va a iniciar también el archivo al que apunta: <span class="solo-color-neon"></span>. Ahora tengo un archivo con banderas rojas por todas partes (¿Por dónde empiezo? Está en la carpeta temporal, usa <span class="solo-color-neon">IFEO</span> para ejecutarse, tiene un nombre para causar confusión y enmascarar acciones). 

Está usando la técnica <span class="solo-color-neon">T1546.012 (Image File Execution Options Injection)</span>. Ya lo he pillado bien, pero voy a seguir investigando con la última que quiero poner en este post.

La última, y no menos importante, es también bastante conocida, casi tanto como la de la <span class="solo-color-neon">clave Run</span>, pero, desde mi punto de vista, me parece más sofisticada. Estoy hablando de las <span class="solo-color-neon">tareas programadas</span>. Una tarea programada es una tarea que se ejecuta cada x tiempo o a una hora concreta. Esto quiere decir que, si esa tarea apunta a un ejecutable, ejecutará ese ejecutable con las órdenes que le dio el hacker (por ejemplo, cada día, cada x minutos, a una hora concreta...). Se encuentra en la siguiente ruta (aunque también se puede encontrar en C:\windows\Tasks):

![Tareas ruta](https://github.com/user-attachments/assets/bf6adc81-9c38-42b6-9504-81ed570843f3)

Aquí te vas a encontrar archivos (aunque puede haber primero una carpeta) en la que puede haber ejecuciones de binarios (aunque no todas, no todas las tareas ejecutan un archivo). En este caso, yo veo una rara en mi equipo:

![TareaListado](https://github.com/user-attachments/assets/af777c24-5156-40bf-a6fd-5b5f5cad64f6)

¿Actualización? ¿No es un poco… curioso? Por lo menos a mí me lo parece, así que voy a abrir el archivo con el bloc de notas:

![Tarea](https://github.com/user-attachments/assets/af0f4640-d9df-4dd6-a13a-fae5175ff048)

Aquí hay varias cosas, la primera que quiero resaltar es la de <span class="solo-color-neon">StartBoundary</span>, esa es la fecha y hora en la que la tarea se ejecutará por primera vez. En <span class="solo-color-neon">DaysInterval</span> define cada cuántos días se va a ejecutar, en nuestro caso, cada 1. Y en el <span class="solo-color-neon">command</span> está la joya de la corona, la ruta de nuestro querido malware. En el caso de que el usuario no haga uso del Notepad eso asegurará que por lo menos cada día se intente ejecutar el archivo si el equipo está activo. También hay otro comando a tener en cuenta, aunque aquí no aparece, que es <span class="solo-color-neon">Arguments</span>, los cuales puede funcionar de la misma manera que el <span class="solo-color-neon">command</span>.

En este caso está usando la técnica <span class="solo-color-neon">T1053.005 (Scheduled Task/Job)</span>.

Como mirar todo esto es sumamente aburrido (pero necesario, hay que conocer el sistema sin el uso de herramientas) yo voy a usar una herramienta creada por mí, llamada Génesis (nombre y GUI todavía en fase Beta, aunque código funcional), para localizar estos puntos de <span class="solo-color-neon"persistencia</span> (tiene muchísimas más funciones, pero me voy a centrar en donde hemos detectado el malware).

![Genesis1](https://github.com/user-attachments/assets/b0b46111-8b61-4fd9-92ea-a5e8721de56a)

![Genesis2](https://github.com/user-attachments/assets/77c8492b-f287-4718-b20f-7fb72d6dbd52)

Al terminar, en el informe, voy a irme a la sección <span class="solo-color-neon"31M</span> y <span class="solo-color-neon"44M</span> (Ambas correspondientes a la sección de tareas y IFEO, con el valor "M" en tareas por recurrencia con las demás entradas y en IFEO por ser del HKLM (También hay una x64, que corresponde al WOW6432NODE, pero en este caso, aunque eliminaré las 2, con eliminar la de HKLM normal es suficiente). En este caso, la sección 31 corresponde a las tareas y la 44 al IFEO, pero también pondré la ruta del ejecutable para que lo mueva a cuarentena. Si te fijas, aparece al lado del archivo <span class="solo-color-neon"(Desconocido)</span>, indicando que no tiene un nombre de compañía válido, lo que refuerza la hipótesis de que podría ser potencialmente un malware.

En la entrada 31M:

![Genesis3](https://github.com/user-attachments/assets/9ae9bc28-abac-41ba-b40b-bae24438d103)

En la entrada 44M:

![Genesis4](https://github.com/user-attachments/assets/a19c7161-6d56-486f-91b5-c9c732fd5915)

Pongo en el <span class="solo-color-neon"script semiautomatizado</span> lo que quiero eliminar (automáticamente el script se encargará de verificar los números de la lista para saber dónde buscar y eliminar el contenido marcado). 

![Genesis5](https://github.com/user-attachments/assets/1cc3dded-9878-4db0-883e-d2cc2b9158a5)

Al pulsar el botón de “Ejecutar Script” el script empezará a eliminar lo marcado:

![Genesis6](https://github.com/user-attachments/assets/768bed52-0fff-44e3-8217-19c9ccb09f7b)

Aquí vemos el archivo ya en la cuarentena (Quiero implementar una función para renombrar el archivo y que así quede totalmente contenido). Si te fijas, la carpeta donde se guarda el archivo se genera en la cuarentena de Génesis siguiendo toda la ruta donde estaba el ejecutable (así, si se quiere restaurar, será mucho más fácil).

![Genesis8](https://github.com/user-attachments/assets/2aa22023-034b-472d-bc77-197d4b786bad)

Una vez el script acabe, un reporte se mostrará (que no haya podido eliminar la entrada “x64” es normal, porque ya la ha eliminado en la otra clave. También, hubo que forzar a la eliminación forzosa de la tarea, ya que no se registró la tarea para la simulación, entonces el programa entendió que no se pudo borrar, porque no existe, y pasó directamente a la eliminación del archivo de tarea, es una cosa que tengo que arreglar).

![Genesis7](https://github.com/user-attachments/assets/a90f4324-b5cf-4994-853b-6234e8a7c77f)

Con esto, sólo quería bucear en algunas técnicas de la táctica Persistencia, me parece muy útil porque la inmensa mayoría del malware la usa. Hay aún más técnicas que iré poniendo en próximos posts.

- <span class="solo-color-neon">Laboratorio creado: </span> 06/01/2026
- <span class="solo-color-neon">Laboratorio editado y publicado: </span> 11/01/2026
