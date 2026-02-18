---
layout: post
title: "Adversarial Hacking y Training: Evaluación de Robustez en Modelos de Detección de Phishing"
date: 2026-02-18
categories: [ia-secops]
permalink: /ml-secops-adversarialphishing/
---

# Adversarial Hacking y Training: Evaluación de Robustez en Modelos de Detección de Phishing

Hoy voy a hacer algo diferente. He creado un pequeño (comparado con los modelos del mercado, es diminuto) modelo de <span class="solo-color-neon">machine learning</span> para detectar <span class="solo-color-neon">Phishing</span>. Tiene pocas url y pocos parámetros de detección, pero quiero enseñarte cómo funciona el <span class="solo-color-neon">modelo de evasión de machine learning</span>, lo que viene siendo un <span class="solo-color-neon">ataque adversario</span>. Como digo, es un modelo muy simple pero la metodología es la misma. Primero, este modelo funciona bajo un modelo <span class="solo-color-neon">white-box</span>, es decir, yo veo el modelo (ya sea por filtración del modelo o por ingeniería inversa para entender los pesos de las características). Lo primero que haré será presentar el modelo (Si sabes de machine learning, no vas a ver gran cosa):

![Mostrar Codigo1](https://github.com/user-attachments/assets/12caed8c-4429-4666-96c1-be2192b1a593)


![Mostrar Codigo2](https://github.com/user-attachments/assets/fbdd6963-b218-4b2b-b435-6626ad6bc8b6)

He utilizado la arquitectura del <span class="solo-color-neon">random forest</span>, es decir, <span class="solo-color-neon">aprendizaje supervisado</span>. Creo que para todo lo que tiene que ver con <span class="solo-color-neon">phishing</span> y <span class="solo-color-neon">malware</span> es el mejor que puedes adaptar a tu modelo. En este caso no hay mucho misterio, tengo unas funciones que me permiten sacar 4 parámetros de una url. La primera es la longitud de la página web (los dominios <span class="solo-color-neon">phishing</span> suelen ser muy largos), la segunda es la cantidad de puntos (los dominios <span class="solo-color-neon">phishing</span> suelen tener muchos puntos en su url), la tercera son las palabras clave, que identifiqué arriba, mirando <span class="solo-color-neon">phistank</span> vi que esas palabras son las que más aparecían en dominios que intentaban ocultarse un poco más (ninguna en español, una pena). La cuarta lo llamé seguro, es si tiene <span class="solo-color-neon">https</span> o <span class="solo-color-neon">http</span> (Antes los <span class="solo-color-neon">phishing</span> eran en http, pero hoy en día casi todos tienen https, pero lo dejo para el modelo porque habrá algunos que aún lo usen así). Luego, catalogo el modelo, <span class="solo-color-neon">0</span> es seguro y <span class="solo-color-neon">1</span> es phishing. 

Luego viene la parte de asignar los parámetros la primera url será la columna uno, teniendo una longitud de 15 letras, un punto, no tiene palabras clave y tiene cifrado, al modelo le digo que no es <span class="solo-color-neon">phishing</span>. Va hacer lo mismo con todas las demás, así es cómo el modelo aprende. Lógicamente, cuantas más webs tengas más fiable es tu modelo y más difícil de romper, este va a ser fácil, iré poniendo más complicados a medida que vaya cogiendo datasets públicos. 

Para esta tarea, como dije antes, he usado <span class="solo-color-neon">random forest</span>, con 100 árboles. ¿Qué es eso? Pues un <span class="solo-color-neon">árbol de decisión</span> es un nodo en el que cada nodo llega a una conclusión. Por ejemplo, el nodo 1 puede analizar todo y decir que es <span class="solo-color-neon">phishing</span>, el 2 puede decir que no lo es y así hasta 100. Si 90 árboles dicen que no es <span class="solo-color-neon">phishing</span>, el modelo lo catalogará como <span class="solo-color-neon">no phishing</span> al 90% de seguridad. Muchos analistas confían muchísimo en sus herramientas comerciales (No les voy a quitar toda la razón, algunas son muy buenas), pero si funcionan con <span class="solo-color-neon">machine learning</span>, la confianza no debería ser ciega. Como ahora veremos, si cambios los parámetros correctos, el modelo se rompe. Podría dejar pasar al navegador a una web infectada de malware y el modelo pensar que es una web legítima. Por eso, aunque sea confíe en las herramientas, debes conservar el <span class="solo-color-neon">juicio crítico</span> y no fiarte de nada al 100%.

Uso la biblioteca <span class="solo-color-neon">matplotlib</span> para luego mostrarte unos gráficos con los pesos. Como puedes intuir en el resto del código, si el bosque decide que es un <span class="solo-color-neon">phishing</span> (==1) te mostrará que es una url sospechosa con tanto por ciento de seguridad, pero si considera que <span class="solo-color-neon">no es phishing</span> (==0) te dirá que parece ser segura con un tanto por ciento de seguridad. Para escoger el caso, he cogido una página de <span class="solo-color-neon">phishing</span> que emula a <span class="solo-color-neon">PayPal</span> (recuerda, PayPal es una web confiable y muy segura, no estamos juzgando eso) si no que están intentando suplntarla. He escogido la web: <span class="solo-color-neon">http://verify-update-paypal-security[.]com/login[.]php </span>

Yo sólo con mirar los parámetros de construcción, sé que donde va a ir el mayor peso, seguramente, sea a la <span class="solo-color-neon">longitud</span>. ¿Por qué? Porque el modelo va a ver que cuando es una web larga es <span class="solo-color-neon">phishing</span> siempre, va a tener lo que se conoce como <span class="solo-color-neon">overfitting</span>, y si eso pasa, tu modelo deja de aprender y empieza a memorizar. Esto parece algo muy bueno, pero es fatal para tu modelo. Imagina, tu modelo ya no va a buscar patrones complejos, va a poner casi todo el peso en la longitud y cuando se encuentre una web legítima con una url larga la va a catalogar directamente de <span class="solo-color-neon">phishing</span>, aunque otros patrones digan que es una web benigna. Voy a usar <span class="solo-color-neon">matplotlib</span> para que veas:

![Importanciaprimermodelo](https://github.com/user-attachments/assets/116a6a75-e37c-46c6-a713-c291e31c9a7c)

El modelo tiene su mayor peso en la longitud (te lo dije) y también en los puntos (ha visto lo mismo que con la longitud, cada vez que había muchos puntos era <span class="solo-color-neon">phishing</span>). El <span class="solo-color-neon">60%</span> de la decisión recae si tiene algunos puntos y una longitud larga (hay webs legítimas que esto lo cumplen perfectamente, sobre todo si hablamos de subdominios). El siguiente mayor peso es para el http/https y el otro es para las palabras clave. Viendo esto, yo sé dónde tengo que atacar, debo atacar la longitud de la url y los puntos en mayor proporción, pero el http/https también es un destino jugoso, porque con <span class="solo-color-neon">Let's encrypt</span> puedes tener un certificado https rápidamente. 

Primer paso, vamos a realizar la evasión. Lo primero, al poner la web, efectivamente, tiene una seguridad del <span class="solo-color-neon">80% de que es phishing</span>, hasta aquí, nada impresionante:

![Iteraccion1](https://github.com/user-attachments/assets/a2cab53a-bd76-4eec-8316-87491c0211cd)

Sé que mi modelo confía mucho en el protocolo http, por lo tanto, voy a adaptar la web, voy a ponerla <span class="solo-color-neon">https</span>:

![Iteraccion2](https://github.com/user-attachments/assets/f78c9d96-ca55-4e9f-9d9d-4a2190bcedbd)

Sigue mostrándola sospechosa, pero ahora su porcentaje de seguridad es del <span class="solo-color-neon">60%</span>. Es decir, ha pasado de estar muy seguro a estar algo seguro, vamos por el buen camino. Ahora, lo que haré, será reducir los puntos, la longitud y quitarle una palabra clave (la de login):

![Iteraccion3](https://github.com/user-attachments/assets/7a130ea0-f430-492a-ae6b-f4d2f60b9232)

Boom, <span class="solo-color-neon">acabamos de romper el modelo</span>. El modelo ahora está al <span class="solo-color-neon">75%</span> seguro de que es una web legítima, cuando para un humano apenas hubo cambios. Ya está, si estamos en un modelo empresarial, nos comemos la web maliciosa con patatas. Ahora voy a ir más allá, quitaré más palabras clave y reduciré aún más la longitud (uno de los pilares del modelo):

![Iteraccion4](https://github.com/user-attachments/assets/cd6a4d07-5ee5-4a22-b8b1-e1970a9d448d)

Un 94% de que es una web legítima. Es decir, podrías tener un cartelito en la web que pusiera en letras grandes y fluorescentes que te van a meter el malware de su vida que el modelo seguiría pensando que estás a salvo. En este caso, fíjate en la web... No es de PayPal, repito (para mí es una web muy buena para comprar online, por la seguridad que ofrece por la ocultación de tus datos bancarios), no está en su subdominio, no es security.paypal.com, es paypal-security.com, es una web completa, no un subdominio. 

Bien, ahora voy a hacer un adversarial training, es decir, voy a pasarle los parámetros de esa web al modelo para que se reajuste y ahora considere que puede haber phishing con una web url corta, con https, con menos puntos y con menos palabras clave:

![Defensa1](https://github.com/user-attachments/assets/d4603b49-171d-4f6c-aeb8-d062ccd44df2)

He añadido una nueva columna, 31 caracteres, 2 puntos, 1 palabra clave, https y lo catalogo como phishing. Reentreno el modelo y vamos a ver matplotlib a ver qué pesos tiene ahora:

![Importanciasegundomodelo](https://github.com/user-attachments/assets/f38d3f65-75e0-497b-b43c-e49c5a69a8ad)

Ahora confía muchísimo más en las palabras clave y en los puntos, pero su confianza en la longitud y el https ha bajado considerablemente. El modelo se ha dado cuenta de que hay webs con menos longitud y con https que son phishing, debe reestructurar el bosque. Como digo, es un dataset muy pequeño y muy pocos datos, los modelos de alta gama tienen cientos de parámetros y millones de páginas web, romperlos lleva más tiempo que este.

Vamos a por la prueba de la verdad, ¿detectará ahora nuestra web como phishing? Vamos a ver:

![Defensa2](https://github.com/user-attachments/assets/5dbf5da9-df76-417f-b7f3-78f3170418e7)

Esto es excelente. Aunque siga diciendo que es segura, su confianza ha caído en picado, ha caído del 94% al 53%, casi a la mitad. Ahora, con un filtro que combine seguridad y operatividad (por ejemplo, que el modelo tiene que tener más de un 75% de certeza que sea web para dejarlo pasar) el phishing se quedaría fuera, no te dejaría entrar (a esto en ciberseguridad se le llama <span class="solo-color-neon">ajustar el threshold</span>, o sea, ajustar el umbral) </span>. Y es más, si pasas más datos de webs de phishing y legítimas, el modelo probablemente ya empiece a verlo como <span class="solo-color-neon">phishing</span> y costará mucho más mover esa certeza (no es lo mismo cambiar 4 parámetros que cambiar 150).

Con este simple modelo, se ha comprendido 2 cosas fundamentales: Si sabes <span class="solo-color-neon">los pesos</span>, controlas el modelo y lo segundo, es tan importante poner a prueba tu modelo como mejorarlo nuevamente. Si atacas y no defiendes, estás fuera, si defiendes pero no lo atacas, estás fuera también. Combinar <span class="solo-color-neon">el adversarial hacking con el adversarial training</span> es la mejor receta para el éxito de cualquier modelo.

- <span class="solo-color-neon">Laboratorio creado:</span> 18/02/2026
- <span class="solo-color-neon">Laboratorio subido:</span> 18/02/2026
