# Ejercicio 2---Siguelíneas y PID

## Planteamiento
El objetivo de este ejercicio es crear un programa que controle un fórmula 1 de tal forma que siga una línea roja. La correcta implementación debe tener mecanismos de recuperación,
un PID que regule V y W lo más óptimo posible y una versatilidad en varios circuitos.

## Desarrollo
A la hora de realizar este ejercicio lo primero que implementé es un filtrado de la imagen con OpenCV de tal forma que obtenía el error en el eje x respecto al centro de la imagen.
Este error lo obtengo del centro de todos los puntos rojos. También le puse un margen donde la velocidad debería ser solo V.

Posteriormente realicé el PID de la velocidad en W. En este momento me di cuenta que al tener el centro del error muy cerca del coche no reacciona bien a los giros. Además, al ponerle un
margen donde la velocidad solo debe ser V no reacciona bien a un PID que funciona mejor sin margen por eso lo quité.

En la siguiente fase del proyecto añadí el controlador PID para V de tal forma que poco a poco lo fui complementando con el PID de W. Este proceso fue el más exhausto debido a los
constantes micro ajustes que se tuvieron que realizar para optimizar. En este proceso se tuvieron que tener en cuenta las velocidades que podía alcanzar el coche, cuánto de reactivo es v
y W. También mencionar que el PID de W funciona distinto del de v, el PID de W reacciona respecto al error del eje X de la imagen y busca ser 0 mientras que el PID de V lo que hace es restar
el error a una velocidad Máxima dada y con alguna modificación y ajustar esa velocidad resultante mediante PID. Tiene algunos factores más como que si el PID pudiese dar una V negativa o superior
a V_MAX lo limitara entre 0 y V_MAX, pero esto en los ajustes de PID finales no se puede alcanzar. Aún así lo mantuve como medida de protección y escalabilidad a otros proyectos.

Para completar el control del coche, realice una serie de medidas para manejar errores de la cámara, pérdida de datos y pérdida de la línea:
- En cuanto a pérdida de la cámara, hay un mecanismo que si se pierde consecutivamente se para el coche. Esto se debe a que no hay forma de recuperar el movimiento y en un entorno real es peligroso manejar sin percepción.

- Hay momentos en los que el filtrado de la línea roja falla, en este caso reduce la velocidad entre 2 hasta recuperar o en el caso de que ha perdido la línea roja se activa el siguiente caso

- La pérdida de la línea es un error que se debe contemplar por ejemplo si se ha perdido el seguimiento durante unos segundos a causa de la cámara. Video:
<video width="640" height="360" controls>
  <source src="video/p2_buscar.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>

Por último, realicé una mejora de la percepción del usuario de la cámara. Fusione la imagen obtenida junto con la imagen filtrada, la primera la opacé un poco y muestro las velocidades que toma el coche junto con su giro.
Las velocidades y giro se vuelven rojas en el caso de que se estén perdiendo una o más veces el filtrado de la línea roja. Respecto a esto último recordar que para el filtro lo que hay por debajo de la línea verde horizontal
no existe ya que está muy pegado al coche y sería muy poco reactivo (esto se puede modificar en la proporción en el código). Aun así en momentos de los vídeos vemos que en líneas que no debería perderse lo hace.
No se muestra cuando pierde directamente la imagen.

## Circuitos Básicos
Como apunte mencionar que el resultado del PID es bastante preciso pero los problemas que presentará son debidos al coste de procesar en el simulador en algunos mapas (se observa más lento y con más pérdidas, recordar que
cuando se pierde la imagen no se muestra nada pero se mantiene la velocidad) y tiene un poco de ocilación al recuperar de curva arecta pero apenas se aleja de la línea roja.

### 1. Simple Circuit
Es el circuito que mejor funciona entre que el simulador corre mejor y que es donde se optimizó el PID

<video width="640" height="360" controls>
  <source src="video/p2_v1.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>

### 2. Montreal Circuit
En las ligeras curvas de la parte superior el coche presenta bastante oscilación, por lo que he visto se debe a la línea central que no son curvas como tal sino una serie de líneas rectas 
(esto también se puede ver en algunos frames a partir del 0:33 en el video), esto provoca que no sea fluido como en la curva constante del circuito simple. 
También mencionar que se puede apreciar el efecto que provoca la pérdida de imágenes en algunas curvas como en el tiempo 0:20 del video en el que esto hace que la rectificación sea brusca.

<video width="640" height="360" controls>
  <source src="video/p2_v2.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>

### 3. Montmelo Circuit
En general este circuito lo realiza bien exceptuando alguna perdiad de imagen como en el caso del segundo 0:19, donde apreciamos que la línea se sigue viendo al ser línea recta y aún así no la detecta en el error
(se sigue obteniendo imagen), seguramente error del simulador. Se aprecia la reducción de velocidad de emergencia. 
Con todo esto en la doble curva final de más de 90º se pierde por la velocidad pero el mecanismo de recuperación de línea funciona y continúa el circuito perdiendo unas milésimas.
Lo considero aceptable al no perder demasiado tiempo y al considerar que la velocidad es necesaria para obtener buenos tiempos en los circuitos.

<video width="640" height="360" controls>
  <source src="video/p2_v3.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>

### 4. Nurburgring Circuit
En general lo realiza bien pero en algunos instantes se aprecian pérdidas. En la primera curva por lo pronunciada que es y la velocidad del vehículo se pierde un instante.

<video width="640" height="360" controls>
  <source src="video/p2_v4.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>

## Circuitos Ackermann
En estos circuitos no funciona debido a las velocidades en el simulador, al ser distintas de los simples lo que provoca que el giro del PID W este descompensado y el PID de V parezca lento.
El giro adaptado a altas velocidades genera sobreoscilación que crece. También denotar que he probado aumentar en gran medida la velocidad de V, pero esto parece no afectar, como si no tuviesen apenas agarre las ruedas.
