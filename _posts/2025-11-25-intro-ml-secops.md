---
layout: post
title: "Matriz Bow aplicada a Ciberseguridad"
date: 2025-11-25
categories: [ia-secops]
permalink: /ml-secops-introduccion/
---
# USAR MATRIZ BOW PARA DETECTAR COMPORTAMIENTO ANÓMALO

Antes de empezar con el artículo quiero decirte una cosa... <span class="solo-color-neon">Los modelos de machine learning (La base de los llms actuales) no entienden palabras </span>... Sí, cuando tú hablas con una IA no va a entender nada de lo que le dices, porque para ello, necesita convertir todo a números... Esto se hace con tokens, matrices (bueno, aquí en verdad una matriz es una colección de vectores)… Así que aclarado esto, empiezo con el artículo.

He simulado un entorno muy minimalista de gente trabajando en una ofician. En teoría, a menos que haya un insider o un ordenador comprometido, <span class="solo-color-neon">la mayoría de personas en la oficina deberían usar programas bastante parecidos...</span> El hecho de que haya cambios drásticos en las labores que realizan, es un indicador de que algo no está bien. Esto se conoce como UBA, o comportamiento anómalo de usuarios. En el UBA una desviación de lo típico se considera una anomalía, lo cual podría ser un falso positivo o un indicador potencial de compromiso. En este ejemplo tenemos 5 usuarios (inventados, obviamente) que están realizando sus labores ejecutando los programas necesarios para su función.
```python
import numpy as np
logs = [
"Manuel ha usado word excel notepad firefox",
"Carlos ha usado excel word firefox notepad",
"Antonio ha usado word firefox excel notepad",
"Juan ha usado notepad firefox word excel",
"Pepe ha usado powershell wmic chrome python"
]

palabras=[linea.split() for linea in logs]
columnas=sorted({unica for w in palabras for unica in w})
matrizbow= np.zeros((len(logs),len(columnas)),dtype=int)
for i, t in enumerate(palabras):
    for palabra in t:
        j = columnas.index(palabra)
        matrizbow[i,j]+=1
nor=np.linalg.norm(matrizbow,axis=1)
matrizbowN=matrizbow/nor[:,np.newaxis]
matriz2=matrizbowN @ matrizbowN.T
print (matriz2)
```

Aquí la lista es bastante pequeña (<span class="solo-color-neon">los modelos de machine learning suelen tener miles, decenas de miles de líneas...</SPAN> Lo siento, pero tanto tiempo libre no tengo). Tenemos 5 usuarios y queremos saber cuál se desvía de lo normal, queremos el comportamiento anómalo. El objetivo de este code es convertir una lista de textos (la matriz logs) en una tabla de números llamada Matriz de frecuencia de términos (Bow).


Para poder construir una matriz bow, primero vamos a calcular las filas y las columnas. 
```python
palabras=[linea.split() for linea in logs]
columnas=sorted({unica for w in palabras for unica in w})
```
Estos son dos bucles for en Python (sí, los bucles pueden condensarse en una línea, pero eso ya lo sabes, ¿verdad?). En el primero vamos a separar las palabras por espacios. Por ejemplo, en el primer vector de la matriz tendríamos: Manuel,ha,usado,word,excel,notepad,firefox. Con esto hemos troceado las palabras y esto lo hará con el resto de las líneas. En el segundo, tenemos un conjunto, lo que va a eliminar duplicados (no vamos a tener la palabra Firefox, Notepad o lo que sea más de una vez). <span class="solo-color-neon">Esto es muy útil porque van a formar las columnas</span> y, para que no se rompa el conteo, sólo necesitamos una palabra, no varias. Luego, con el sorted las ordenamos las palabras alfabéticamente (tanto mi casa como las palabras me gusta tenerlas ordenadas).
```python
matrizbow= np.zeros((len(logs),len(columnas)),dtype=int)
```
Esto es algo más simple de entender (te doy un respiro, colega), porque vamos a formar <span class="solo-color-neon">nuestra matriz bow</span>. En este caso, construimos las filas con len(logs) el cual nos dará 5 (Tenemos 5 líneas, 5 usuarios diferentes). Las columnas serán las palabras que hemos troceado y ordenado antes. Y la matriz será de tipo int. Esto es una matriz que, si no me equivoco, es de 5x15 (5 filas y 15 columnas). Iniciamos esta matriz a 0, es decir, en todos sus ejes fila-columna habrá 0.
```python
for i, t in enumerate(palabras):
    for palabra in t:
        j = columnas.index(palabra)
        matrizbow[i,j]+=1
```
Sin ninguna duda, y como experiencia personal, esta es la parte del code más difícil de entender, <span class="solo-color-neon">porque queremos ver cómo se estructura la matriz bow </span>. Vamos a hacer un bucle for en el que el valor de i recogerá las filas y t recogerá el elemento que estamos procesando (es lo que hace enumerate, devuelve 2 valores). Por ejemplo, con i=0 la variable t será <span class="solo-color-neon">["Manuel","ha","usado","word","excel","notepad","firefox"]</span>. Es decir, con el primer bucle, vamos a ir línea por línea de la matriz logs. Con el segundo bucle, vamos a recorrer justamente esas palabras que están dentro ("Manuel", "ha"...). En el caso de que cojamos Manuel, va a sumar un valor en la columna llamada "Manuel". Luego irá con la palabra "ha" (recuerda, seguimos en la primera fila aún" y va a hacer lo mismo,  donde haya un "ha" sumará uno a eso. Va a recorrer TODAS las palabras que están en el vector de Manuel en la primera fila. Luego, pasará a la segunda columna y hará exactamente lo mismo, va a ir palabra por palabra de la fila y donde coincida esa palabra y la columna, sumará uno al valor. Porque si prestas atención, hemos ordenado las columnas en orden alfabético, nos va a quedar así:

```python
['Antonio', 'Carlos', 'Juan', 'Manuel', 'Pepe', 'chrome', 'excel', 'firefox', 'ha', 'notepad', 'powershell', 'python', 'usado', 'wmic', 'word']
```
(Aquí te estarás preguntando, ¿por qué por ejemplo Chrome va después de Juan, Manuel o Pepe? Has sido avispado, Python ordena y prioriza primero las mayúsculas (por el formato ASCII) y luego va con las minúsculas.

Vamos con la primera fila: Manuel,ha,usado,word,excel,notepad,firefox

Primero cogemos Manuel, que coincide con Manuel en la 4 columna. En todas las demás dará 0. Vamos con "ha" (recuerda, seguimos en la primera fila aún), aquí se encuentra en la columna 9, lo que en ese lugar, el valor cambiará de 0 a 1. Seguimos con "usado", cuya columna es la 13, donde coincide su valor, y pasará de 0 a 1. Seguimos aún en la fila 1, y ahora nos toca la palabra Word, que está en la columna 15, lo que cambiará ese valor de 0 a 1. Seguimos con Excel, que está en la columna 7, como palabra y nombre de columna coinciden, cambiará de 0 a 1. Vamos con Notepad, que está en la columna 10,  cambiará el valor de 0 a 1 nuevamente. Falta el último valor de esa fila, Firefox, que está en la columna 8, bueno, ya sabes lo que pasará. <span class="solo-color-neon">Ahora tenemos nuestra fila del vector bow terminada</span>: 

```python
[0 0 0 1 0 0 1 1 1 1 0 0 1 0 1]
```
Esto pasará en cada fila, por lo tanto, al final tendremos esta matriz con un print:
```python
[[0 0 0 1 0 0 1 1 1 1 0 0 1 0 1]
 [0 1 0 0 0 0 1 1 1 1 0 0 1 0 1]
 [1 0 0 0 0 0 1 1 1 1 0 0 1 0 1]
 [0 0 1 0 0 0 1 1 1 1 0 0 1 0 1]
 [0 0 0 0 1 1 0 0 1 0 1 1 1 1 0]]
```
Aquí, un modelo de machine learning estaría en su salsa, porque hemos pasado <span class="solo-color-neon">vectores de texto a vectores numéricos</span>, nuestra IA podría trabajar de manera óptima. Pasamos a la siguiente parte del code:
```python
nor=np.linalg.norm(matrizbow,axis=1)
matrizbowN=matrizbow/nor[:,np.newaxis]
```
Aquí, la función numpy (np, la hemos renombrado por facilidad), lo cual creará una <span class="solo-color-neon">norma eucladiana</span> para cada fila. La siguiente linea lo que hará es que vamos a <span class="solo-color-neon">NORMALIZAR</span> los vectores de la matriz bow. Normalizar es obligatorio, porque si una persona usó 35 aplicaciones y otra sólo 4, la de 35 tendrá números mayores y una magnitud mucho mayor. Con la normalización, <span class="solo-color-neon">eliminamos el efecto del tamaño de lo que analizamos para sólo comparar el patrón relativo de uso</span>. Te doy mi truco, normaliza siempre vectores antes de comparar, pase lo que pase.

```python
matriz2=matrizbowN @ matrizbowN.T
```
Aquí calculamos <span class="solo-color-neon">la similitud del coseno</span>, porque una matriz multiplicada por su traspuesta, <span class="solo-color-neon">su resultado es el coseno del ángulo que forman ambos vectores</span>.

Si hacemos un print a matriz2, nos va a quedar algo interesante (y probablemente aquí es cuando veas la luz):
```python
[[1.         0.85714286 0.85714286 0.85714286 0.28571429]
 [0.85714286 1.         0.85714286 0.85714286 0.28571429]
 [0.85714286 0.85714286 1.         0.85714286 0.28571429]
 [0.85714286 0.85714286 0.85714286 1.         0.28571429]
 [0.28571429 0.28571429 0.28571429 0.28571429 1.        ]]
```
Explico, porque en ciberseguridad, esto es lo realmente importante. Aquí se está haciendo <span class="solo-color-neon">comparaciones entre filas</span> (coseno de dos vectores). Por ejemplo, tenemos en la primera posición un 1, ¿por qué? Porque estamos comparando la frase (Vector) Manuel ha usado Word Excel Notepad Firefox (es para que lo veas, en realidad estamos comparando su vector [0 0 0 1 0 0 1 1 1 1 0 0 1 0 1]) consigo mismo. Como es exactamente igual, aquí tenemos un ángulo de 0º, lo cual su coseno es 1, eso indica que es lo mismo. El segundo resultado, 0,85714286 es porque está comparando la misma frase con la segunda Carlos ha usado Excel Word Firefox Notepad (recuerda, el vector es [0 1 0 0 0 0 1 1 1 1 0 0 1 0 1]) suponiendo un coseno del ángulo entre los dos vectores de eso. ¿Qué quiere decir el valor 0,8571...? Que la primera frase y la segunda <span class="solo-color-neon">son muy parecidas</span> (si te fijas, sólo cambian los nombres, todas las demás palabras, aunque estén en otro orden, son exactamente las mismas). Vamos a avanzar hacia la última, la del 0,2857... Ese valor está comparando la primera frase con la última, y su valor es muy cercano a 0, ¿qué significa? significa que estamos calculando el coseno <span class="solo-color-neon">de un ángulo cercano a 90</span> (si en la comparación, hay un ángulo de 90 entre dos vectores, significa que son totalmente diferentes). Esto indica que las frases <span class="solo-color-neon">son muy distintas</span> (sólo coinciden en "ha" y "usado"), son dos vectores distintos. 

Siguiendo cada fila, puedes adivinar los valores que darán, porque es la misma tónica. 

Como vemos, aquí se ha detectado un valor muy cercano a 90º o con un coseno de un ángulo muy cercano a 0. Cuanto más cerca esté el coseno a 1, más se parecen los vectores, cuanto más cercano a 0 es que son muy diferentes. Ten siempre esto presente:

Si comparas dos vectores y son totalmente diferentes, <span class="solo-color-neon">su valor será 0</span>. Si son idénticos, <span class="solo-color-neon">su valor será 1</span>.

Con esta matriz terminada lo que tenemos es que, por ejemplo, si el valor baja de un 0,5 lo detectaremos como anomalía. Imagina esto con 100.000 entradas y sólo una es maligna, te perderías intentando encontrarla, pero con esto, en un par de segundos lo tendrías, detectarías una anomalía en cuestión de segundos en tus logs.

Según esto, puede haber dos interpretaciones:

- <span class="solo-color-neon">Pepe es un insider malicioso o tiene el ordenador comprometido</span>. No hay más, está usando herramientas que no tienen nada que ver con las de oficina y, además, son herramientas que ejecutan scripts, algo que un oficinista no debería estar haciendo.

- <span class="solo-color-neon">El ordenador de Pepe está revisándolo un técnico por un fallo de software</span>. En este caso sería un falso positivo, ya que se está implementando una acción legítima que, en los logs, aparece como anomalía.

Y ambas están bien, porque las anomalías te permiten <span class="solo-color-neon">detectar ataques que no están en tus firmas estáticas</span>, porque todo comportamiento que se salga de lo normal es visto como malicioso. Pero también puede generar <span class="solo-color-neon">falsos positivos</span>, por la razón descrita arriba. Es trabajo del analista corroborar esa información y catalogar el incidente de ataque o como falso positivo.

Laboratorio creado: 04/11/2025

Laboratorio publicado: 25/11/2025
