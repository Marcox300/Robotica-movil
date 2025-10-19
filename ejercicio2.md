# Ejercicio 2 Siguelineas y PID

## Planteamiento
El objetivo de este ejercicio es crear un programa que controle un formula 1 de tal forma que siga una linea roja. La corecta implemetación debe tener mecanismos de recuperación,
un PID que reguel V y W lo más optimo posible y una versatilidad en varios circuitos.

## Desarrollo
A la hora de realizar este ejercicio lo primero que implemente es un filtrado de la imagen con openCV de tal forma que obtenía el error en el eje x respecto al centro de la imagen.
Este error lo obtengo del centro de todos los puntos rojos. Tambien le puse un margen donde la velocidad devería ser solo V.

Posteriormente realice el PID de la velocidad en W. En este momento me di cuenta que al tener el centro del error muy cerca del coche no reacciona bien a los giros. Además, al ponerle un
margen donde la velocidad solo debe ser V no reacciona bien a un PID que funciona mejor sin margen por eso lo quite.

En la diguiente fase del proyecto añadi el controlador PID para V de tal forma que poco a poco lo fui complementando con el PID de W. Este proceso fué el más exausto debido a los
constantes micro ajustes que se tuvieron que realizar para optimizar. En este proceso se tubieron que tener en cuenta las velocidades que podía alcanzar el coche, cuanto de reactivo es v
y W. Tambien mencionar que el PID de W funciona distinto del de v, el PID de W reaciona respecto al error del eje X de la imagen y busca ser 0 mientras que el PID de V lo que hace es restar
el error a una velocidad Máxima dada y con alguna modificación y ajustar esa velocidad resultante mediante PID. Tiene algunos factores más como que si el PID pudiese dar una V negativa o superior
a V_MAX lo limitara entre 0 y V_MAX, pero esto en los ajustes de PID finales no se puede alcanzar. Aun así lo mantuve como medida de protección y escalabilidad a otros proyectos.

Para completar el contro del coche, realice una serie de medidas para manejar errores de la cámara, pérdida de datos y pérdida de la linea:
- En cuanto a pérdida de la cámara, hay un mecanismo que si se pierde consecutivamente se para el coche. Esto se debe a que no hay forma de recuperar el movimiento y en un entorno real es peligroso manejar sin percepción.

- Hay momentos en los que el filtrado de la linea roja falla, en este caso reduce la velocidad entre 2 hasta recuperar o en el caso de que ha perdido la linea roja se activa el siguiente caso

- La pérdida de la linea es un error que se debe contemplar por ejemplo si se ha perdido el deguimiento durante unos segundos a causa de la cámara. Video:
<video width="640" height="360" controls>
  <source src="video/p2_buscar.mp4" type="video/mp4">
  Tu navegador no soporta el video.
</video>

Por último, realice una mejora de la percepción del usuario de la cámara. Fusione la imagen obtenida junto con la imagen filtrada, la primera la opaco un poco y muestro las velocidades que toma el coche junto con su giro.
Las velocidades y giro se vuelven rojas en el caso de que se esten perdiendo una o más veces el filtrado de la linea roja. Respecto a esto último recordar que para el filtro lo que hay por debajo de la linea verde horizontal
no existe ya que está muy pegado al coche y sería muy poco reactivo (esto se puede modificar en la proporción en el código). Aun así en momentos de los videos vemos que en lineas que no debería perderse lo hace.
No se muestra cuando pierde directamente la imagen.


